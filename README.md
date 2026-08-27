# forgejo-runner-k8s

An event-driven, autoscaling Forgejo Actions runner for Kubernetes. Designed specifically for self-hosted ARM64/AMD64 clusters using **KEDA** (Kubernetes Event-driven Autoscaling) and **Docker-in-Docker (DinD)** native sidecars.

```mermaid
flowchart TD
    subgraph Forgejo Server ["Forgejo Server (forgejo.example.com)"]
        Queue["Actions Job Queue (runs-on: arm64)"]
    end

    subgraph KEDA ["KEDA Controller"]
        Scaler["forgejo-runner Trigger"]
    end

    subgraph K8s ["Kubernetes Cluster"]
        Job["ScaledJob Pod (0 -> N Max)"]
        DinD["Sidecar: docker:dind"]
        Runner["Container: forgejo-runner"]
        Cache["Registry Cache Mirror (Optional)"]
    end

    Queue -. Polled every 10s .-> Scaler
    Scaler -- "Scale up (1-N jobs)" --> Job
    Job --> DinD
    Job --> Runner
    DinD -. Pull layers .-> Cache
    Runner -- "Registers (ephemeral)" --> Forgejo Server
    Runner -- "Executes job" --> DinD
    Runner -- "Deregisters & Exits" --> Job
```

---

## Features

- **True Event-Driven Autoscaling (0 to N)**: Consumes **0 CPU and 0 RAM** when idle. Pods are only spawned when jobs targeting specific labels (e.g., `arm64`) enter the Forgejo queue.
- **Auto-Deregistration (No Zombie Runners)**: Runners register dynamically with `--ephemeral`. When a job finishes, Forgejo automatically deregisters the runner, and Kubernetes terminates the Job pod.
- **Native K8s Sidecar DinD**: Leverages Kubernetes 1.28+ native sidecar containers (`restartPolicy: Always`) ensuring Docker is healthy before the runner starts, and terminates cleanly when the runner exits.
- **Pull-Through Registry Cache Support**: Connects directly to local registry mirrors (e.g., in-cluster registry cache) to accelerate container pulls and avoid Docker Hub rate limits.
- **Cluster Node Distribution**: Pod anti-affinity automatically spreads concurrent jobs across physical cluster nodes.
- **External Secrets Operator (ESO)**: Native support for syncing the runner registration token directly from external secret managers (Vault, Bitwarden, AWS Secrets Manager, etc.).

---

## Prerequisites

1. **Kubernetes 1.28+** (with privileged container support for DinD).
2. **KEDA v2.18+** installed in the cluster (`kedacore/keda` Helm chart).
3. A Forgejo instance (v15.0+ recommended) with Actions enabled.
4. An Actions Runner Registration Token from Forgejo:
   - Global: `https://your-forgejo/admin/actions/runners`
   - Or API: `GET /api/v1/admin/actions/runners/registration-token`

---

## Architecture Modes

### 1. `ScaledJob` Mode (Default & Recommended)
Spawns a dedicated, single-use Kubernetes `Job` for each queued workflow.
- Complete isolation between jobs.
- Clean shutdown and automatic pod cleanup (`ttlSecondsAfterFinished: 60`).
- No risk of rolling updates killing an in-flight build.

### 2. `ScaledObject` Mode
Scales a standard Kubernetes `Deployment` from 0 to N replicas based on queue depth.
- Replicas stay alive during `cooldownPeriod` (default 300s) to handle rapid back-to-back commits.

---

## Quickstart

### 1. Configure Secret (Manual or ExternalSecrets)

#### Via Manual Kubernetes Secret:
```bash
kubectl create namespace forgejo-runner
kubectl create secret generic forgejo-runner-secret \
  -n forgejo-runner \
  --from-literal=token="YOUR_REGISTRATION_TOKEN"
```

#### Via External Secrets Operator:
Set `secrets.externalSecrets.enabled=true` and specify your `secretStoreRef` and `remoteKey`.

### 2. Install Chart via Helm

```bash
helm upgrade --install forgejo-runner ./ \
  --namespace forgejo-runner \
  --create-namespace \
  --set forgejo.url="https://forgejo.example.com" \
  -f values.yaml
```

---

## Workflow Configuration

In your repository `.forgejo/workflows/` (e.g. `docker.yml`), target the runner using the configured labels:

### Native ARM64 Build Example
```yaml
name: Native ARM64 Build
on: [push]

jobs:
  build:
    runs-on: arm64
    steps:
      - uses: actions/checkout@v4
      - name: Verify Architecture
        run: |
          echo "Building natively on $(uname -m)"
          docker version
```

### Dual-Architecture Matrix (x86 + Native ARM64)
```yaml
name: Multi-Arch Build Matrix
on: [push]

jobs:
  build:
    strategy:
      matrix:
        include:
          - arch: amd64
            runs-on: docker # Runs on your x86 Docker runner
          - arch: arm64
            runs-on: arm64  # Autoscales onto your K8s ARM64 nodes!
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: Build Slice
        run: |
          docker build --platform linux/${{ matrix.arch }} -t myapp:${{ matrix.arch }} .
```

---

## Configuration Reference

| Parameter | Description | Default |
| :--- | :--- | :--- |
| `forgejo.url` | Forgejo instance base URL | `https://forgejo.example.com` |
| `forgejo.scope` | Scope of runner (`global`, `owner`, `org`, `repo`) | `global` |
| `runner.image.repository` | Forgejo runner image | `code.forgejo.org/forgejo/runner` |
| `runner.image.tag` | Image tag | `12` |
| `runner.labels` | Runner labels registered in Forgejo | `["arm64:...", "linux-arm64:..."]` |
| `runner.capacity` | Max concurrent jobs per runner pod | `1` |
| `dind.image.tag` | Docker-in-Docker image tag | `28-dind` |
| `dind.privileged` | Enable privileged mode for DinD | `true` |
| `dind.registryMirror` | Local pull-through cache mirror | `""` |
| `autoscaling.mode` | Scaling mode (`ScaledJob` or `ScaledObject`) | `ScaledJob` |
| `autoscaling.minReplicas` | Minimum replica count | `0` |
| `autoscaling.maxReplicas` | Maximum concurrent runners | `4` |
| `autoscaling.pollingInterval` | Seconds between queue checks | `10` |
| `autoscaling.triggerLabels` | Labels KEDA monitors for queue depth | `arm64` |
| `nodeSelector` | Node architecture selector | `kubernetes.io/arch: arm64` |
