# llm-d on DigitalOcean Kubernetes Service (DOKS)

This document covers configuring DOKS clusters for running high performance LLM inference with llm-d.

## Prerequisites

llm-d on DOKS is tested with the following configurations:

* GPU types: NVIDIA H100, NVIDIA RTX 6000 Ada, NVIDIA RTX 4000 Ada, NVIDIA L40S
* Versions: DOKS 1.33.1-do.3
* Networking: VPC-native clusters (required)

## Configuration Architecture

The DigitalOcean deployment follows clean configuration principles:

* **Base Configuration**: `values.yaml` files maintain original design intent with high-end specs
* **Platform Overrides**: `digitalocean-values.yaml` files contain ONLY DigitalOcean-specific modifications
* **Conditional Loading**: Using `digitalocean` environment selectively applies DigitalOcean overrides

This approach ensures:
- Original configurations remain unchanged for other platforms
- DigitalOcean optimizations are isolated and maintainable
- Clear separation between base architecture and platform adaptations

## Cluster Configuration

The DOKS cluster should be configured with the following settings:

* [GPU-enabled node pools](https://docs.digitalocean.com/products/kubernetes/details/supported-gpus/) with at least 2 GPU nodes for P/D disaggregation
* [VPC-native networking](https://docs.digitalocean.com/products/kubernetes/details/networking/) (default for new clusters)
* [kubectl configured](https://docs.digitalocean.com/products/kubernetes/how-to/connect-to-cluster/) for cluster access

### GPU Driver Management

DigitalOcean automatically installs and manages GPU drivers on DOKS clusters:

* **NVIDIA Device Plugin**: Automatic installation for GPU discovery and scheduling
* **Driver Updates**: Managed alongside cluster updates
* **GPU Monitoring**: Built-in metrics collection via DCGM Exporter

Verify automatic GPU setup:
```bash
kubectl get pods -n nvidia-device-plugin-system
kubectl get nodes -o custom-columns="NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu"
```

## Quick Start

### Step 1: Install Prerequisites

Before deploying llm-d workloads, install the required components:

```bash
# Navigate to gateway provider prerequisites
cd guides/prereq/gateway-provider

# Install Gateway API and Inference Extension CRDs
./install-gateway-provider-dependencies.sh

# Install Istio control plane
helmfile apply -f istio.helmfile.yaml
```

### Step 2: Cluster Validation

Verify your cluster setup:

```bash
# Verify cluster access and GPU nodes
kubectl cluster-info
kubectl get nodes -l doks.digitalocean.com/gpu-brand=nvidia

# Verify components are ready
kubectl get pods -n istio-system
```

### Step 3: Deploy Workloads

Use the `digitalocean` environment to automatically load DigitalOcean-specific value overrides:

```bash
# For inference scheduling (2 decode pods)
cd guides/inference-scheduling
export NAMESPACE=llm-d-inference-scheduling
helmfile apply -e digitalocean -n ${NAMESPACE}

# For P/D disaggregation (1 prefill + 1 decode pod)
cd guides/pd-disaggregation
export NAMESPACE=llm-d-pd
helmfile apply -e digitalocean -n ${NAMESPACE}
```

**Key DigitalOcean Optimizations Applied Automatically:**
- **Smaller Models**: Uses `Qwen3-0.6B` (inference-scheduling) and `Qwen2.5-3B-Instruct` (P/D) that don't require HuggingFace tokens
- **Stable Images**: Uses production-ready `ghcr.io/llm-d/llm-d:v0.2.0` instead of development builds
- **DOKS-Optimized Resources**: Reduced memory/CPU requirements suitable for DOKS GPU nodes
- **GPU Tolerations**: Automatic scheduling on DigitalOcean GPU nodes with `nvidia.com/gpu` taints
- **No RDMA**: Removes InfiniBand requirements not available on DOKS

**Architecture Overview:**

- **Inference Scheduling**: 2 decode pods with intelligent routing via InferencePool
- **P/D Disaggregation**: 1 prefill pod + 1 decode pod with automatic InferencePool and HTTPRoute creation

### Step 4: Testing

Verify deployment success:

```bash
# Check deployment status for inference scheduling
kubectl get pods -n llm-d-inference-scheduling
kubectl get gateway -n llm-d-inference-scheduling

# Check deployment status for P/D disaggregation
kubectl get pods -n llm-d-pd
kubectl get gateway,inferencepool,httproute -n llm-d-pd

# Test inference endpoint (P/D disaggregation example)
kubectl port-forward -n llm-d-pd svc/infra-pd-inference-gateway-istio 8080:80

curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen2.5-3B-Instruct", "messages": [{"role": "user", "content": "hello"}], "max_tokens": 20}'

# Test inference endpoint (inference scheduling example)
kubectl port-forward -n llm-d-inference-scheduling svc/infra-inference-scheduling-inference-gateway-istio 8080:80

curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3-0.6B", "messages": [{"role": "user", "content": "hello"}], "max_tokens": 20}'
```

## Monitoring (Optional)
Deploy Prometheus and Grafana for observability:

```bash
cd monitoring
./setup-monitoring.sh

# Access Grafana dashboard
kubectl port-forward -n llm-d-monitoring svc/prometheus-grafana 3000:80
```

We recommend enabling the monitoring stack to track:
- GPU utilization per deployment
- Inference request latency and throughput
- Memory usage and KV cache efficiency
- Network performance between prefill/decode pods

## DigitalOcean-Specific Configuration Details

### Model Selection

The DigitalOcean deployment uses smaller, optimized models:

| Architecture | Original Model | DigitalOcean Model | Benefits |
|-------------|----------------|--------------------|---------|
| Inference Scheduling | `Qwen3-0.6B` + HF Token | `Qwen3-0.6B` (no token) | No authentication required |
| P/D Disaggregation | `Llama-3.3-70B` + HF Token | `Qwen2.5-3B-Instruct` (no token) | 8x smaller, faster loading |

### Resource Optimization

**Original Specs (for reference):**
```yaml
# P/D Disaggregation - Original
resources:
  limits:
    memory: 64Gi
    cpu: "16"  # decode: 16, prefill: 8
    nvidia.com/gpu: "4"  # decode: 4, prefill: 1
    rdma/ib: 1
replicas: 4  # prefill replicas
```

**DigitalOcean Optimized:**
```yaml
# P/D Disaggregation - DigitalOcean
resources:
  limits:
    memory: 16Gi
    cpu: "4"
    nvidia.com/gpu: "1"
    # No RDMA requirement
replicas: 1  # prefill replicas
```

### GPU Node Configuration

DigitalOcean DOKS GPU nodes use taints to prevent non-GPU workloads from scheduling:

```yaml
# Automatically applied tolerations
tolerations:
- key: "nvidia.com/gpu"
  operator: "Exists"
  effect: "NoSchedule"
```

### Architecture Differences

**Inference Scheduling on DOKS:**
- 2 decode pods with InferencePool routing
- Single GPU per pod (optimal for DOKS node sizes)
- Intelligent request distribution

**P/D Disaggregation on DOKS:**
- 1 prefill pod (handles initial token processing)
- 1 decode pod (handles generation)
- InferencePool + HTTPRoute for proper routing
- Reduced from 4+4 GPUs to 1+1 GPUs total

## Troubleshooting

### Common Issues

#### 1. CRD Not Found Errors During Deployment

**Error**: `resource mapping not found for name: "..." kind: "Gateway"`

**Cause**: Required CRDs not installed before deployment

**Solution**: Install CRDs before any helmfile deployment:
```bash
cd guides/prereq/gateway-provider
./install-gateway-provider-dependencies.sh
helmfile apply -f istio.helmfile.yaml
```

#### 2. LoadBalancer Pending or API Errors

**Error**: LoadBalancer stuck in `<pending>` state with API errors

**Cause**: DigitalOcean API rate limiting or concurrent LoadBalancer operations

**Solution**:
```bash
# Check LoadBalancer status
kubectl describe svc <service-name> -n <namespace>

# Wait for API operations to complete (typically 2-3 minutes)
# Sequential deployments avoid conflicts
```

#### 3. Pods Fail to Schedule on GPU Nodes

**Error**: `untolerated taint {nvidia.com/gpu}`

**Cause**: DigitalOcean GPU nodes have automatic taints to prevent non-GPU workloads

**Solution**: DigitalOcean values automatically include required tolerations. Verify they're applied:
```bash
kubectl describe pod <pod-name> -n <namespace> | grep Tolerations

# Should show:
# Tolerations: nvidia.com/gpu:NoSchedule op=Exists
```

If tolerations are missing, ensure you're using the `digitalocean` environment which loads DigitalOcean overrides.

#### 4. InferencePool or HTTPRoute Not Created (P/D Disaggregation)

**Error**: HTTPRoute references non-existent InferencePool

**Solution**: Verify DigitalOcean values override is loading correctly:
```bash
kubectl get inferencepool -n llm-d-pd
kubectl get httproute -n llm-d-pd
```

The DigitalOcean configuration automatically enables InferencePool and HTTPRoute creation for proper P/D routing.

#### 5. Gateway Not Programmed

**Error**: Gateway shows `PROGRAMMED: False`

**Solution**: Verify Istio is running and LoadBalancer IP is assigned:
```bash
kubectl get pods -n istio-system
kubectl get gateway -n <namespace>
```


## Cleanup

```bash
# Remove specific deployment
export NAMESPACE=llm-d-pd # or llm-d-inference-scheduling
helmfile destroy -e digitalocean -n ${NAMESPACE}

# Remove prerequisites (affects all deployments)
cd guides/prereq/gateway-provider
helmfile destroy -f istio.helmfile.yaml
./install-gateway-provider-dependencies.sh delete
```

## Configuration Files Reference

### Base Configurations (Unchanged)
- `guides/inference-scheduling/ms-inference-scheduling/values.yaml`
- `guides/pd-disaggregation/ms-pd/values.yaml`

### DigitalOcean Overrides (Platform-Specific)
- `guides/inference-scheduling/ms-inference-scheduling/digitalocean-values.yaml`
- `guides/pd-disaggregation/ms-pd/digitalocean-values.yaml`

### Helmfile Configuration
- Uses `digitalocean` environment to conditionally load DigitalOcean overrides
- Only applies platform-specific configurations when explicitly using `-e digitalocean`
- Follows clean configuration architecture principles with proper environment separation

For detailed configuration options and advanced setups, see the main [llm-d guides](../../../guides/).
