# OpAMP Supervisor + OTel Collector on EKS
## End-to-End Reproducible Lab

## Goal

Build and verify an OpAMP-managed OpenTelemetry Agent where:

- OpAMP Supervisor runs as PID 1
- Supervisor connects to OpAMP Server
- Supervisor spawns otelcol-contrib
- Collector runs successfully
- Remote config pushed from UI is applied live
- Changes appear in `/storage/effective.yaml`

## Architecture (Mental Model)

```
OpAMP Server (Control Plane)
        │
        │ WebSocket (OpAMP)
        ▼
OpAMP Supervisor (PID 1, inside pod)
        │
        │ Local WS (127.0.0.1:<random>)
        ▼
OTel Collector (otelcol-contrib)
```

**Key Rule:** Supervisor manages the collector lifecycle. Collector never talks to OpAMP Server directly.

## Step 1: Build the OpAMP Agent Image

### Dockerfile (Final, Working)

```dockerfile
FROM otel/opentelemetry-collector-opampsupervisor:0.136.0 AS supervisor
FROM otel/opentelemetry-collector-contrib:0.136.0 AS otel
FROM alpine:3.20

COPY --from=supervisor /usr/local/bin/opampsupervisor /opampsupervisor
COPY --from=otel /otelcol-contrib /otelcol-contrib

COPY supervisor.yaml /supervisor.yaml
COPY collector.yaml /collector.yaml

ENTRYPOINT ["/opampsupervisor", "--config=/supervisor.yaml"]
```

### Build and Push to ECR

```bash
docker build -t opamp-agent:v0.1 .
docker tag opamp-agent:v0.1 <ECR_REPO>:v0.1
docker push <ECR_REPO>:v0.1
```

## Step 2: Base Collector Config (collector.yaml)

Minimal, safe, debug-only pipeline:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

exporters:
  debug:
    verbosity: detailed

service:
  pipelines:
    logs:
      receivers: [otlp]
      exporters: [debug]
    metrics:
      receivers: [otlp]
      exporters: [debug]
    traces:
      receivers: [otlp]
      exporters: [debug]
```

## Step 3: Supervisor Config (supervisor.yaml)

**Important:**
- No `collector:` block
- Collector is started via `agent.executable`

```yaml
server:
  endpoint: wss://opamp-server.opamp.svc.cluster.local:4320/v1/opamp
  tls:
    insecure_skip_verify: true

storage:
  directory: /storage

capabilities:
  accepts_remote_config: true
  reports_effective_config: true
  reports_own_metrics: true
  reports_own_logs: true
  reports_health: true
  reports_remote_config: true

agent:
  executable: /otelcol-contrib
  args:
    - "--config=/collector.yaml"
```

## Step 4: Deploy OpAMP Agent to EKS

### ConfigMap

```bash
kubectl -n opamp create configmap opamp-agent-config \
  --from-file=supervisor.yaml \
  --from-file=collector.yaml \
  -o yaml --dry-run=client | kubectl apply -f -
```

### Deployment Requirements

Your deployment must:
- Mount configmap files at `/supervisor.yaml` and `/collector.yaml`
- Mount `/storage` as emptyDir
- Use the built image

## Step 5: Rollout / Update Workflow (Repeatable)

This is the official workflow you should always use.

**Update config → restart → verify**

```bash
kubectl -n opamp rollout restart deploy/opamp-agent
kubectl -n opamp rollout status deploy/opamp-agent --timeout=180s
```

## Step 6: Verify Processes (Hard Proof)

### Confirm supervisor + collector are running

```bash
POD=$(kubectl -n opamp get pod -l app=opamp-agent -o jsonpath='{.items[0].metadata.name}')

kubectl -n opamp exec -it $POD -c agent -- sh -c \
'ps -ef | egrep "opampsupervisor|otelcol-contrib"'
```

**Expected:**
```
1   root  /opampsupervisor --config=/supervisor.yaml
XX  root  /otelcol-contrib ...
```

**If collector is missing** → supervisor config is wrong.

## Step 7: Verify Supervisor ↔ Server Connection

```bash
kubectl -n opamp logs deploy/opamp-agent -c agent --since=5m
```

**Look for:**
- `Supervisor starting`
- `Connected to the server.`

**Ignore single occurrences of:**
- `connection reset by peer`

Those are local WS restarts during reload.

## Step 8: Verify Effective Config Generation

```bash
kubectl -n opamp exec -it $POD -c agent -- sh -c \
'ls -l /storage/effective.yaml && head -n 40 /storage/effective.yaml'
```

**This proves:**
- Supervisor is managing config
- Collector is running under OpAMP control

## Step 9: Push Remote Config from UI (Critical Step)

### UI Access

Open: `http://<opamp-server>:4321`

Go to **Configuration** → **Additional Configuration**

### Paste this overlay (safe test)

```yaml
processors:
  attributes/test_opamp:
    actions:
      - key: opamp.ui_test
        value: "hello-from-ui"
        action: insert

service:
  pipelines:
    traces:
      processors: [attributes/test_opamp]
```

Click **Save and Send to Agent**

## Step 10: Verify Remote Config Applied

### Effective config proof (strongest signal)

```bash
kubectl -n opamp exec -it $POD -c agent -- sh -c \
'grep -R "test_opamp" -n /storage/effective.yaml'
```

**Expected output:**
```
attributes/test_opamp
```

**This confirms:** UI → OpAMP Server → Supervisor → Collector end-to-end.

## Step 11: Common Failure Modes (Lessons Learned)

### ❌ Supervisor crashes at startup

**Cause:**
- Invalid keys (`collector:` block, `agent.id`, etc.)

**Fix:**
- Use only supported schema
- Collector started via `agent.executable`

### ❌ websocket: bad handshake

**Cause:**
- Wrong port / protocol / path

**Fix:**
Ensure:
- 4320 is WS
- `/v1/opamp` is correct
- `ws` vs `wss` matches server

### ❌ Collector not running

**Cause:**
- Supervisor connected but no collector spawn

**Fix:**
- Define `agent.executable`
- Ensure binary exists
- Check `ps -ef`

## Final Mental Model (Lock This In)

1. **Supervisor is the agent**
2. **Collector is a managed subprocess**
3. **UI pushes overlays, not full configs**
4. **effective.yaml is the truth**
5. **Logs are secondary**

## Troubleshooting Commands

```bash
# Get pod name
POD=$(kubectl -n opamp get pod -l app=opamp-agent -o jsonpath='{.items[0].metadata.name}')

# Check processes
kubectl -n opamp exec -it $POD -c agent -- ps -ef

# Check logs
kubectl -n opamp logs deploy/opamp-agent -c agent --tail=50

# Check effective config
kubectl -n opamp exec -it $POD -c agent -- cat /storage/effective.yaml

# Check storage directory
kubectl -n opamp exec -it $POD -c agent -- ls -la /storage/

# Restart deployment
kubectl -n opamp rollout restart deploy/opamp-agent
```

## Success Criteria

✅ **Supervisor starts as PID 1**  
✅ **Collector spawns as child process**  
✅ **WebSocket connection established**  
✅ **effective.yaml generated**  
✅ **Remote config from UI applied**  
✅ **Changes visible in effective.yaml**  

When all criteria are met, you have a fully functional OpAMP-managed OpenTelemetry agent running on EKS.
