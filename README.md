# vLLM on K3s Setup with Ansible

Automated setup of vLLM inference server on K3s with GPU support and security verification.

## Prerequisites

- Ubuntu 20.04+ or WSL2 with Ubuntu
- NVIDIA GPU with drivers installed
- Ansible installed: `sudo apt install ansible`

## Security Features

This playbook includes checksum verification for all downloaded scripts to prevent supply chain attacks.

### Initial Setup (First Run)

On your first run, checksums are not yet configured:
```bash
# Run with verification disabled to see checksums
ansible-playbook playbook.yml
```

The playbook will display the checksum of the k3s install script. **Verify this checksum manually** by:

1. Visiting: https://raw.githubusercontent.com/k3s-io/k3s/v1.28.5+k3s1/install.sh
2. Or running: `curl -sL https://raw.githubusercontent.com/k3s-io/k3s/v1.28.5+k3s1/install.sh | sha256sum`

### Enable Checksum Verification

Once you've verified the checksum, update `group_vars/all/defaults.yml`:
```yaml
k3s_install_script_checksum: "sha256:YOUR_VERIFIED_CHECKSUM_HERE"
k3s_verify_checksum_strict: true
```

Now the playbook will fail if the checksum doesn't match, protecting against tampered scripts.

## Configuration

Default variables are in `group_vars/all/defaults.yml`. 

To customize, create `custom_vars.yml`:
```yaml
---
# Model configuration
vllm_model: "mistralai/Mistral-7B-Instruct-v0.2"
vllm_gpu_memory_utilization: 0.85
vllm_max_model_len: 2048
```

## Usage
```bash
# First run (displays checksums)
ansible-playbook playbook.yml

# After verifying and configuring checksums
ansible-playbook playbook.yml

# With custom vars
ansible-playbook playbook.yml -e @custom_vars.yml

# Override specific variables
ansible-playbook playbook.yml -e "vllm_model=microsoft/phi-2"
```

## Testing the Deployment

Once complete, test the vLLM API:
```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "TheBloke/Mistral-7B-Instruct-v0.2-AWQ",
    "messages": [
      {"role": "user", "content": "What is Kubernetes?"}
    ]
  }'
```

Check deployment status:
```bash
kubectl get pods
kubectl logs -l app=vllm-mistral -f
```

## Available Variables

See `group_vars/all/defaults.yml` for all configurable options:

### Model Configuration
- `vllm_model`: Model to deploy
- `vllm_quantization`: Quantization method (awq, gptq, or empty)
- `vllm_gpu_memory_utilization`: GPU memory % to use (0.0-1.0)
- `vllm_max_model_len`: Maximum context length
- `vllm_port`: Service port (default: 8000)

### Security Configuration
- `k3s_version`: Pinned k3s version
- `k3s_install_script_checksum`: Verified SHA256 checksum
- `k3s_verify_checksum_strict`: Enable/disable verification

### Storage Configuration
- `huggingface_cache`: Model cache directory

## Updating k3s Version

To update k3s:

1. Change `k3s_version` in `group_vars/all/defaults.yml`
2. Get new checksum: `curl -sL https://raw.githubusercontent.com/k3s-io/k3s/NEW_VERSION/install.sh | sha256sum`
3. Update `k3s_install_script_checksum`
4. Re-run playbook

## Troubleshooting

### Checksum mismatch error

If you get a checksum mismatch:

1. **DO NOT bypass** without investigating
2. Verify the script manually at the GitHub URL shown in the error
3. If legitimate, update the checksum in vars
4. If suspicious, investigate before proceeding

### Pod not starting
```bash
# Check pod status
kubectl describe pod -l app=vllm-mistral

# Check logs
kubectl logs -l app=vllm-mistral
```

### GPU not detected
```bash
# Verify NVIDIA plugin
kubectl get pods -n kube-system | grep nvidia

# Check node GPU resources
kubectl describe node
```

## Directory Structure
```
vllm-k8s-setup/
├── ansible.cfg
├── playbook.yml
├── README.md
├── .gitignore
├── custom_vars.yml (optional)
├── group_vars/
│   └── all/
│       └── defaults.yml
├── inventory/
│   └── hosts
└── roles/
    ├── docker/
    │   ├── tasks/
    │   │   └── main.yml
    │   └── handlers/
    │       └── main.yml
    ├── k3s/
    │   └── tasks/
    │       └── main.yml
    └── vllm/
        ├── tasks/
        │   └── main.yml
        └── templates/
            └── vllm-deployment.yaml.j2
```

## License

MIT
