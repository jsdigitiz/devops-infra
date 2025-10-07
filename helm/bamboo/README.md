# Bamboo Data Center on Kubernetes

Production-ready Bamboo deployment on K3s using official Atlassian Helm charts.

## Prerequisites

- K3s cluster up and running
- PostgreSQL database (deployed via [ansible/postgresql-server](../../ansible/postgresql-server/README.md))
- kubectl configured to access your cluster
- Helm 3.x installed
- Valid Bamboo Data Center license

## Architecture

```
┌─────────────────────────────────────────┐
│           K3s Cluster (Node)            │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │     Traefik Ingress              │  │
│  │  (bamboo.example.com)            │  │
│  └──────────────┬───────────────────┘  │
│                 │                       │
│  ┌──────────────▼───────────────────┐  │
│  │     Bamboo Service (ClusterIP)   │  │
│  └──────────────┬───────────────────┘  │
│                 │                       │
│  ┌──────────────▼───────────────────┐  │
│  │     Bamboo StatefulSet (1 pod)   │  │
│  │  - CPU: 1-2 cores                │  │
│  │  - Memory: 3-4Gi                 │  │
│  │  - JVM Heap: 1-2g                │  │
│  └─────┬────────────────────┬───────┘  │
│        │                    │           │
│  ┌─────▼────────┐    ┌─────▼────────┐  │
│  │ Local Home   │    │ Shared Home  │  │
│  │ PVC (10Gi)   │    │ PVC (20Gi)   │  │
│  └──────────────┘    └──────────────┘  │
└─────────────────────────────────────────┘
                 │
                 │ JDBC Connection
                 │
┌────────────────▼─────────────────────────┐
│     PostgreSQL Server (192.168.3.100)    │
│  - Database: bamboo                      │
│  - Port: 5432                            │
│  - Deployed via Ansible                  │
└──────────────────────────────────────────┘
```

## Installation

### Step 1: Prepare PostgreSQL Database

Ensure PostgreSQL is set up and accessible:

```bash
# From ansible/postgresql-server directory
ansible-playbook -i inventory/production.ini site.yml --ask-vault-pass
```

Verify database connection from K3s node:
```bash
psql -h 192.168.3.100 -U bamboo -d bamboo -c "SELECT version();"
```

### Step 2: Add Atlassian Helm Repository

```bash
# Add the official Atlassian Helm repo
helm repo add atlassian-data-center https://atlassian.github.io/data-center-helm-charts

# Update repo
helm repo update
```

### Step 3: Create Namespace

```bash
kubectl create namespace bamboo
```

### Step 4: Configure Secrets

```bash
# Copy example secrets file
cp secrets.yaml.example secrets.yaml

# Edit with your credentials
nano secrets.yaml

# Apply secrets
kubectl apply -f secrets.yaml -n bamboo
```

**Important:** Update these values in `secrets.yaml`:
- `bamboo-db-credentials`: PostgreSQL username/password (from Ansible vault)
- `bamboo-admin-credentials`: Initial Bamboo admin user
- `bamboo-security-token`: Generate with `openssl rand -base64 32`
- `bamboo-license`: Your Bamboo Data Center license key

### Step 5: Customize Values

Edit [values-production.yaml](values-production.yaml) to match your environment:

```yaml
# Key settings to update:
database:
  url: jdbc:postgresql://192.168.3.100:5432/bamboo  # Your DB host

ingress:
  host: bamboo.example.com  # Your domain

volumes:
  localHome:
    persistentVolumeClaim:
      storageClassName: local-path  # Your storage class
```

Check available storage classes:
```bash
kubectl get storageclass
```

### Step 6: Install Bamboo

```bash
# Install Bamboo using Helm (Chart v2.0.4 with Bamboo 11.0.4)
helm install bamboo \
  atlassian-data-center/bamboo \
  --namespace bamboo \
  --values values-production.yaml \
  --version 2.0.4

# Watch the deployment
kubectl get pods -n bamboo -w
```

### Step 7: Verify Installation

```bash
# Check pod status
kubectl get pods -n bamboo

# Check services
kubectl get svc -n bamboo

# Check ingress
kubectl get ingress -n bamboo

# View logs
kubectl logs -n bamboo -l app=bamboo --tail=100 -f
```

## Accessing Bamboo

### Via Ingress (Production)

```bash
# Add to /etc/hosts or configure DNS
192.168.3.100  bamboo.example.com

# Access via browser
https://bamboo.example.com
```

### Via Port Forward (Testing)

```bash
kubectl port-forward -n bamboo svc/bamboo 8085:80

# Access at: http://localhost:8085
```

## Configuration

### Update Configuration

```bash
# Edit values
nano values-production.yaml

# Upgrade release
helm upgrade bamboo \
  atlassian-data-center/bamboo \
  --namespace bamboo \
  --values values-production.yaml
```

### Scale Resources

For larger workloads, update in [values-production.yaml](values-production.yaml):

```yaml
resources:
  jvm:
    maxHeap: "4g"
    minHeap: "2g"
  container:
    requests:
      cpu: "2"
      memory: "6Gi"
    limits:
      cpu: "4"
      memory: "8Gi"

volumes:
  localHome:
    persistentVolumeClaim:
      resources:
        requests:
          storage: 50Gi
```

