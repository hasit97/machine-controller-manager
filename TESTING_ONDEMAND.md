# Testing Machine Drain Timeout Changes in an On-Demand Environment

This guide walks through testing the **Disable Machine Drain Timeout** changes (`--disable-machine-drain-timeout`) in a Gardener on-demand environment.

---

## Overview of the Changes

The drain logic and flags live in `pkg/util/provider`, which is consumed by the provider-specific machine controller (`machine-controller-manager-provider-<provider>`):

1. **Global CLI Flag** (`pkg/util/provider/app/options/options.go`):
   - `--disable-machine-drain-timeout=true`: Globally disables the machine drain timeout.
2. **Drain Behavior When Disabled** (`pkg/util/provider/drain/drain.go`, `pkg/util/provider/machinecontroller/machine_util.go`):
   - `RunDrain` uses a context without a deadline (`context.WithCancel`) when `DisableMachineDrainTimeout` is enabled.
   - `HasTimeOutOccurred` is bypassed so the controller never switches to `ForceDeleteMachine`.
   - When eviction retries are exhausted (e.g., blocked by a `PodDisruptionBudget`), pods are **not** force-deleted; instead, the drain returns an error and the `Machine` remains in `Terminating` phase and retries on the next sync until the PDB allows eviction.

---

## Prerequisites

