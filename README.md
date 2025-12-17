# 🛠 Runbook: Node Drain Stuck on Zombie Pods (K8s v1.23)

## 📌 Overview
Node drain hangs because a pod with a **custom finalizer** never completes termination. The controller responsible for removing the finalizer crashed, leaving the pod stuck in **Terminating**.

Environment: **Kubernetes v1.23**, On-prem bare metal, systemd cgroups.

---

## 🔍 Symptoms
- `kubectl drain <node>` hangs indefinitely.
- `kubectl get pods` shows pod(s) in **Terminating** >20 minutes.

---

## 🧪 Diagnosis
1. List pods stuck in Terminating:
   ```bash
   kubectl get pods --all-namespaces --field-selector=status.phase!=Running -o wide | grep Terminating
   ```
2. Check finalizers:
   ```bash
   kubectl get pod <pod> -n <ns> -o jsonpath='{.metadata.finalizers}'
   ```
3. Inspect controller logs:
   ```bash
   kubectl logs deploy/<controller> -n <controller-ns> --tail=500
   ```

---

## ✅ Quick Fix
> **Use with caution**: Skips cleanup logic.
```bash
kubectl patch pod <pod-name> -n <ns> -p '{"metadata":{"finalizers":[]}}' --type=merge
```

---

## 🧭 Detailed Steps
1. Detect stuck pods (>20m):
   ```bash
   kubectl get pods --all-namespaces -o json | jq -r '.items[] | select(.metadata.deletionTimestamp!=null) | "\(.metadata.namespace)/\(.metadata.name) \(.metadata.deletionTimestamp)"'
   ```
2. Confirm finalizer presence.
3. Restore controller OR remove finalizer if safe.
4. Retry drain:
   ```bash
   kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force --grace-period=60 --timeout=10m
   ```

---

## 🛡 Prevention
- Avoid unnecessary finalizers.
- Add timeout/retry logic in controllers.
- Monitor pods stuck in Terminating >N minutes.
- Provide override annotation (e.g., `ops/override-finalizer=true`).

---

## ✅ TL;DR Checklist
- [ ] List Terminating pods >20m
- [ ] Check `metadata.finalizers`
- [ ] Fix controller or patch finalizer
- [ ] Re-run drain with flags
- [ ] Add monitoring & pre-drain checks

