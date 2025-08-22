# SemaphoreUI Manifest

A Kubernetes application manifest for deploying [SemaphoreUI](https://semaphoreui.com) with PostgreSQL database using CloudNativePG (CNPG) and auto-registering runner pods. This project is designed to be deployed using ArgoCD for GitOps-based continuous deployment.

## Overview

This manifest deploys a complete SemaphoreUI environment including:

- **SemaphoreUI Server**: The main application server with web interface
- **PostgreSQL Database**: Managed by CloudNativePG for high availability and reliability
- **Auto-registering Runners**: Kubernetes pods that automatically register with the SemaphoreUI server to execute tasks
- **OIDC Authentication**: Integration with Authentik for single sign-on

## Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   ArgoCD        │    │   SemaphoreUI    │    │   PostgreSQL    │
│   Application   │───▶│   Server         │───▶│   (CNPG)        │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                              │
                              ▼
                       ┌──────────────────┐
                       │   Runner Pods    │
                       │   (Auto-register)│
                       └──────────────────┘
```

## Prerequisites

- Kubernetes cluster (1.20+)
- ArgoCD installed and configured
- CloudNative PostgreSQL (for PostgreSQL server management)
- cert-manager (for TLS certificates)
- Storage class for persistent volumes
- Authentik instance (for OIDC authentication)
- Vault or OpenBAO for secrets management

## Components

### 1. SemaphoreUI Server

- Deployed using the official SemaphoreUI Helm chart
- Configured with OIDC authentication via Authentik
- Exposed via Ingress with TLS termination
- Uses persistent storage for data persistence

### 2. PostgreSQL Database

- Managed by CloudNativePG operator
- High availability with pod anti-affinity
- Write-Ahead Log (WAL) storage enabled
- Monitoring with Prometheus integration
- Backup configuration (optional)

### 3. Runner Pods

- Auto-registering SemaphoreUI runners
- 3 replicas by default for high availability
- Ephemeral storage for task execution
- Automatic token management and registration

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/mattboston/semaphoreui-manifest.git
cd semaphoreui-manifest
```

### 2. Configure Secrets

Create the required Kubernetes secrets before deployment:

```bash
# OIDC credentials for Authentik
kubectl create secret generic semaphoreui-oidc-creds \
  --from-literal=provider_url="https://your-authentik-instance.com" \
  --from-literal=redirect_url="https://semaphoreui.mydomain.com/api/auth/authentik/callback" \
  --from-literal=client_id="your-client-id" \
  --from-literal=client_secret="your-client-secret" \
  -n semaphoreui

# Runner registration token
kubectl create secret generic semaphoreui-runner \
  --from-literal=token="your-runner-registration-token" \
  -n semaphoreui
```

### 3. Deploy with ArgoCD

Apply the ArgoCD application:

```bash
kubectl apply -f argocd-application.yaml
```

Or create the application through the ArgoCD UI using the following configuration:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: semaphoreui
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://github.com/mattboston/semaphoreui-manifest.git
    path: charts
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: semaphoreui
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

## Configuration

### Customizing the Deployment

Edit `charts/values.yaml` to customize the deployment:

#### SemaphoreUI Server Configuration

```yaml
semaphore:
  replicaCount: 1
  image:
    repository: public.ecr.aws/semaphore/pro/server
    tag: "latest"
  ingress:
    hosts:
      - host: semaphoreui.mydomain.com  # Change to your domain
  persistence:
    size: 100Gi
```

#### PostgreSQL Configuration

```yaml
cluster:
  cluster:
    resources:
      limits:
        cpu: 2
        memory: 2Gi
      requests:
        cpu: 250m
        memory: 256Mi
```

#### Runner Configuration

Edit `charts/templates/runner.yaml` to modify:

- Number of replicas (default: 3)
- Resource limits and requests
- Storage sizes

### Environment Variables

The following environment variables are automatically configured:

- `SEMAPHORE_WEB_ROOT`: Points to the SemaphoreUI server
- `SEMAPHORE_RUNNER_REGISTRATION_TOKEN`: Runner registration token
- `SEMAPHORE_RUNNER_PRIVATE_KEY_FILE`: Path to runner private key
- `ANSIBLE_HOST_KEY_CHECKING`: Disabled for automated deployments

## Monitoring

### PostgreSQL Monitoring

The PostgreSQL cluster includes Prometheus monitoring:

- PodMonitor enabled for metrics collection
- Resource usage monitoring
- Database performance metrics

### Application Monitoring

- Kubernetes native monitoring via ArgoCD
- Health checks and readiness probes
- Resource usage tracking

## Backup and Recovery

### PostgreSQL Backups

Backup configuration is available in the values file:

```yaml
cluster:
  backups:
    enabled: true
    s3:
      bucket: platform-postgresql
```

### Application Data

- SemaphoreUI data is persisted in PVCs
- Runner configurations are stored in ephemeral volumes
- Database backups can be configured for disaster recovery

## Known Issues

### Runner Auto-Registration

**Issue**: Runners register but are not enabled and hostname not set in SemaphoreUI

- **Cause**: `semaphore runner register` lacks the ability to set the runner to be enabled or set the hostname
- **Workaround**: None
- **Status**: Feature request submitted to SemaphoreUI

**Issue**: Semaphore Server and Runner pods re-provision whenever chart is updated

- **Cause**: Not known yet
- **Workaround**: None
- **Status**: None

## Troubleshooting

### Common Issues

1. **Runner Registration Fails**
   - Check if the SemaphoreUI server is ready
   - Verify the runner token secret exists
   - Check runner pod logs

2. **Database Connection Issues**
   - Verify PostgreSQL cluster is healthy
   - Check database credentials in secrets
   - Ensure network policies allow connections

3. **OIDC Authentication Issues**
   - Verify Authentik configuration
   - Check OIDC secret values
   - Ensure redirect URLs are correct

### Logs and Debugging

```bash
# Check SemaphoreUI server logs
kubectl logs -f deployment/semaphoreui -n semaphoreui

# Check runner logs
kubectl logs -f deployment/semaphoreui-runner -n semaphoreui

# Check PostgreSQL cluster status
kubectl get cluster -n semaphoreui
kubectl describe cluster semaphoreui-postgresql -n semaphoreui
```

## Security Considerations

- All sensitive data is stored in Kubernetes secrets
- OIDC authentication provides secure access control
- Network policies should be configured for production use
- TLS termination is handled by cert-manager
- Runner pods run with minimal privileges

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test the deployment
5. Submit a pull request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Support

For issues and questions:

- Create an issue in this repository
- Review Kubernetes and ArgoCD documentation
- Check the [SemaphoreUI documentation](https://docs.semaphoreui.com)
- Check the [SemaphoreUI GitHub Issues/Discussion](https://github.com/semaphoreui/semaphore)
- Check the [SemaphoreUI Discord channel](https://discord.gg/5R6k7hNGcH)
