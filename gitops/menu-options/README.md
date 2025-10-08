# SNO-AI Menu Options

This directory contains the GitOps manifests for each SNO-AI menu configuration. Each option provides a different RHOAI deployment optimized for specific use cases.

## Available Menu Options

### Option 1: Full RHOAI (v2.25)

**Use Case**: Complete AI/ML development and production environment

**Components**:
- Full Data Science Pipelines
- Jupyter Workbenches
- Model Serving (ModelMesh + KServe)
- Distributed Training (Ray, CodeFlare, Training Operator)
- Model Registry
- TrustyAI for monitoring
- Kueue for job queueing

**Resource Requirements**:
- CPU: 16+ cores
- Memory: 64GB+
- Storage: 500GB+

**Best For**:
- Complete AI/ML development lifecycle
- Training and serving workflows
- Multi-user environments
- Production-grade deployments

---

### Option 2: Model Serving Focus (v2.22) with Granite

**Use Case**: Optimized for serving pre-trained models with minimal overhead

**Components**:
- ModelMesh Serving
- OpenVINO Model Server (OVMS) runtime
- Pre-configured IBM Granite 7B Instruct model
- Minimal workbenches for testing
- TrustyAI for model monitoring
- Model Registry

**Disabled Components**:
- Data Science Pipelines
- KServe
- Distributed training components (Ray, CodeFlare)

**Resource Requirements**:
- CPU: 8+ cores
- Memory: 32GB+
- Storage: 200GB+

**Best For**:
- Code generation and assistance
- Pre-trained model deployment
- Resource-constrained environments
- Model serving-only workloads

**Pre-configured Models**:
- IBM Granite 7B Instruct (code generation)

---

### Option 3: LLM Serving Focus (v3.0) with LLamaScout 20B

**Use Case**: High-performance large language model serving with vLLM

**Components**:
- KServe for advanced serving
- vLLM runtime for optimized inference
- Pre-configured LLamaScout 20B model
- GPU support (required)
- TrustyAI for LLM monitoring
- Model Registry

**Disabled Components**:
- ModelMesh
- Data Science Pipelines
- Distributed training components

**Resource Requirements**:
- CPU: 16+ cores
- Memory: 64GB+
- GPU: 1x NVIDIA GPU (16GB+ VRAM)
- Storage: 500GB+

**Instance Types**:
- AWS: p3.2xlarge, g5.xlarge, g5.2xlarge
- Bare Metal: NVIDIA A10, A100, or similar

**Best For**:
- Large language model inference
- Chat applications
- Text generation workloads
- GPU-accelerated serving

**Pre-configured Models**:
- LLamaScout 20B (general-purpose LLM)

---

### Option 4: Custom Configuration

**Use Case**: User-defined RHOAI configuration

**Components**: User-specified

**Best For**:
- Custom requirements
- Experimental configurations
- Advanced users

---

## Switching Between Menu Options

### At Deployment Time

Pass the menu option via user-data:

```yaml
#cloud-config
write_files:
  - path: /etc/snoai-config.env
    content: |
      SNOAI_MENU_ITEM=2
```

### Post-Deployment Toggle

Update the ConfigMap to switch configurations:

```bash
# Switch to Option 3 (LLM Serving)
oc patch configmap snoai-menu-config \
  -n openshift-gitops \
  -p '{"data":{"MENU_ITEM":"3"}}'

# Wait for ArgoCD to sync (automatic with selfHeal enabled)
oc get applications -n openshift-gitops -w
```

**Note**: Switching menu options will:
1. Remove components from the previous option
2. Deploy components for the new option
3. Preserve data in PVCs (unless explicitly pruned)
4. May cause temporary service interruption

## Directory Structure

```
menu-options/
├── README.md                   # This file
├── option-1/                   # Full RHOAI
│   ├── kustomization.yaml
│   ├── datasciencecluster.yaml
│   └── dashboard-config.yaml
├── option-2/                   # Model Serving with Granite
│   ├── kustomization.yaml
│   ├── datasciencecluster.yaml
│   ├── serving-runtime-granite.yaml
│   └── inference-service-granite.yaml
├── option-3/                   # LLM Serving with LLamaScout
│   ├── kustomization.yaml
│   ├── datasciencecluster.yaml
│   ├── serving-runtime-vllm.yaml
│   └── inference-service-llamascout.yaml
└── option-4/                   # Custom (template)
    ├── kustomization.yaml
    └── README.md
```

## Testing Menu Options

### Validate Manifests

```bash
# Test Option 1
oc kustomize gitops/menu-options/option-1

# Test Option 2
oc kustomize gitops/menu-options/option-2

# Test Option 3
oc kustomize gitops/menu-options/option-3
```

### Deploy Locally for Testing

```bash
# Deploy Option 2 directly (without GitOps)
oc apply -k gitops/menu-options/option-2

# Verify deployment
oc get datasciencecluster -n redhat-ods-operator
oc get servingruntime -n snoai-system
oc get inferenceservice -n snoai-system
```

## Customization

Each menu option can be customized by:

1. **Forking this repository**
2. **Modifying the manifests** in the respective option directory
3. **Updating the GitOps repository URL** in the seed cluster or deployment config

### Example Customization: Change Granite Model Version

```yaml
# In option-2/serving-runtime-granite.yaml
spec:
  containers:
    - name: ovms
      image: quay.io/opendatahub/openvino_model_server:2024.1  # Updated version
```

### Example Customization: Adjust Resource Limits

```yaml
# In option-3/serving-runtime-vllm.yaml
resources:
  requests:
    cpu: "16"          # Increased from 8
    memory: 64Gi       # Increased from 32Gi
    nvidia.com/gpu: "2"  # Multiple GPUs
```

## Troubleshooting

### Menu Option Not Syncing

```bash
# Check ArgoCD application status
oc get application -n openshift-gitops

# Check ArgoCD logs
oc logs -n openshift-gitops -l app.kubernetes.io/name=openshift-gitops-application-controller

# Force sync
oc patch application snoai-full-rhoai \
  -n openshift-gitops \
  --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{}}}'
```

### Components Not Deploying

```bash
# Check DataScienceCluster status
oc get datasciencecluster -n redhat-ods-operator -o yaml

# Check RHOAI operator logs
oc logs -n redhat-ods-operator -l name=rhods-operator

# Check for resource constraints
oc top nodes
oc describe node <node-name>
```

### Model Serving Not Working

```bash
# Check serving runtime status
oc get servingruntime -n snoai-system

# Check inference service
oc get inferenceservice -n snoai-system

# Check pod logs
oc logs -n snoai-system -l serving.kserve.io/inferenceservice=<model-name>
```

## Contributing

To add a new menu option:

1. Create a new directory: `option-N/`
2. Add required manifests (kustomization.yaml, datasciencecluster.yaml, etc.)
3. Update `gitops/bootstrap/menu-router-appset.yaml` to include the new option
4. Update this README with the new option details
5. Test thoroughly before committing

## References

- [RHOAI Documentation](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai/)
- [KServe Documentation](https://kserve.github.io/website/)
- [vLLM Documentation](https://docs.vllm.ai/)
- [OpenVINO Model Server](https://docs.openvino.ai/latest/ovms_what_is_openvino_model_server.html)
