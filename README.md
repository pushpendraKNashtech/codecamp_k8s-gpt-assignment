# K8sGPT Troubleshooting Lab — Kubernetes Service Selector Mismatch

## Overview

This lab demonstrates how K8sGPT can be used to troubleshoot a Kubernetes application with a Service selector mismatch.

The scenario intentionally introduces a configuration problem where:

* The `web` Deployment creates healthy Pods labeled `app=web`.
* The `web-svc` Service incorrectly selects Pods labeled `app=frontend`.
* The Pods are running successfully, but the Service has no endpoints.
* K8sGPT's deterministic analyzer identifies the missing Service endpoints and the expected selector label.
* The Service selector is corrected.
* The application is verified by checking Service endpoints and accessing nginx through the Kubernetes Service.

The troubleshooting workflow used in this lab is:

**Deploy → Break → Analyze → Explain → Fix → Verify**

---

## Scenario

The application consists of:

* A Kubernetes namespace: `k8sgpt-assignment`
* A Deployment: `web`
* Two nginx Pods
* A ClusterIP Service: `web-svc`

The Deployment creates Pods with the following label:

```yaml
app: web
```

The intentionally broken Service uses the following selector:

```yaml
selector:
  app: frontend
```

Because `app=frontend` does not match the Pod label `app=web`, Kubernetes does not associate the Pods with the Service.

As a result, the Service has no endpoints even though the Pods themselves are healthy.

---

## Prerequisites

The following tools were used:

* Kubernetes
* kubectl
* kind
* K8sGPT
* Ollama

Versions used during the lab included:

```text
K8sGPT: 0.4.39
Ollama: 0.34.0
Kubernetes node: v1.34.0
```

The Kubernetes cluster used for the exercise was a kind cluster named:

```text
k8sgpt-lab
```

The current Kubernetes context was:

```text
kind-k8sgpt-lab
```

---

## Project Structure

```text
k8sgpt-assignment/
├── README.md
├── manifests/
│   ├── broken-app.yaml
│   └── fixed-app.yaml
└── reports/
    ├── before.txt
    ├── k8sgpt-analysis.txt
    └── after.txt
```

---

# 1. Deploy the Broken Application

The intentionally broken configuration is defined in:

```text
manifests/broken-app.yaml
```

It can be deployed with:

```bash
kubectl apply -f manifests/broken-app.yaml
```

The Deployment creates two nginx Pods.

The Pod template uses:

```yaml
labels:
  app: web
```

However, the Service uses:

```yaml
selector:
  app: frontend
```

This creates the intentional Service selector mismatch.

---

# 2. Observe the Broken State

The Pods were healthy:

```text
NAME                  READY   STATUS    RESTARTS
web-f4dd7dd4f-kfj5m   1/1     Running   0
web-f4dd7dd4f-lfz27   1/1     Running   0
```

The Service was available:

```text
NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)
web-svc   ClusterIP   10.96.19.51   <none>        80/TCP
```

However, the Service had no endpoints:

```text
NAME      ENDPOINTS
web-svc   <none>
```

This demonstrated that the Pods were running but were not being selected by the Service.

The initial state is captured in:

```text
reports/before.txt
```

---

# 3. Analyze the Problem with K8sGPT

## Deterministic Analysis

The deterministic K8sGPT analyzer was run with:

```bash
k8sgpt analyze --filter=Service --namespace=k8sgpt-assignment
```

K8sGPT reported:

```text
AI Provider: AI not used; --explain not set

0: Service k8sgpt-assignment/web-svc()
- Error: Service has no endpoints, expected label app=frontend
```

This correctly identified the important symptom:

* The Service had no endpoints.
* The Service expected Pods matching `app=frontend`.

The deterministic analysis is preserved in:

```text
reports/k8sgpt-analysis.txt
```

---

# 4. Root Cause

The root cause was a mismatch between the Kubernetes Service selector and the Pod labels.

The Deployment creates Pods labeled:

```yaml
app: web
```

The broken Service selects:

```yaml
selector:
  app: frontend
```

Therefore:

```text
Service selector: app=frontend
Pod label:       app=web
                     ↓
                 No match
                     ↓
               No endpoints
```

The Deployment itself was not the problem. The Pods were healthy and running normally.

The problem was specifically the Service selector.

---

# 5. AI-Assisted K8sGPT

K8sGPT was configured to use the local Ollama backend instead of an external OpenAI API.

Ollama was installed and configured with the local model:

```text
llama3.2:3b
```

The AI-assisted command used was:

```bash
k8sgpt analyze --explain --backend ollama --filter=Service --namespace=k8sgpt-assignment
```

The AI-assisted analysis was executed using the local Ollama backend.

The final AI-assisted output is preserved in:

```text
reports/k8sgpt-analysis.txt
```

### AI Analysis Observation

The AI-assisted run performed after the Service had already been corrected returned:

```text
AI Provider: ollama

No problems detected
```

This result is consistent with the corrected Kubernetes state at that point: the Service had valid endpoints and no longer had the original selector problem.

Therefore, this lab does not claim that the AI-assisted analysis independently detected the original selector mismatch.

The deterministic K8sGPT analyzer provided the explicit diagnosis of:

```text
Service has no endpoints, expected label app=frontend
```

This diagnosis was then verified directly against the Kubernetes manifests and cluster state.

---

# 6. Fix the Service

The corrected configuration is stored in:

```text
manifests/fixed-app.yaml
```

The fixed Service selector is:

```yaml
selector:
  app: web
```

The fixed manifest was applied using:

```bash
kubectl apply -f manifests/fixed-app.yaml
```

The command completed successfully:

```text
namespace/k8sgpt-assignment unchanged
deployment.apps/web unchanged
service/web-svc configured
```

The important change was:

### Broken

```yaml
selector:
  app: frontend
```

### Fixed

```yaml
selector:
  app: web
```

This makes the Service selector match the labels on the Pods.

---

# 7. Verify the Fix

After applying the fixed manifest, both Pods were healthy:

```text
NAME                  READY   STATUS    RESTARTS
web-f4dd7dd4f-kfj5m   1/1     Running   0
web-f4dd7dd4f-lfz27   1/1     Running   0
```

The Service remained available:

```text
NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)
web-svc   ClusterIP   10.96.19.51   <none>        80/TCP
```

Most importantly, the Service now had two endpoints:

```text
NAME      ENDPOINTS
web-svc   10.244.0.5:80,10.244.0.6:80
```

The final state is captured in:

```text
reports/after.txt
```

> Note: Kubernetes v1.33+ reports that the legacy `v1 Endpoints` API is deprecated in favor of `discovery.k8s.io/v1 EndpointSlice`. The assignment explicitly uses `kubectl get endpoints`, so that command was retained for the required evidence.

---

# 8. Functional Verification

The Service was tested from inside the Kubernetes namespace using a temporary curl Pod:

```bash
kubectl run curl-test \
  --image=curlimages/curl:8.10.1 \
  --rm -it \
  --restart=Never \
  -n k8sgpt-assignment \
  -- curl -s http://web-svc
```

The request successfully returned the nginx Welcome page:

```text
Welcome to nginx!
```

This confirms that:

1. The Service exists.
2. The Service has valid endpoints.
3. The Service can route traffic to the nginx Pods.
4. The backend application is responding successfully.

The temporary `curl-test` Pod was automatically deleted after the test.

---

# 9. Before vs After

| State  | Pod Status | Service Selector | Service Endpoints                |
| ------ | ---------- | ---------------- | -------------------------------- |
| Broken | Running    | `app=frontend`   | None                             |
| Fixed  | Running    | `app=web`        | `10.244.0.5:80`, `10.244.0.6:80` |

The key difference was the Service selector.

---

# 10. Troubleshooting Summary

### Problem

The Kubernetes Pods were healthy, but the Service had no endpoints.

### K8sGPT Diagnosis

K8sGPT's deterministic analyzer reported:

```text
Service has no endpoints, expected label app=frontend
```

### Root Cause

The Service selected:

```text
app=frontend
```

while the Pods were labeled:

```text
app=web
```

### Fix

Changed the Service selector to:

```text
app=web
```

### Verification

The Service obtained two endpoints:

```text
10.244.0.5:80
10.244.0.6:80
```

A request to:

```text
http://web-svc
```

successfully returned the nginx Welcome page.

---

# 11. Conclusion

This lab demonstrates a practical Kubernetes troubleshooting workflow using K8sGPT.

The deterministic K8sGPT analyzer successfully identified the Service with no endpoints and the expected selector label. The root cause was confirmed by inspecting the Service selector and Pod labels.

After correcting the Service selector, Kubernetes created the expected endpoints and a request through the Service successfully reached the nginx application.

The AI-assisted K8sGPT workflow was also tested using a local Ollama backend. The observed AI result is preserved in the report without modifying or inventing the generated output.

This demonstrates the importance of combining automated analysis with direct Kubernetes inspection and functional verification.