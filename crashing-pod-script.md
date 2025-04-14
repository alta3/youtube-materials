# 🧪 **Kubernetes CrashLoopBackOff Recovery Lab**

**Objective:**  
Simulate three common pod failure scenarios and write a Bash script to detect and restart those pods automatically.

**Estimated Time:** 15–20 minutes  
**Prerequisites:**  
- A running Kubernetes cluster (e.g. Minikube, kind, GKE, etc.)
- `kubectl` access
- Basic knowledge of YAML and Bash

---

## 🔧 Part 1: Simulate Failing Pods

### 1. Create the failure manifest

Save the following to a file called `wile-e-pods.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: wile-e-coyote
spec:
  containers:
  - name: fail-fast
    image: busybox
    command: ["sh", "-c", "echo 'Failing...'; sleep 1; exit 1"]
  restartPolicy: Always
---
apiVersion: v1
kind: Pod
metadata:
  name: wile-e-coyote2
spec:
  containers:
  - name: memory-hog
    image: python:3.9
    command: ["python3", "-c", "a = ' ' * (10**9)"]
    resources:
      limits:
        memory: "50Mi"
  restartPolicy: Always
---
apiVersion: v1
kind: Pod
metadata:
  name: wile-e-coyote3
spec:
  containers:
  - name: probe-fail
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 5
    ports:
    - containerPort: 8080
  restartPolicy: Always
```

### 2. Apply it

```bash
kubectl apply -f wile-e-pods.yaml
```

### 3. Observe their status

```bash
kubectl get pods
```

You should see all three enter `CrashLoopBackOff`.

---

## 🧰 Part 2: Create the Recovery Script

### 1. Save the following as `acme.sh`

```bash
#!/bin/bash

# Namespace to monitor
NAMESPACE="default"
NOW=$(date +"%Y-%m-%d %H:%M:%S")
echo "[$NOW] Starting CrashLoopBackOff scan in namespace: $NAMESPACE"
echo "=============================================================="

# Find all pods in CrashLoopBackOff within the specified namespace
crashing_pods=$(kubectl get pods -n $NAMESPACE --field-selector=status.phase=Running \
  | awk '$3=="CrashLoopBackOff" {print $1}')

# Exit early if no crashing pods found
if [[ -z "$crashing_pods" ]]; then
  echo "[$(date +"%Y-%m-%d %H:%M:%S")] No crashing pods found."
  exit 0
fi

# Loop through each crashing pod
for pod in $crashing_pods; do
  echo
  echo "--------------------------------------------------------------"
  echo "[$(date +"%Y-%m-%d %H:%M:%S")] 📛 Pod: $pod"
  echo "--------------------------------------------------------------"

  echo "▶ Log Snippet:"
  kubectl logs -n $NAMESPACE $pod --tail=5 2>/dev/null || echo "(No logs found)"
  echo

  echo "▶ Restart Count:"
  kubectl get pod -n $NAMESPACE $pod -o jsonpath='{.status.containerStatuses[0].restartCount}' || echo "(N/A)"
  echo -e "\n"

  echo "▶ Last Crash Reason:"
  kubectl get pod -n $NAMESPACE $pod -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}' || echo "(N/A)"
  echo -e "\n"

  echo "▶ Liveness Probe Configuration:"
  PROBE=$(kubectl get pod -n $NAMESPACE $pod -o jsonpath='{.spec.containers[0].livenessProbe}' 2>/dev/null)
  if [[ -z "$PROBE" ]]; then
    echo "(No liveness probe configured)"
  else
    echo "$PROBE"
  fi
  echo

  echo "▶ Recent Events:"
  kubectl describe pod -n $NAMESPACE $pod | awk '/Events:/,/^$/'
  echo

  echo "▶ Action: Deleting pod to trigger restart..."
  kubectl delete pod -n $NAMESPACE $pod
  echo "--------------------------------------------------------------"
done

echo "[$(date +"%Y-%m-%d %H:%M:%S")] ✅ Scan complete."
```

### 2. Make it executable

```bash
chmod +x acme.sh
```

### 3. Run the script

```bash
./acme.sh
```

You’ll see output for each pod including:
- Recent logs
- Restart count
- Crash reason
- Liveness probe (if applicable)
- Events
- A deletion trigger

---

## 🕒 Bonus: Automate with Cron

To run this every 5 minutes automatically:

```bash
crontab -e
```

Add:

```bash
*/5 * * * * /full/path/to/acme.sh >> /tmp/acme.log 2>&1
```

---

## ✅ Lab Complete!

By the end of this lab, you’ve:
- Simulated 3 common pod failure scenarios
- Written a script to detect and remediate them
- Practiced using logs, JSONPath, `kubectl describe`, and event parsing
- Built a script that can be reused across namespaces or production environments

---

## 🧠 Try This Next

- Modify the script to only restart pods that have restarted more than 3 times
- Add Slack or Discord alerts using a webhook
- Turn this into a Kubernetes Job or CronJob inside the cluster itself
