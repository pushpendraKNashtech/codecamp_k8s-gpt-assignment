# K8sGPT Troubleshooting Assignment

## Objective

Create a small Kubernetes troubleshooting lab that demonstrates one realistic failure mode, analyzes it with K8sGPT, documents the root cause, fixes it, and verifies the result.

Use this workflow:

```text
Deploy -> Break -> Analyze -> Explain -> Fix -> Verify
```

## Scenario

This starter repository uses a Service selector mismatch:

- The `web` Deployment creates healthy pods labeled `app=web`.
- The `web-svc` Service selects `app=frontend`.
- The pods run, but the Service has no endpoints.

You may complete this scenario or replace it with another failure such as `ImagePullBackOff`, `CrashLoopBackOff`, a missing ConfigMap, `OOMKilled`, or an unschedulable pod.

## Prerequisites

- A Kubernetes cluster, such as Minikube, kind, or a shared development cluster
- `kubectl`
- K8sGPT
- An optional AI backend for `--explain` (Ollama, OpenAI, Azure OpenAI, or another supported provider)

Do not commit API keys, kubeconfig files, model credentials, or other secrets.

## Task

### 1. Deploy the broken scenario

```bash
kubectl apply -f manifests/broken-app.yaml
kubectl get pods -n k8sgpt-assignment
kubectl get svc -n k8sgpt-assignment
kubectl get endpoints -n k8sgpt-assignment
```

Save the command output in `reports/before.txt`.

### 2. Run K8sGPT in two layers

First run the deterministic facts-only analysis:

```bash
k8sgpt analyze --filter=Service --namespace=k8sgpt-assignment
```

Then run the AI-assisted explanation:

```bash
k8sgpt analyze --explain --filter=Service --namespace=k8sgpt-assignment
```

Save both outputs in `reports/k8sgpt-analysis.txt`.

If you use a local Ollama backend, you may need:

```bash
k8sgpt analyze --explain --backend ollama \
  --filter=Service --namespace=k8sgpt-assignment
```

### 3. Explain the root cause

In the README, explain:

- What Kubernetes observed
- What K8sGPT detected
- What the AI explanation suggested
- Why the Service had no endpoints
- Whether the AI recommendation was accurate

### 4. Apply the fix

```bash
kubectl apply -f manifests/fixed-app.yaml
kubectl get pods -n k8sgpt-assignment
kubectl get endpoints -n k8sgpt-assignment
```

Save the final output in `reports/after.txt`.

The Service should now have one or more endpoints.

### 5. Complete the submission

Update this README with your findings and open a Pull Request titled:

```text
Add K8sGPT troubleshooting lab for Service selector mismatch
```

## Required repository structure

```text
k8sgpt-assignment/
├── README.md
├── manifests/
│   ├── broken-app.yaml
│   └── fixed-app.yaml
├── reports/
│   ├── before.txt
│   ├── k8sgpt-analysis.txt
│   └── after.txt
```

## Acceptance criteria

- The broken state is reproducible from the repository.
- The initial Kubernetes state is captured.
- Both facts-only and AI-assisted K8sGPT outputs are included.
- The actual root cause is clearly documented.
- A corrected manifest is provided.
- The fix is verified with Kubernetes commands.
- No secrets are committed.
- The README explains what the AI got right or wrong.


## Optional extensions

- Replace the starter scenario with another Kubernetes failure.
- Add a second failure and compare the analyzer output.