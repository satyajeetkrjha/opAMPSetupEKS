# OpenTelemetry OpAMP Server on EKS Lab

This repository contains a comprehensive guide for deploying an OpenTelemetry OpAMP (Open Agent Management Protocol) server on Amazon EKS. The lab demonstrates the complete journey from local validation to production deployment.

## High-Level Architecture

```
┌────────────────────────────┐
│        EKS Cluster          │
│                             │
│  Namespace: opamp           │
│                             │
│  ┌──────────────────────┐  │
│  │   OpAMP Server Pod    │  │
│  │                      │  │
│  │  - Built from source │  │
│  │  - Runs on amd64     │  │
│  │  - Port 4320 (WS)    │  │
│  │  - Port 4321 (HTTP)  │  │
│  │  └──────────────────────┘  │
│                             │
└────────────────────────────┘
```

*Note: Agents will be added in future phases.*

## Prerequisites

- AWS CLI configured with appropriate permissions
- Docker with buildx support
- kubectl
- eksctl
- Git

## Phase 0: Local Validation (Critical First Step)

**🚨 Important:** Never deploy to EKS before validating locally!

Before touching Kubernetes, we validated the OpAMP flow locally using:
- `opampsupervisor`
- `otelcol-contrib`
- Local WebSocket connection

**Verified locally:**
- ✅ Agent connects successfully
- ✅ Effective config is generated
- ✅ Collector runs under supervisor control
- ✅ Metrics/logs are exported correctly

**Key Takeaway:** Always understand the local mental model before moving to distributed systems.

## Phase 1: Create EKS Cluster

Create the EKS cluster using eksctl:

```bash
eksctl create cluster \
  --name opamp-lab \
  --region us-east-1 \
  --version 1.29 \
  --nodegroup-name ng-1 \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 2 \
  --managed \
  --with-oidc
```

Verify cluster creation:

```bash
kubectl get nodes
kubectl get ns
```

Create the OpAMP namespace:

```bash
kubectl create namespace opamp
```

## Phase 2: Image Registry Challenges (Important Lesson)

### Initial Failure with GHCR

Our first attempt used the public GitHub Container Registry:

```bash
ghcr.io/open-telemetry/opamp-server/opamp-server:latest
```

**This failed with:**
- `403 Forbidden`
- `ImagePullBackOff`

**Root Cause:**
- GHCR requires authentication for this repository
- EKS nodes cannot pull it anonymously
- This is a very common real-world failure scenario

**Decision:** Build and own the image ourselves for full control.

## Phase 3: Build OpAMP Server from Source

Clone the OpAMP Go repository:

```bash
git clone https://github.com/open-telemetry/opamp-go.git
cd opamp-go
```

### Dockerfile (Critical Implementation Details)

```dockerfile
FROM golang:1.23-alpine AS build
RUN apk add --no-cache git make
WORKDIR /src
COPY . .
RUN make build-example-server

FROM alpine:3.20
RUN adduser -D -u 10001 app
USER 10001
WORKDIR /app
COPY --from=build /src/internal/examples/server/bin/server /app/opamp-server
EXPOSE 4320 4321
ENTRYPOINT ["/app/opamp-server"]
```

### Key Lessons Learned

1. **Go Version Compatibility:** 
   - `go.mod` required Go >= 1.23
   - Local Go version must match requirements

2. **Missing Build Tools:**
   - `make` was initially missing from container
   - Build failed until complete toolchain was available

3. **Source Code Structure:**
   - OpAMP server example is in `internal/examples/server/`
   - Build target: `make build-example-server`

## Phase 4: Architecture Mismatch Resolution

### The Problem
- **Local Development:** Mac ARM64
- **EKS Nodes:** AMD64

**Error Encountered:**
```
exec format error
```

### The Solution

Use Docker buildx for cross-platform builds:

```bash
# Create and use buildx builder
docker buildx create --name opampbuilder --use
docker buildx inspect --bootstrap

# Build for AMD64 platform
docker buildx build \
  --platform linux/amd64 \
  -t <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/opamp-go-example-server:v0.1 \
  --push .
```

## Phase 5: Amazon ECR Setup

Create ECR repository:

```bash
aws ecr create-repository \
  --repository-name opamp-go-example-server \
  --region us-east-1
```

Authenticate Docker with ECR:

```bash
aws ecr get-login-password --region us-east-1 \
 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

## Phase 6: Deploy OpAMP Server to EKS

### Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opamp-server
  namespace: opamp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: opamp-server
  template:
    metadata:
      labels:
        app: opamp-server
    spec:
      containers:
      - name: server
        image: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/opamp-go-example-server:v0.1
        ports:
        - containerPort: 4320  # WebSocket
        - containerPort: 4321  # HTTP
```

### Deploy and Verify

```bash
# Apply deployment
kubectl apply -f deployment.yaml

# Check rollout status
kubectl -n opamp rollout status deploy/opamp-server

# Verify pods are running
kubectl -n opamp get pods

# Check logs
kubectl -n opamp logs deploy/opamp-server
```

**Success Indicators:**
```
[MAIN] OpAMP Server starting...
[MAIN] OpAMP Server running...
```

## Phase 7: Service Exposure

### Internal Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: opamp-server
  namespace: opamp
spec:
  selector:
    app: opamp-server
  ports:
  - name: ws
    port: 4320
    targetPort: 4320
  - name: http
    port: 4321
    targetPort: 4321
```

### Testing with Port Forward

```bash
kubectl -n opamp port-forward svc/opamp-server 4320:4320 4321:4321
```

## Key Lessons Learned

1. **Always Validate Locally First**
   - Understand the technology stack before deploying to Kubernetes
   - Local validation saves hours of debugging in distributed environments

2. **Image Registry Authentication**
   - Public registries may require authentication
   - Building your own images provides full control
   - ECR integration with EKS is seamless

3. **Platform Architecture Matters**
   - Development and production architectures must match
   - Use buildx for cross-platform builds
   - Always specify target platform explicitly

4. **Incremental Deployment**
   - Start with basic deployment
   - Add complexity gradually
   - Verify each phase before proceeding

## Next Steps

- [ ] Add OpAMP agents to the cluster
- [ ] Implement configuration management
- [ ] Set up monitoring and observability
- [ ] Add security policies and RBAC
- [ ] Implement CI/CD pipeline

## Troubleshooting

### Common Issues

1. **ImagePullBackOff**
   - Check ECR authentication
   - Verify image exists and platform matches

2. **exec format error**
   - Rebuild image for correct platform (linux/amd64)

3. **Connection Refused**
   - Verify service is running
   - Check port-forward configuration
   - Confirm firewall/security group settings

### Useful Commands

```bash
# Check pod status
kubectl -n opamp get pods -o wide

# Describe pod for detailed info
kubectl -n opamp describe pod <pod-name>

# Check service endpoints
kubectl -n opamp get endpoints

# View recent events
kubectl -n opamp get events --sort-by='.lastTimestamp'
```

## Contributing

This lab is part of a larger OpenTelemetry hands-on learning series. Contributions and improvements are welcome!

## License

This project follows the OpenTelemetry project licensing.
