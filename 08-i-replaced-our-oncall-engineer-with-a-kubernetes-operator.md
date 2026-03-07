# I Replaced Our On-Call Engineer with a Kubernetes Operator — Here's What Happened at 3 AM

*How we built an autonomous remediation system that handles 73% of production incidents without waking anyone up.*

---

## The 3 AM Call That Changed Everything

It was a Tuesday. 3:14 AM. My phone buzzed with that distinctive PagerDuty tone that every on-call engineer learns to dread.

`CRITICAL: Pod CrashLoopBackOff — payment-service — production cluster`

I fumbled for my laptop, SSH'd into the cluster, and ran the same commands I had run a hundred times before:

```bash
kubectl get pods -n payments
kubectl describe pod payment-service-7d4f8b6c9-x2k4p -n payments
kubectl logs payment-service-7d4f8b6c9-x2k4p -n payments --previous
```

The root cause? A downstream database connection pool was exhausted. The fix? Restart the pod, which grabs a fresh connection pool. Total time to diagnose and fix: 4 minutes. Total time lost from my sleep: 2 hours (because who falls back asleep after a production alert?).

Here is the thing — this was the **third time that month** we had the same incident. Same root cause. Same fix. Same 3 AM wake-up call.

That is when I asked the question that changed our entire operations model: **What if Kubernetes could fix this itself?**

## The Problem with Modern Incident Response

Our team ran 47 microservices across 3 Kubernetes clusters. We had Prometheus for metrics, Grafana for dashboards, PagerDuty for alerting, and a well-documented runbook wiki. By industry standards, we were doing everything right.

But the numbers told a different story:

| Metric | Our Reality |
|--------|------------|
| Monthly incidents | 89 |
| Mean Time to Acknowledge (MTTA) | 8 minutes |
| Mean Time to Resolve (MTTR) | 23 minutes |
| Incidents requiring human creativity | 24 (27%) |
| Incidents with known, repeatable fixes | **65 (73%)** |
| On-call engineer burnout rate | High |

**73% of our incidents had known, repeatable fixes.** We were waking people up at 3 AM to execute the same `kubectl` commands they had executed dozens of times before. That is not engineering — that is a cron job with a human in the loop.

## The Architecture of an Autonomous Remediation Operator

Instead of writing another runbook, I wrote a Kubernetes operator. Here is the architecture:

```
┌─────────────────────────────────────────────────────────┐
│                 Kubernetes Cluster                        │
│                                                          │
│  ┌──────────────┐    ┌──────────────────────────────┐   │
│  │  Prometheus   │───▶│   Remediation Operator        │   │
│  │  (Metrics)    │    │                              │   │
│  └──────────────┘    │  1. Detect anomaly            │   │
│                      │  2. Match remediation rule     │   │
│  ┌──────────────┐    │  3. Validate safety gates     │   │
│  │  Pod Events   │───▶│  4. Execute fix               │   │
│  │  (Watcher)    │    │  5. Verify recovery           │   │
│  └──────────────┘    │  6. Log & notify              │   │
│                      └──────────┬───────────────────┘   │
│  ┌──────────────┐               │                       │
│  │  Custom       │◀──────────────┘                       │
│  │  Resources    │  RemediationRule CRD                  │
│  └──────────────┘                                       │
│                                                          │
│  Safety Gates:                                           │
│  ✓ Max 3 auto-remediations per hour per service         │
│  ✓ No auto-remediation during deploy windows            │
│  ✓ Escalate to human if fix fails twice                 │
│  ✓ Never auto-remediate stateful workloads              │
└─────────────────────────────────────────────────────────┘
```

The operator watches for specific failure patterns and applies predefined fixes — but only when safety conditions are met.

## The Custom Resource: RemediationRule

The core abstraction is a `RemediationRule` CRD. Each rule defines a detection pattern and a remediation action:

```yaml
apiVersion: selfheal.io/v1alpha1
kind: RemediationRule
metadata:
  name: fix-crashloopbackoff
  namespace: payments
spec:
  # DETECTION: What triggers this rule?
  detection:
    type: PodStatus
    match:
      status: CrashLoopBackOff
      labelSelector:
        app: payment-service
    # Only trigger after 3 consecutive crashes
    threshold:
      count: 3
      window: 5m

  # REMEDIATION: What should we do?
  remediation:
    action: RestartDeployment
    target:
      kind: Deployment
      name: payment-service
    params:
      strategy: rolling    # rolling restart, not delete-all
      maxUnavailable: 1

  # SAFETY: When should we NOT auto-fix?
  safety:
    maxRemediationsPerHour: 3
    cooldownMinutes: 10
    excludeWindows:
      - cron: "0 14-16 * * 1-5"  # No auto-fix during deploy window
    escalateAfterFailures: 2     # Page human after 2 failed fixes
    excludeStateful: true        # Never touch StatefulSets

  # NOTIFICATION: Always tell someone
  notification:
    slack:
      channel: "#incidents"
      onTrigger: true
      onSuccess: true
      onFailure: true
      onEscalate: true
```