### Enable HTTPS with cert-manager

1. Install cert-manager:
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

2. Create ClusterIssuer:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
```

3. Apply: `kubectl apply -f clusterissuer.yaml`

## Bamboo Agents

Deploy agents to run build jobs:

```bash
# Install Bamboo Agent
helm install bamboo-agent \
  atlassian-data-center/bamboo-agent \
  --namespace bamboo \
  --set agent.server.url=http://bamboo.bamboo.svc.cluster.local \
  --set agent.securityToken.secretName=bamboo-security-token \
  --set replicaCount=2
```

For more details, see [Bamboo Agent Helm Chart](https://artifacthub.io/packages/helm/atlassian-data-center/bamboo-agent).

## Backup and Restore

### Backup Bamboo Home

```bash
# Create backup job
kubectl create job bamboo-backup-$(date +%Y%m%d) \
  --from=cronjob/backup-cronjob \
  -n bamboo

# Or manually backup PVCs
kubectl exec -n bamboo bamboo-0 -- tar czf /tmp/bamboo-home-backup.tar.gz \
  /var/atlassian/application-data/bamboo
```

### Database Backup

Automated backups are configured via Ansible. See [ansible/postgresql-server/README.md](../../ansible/postgresql-server/README.md).

Manual backup:
```bash
ssh ops@192.168.3.100
sudo -u postgres pg_dump bamboo | gzip > bamboo_backup_$(date +%Y%m%d).sql.gz
```

## Monitoring

### Check Pod Status

```bash
# Pod details
kubectl describe pod -n bamboo bamboo-0

# Resource usage
kubectl top pod -n bamboo

# Logs
kubectl logs -n bamboo bamboo-0 -f
```

### JMX Metrics

JMX metrics are exposed on port 9999. Configure Prometheus to scrape:

```yaml
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9999"
  prometheus.io/path: "/metrics"
```

### Health Checks

```bash
# Readiness probe
kubectl exec -n bamboo bamboo-0 -- curl -f http://localhost:8085/status

# Liveness probe
kubectl exec -n bamboo bamboo-0 -- curl -f http://localhost:8085/
```

## Troubleshooting

### Pod Not Starting

```bash
# Check events
kubectl describe pod -n bamboo bamboo-0

# Check logs
kubectl logs -n bamboo bamboo-0

# Common issues:
# - Database connection failed
# - Insufficient resources
# - PVC mount issues
```

### Database Connection Issues

```bash
# Test from pod
kubectl exec -n bamboo bamboo-0 -- nc -zv 192.168.3.100 5432

# Check secrets
kubectl get secret -n bamboo bamboo-db-credentials -o yaml

# Verify PostgreSQL allows connection
# Update pg_hba.conf to allow K3s pod network
```

### Storage Issues

```bash
# Check PVCs
kubectl get pvc -n bamboo

# Check PV
kubectl get pv

# Verify storage class
kubectl describe storageclass local-path
```

### Performance Issues

1. Check resource usage:
```bash
kubectl top pod -n bamboo
```

2. Increase JVM heap:
```yaml
resources:
  jvm:
    maxHeap: "4g"
```

3. Check database performance:
```bash
sudo -u postgres psql bamboo -c "SELECT * FROM pg_stat_activity;"
```

## Uninstallation

```bash
# Delete Helm release
helm uninstall bamboo -n bamboo

# Delete PVCs (WARNING: This deletes all data!)
kubectl delete pvc -n bamboo --all

# Delete namespace
kubectl delete namespace bamboo
```

## Security Considerations

- ✅ Database credentials stored in Kubernetes secrets
- ✅ Run as non-root user (UID 2005)
- ✅ Security context with dropped capabilities
- ✅ HTTPS enabled with TLS certificates
- ✅ Network policies recommended (not included)
- ✅ Regular automated backups
- ✅ PostgreSQL uses scram-sha-256 authentication

## Production Checklist

Before going to production:

- [ ] Valid Bamboo Data Center license configured
- [ ] PostgreSQL database properly sized and backed up
- [ ] Persistent volumes configured with appropriate storage class
- [ ] Ingress configured with valid domain and TLS certificate
- [ ] Resource limits set appropriately for workload
- [ ] Secrets properly secured and rotated regularly
- [ ] Backup strategy implemented and tested
- [ ] Monitoring and alerting configured
- [ ] Disaster recovery plan documented
- [ ] Network policies configured (if required)

## Additional Resources

- [Atlassian Bamboo Helm Charts](https://atlassian.github.io/data-center-helm-charts/)
- [Bamboo Documentation](https://confluence.atlassian.com/bamboo/)
- [GitHub Repository](https://github.com/atlassian/data-center-helm-charts)
- [Artifact Hub](https://artifacthub.io/packages/helm/atlassian-data-center/bamboo)

## Support

For issues related to:
- **Helm chart**: [atlassian/data-center-helm-charts](https://github.com/atlassian/data-center-helm-charts/issues)
- **Bamboo product**: [Atlassian Support](https://support.atlassian.com/)
- **This deployment**: Open an issue in this repository
