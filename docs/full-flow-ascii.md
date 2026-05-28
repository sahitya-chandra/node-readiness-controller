# Complete End-to-End Flow (ASCII)

```
╔══════════════════════════════════════════════════════════════════════════════════════════════╗
║                          NODE READINESS CONTROLLER — COMPLETE FLOW                           ║
╚══════════════════════════════════════════════════════════════════════════════════════════════╝


┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 0 — CONTROLLER STARTUP                                                               │
│                                                                                             │
│   cmd/main.go                                                                               │
│   ┌──────────────────────────────────────────────────────────────────────────────────────┐  │
│   │  1. Connect to Kubernetes API server (in-cluster config)                             │  │
│   │  2. Acquire leader election lock  ──► only ONE replica becomes active                │  │
│   │  3. Register RuleReconciler       ──► watches NodeReadinessRule objects               │  │
│   │  4. Register NodeReconciler       ──► watches Node objects                            │  │
│   │  5. Register Webhook (optional)   ──► ValidatingWebhookConfiguration                 │  │
│   │  6. Add controller tolerations    ──► tolerate ALL readiness.k8s.io/* taints          │  │
│   │     (so controller runs on nodes it manages)                                         │  │
│   │  7. Start health probe server     ──► :8081/healthz  :8081/readyz                   │  │
│   │  8. Start metrics server          ──► :8080/metrics                                 │  │
│   │  9. Seed rule cache               ──► LIST all existing NodeReadinessRules           │  │
│   └──────────────────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 1 — ADMIN CREATES A NodeReadinessRule                                                │
│                                                                                             │
│   ADMIN (kubectl apply -f rule.yaml)                                                        │
│        │                                                                                    │
│        │  HTTP POST /apis/readiness.node.x-k8s.io/v1alpha1/nodereadinessrules              │
│        ▼                                                                                    │
│   ┌──────────────────────────────────────────────────────────────────┐                     │
│   │  KUBERNETES API SERVER — Admission Webhook intercepts            │                     │
│   │                                                                  │                     │
│   │  internal/webhook/nodereadinessgaterule_webhook.go               │                     │
│   │                                                                  │                     │
│   │  ValidateCreate():                                               │                     │
│   │    ① taint.key must start with "readiness.k8s.io/"              │                     │
│   │    ② nodeSelector must not be empty                             │                     │
│   │    ③ scan all existing rules for:                               │                     │
│   │         same taint key+effect  +  overlapping nodeSelector      │                     │
│   │         → REJECT if conflict found                              │                     │
│   │    ④ if enforcementMode=continuous + effect=NoExecute           │                     │
│   │         → WARN (will evict running pods on condition loss)      │                     │
│   │                                                                  │                     │
│   │  ADMIT ──────────────────────────────────────────────────────►  │                     │
│   │         rule stored in etcd as NodeReadinessRule object          │                     │
│   └──────────────────────────────────────────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              │  informer fires ADDED event
                                              ▼
╔═════════════════════════════════════════════════════════════════════════════════════════════╗
║  PHASE 2 — RuleReconciler takes over                                                        ║
║  FILE: internal/controller/nodereadinessrule_controller.go                                  ║
╚═════════════════════════════════════════════════════════════════════════════════════════════╝

   Reconcile(ctx, req) called with ruleName
        │
        ├─① GET rule from API server
        │       updates: rule struct in memory
        │
        ├─② Add finalizer  "readiness.node.x-k8s.io/cleanup"
        │       PATCH NodeReadinessRule.metadata.finalizers
        │       FILE UPDATED: NodeReadinessRule object (etcd)
        │
        ├─③ Detect nodeSelector change?
        │       compare rule.spec.nodeSelector vs cached version
        │       FILE READ: internal/controller/helper.go  nodeSelectorChanged()
        │       if changed → LIST old matching nodes → remove taint from them
        │
        ├─④ Update in-memory Rule Cache
        │       WRITE: RuleReconciler.ruleCache  map[ruleName]→Rule
        │       (protected by sync.RWMutex)
        │
        ├─⑤ LIST all nodes matching rule.spec.nodeSelector
        │       GET /api/v1/nodes?labelSelector=...
        │
        └─⑥ For each matching node → evaluateRuleForNode()
                │
                │  dryRun=true?
                ├──YES──► simulate only, write to status.dryRunResults
                │          NO taint changes made
                │          NodeReadinessRule.status.dryRunResults UPDATED
                │
                └──NO───► real evaluation (see Phase 3 below)
                           then:
                           PATCH NodeReadinessRule.status
                           FILE UPDATED: NodeReadinessRule.status.nodeEvaluations
                                         NodeReadinessRule.status.appliedNodes
                                         NodeReadinessRule.status.failedNodes
                           (retry.RetryOnConflict wraps this patch)


┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  evaluateRuleForNode()  — shared logic used by BOTH reconcilers                             │
│  FILE: internal/controller/nodereadinessrule_controller.go                                  │
│                                                                                             │
│   for each condition in rule.spec.conditions:                                               │
│       look up condition.type in node.status.conditions                                      │
│       ┌──────────────────────────────────────────────┐                                     │
│       │  condition found?                            │                                     │
│       │    YES → compare .status to required value  │                                     │
│       │    NO  → treat as Unknown  (FAIL-SAFE)       │                                     │
│       └──────────────────────────────────────────────┘                                     │
│       record result: { type, required, actual, satisfied }                                  │
│                                                                                             │
│   ALL satisfied?                                                                            │
│   ┌─YES──────────────────────────────────────────────────────────────┐                     │
│   │  node has managed taint?                                         │                     │
│   │    YES → REMOVE taint                                            │                     │
│   │          PATCH Node.spec.taints  (MergeFromWithOptimisticLock)   │                     │
│   │          FILE UPDATED: Node object (etcd)                        │                     │
│   │          metrics.TaintOperations.Inc(rule, "remove")             │                     │
│   │          if bootstrap-only:                                      │                     │
│   │            SET annotation on node:                               │                     │
│   │            readiness.k8s.io/bootstrap-completed-<ruleName>=true  │                     │
│   │            PATCH Node.metadata.annotations                       │                     │
│   │            metrics.BootstrapCompleted.Inc(rule)                  │                     │
│   └──────────────────────────────────────────────────────────────────┘                     │
│   ┌─NO───────────────────────────────────────────────────────────────┐                     │
│   │  node missing managed taint?                                     │                     │
│   │    YES → ADD taint                                               │                     │
│   │          PATCH Node.spec.taints  (MergeFromWithOptimisticLock)   │                     │
│   │          FILE UPDATED: Node object (etcd)                        │                     │
│   │          metrics.TaintOperations.Inc(rule, "add")                │                     │
│   └──────────────────────────────────────────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────────────────────────────────┘


                    ┌─────────────────────────────────────────┐
                    │  MEANWHILE — Infrastructure side        │
                    │                                         │
                    │  cmd/readiness-condition-reporter/      │
                    │  main.go                                │
                    │                                         │
                    │  env: NODE_NAME                         │
                    │       CONDITION_TYPE                    │
                    │       CHECK_ENDPOINT                    │
                    │                                         │
                    │  every 30s:                             │
                    │    GET CHECK_ENDPOINT                   │
                    │    HTTP 2xx → condition status = True   │
                    │    other   → condition status = False   │
                    │                                         │
                    │    PATCH node.status.conditions         │
                    │    FILE UPDATED: Node object (etcd)     │
                    └─────────────────────────────────────────┘
                                        │
                                        │ node MODIFIED event fired
                                        ▼
╔═════════════════════════════════════════════════════════════════════════════════════════════╗
║  PHASE 3 — NodeReconciler reacts to node condition change                                   ║
║  FILE: internal/controller/node_controller.go                                               ║
╚═════════════════════════════════════════════════════════════════════════════════════════════╝

   Node event arrives in informer queue
        │
        ├─① Predicate filter  (before reconcile is even called)
        │   FILE: internal/controller/helper.go
        │   ┌──────────────────────────────────────────────────────────┐
        │   │  conditionsEqual(old, new)?   → skip if no change        │
        │   │  taintsEqual(old, new)?       → skip if no change        │
        │   │  labelsEqual(old, new)?       → skip if no change        │
        │   │                                                          │
        │   │  ALL equal → DROP EVENT (no reconcile triggered)         │
        │   │  ANY differ → ENQUEUE for reconcile                      │
        │   └──────────────────────────────────────────────────────────┘
        │
        ├─② GET fresh Node from API server
        │
        ├─③ READ Rule Cache (RLock)
        │   get all NodeReadinessRules that match this node's labels
        │   FILE: RuleReconciler.ruleCache (shared memory, read-locked)
        │
        └─④ For each matching rule:
                │
                ├── dryRun=true?        → SKIP (no real changes)
                │
                ├── bootstrap-only AND node has annotation
                │   readiness.k8s.io/bootstrap-completed-<ruleName>?
                │                       → SKIP (rule already completed)
                │
                └── evaluateRuleForNode()   ◄── same function as Phase 2
                        │
                        ├── taint added/removed on Node
                        │   FILE UPDATED: Node.spec.taints  (etcd)
                        │
                        └── PATCH rule.status for THIS NODE ONLY
                            FILE UPDATED: NodeReadinessRule.status.nodeEvaluations[nodeName]
                            (retry.RetryOnConflict — multiple nodes can patch same rule concurrently)


┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 4 — TAINT REMOVED → KUBELET REACTS                                                   │
│                                                                                             │
│   Node.spec.taints no longer contains readiness.k8s.io/... taint                           │
│        │                                                                                    │
│        ▼                                                                                    │
│   Kubelet watches Node object                                                               │
│   Node is now schedulable (no blocking taint)                                               │
│        │                                                                                    │
│        ▼                                                                                    │
│   Kube-scheduler assigns pending pods to this node                                         │
│   Kubelet starts pulling images and running containers                                      │
└─────────────────────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 5 — RULE DELETED                                                                     │
│  FILE: internal/controller/nodereadinessrule_controller.go                                  │
│                                                                                             │
│   Admin: kubectl delete nodereadinessrule <name>                                            │
│        │                                                                                    │
│        │  DeletionTimestamp set on object (not immediately gone — finalizer blocks it)      │
│        ▼                                                                                    │
│   RuleReconciler.Reconcile() detects DeletionTimestamp                                      │
│        │                                                                                    │
│        ├─① LIST all nodes that have the managed taint                                      │
│        ├─② REMOVE taint from each node                                                     │
│        │   FILE UPDATED: Node.spec.taints  (etcd)                                          │
│        ├─③ REMOVE rule from in-memory cache                                                │
│        │   FILE UPDATED: RuleReconciler.ruleCache                                          │
│        └─④ REMOVE finalizer from rule                                                      │
│            PATCH NodeReadinessRule.metadata.finalizers = []                                 │
│            → API server now fully deletes the object from etcd                             │
└─────────────────────────────────────────────────────────────────────────────────────────────┘


╔═════════════════════════════════════════════════════════════════════════════════════════════╗
║  WHO UPDATES WHAT — QUICK REFERENCE                                                         ║
╚═════════════════════════════════════════════════════════════════════════════════════════════╝

  ┌─────────────────────────────────────────┬──────────────────────────────────────────────┐
  │  FILE / OBJECT UPDATED                  │  WHO UPDATES IT                              │
  ├─────────────────────────────────────────┼──────────────────────────────────────────────┤
  │  NodeReadinessRule.metadata.finalizers  │  RuleReconciler  (add on create, rm on del)  │
  │  NodeReadinessRule.status.nodeEvaluations│ Both reconcilers (per-node patch)           │
  │  NodeReadinessRule.status.dryRunResults │  RuleReconciler  (dryRun=true only)          │
  │  NodeReadinessRule.status.appliedNodes  │  RuleReconciler                              │
  │  NodeReadinessRule.status.failedNodes   │  RuleReconciler                              │
  ├─────────────────────────────────────────┼──────────────────────────────────────────────┤
  │  Node.spec.taints                       │  Both reconcilers (via evaluateRuleForNode)  │
  │  Node.metadata.annotations              │  RuleReconciler  (bootstrap-completed annot) │
  │  Node.status.conditions                 │  readiness-condition-reporter (external)     │
  ├─────────────────────────────────────────┼──────────────────────────────────────────────┤
  │  In-memory ruleCache                    │  RuleReconciler  (write on reconcile)        │
  │                                         │  NodeReconciler  (read only)                 │
  ├─────────────────────────────────────────┼──────────────────────────────────────────────┤
  │  Prometheus metrics                     │  Both reconcilers + evaluateRuleForNode()    │
  └─────────────────────────────────────────┴──────────────────────────────────────────────┘


╔═════════════════════════════════════════════════════════════════════════════════════════════╗
║  CONCURRENCY GUARDS                                                                         ║
╚═════════════════════════════════════════════════════════════════════════════════════════════╝

  ┌──────────────────────────────────────────────────────────────────────────────────────────┐
  │  PROBLEM                            GUARD USED                                           │
  ├──────────────────────────────────────────────────────────────────────────────────────────┤
  │  Two nodes reconcile same rule      retry.RetryOnConflict on status patch                │
  │  simultaneously                                                                          │
  ├──────────────────────────────────────────────────────────────────────────────────────────┤
  │  Two goroutines patch same          MergeFromWithOptimisticLock on taint patch           │
  │  Node.spec.taints at once                                                                │
  ├──────────────────────────────────────────────────────────────────────────────────────────┤
  │  NodeReconciler reads cache while   sync.RWMutex  (RLock for reads,                     │
  │  RuleReconciler writes it           Lock for writes)                                     │
  ├──────────────────────────────────────────────────────────────────────────────────────────┤
  │  Multiple replicas of controller    Leader Election  (only 1 active manager)             │
  │  running in HA setup                                                                     │
  └──────────────────────────────────────────────────────────────────────────────────────────┘


╔═════════════════════════════════════════════════════════════════════════════════════════════╗
║  ENFORCEMENT MODE DECISION TREE (inside evaluateRuleForNode)                                ║
╚═════════════════════════════════════════════════════════════════════════════════════════════╝

                      all conditions satisfied?
                      /                       \
                    YES                        NO
                     │                          │
             has taint?                   missing taint?
            /         \                  /            \
          YES          NO              YES             NO
           │            │              │               │
        REMOVE        nothing        nothing          ADD
         taint         (already      (already          taint
           │           clean)        blocked)            │
           │                                            ▼
           ▼                               Node stays unschedulable
    bootstrap-only?
    /           \
  YES            NO (continuous)
   │               │
  SET              done — will re-evaluate
  annotation       on next condition change
  (node skipped
  forever after)
```
