# AI381 Part 1 — Kubernetes runbook

**Status: NOT YET TESTED.** Deployment, endpoint, recovery, and rollback tests
have not been performed. A minikube dry-run reportedly succeeded with a memory
headroom warning; this does not prove that a cluster or application will run.
Commands below are procedures for later manual execution, not recorded results.

Run commands one at a time and inspect each result. **Stop immediately if any
command fails unexpectedly**, including a timeout or an incorrect response.
Do not run subsequent commands blindly or paste whole blocks expecting them to
stop automatically. Diagnose the failure before deliberately choosing a retry,
recovery, or cleanup step.

## 1. Prerequisites and image integration

- Run from `/home/qatada/ai381-project1/group08` in WSL2. Have minikube v1.39.0,
  kubectl, curl, and Docker Desktop with WSL integration and Linux containers.
  Confirm Docker is accessible from this terminal before proceeding.
- Perform a fresh memory assessment before startup: recheck `free -h` in WSL,
  Windows Task Manager's available memory, and Docker's resource budget. Our
  previous available RAM (about 1.2–1.6 GiB in WSL and 0.62 GiB in Windows) is
  insufficient for confidently starting a 2 GiB minikube node. These figures
  are not independent pools to add together. Require enough current headroom
  for the node plus Windows, WSL, Docker, and other workloads; otherwise defer
  startup. Never bypass resource checks with `--force`. Agree before temporarily
  stopping unrelated workloads such as MLflow or the standalone FastAPI
  container, then reassess memory. Avoid optional addons.
- **Student 3 must supply the actual GHCR image**, preferably an immutable
  digest or unique version tag, and confirm minikube can pull it. If private,
  agree on registry authentication separately; no pull secret is configured.
- In a separate reviewed integration change, replace
  `example.invalid/ai381-inference:replace-me` in `deployment.yaml` with that
  image. **Do not apply the current placeholder Deployment.** This runbook does
  not change either manifest or assume any GHCR image exists.
- Confirm with API/model teammates: the image starts without extra arguments,
  listens on `0.0.0.0:8000`, supplies its model/runtime dependencies, and defines
  a numeric non-zero user with appropriate file permissions. No environment
  variables, model volumes, or secrets are currently supplied by the manifests.
- Confirm GET `/health` is quick and unauthenticated, returning 200 when ready,
  and suitable for liveness checks. Obtain a valid POST `/predict` JSON payload
  and expected response from the API teammate; no request schema is assumed.

The Deployment `ml-inference` runs two Pods labeled `app: ml-inference` with
container `api`, named port `http` (8000). The ClusterIP Service `ml-inference`
selects those Pods and forwards port 8000 to `http`. All probes use `/health`:
startup allows about 150 seconds; readiness checks every 5 seconds; liveness
checks every 10 seconds. Each has a 2-second timeout. Startup gates the other
probes. The Pod requires non-root execution; privilege escalation is disabled
and Linux capabilities are dropped.

Each Pod requests 100m CPU / 128Mi and is limited to 500m / 512Mi. Two Pods can
use up to 1Gi memory. With `maxUnavailable: 0` and `maxSurge: 1`, a rolling
update normally requires up to three Pods (1.5Gi), excluding Kubernetes
overhead. Terminating Pods may remain temporarily, increasing actual Pod count
and resource pressure beyond that estimate. A rollout can stall if there is
no spare capacity. A 2-CPU, 2048mb node is only a minimal trial,
not a reliability guarantee. Reassess capacity before starting; do not change
replicas, limits, or rollout policy without a team decision.

## 2. Start and validate — NOT YET TESTED

Run only after prerequisites and a fresh memory assessment confirm sufficient
headroom. Do not proceed using the previously insufficient RAM figures:

```bash
minikube start --profile=minikube --driver=docker --cpus=2 --memory=2048mb --nodes=1
```

Use an explicit context and namespace to avoid another cluster. Define this
helper in each terminal where the following `k` commands are used:

```bash
cd /home/qatada/ai381-project1/group08
k() { kubectl --context=minikube --namespace=default "$@"; }
k get nodes
k get pods -n kube-system
k apply --dry-run=server -f k8s/deployment.yaml -f k8s/service.yaml
```

Expect a Ready node and healthy system Pods before proceeding. Server dry-run
needs the running API server, validates without persisting these resources,
and does not test image pulls or inference. A localhost:8080 connection error
indicates missing/unusable cluster configuration, not proof of invalid YAML.

## 3. Deploy and verify — NOT YET TESTED

After the real image has been integrated and validation succeeds:

```bash
k apply -f k8s/deployment.yaml -f k8s/service.yaml
k rollout status deployment/ml-inference --timeout=300s
k get deployment ml-inference
k get pods -l app=ml-inference -o wide
k get service ml-inference
k get endpointslices -l kubernetes.io/service-name=ml-inference
```

Expect two available replicas, two Running Pods each showing `1/1` Ready,
and ready Service endpoints. A timeout requires diagnosis, not a success claim.
ClusterIP is internal; in a separate terminal with the helper defined, run:

```bash
k port-forward service/ml-inference 18000:8000 --address=127.0.0.1
```

Keep that terminal open. In another WSL terminal:

```bash
curl --fail-with-body -i http://127.0.0.1:18000/health
# Set PREDICT_JSON to the exact JSON supplied by the API teammate first.
: "${PREDICT_JSON:?Set PREDICT_JSON to a teammate-confirmed valid request}"
curl --fail-with-body -i http://127.0.0.1:18000/predict \
  -H 'Content-Type: application/json' --data-binary "$PREDICT_JSON"
```

