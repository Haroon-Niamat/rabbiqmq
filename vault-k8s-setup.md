# Setting Up HashiCorp Vault on Kubernetes (Minikube)

## Prerequisites
- Minikube running
- kubectl configured
- Helm 3.x installed

## Step 1: Add HashiCorp Helm Repository
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

## Step 2: Create Vault Namespace
```bash
kubectl create namespace vault
```

## Step 3: Create values.yaml for Vault Configuration
```yaml
server:
  dev:
    enabled: true  # Enable dev mode for testing
  standalone:
    enabled: true  # Use standalone mode
    config: |
      ui = true
      listener "tcp" {
        tls_disable = 1
        address = "[::]:8200"
        cluster_address = "[::]:8201"
      }
      storage "file" {
        path = "/vault/data"
      }

  service:
    enabled: true

ui:
  enabled: true
  serviceType: NodePort  # Makes UI accessible via NodePort
```

## Step 4: Install Vault using Helm
```bash
helm install vault hashicorp/vault \
  --namespace vault \
  --values values.yaml
```

## Step 5: Verify Installation
```bash
# Check if pods are running
kubectl get pods -n vault

# Check if services are created
kubectl get svc -n vault
```

## Step 6: Initialize and Unseal Vault
```bash
# Get the pod name
export VAULT_POD=$(kubectl get pods -n vault -l app.kubernetes.io/name=vault -o jsonpath='{.items[0].metadata.name}')

# Initialize Vault
kubectl exec -n vault $VAULT_POD -- vault operator init

# The above command will output 5 unseal keys and an initial root token
# Save these securely!
```

## Step 7: Access Vault UI
```bash
# Create a port forward to access the UI
kubectl port-forward -n vault svc/vault-ui 8200:8200
```
The UI will be available at: http://localhost:8200

## Step 8: Configure kubectl for Vault Access
```bash
# Set environment variables for Vault CLI
export VAULT_ADDR='http://localhost:8200'
export VAULT_TOKEN='<your-root-token>'  # Use the token from initialization
```

## Common Operations

### Check Vault Status
```bash
kubectl exec -n vault $VAULT_POD -- vault status
```

### Create a Secret
```bash
kubectl exec -n vault $VAULT_POD -- vault kv put secret/myapp/config username="myuser" password="mypassword"
```

### Read a Secret
```bash
kubectl exec -n vault $VAULT_POD -- vault kv get secret/myapp/config
```

## Important Notes:
1. In production:
   - Disable dev mode
   - Enable high availability
   - Configure proper storage backend (like Consul)
   - Enable TLS
   - Use proper authentication methods
   
2. Security Considerations:
   - Never store root tokens in plain text
   - Use proper secret management for unseal keys
   - Configure proper RBAC
   - Enable audit logging