This is declarative incident response. The rule says: *"If payment-service enters CrashLoopBackOff 3 times in 5 minutes, do a rolling restart. But never more than 3 times an hour, never during deploy windows, and if it fails twice, wake up a human."*

## The Operator Logic

The operator's reconciliation loop follows a strict decision tree:

```go
func (r *RemediationReconciler) Reconcile(ctx context.Context, 
    req ctrl.Request) (ctrl.Result, error) {
    
    // 1. Fetch the RemediationRule
    rule := &selfhealv1alpha1.RemediationRule{}
    if err := r.Get(ctx, req.NamespacedName, rule); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. Check if the detection condition is met
    detected, err := r.detectAnomaly(ctx, rule)
    if err != nil || !detected {
        return ctrl.Result{RequeueAfter: 30 * time.Second}, err
    }

    // 3. Check ALL safety gates before proceeding
    safe, reason := r.checkSafetyGates(ctx, rule)
    if !safe {
        r.log.Info("Safety gate blocked remediation", 
            "rule", rule.Name, "reason", reason)
        r.notify(ctx, rule, "blocked", reason)
        return ctrl.Result{RequeueAfter: 60 * time.Second}, nil
    }

    // 4. Execute the remediation
    r.log.Info("Executing auto-remediation", 
        "rule", rule.Name, "action", rule.Spec.Remediation.Action)
    r.notify(ctx, rule, "triggered", "Auto-remediation started")
    
    if err := r.executeRemediation(ctx, rule); err != nil {
        r.recordFailure(ctx, rule)
        
        // Check if we should escalate to human
        if r.shouldEscalate(ctx, rule) {
            r.escalateToHuman(ctx, rule, err)
        }
        return ctrl.Result{}, err
    }

    // 5. Verify the fix worked
    time.Sleep(30 * time.Second) // Wait for pods to stabilize
    healthy, err := r.verifyRecovery(ctx, rule)
    if !healthy {
        r.recordFailure(ctx, rule)
        r.notify(ctx, rule, "fix-failed", 
            "Auto-remediation did not resolve the issue")
        return ctrl.Result{RequeueAfter: 60 * time.Second}, err
    }

    // 6. Success — record and notify
    r.recordSuccess(ctx, rule)
    r.notify(ctx, rule, "resolved", 
        "Issue auto-remediated successfully")
    
    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}
```

The critical insight: **step 3 (safety gates) runs before step 4 (execution)**. The operator is conservative by default. It will not act unless every safety condition is satisfied.

## The Safety Gates: Why This Is Not Reckless

The biggest pushback I got was: *"You are letting a program make changes to production automatically? That is terrifying."*

Fair concern. Here is why our safety model works:

### Gate 1: Rate Limiting

```go
func (r *RemediationReconciler) checkRateLimit(
    ctx context.Context, rule *Rule) bool {
    
    // Count remediations in the last hour
    recent := r.getRecentRemediations(ctx, rule, 1*time.Hour)
    return len(recent) < rule.Spec.Safety.MaxRemediationsPerHour
}
```

If a service is crash-looping despite restarts, the operator stops after 3 attempts and pages a human. This prevents infinite restart loops.

### Gate 2: Deploy Window Exclusion

```go
func (r *RemediationReconciler) isInExcludeWindow(rule *Rule) bool {
    now := time.Now()
    for _, window := range rule.Spec.Safety.ExcludeWindows {
        if window.Matches(now) {
            return true  // Do NOT auto-remediate during deploys
        }
    }
    return false
}
```

If a service breaks during a deployment, the problem is probably the deployment — not something a restart will fix. The operator backs off and lets the deploy team handle it.

### Gate 3: Escalation After Repeated Failures

If the same fix fails twice in a row, the operator assumes the problem is deeper than its rules can handle and escalates to a human with full context:

```
🚨 ESCALATION: payment-service
Auto-remediation failed 2 times in 30 minutes.

Detection: CrashLoopBackOff (5 restarts in 3m)
Attempted fix: Rolling restart (2x)
Result: Pods still crashing after restart

Last 20 log lines:
  [ERROR] Connection refused: db-payments:5432
  [ERROR] Connection pool exhausted, max_connections=100
  [ERROR] Health check failed: /healthz returned 503

Suggested investigation:
  - Database connection pool at capacity
  - Check db-payments pod health
  - Review recent connection count metrics
```

The human gets the diagnosis, the context, and a suggested starting point — not just a "something is broken" page.

### Gate 4: Stateful Workload Protection

The operator never auto-remediates StatefulSets, databases, or persistent workloads. These require careful, ordered operations that should always involve a human.

## The Rules We Built

Over three months, we codified our top incident runbooks into RemediationRules:

| Rule | Detection | Action | Frequency |
|------|-----------|--------|-----------|
| CrashLoopBackOff restart | 3+ crashes in 5 min | Rolling restart | 18/month |
| OOMKilled memory bump | OOMKilled event | Increase memory limit by 25% (up to cap) | 8/month |
| Stuck Pending pod | Pending > 10 min | Delete pod (scheduler retry) | 6/month |
| Certificate expiry | Cert expires < 24h | Trigger cert-manager renewal | 2/month |
| Connection pool exhaustion | Error log pattern match | Restart + alert DB team | 12/month |
| Disk pressure | Node disk > 90% | Evict non-critical pods, clean logs | 4/month |
| Image pull failure | ImagePullBackOff > 3 | Re-tag from backup registry | 3/month |
| HPA stuck at max | Max replicas for > 30 min | Alert capacity team | 5/month |

## The Results: Before and After

After running the operator for 6 months, here are the numbers:

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| Monthly incidents (total) | 89 | 91 | +2% (more detection) |
| Incidents requiring human | 89 | 24 | **-73%** |
| Auto-remediated incidents | 0 | 67 | New capability |
| MTTR (human incidents) | 23 min | 19 min | -17% (better context) |
| MTTR (auto-remediated) | N/A | **47 seconds** | — |
| 3 AM pages per month | 12 | 3 | **-75%** |
| On-call satisfaction (1-10) | 4.2 | 8.1 | +93% |
| False positive rate | N/A | 2.3% | Acceptable |

The most impactful number: **3 AM pages dropped from 12 to 3 per month.** The remaining 3 are genuinely novel incidents that require human creativity. That is what on-call should be — solving interesting problems, not executing scripts.

## What I Learned Building This

### 1. Start with the Boring Stuff

Do not try to build an AI that understands all failures. Start with the incidents that have a known, deterministic fix. CrashLoopBackOff → restart. OOMKilled → bump memory. These are boring, but they account for the majority of pages.

### 2. Safety Gates Are Non-Negotiable

Every auto-remediation system needs hard limits. Without rate limiting, you will create a system that restarts your entire production fleet in a loop. Without deploy window exclusion, you will mask deployment failures. Without escalation, you will miss novel incidents.

### 3. Observability of the Observer

Your remediation operator needs its own monitoring. We track:
- How many remediations per hour/day/week
- Success vs. failure rate per rule
- Time from detection to resolution
- How often safety gates block action
- Escalation frequency

If escalations are increasing, your rules are getting stale. If the success rate drops, the failure modes are evolving.

### 4. The Human Stays in the Loop

This is not about replacing engineers. It is about respecting their time. The operator handles the repetitive work. The engineer handles the creative work. When the operator encounters something it does not understand, it escalates with full context — making the human's job easier, not harder.

### 5. Declare Your Runbooks as Code

The biggest cultural shift was moving from wiki-based runbooks to CRD-based rules. When your incident response is in YAML and version-controlled in Git, you get:
- Code review for operational changes
- Audit trail of every rule modification
- Rollback capability if a rule causes problems
- Testing in staging before production

## The Uncomfortable Truth About On-Call

Here is what nobody in our industry wants to say out loud: **most on-call work is not engineering.** It is pattern matching and script execution. We dress it up with terms like "incident response" and "reliability engineering," but the reality is that a large percentage of production incidents follow known patterns with known fixes.

The traditional approach — train humans, write runbooks, set up paging — was designed for an era when systems were simpler and failure modes were fewer. In a world of microservices and Kubernetes, the volume and velocity of incidents has outpaced what human-only response can sustainably handle.

Autonomous remediation is not a luxury. It is a necessity for any team running distributed systems at scale.

## Getting Started

If you want to build something similar, start here:

1. **Audit your incidents** — Export the last 6 months from PagerDuty/OpsGenie. Categorize by root cause and fix. I guarantee 50-70% are repeatable.

2. **Pick the top 3 patterns** — Start with the highest-frequency, lowest-risk incidents. CrashLoopBackOff restarts are a great first rule.

3. **Build the safety model first** — Before writing any remediation logic, implement rate limiting, exclusion windows, and escalation. The safety system is more important than the fix system.

4. **Run in dry-run mode** — Deploy the operator with `dryRun: true`. Let it detect and log what it *would* do for 2 weeks. Review the logs. Adjust thresholds. Only then enable actual remediation.

5. **Measure everything** — Track MTTR, page frequency, false positive rate, and engineer satisfaction. If any metric moves in the wrong direction, dial back.

## Conclusion

Six months ago, I was getting paged at 3 AM to restart pods. Today, a Kubernetes operator handles 73% of our incidents in under a minute, while I sleep.

The operator is not perfect. It cannot debug a race condition or optimize a slow query. But it can restart a crashed pod, bump a memory limit, and clear disk pressure — the same things I was doing at 3 AM with bleary eyes and a bad attitude.

The best on-call engineers are not the ones who respond fastest. They are the ones who automate themselves out of the repetitive work so they can focus on the problems that actually need a human brain.

Build the operator. Get your sleep back.

---

*Pavan Madduri is a Senior DevOps/Platform Engineer and Kubestronaut. His research on autonomous remediation and agentic SRE teams is available on [Google Scholar](https://scholar.google.com/citations?user=au0O-8oAAAAJ).*

**Tags:** `#Kubernetes` `#SRE` `#DevOps` `#OnCall` `#Automation` `#Operators` `#SelfHealing` `#PlatformEngineering`