Compare responses with the agreed contract. Port forwarding exercises one
selected Pod; it does not prove balancing across replicas. Stop forwarding
with Ctrl+C. Restart it after recovery or rollout if its selected Pod exits.

## 4. Diagnose failures — NOT YET TESTED

List Pods, then replace `POD_NAME` below with an actual name (do not paste it
literally). Inspect before changing settings:

```bash
k get pods -l app=ml-inference -o wide
k describe pod POD_NAME
k logs POD_NAME -c api --tail=100
k logs POD_NAME -c api --previous --tail=100
k get events --sort-by=.metadata.creationTimestamp
k describe node minikube
```

Previous logs exist only when that container has restarted.

| Symptom | Check and next decision |
| --- | --- |
| ImagePullBackOff / ErrImagePull | Read Pod events. Check that the placeholder was replaced, tag/digest exists, registry is reachable, and GHCR permissions/authentication are correct. Ask Student 3; do not assume Docker login credentials automatically reach Kubernetes. |
| CrashLoopBackOff | Read current/previous logs and termination reason. Check startup errors, missing image-bundled dependencies, non-root permissions, and startup/liveness failures. Fix the image or agreed configuration rather than disabling security/probes. |
| Running but not Ready | Read probe events. Verify bind address, port, `/health`, model initialization time, and 2-second timeout. No ready endpoints means the Service has no ready backend. |
| Pending / resource exhaustion | Inspect scheduling events, node conditions, and requests. `OOMKilled` suggests the memory limit was exceeded; node `MemoryPressure`/evictions indicate broader pressure. CPU throttling can slow probes. Measure before agreeing on changes. |

`kubectl top` requires Metrics Server; its absence is not an application failure.
No monitoring addon is required for these initial checks.

## 5. Demonstrate Pod recovery — NOT YET TESTED

Only after both replicas are Ready, record their names and UIDs:

```bash
k get pods -l app=ml-inference -o custom-columns=NAME:.metadata.name,UID:.metadata.uid,READY:.status.containerStatuses[0].ready
```

Select **one** actual Pod name. The following intentionally deletes that Pod;
never substitute a selector that deletes both:

```bash
k delete pod POD_NAME
k get pods -l app=ml-inference -w
```

Wait for the replacement to reach `1/1` Ready, then Ctrl+C the watch. Confirm
two available replicas and a new Pod UID; repeat endpoint tests. Record results
and recovery time. This demonstrates controller replacement, not node failure
tolerance or liveness-triggered restart. One local node is still a single
point of failure. Do not use forced deletion.

## 6. Demonstrate rollout and rollback — NOT YET TESTED

Requires a healthy, tested baseline image and a second distinct, pullable image
from Student 3. The initial placeholder is never a valid rollback target.
The Deployment retains three old revisions; rollback needs retained history.
Before recording `GOOD_REVISION`, verify the current baseline rollout:

```bash
k rollout status deployment/ml-inference --timeout=300s
k get deployment ml-inference
k get pods -l app=ml-inference -o wide
```

Require a successful rollout and two available, Ready replicas. Then repeat
the section 3 port-forward and endpoint tests: GET `/health` must return 200,
and POST `/predict` with the teammate-confirmed payload must return the agreed
successful response. Stop if any check fails; do not record that revision as
known-good. Only after all checks pass, in one terminal with the helper defined:

```bash
k rollout history deployment/ml-inference
GOOD_REVISION=$(k get deployment ml-inference -o jsonpath='{.metadata.annotations.deployment\.kubernetes\.io/revision}')
k get deployment ml-inference -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
# Set CANDIDATE_IMAGE to the second teammate-provided tag/digest first.
: "${GOOD_REVISION:?A known-good deployed revision is required}"
: "${CANDIDATE_IMAGE:?Set CANDIDATE_IMAGE to the second agreed image}"
k set image deployment/ml-inference api="$CANDIDATE_IMAGE"
k rollout status deployment/ml-inference --timeout=300s
k rollout history deployment/ml-inference
```

Record images/revisions and test endpoints. If rollout stalls, diagnose it;
Kubernetes does not automatically roll back this Deployment. Restore the
recorded, tested baseline explicitly:

```bash
k rollout undo deployment/ml-inference --to-revision="$GOOD_REVISION"
k rollout status deployment/ml-inference --timeout=300s
k get pods -l app=ml-inference -o wide
k get deployment ml-inference -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Verify the baseline image, two ready replicas, and both endpoints again. Runtime
`set image`/`undo` changes do not edit Git manifests; reconcile the agreed image
in a later reviewed change before reapplying manifests, which would otherwise
overwrite the runtime image. Save evidence only after running the demonstration.

## 7. Cleanup — NOT YET TESTED; choose manually

- Ctrl+C any port-forward or watch commands.
- To stop the cluster while retaining its state for later use:
  `minikube stop --profile=minikube`.
- Only if intentionally removing this application's resources while the cluster
  is running: `k delete -f k8s/deployment.yaml -f k8s/service.yaml`.
- Only if intentionally discarding the entire local cluster and its state:
  `minikube delete --profile=minikube`. This is destructive and optional.
- Restore any unrelated workloads you deliberately stopped after capacity is
  available. None are automatically stopped or restarted by this runbook.

References: [minikube start](https://minikube.sigs.k8s.io/docs/commands/start/),
[Kubernetes Deployments and rollback](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).