- Access to a Gardener environment configured in `gardenctl`.
- A Shoot cluster with at least **2 worker nodes** (so one node can be drained while the other hosts system pods).
- Local checkouts of both repositories side by side:
  - `machine-controller-manager` (this repository with the new changes)
  - `machine-controller-manager-provider-<provider>` (your cloud provider's MCM provider repository)

---

## Step 1: Link Your Local `machine-controller-manager` Changes to the Provider Repository

Because `pkg/util/provider` is executed by the provider binary (`cmd/machine-controller/main.go` in `machine-controller-manager-provider-<provider>`), point the provider's `go.mod` to your local `machine-controller-manager` checkout.

```bash
export MCM_DIR="/path/to/machine-controller-manager"
export PROVIDER_DIR="/path/to/machine-controller-manager-provider-<provider>"

cd "${PROVIDER_DIR}"

# Point to the local machine-controller-manager directory
go mod edit -replace github.com/gardener/machine-controller-manager="${MCM_DIR}"
go mod tidy
```

---

## Step 2: Target the On-Demand Shoot and Prepare Kubeconfigs

### 2a. Target the Shoot in `gardenctl`

```bash
export GCTL_SESSION_ID=$(gardenctl-session use "$(uuidgen)")
eval "$(gardenctl kubectl-env $(basename $SHELL))"

export GARDEN="<garden-name>"
export PROJECT="<project-name>"
export SHOOT="<shoot-name>"

gardenctl target --garden "${GARDEN}" --project "${PROJECT}" --shoot "${SHOOT}"
gardenctl target view
```

### 2b. Export Target & Control Kubeconfigs and Scale Down In-Cluster MCM

Export the **Target (Shoot)** and **Control (Seed)** kubeconfigs and scale down the in-cluster `machine-controller-manager` deployment so your local process can take over without interference from `gardener-resource-manager` or `dependency-watchdog`.

```bash
export WORKDIR="/tmp/mcm-ondemand"
export TARGET_KUBECONFIG="${WORKDIR}/kubeconfig_target.yaml"
export CONTROL_KUBECONFIG="${WORKDIR}/kubeconfig_control.yaml"
mkdir -p "${WORKDIR}"

# 1. Save Shoot (target) kubeconfig
gardenctl target unset --control-plane
eval "$(gardenctl kubectl-env $(basename $SHELL))"
cp "${KUBECONFIG}" "${TARGET_KUBECONFIG}"

# 2. Switch to Seed Control-Plane and save Control kubeconfig
gardenctl target --control-plane
eval "$(gardenctl kubectl-env $(basename $SHELL))"
cp "${KUBECONFIG}" "${CONTROL_KUBECONFIG}"

# 3. Prevent Gardener Resource Manager and Dependency Watchdog from reverting or scaling up MCM
kubectl annotate deployment/machine-controller-manager \
  resources.gardener.cloud/ignore=true \
  dependency-watchdog.gardener.cloud/ignore-scaling=true \
  --overwrite

# 4. Scale down the in-cluster MCM deployment
kubectl scale deployment/machine-controller-manager --replicas=0
```

### 2c. Generate `.env` Files for Local Execution

```bash
export MCM_DIR="/path/to/machine-controller-manager"
export PROVIDER_DIR="/path/to/machine-controller-manager-provider-<provider>"
export PROVIDER="<provider>"
export PROJECT="<project-name>"
export SHOOT="<shoot-name>"
export CONTROL_CLUSTER_NAMESPACE="shoot--${PROJECT}--${SHOOT}"
export WORKDIR="/tmp/mcm-ondemand"
export CONTROL_KUBECONFIG="${WORKDIR}/kubeconfig_control.yaml"
export TARGET_KUBECONFIG="${WORKDIR}/kubeconfig_target.yaml"

cd "${MCM_DIR}"

./hack/non_gardener_local_setup.sh \
  --namespace "${CONTROL_CLUSTER_NAMESPACE}" \
  --control-kubeconfig-path "${CONTROL_KUBECONFIG}" \
  --target-kubeconfig-path "${TARGET_KUBECONFIG}" \
  --provider "${PROVIDER}" \
  --mcm-provider-project-path "${PROVIDER_DIR}"
```

---

## Step 3: Start `machine-controller-manager` and `machine-controller` Locally

Open two terminals:

### Terminal 1: Start `machine-controller-manager`

```bash
export MCM_DIR="/path/to/machine-controller-manager"

cd "${MCM_DIR}"
LEADER_ELECT=false make start
```

### Terminal 2: Start `machine-controller` with Drain Timeout Disabled

To test the global flag (`--disable-machine-drain-timeout=true`) and speed up eviction retry feedback during testing (`--machine-max-evict-retries=3`), run the provider controller.

```bash
export PROVIDER_DIR="/path/to/machine-controller-manager-provider-<provider>"

cd "${PROVIDER_DIR}"
source .env

go run cmd/machine-controller/main.go \
  --control-kubeconfig="${CONTROL_KUBECONFIG}" \
  --target-kubeconfig="${TARGET_KUBECONFIG}" \
  --namespace="${CONTROL_CLUSTER_NAMESPACE}" \
  --disable-machine-drain-timeout=true \
  --machine-max-evict-retries=3 \
  --leader-elect=false \
  --v=4
```

---

## Step 4: Create a Blocking Workload and `PodDisruptionBudget` on a Worker Node

Using `TARGET_KUBECONFIG`, pick one worker node to test draining and schedule a pod protected by a strict `PodDisruptionBudget` (`minAvailable: 1`) on that node.

```bash
export WORKDIR="/tmp/mcm-ondemand"
export TARGET_KUBECONFIG="${WORKDIR}/kubeconfig_target.yaml"

# Pick a worker node
export TARGET_NODE=$(KUBECONFIG="${TARGET_KUBECONFIG}" kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
echo "Target Node: ${TARGET_NODE}"
```

Apply a test Pod pinned to `${TARGET_NODE}` and a `PodDisruptionBudget` that blocks all evictions:

```bash
cat <<EOF | KUBECONFIG="${TARGET_KUBECONFIG}" kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: drain-test
---
apiVersion: v1
kind: Pod
metadata:
  name: pdb-protected-pod
  namespace: drain-test
  labels:
    app: pdb-test
spec:
  nodeName: ${TARGET_NODE}
  terminationGracePeriodSeconds: 30
  containers:
  - name: pause
    image: registry.k8s.io/pause:3.9
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: block-eviction-pdb
  namespace: drain-test
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: pdb-test
EOF
```

Wait until the pod is `Running`:

```bash
KUBECONFIG="${TARGET_KUBECONFIG}" kubectl -n drain-test wait --for=condition=Ready pod/pdb-protected-pod --timeout=60s
```

---

## Step 5: Trigger Machine Drain and Verify Behavior

Using `CONTROL_KUBECONFIG`, find the `Machine` corresponding to `${TARGET_NODE}` and trigger its deletion.

```bash
export WORKDIR="/tmp/mcm-ondemand"
export CONTROL_KUBECONFIG="${WORKDIR}/kubeconfig_control.yaml"
export PROJECT="<project-name>"
export SHOOT="<shoot-name>"
export CONTROL_CLUSTER_NAMESPACE="shoot--${PROJECT}--${SHOOT}"
export TARGET_NODE="<worker-node-name>"

export TARGET_MACHINE=$(KUBECONFIG="${CONTROL_KUBECONFIG}" kubectl -n "${CONTROL_CLUSTER_NAMESPACE}" get machines -l "node=${TARGET_NODE}" -o jsonpath='{.items[0].metadata.name}')
echo "Target Machine: ${TARGET_MACHINE}"

# Trigger deletion (and drain) of the Machine without waiting for completion
KUBECONFIG="${CONTROL_KUBECONFIG}" kubectl -n "${CONTROL_CLUSTER_NAMESPACE}" delete machine "${TARGET_MACHINE}" --wait=false
```

### Expected Results While Blocked by PDB

1. **Node is cordoned** on the Shoot cluster (`SchedulingDisabled`).
2. **Provider logs (Terminal 2)** show eviction attempts failing due to the PDB and exhausting retries **without** falling back to force deletion:
   ```text
   Machine drain started on ... for "<worker-node-name>"
   Evicting pod drain-test/pdb-protected-pod on node "<worker-node-name>" ...
   error when evicting pod "pdb-protected-pod": eviction retries exhausted without force deletion scheduled on node <worker-node-name>
   ```
3. **Pod remains running** on the Shoot cluster (not force-deleted):
   ```bash
   export WORKDIR="/tmp/mcm-ondemand"
   export TARGET_KUBECONFIG="${WORKDIR}/kubeconfig_target.yaml"

   KUBECONFIG="${TARGET_KUBECONFIG}" kubectl -n drain-test get pod pdb-protected-pod
   ```
4. **Machine remains in `Terminating` phase** (not force-deleted at the cloud provider) and reports that drain failed and will retry in the next sync:
   ```bash
   export WORKDIR="/tmp/mcm-ondemand"
   export CONTROL_KUBECONFIG="${WORKDIR}/kubeconfig_control.yaml"
   export PROJECT="<project-name>"
   export SHOOT="<shoot-name>"
   export CONTROL_CLUSTER_NAMESPACE="shoot--${PROJECT}--${SHOOT}"
   export TARGET_MACHINE="<machine-name>"

   KUBECONFIG="${CONTROL_KUBECONFIG}" kubectl -n "${CONTROL_CLUSTER_NAMESPACE}" get machine "${TARGET_MACHINE}" -o jsonpath='{.status.currentStatus.phase}{"\n"}{.status.lastOperation.description}{"\n"}'
   ```

---

## Step 6: Unblock the PDB and Verify Clean Machine Deletion

Delete the `PodDisruptionBudget` on the Shoot cluster to allow the eviction to succeed on the next retry.

```bash
export WORKDIR="/tmp/mcm-ondemand"
export TARGET_KUBECONFIG="${WORKDIR}/kubeconfig_target.yaml"

KUBECONFIG="${TARGET_KUBECONFIG}" kubectl delete pdb -n drain-test block-eviction-pdb
```

Observe that on the next sync cycle:
- `pdb-protected-pod` is gracefully evicted respecting its `terminationGracePeriodSeconds`.
- The node drain succeeds.
- The `Machine` (`${TARGET_MACHINE}`) is deleted from `${CONTROL_CLUSTER_NAMESPACE}`, and the `MachineSet` provisions a replacement node if desired replicas > current replicas.

Clean up the test namespace:

```bash
KUBECONFIG="${TARGET_KUBECONFIG}" kubectl delete namespace drain-test
```

---

## Step 7: Restore the On-Demand Environment

Stop the local `machine-controller` and `machine-controller-manager` processes (`Ctrl+C` in both terminals), then restore the in-cluster `machine-controller-manager` deployment and clean up local artifacts.

```bash
export MCM_DIR="/path/to/machine-controller-manager"
export PROVIDER_DIR="/path/to/machine-controller-manager-provider-<provider>"
export WORKDIR="/tmp/mcm-ondemand"
export CONTROL_KUBECONFIG="${WORKDIR}/kubeconfig_control.yaml"
export PROJECT="<project-name>"
export SHOOT="<shoot-name>"
export CONTROL_CLUSTER_NAMESPACE="shoot--${PROJECT}--${SHOOT}"

# Remove ignore annotations and scale MCM back to 1
KUBECONFIG="${CONTROL_KUBECONFIG}" kubectl -n "${CONTROL_CLUSTER_NAMESPACE}" annotate deployment/machine-controller-manager \
  resources.gardener.cloud/ignore- \
  dependency-watchdog.gardener.cloud/ignore-scaling-
KUBECONFIG="${CONTROL_KUBECONFIG}" kubectl -n "${CONTROL_CLUSTER_NAMESPACE}" scale deployment/machine-controller-manager --replicas=1

# Clean up local .env and temporary kubeconfigs
rm -rf "${WORKDIR}"
rm -f "${MCM_DIR}/.env"
rm -f "${PROVIDER_DIR}/.env"

# Revert the temporary replace directive in the provider repository
cd "${PROVIDER_DIR}"
go mod edit -dropreplace github.com/gardener/machine-controller-manager
go mod tidy
```
