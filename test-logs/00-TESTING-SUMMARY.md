# Test Run Summary — Qwen-Chatbot

**Environment:** GitHub Codespace (isolated), machine type standardLinux32gb (4 cores / 16 GB RAM)
**Date:** 2026-09-26
**Performed by:** subash (GitHub: Subash-1507)

## What was done

1. Created an isolated GitHub Codespace on `Subash-1507/Qwen-Chatbot` (branch `main`).
2. Started minikube (`--driver=docker --memory=8192 --cpus=4`).
3. Built both Docker images locally in the codespace:
   - `chatbot-backend:latest` — bakes in Qwen2.5-0.5B-Instruct at build time (no HF token needed).
   - `chatbot-frontend:latest` — Flask proxy.
4. Loaded both images into the minikube node and applied all manifests in `k8s/`.
5. Enabled the `metrics-server` addon for HPA CPU metrics.
6. Verified `backend` and `frontend` deployments rolled out successfully (both pods `Running`, `1/1`).
7. Port-forwarded `svc/frontend` to `0.0.0.0:8080` and sent a live test request to `/chat`.
8. Confirmed the HPA (`backend-hpa`) is actively reporting live CPU metrics (`cpu: 0%/60%`).

## Issue encountered and fix

Minikube's node stayed `NotReady` because the CNI pod (`kindnet`) and a few `kube-system` addon
pods (`coredns`, `storage-provisioner`, `metrics-server`) could not resolve DNS from *inside* the
minikube node's container to pull their images (`dial tcp: lookup registry-1.docker.io ...: i/o
timeout`), even though the codespace host itself had working internet access. This is a networking
quirk of running Docker-in-Docker minikube inside a GitHub Codespace.

**Fix:** pulled the exact image references on the codespace host with `docker pull`, then used
`minikube image load <image>` to inject them directly into the minikube node's image cache,
bypassing the node's broken DNS path. The `metrics-server` addon also needed its deployment image
patched from a digest-pinned reference to a tag reference (`kubectl set image ...`) so it would
resolve to the locally-loaded image instead of trying to re-resolve the digest over the network.

This is worth knowing if you redeploy this repo inside a fresh Codespace — expect to repeat the
same `docker pull` + `minikube image load` workaround for `kindnetd`, `coredns`,
`storage-provisioner`, and `metrics-server`.

## End-to-end test result

Request:
```
curl -s -X POST http://localhost:8080/chat -H 'Content-Type: application/json' -d '{"message": "What is Kubernetes?"}'
```

Response (see `09-chat-test-response.json`): a real generated reply from the Qwen2.5-0.5B model,
confirming the frontend -> backend -> model inference path all work correctly.

## Log files in this directory

| File | What it shows |
|---|---|
| 01-minikube-start.log | minikube cluster startup |
| 02-build-backend.log / 02-build-frontend.log | Docker image builds |
| 03-image-load-backend.log / 03-image-load-frontend.log | Loading app images into minikube |
| 04-kubectl-apply.log | Applying k8s manifests |
| 05-metrics-server.log | Enabling metrics-server addon |
| 06-rollout-backend.log / 07-rollout-frontend.log | Deployment rollout status |
| 08-port-forward.log | Port-forward process log |
| 09-chat-test-response.json | Live chat endpoint test response |
| 10-final-status.log | Final pods/hpa/svc status |
| 11-backend-logs.log | Backend pod application logs |
