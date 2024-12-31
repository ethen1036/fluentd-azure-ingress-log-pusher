# Fluentd Azure Ingress Logger Helm Chart

## Overview
This Helm chart deploys a Fluentd DaemonSet that collects Nginx ingress controller logs and forwards them to Azure Blob Storage. The solution runs Fluentd on each node of your Kubernetes cluster to ensure comprehensive log collection.

## Prerequisites
- Kubernetes 1.16+
- Helm 3.0+
- Azure Storage Account with appropriate permissions
- Nginx Ingress Controller installed in the cluster

## Installing the Chart

1. Create a `secrets.yaml` file with your Azure credentials:
```yaml
azure:
  accessKey: "your-azure-storage-access-key"
```

2. Install the chart:
```bash
helm install fluentd-logger fluentd-azure-ingress-logger \
  -f secrets.yaml \
  --namespace monitoring \
  --create-namespace
```

## Configuration

The following table lists the configurable parameters of the chart and their default values:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Fluentd image repository | `fluent/fluentd-kubernetes-daemonset` |
| `image.tag` | Fluentd image tag | `v1.17.1-debian-azureblob-1.0` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `resources.limits.memory` | Memory limit | `500Mi` |
| `resources.requests.cpu` | CPU request | `100m` |
| `resources.requests.memory` | Memory request | `200Mi` |
| `azure.storageAccount` | Azure Storage account name | `logbucket` |
| `azure.container` | Azure Blob container name | `nginx-ingress-logs` |
| `azure.autoCreateContainer` | Auto-create container if not exists | `true` |
| `azure.compress` | Enable log compression | `true` |
| `azure.timeSliceFormat` | Time slice format for log files | `%Y%m%d-%H` |
| `fluentd.posFilePath` | Path to position file | `/var/log/nginx-ingress.log.pos` |
| `fluentd.bufferPath` | Path to buffer directory | `/var/log/fluent/azurestorage` |
| `fluentd.timeKey` | Time key for buffer | `24h` |
| `fluentd.timeKeyWait` | Time to wait before flushing buffer | `300` |
| `tolerations` | Pod tolerations | `[{"key": "node-role.kubernetes.io/master", "effect": "NoSchedule"}]` |
| `namespace` | Namespace to deploy resources | `monitoring` |

## Customization Examples

### Setting Custom Resource Limits
```yaml
resources:
  limits:
    memory: 1Gi
  requests:
    cpu: 200m
    memory: 512Mi
```

### Configuring Azure Storage
```yaml
azure:
  storageAccount: "mycompanylogs"
  container: "prod-nginx-logs"
  timeSliceFormat: "%Y%m%d-%H%M"
  compress: true
```

### Modifying Buffer Settings
```yaml
fluentd:
  timeKey: 12h
  timeKeyWait: 600
```

## Log Format

The Fluentd configuration captures the following fields from Nginx ingress logs:
- `remote_addr`: Client IP address
- `geoip_country_name`: Country name from GeoIP (if enabled)
- `timestamp`: Request timestamp
- `request`: HTTP request line
- `status`: HTTP status code
- `body_bytes_sent`: Response size
- `http_referer`: Referer header
- `http_user_agent`: User agent
- `http_x_forwarded_for`: X-Forwarded-For header
- And more...

## Storage Structure

Logs are stored in Azure Blob Storage with the following structure:
```
container_name/
  ├── YYYYMMDD-HH-0.log.gz
  ├── YYYYMMDD-HH-1.log.gz
  └── ...
```

## Monitoring and Troubleshooting

### Viewing Fluentd Logs
```bash
kubectl logs -n monitoring -l app=fluentd-azure --tail=100
```

### Common Issues

1. **Pod Startup Failures**
   - Check Azure credentials
   - Verify storage account accessibility
   ```bash
   kubectl describe pod -n monitoring -l app=fluentd-azure
   ```

2. **Log Collection Issues**
   - Verify Nginx ingress controller pod naming pattern
   - Check position file existence
   ```bash
   kubectl exec -n monitoring -l app=fluentd-azure -- ls -l /var/log/nginx-ingress.log.pos
   ```

3. **Buffer Issues**
   - Check buffer directory permissions
   - Monitor buffer size
   ```bash
   kubectl exec -n monitoring -l app=fluentd-azure -- du -sh /var/log/fluent/azurestorage
   ```

## Security Considerations

1. The chart creates necessary RBAC resources with minimal permissions
2. Azure credentials are stored as Kubernetes secrets
3. Container logs are mounted read-only
4. ServiceAccount tokens are automatically mounted

## Upgrade Instructions

1. Update values as needed in your custom values file
2. Upgrade the release:
```bash
helm upgrade fluentd-logger fluentd-azure-ingress-logger \
  -f values.yaml \
  -f secrets.yaml \
  --namespace monitoring
```

## Uninstalling the Chart

```bash
helm uninstall fluentd-logger -n monitoring
```

Note: This will not delete the PersistentVolumes used for buffer storage. To clean up completely:
```bash
kubectl delete pv -l app=fluentd-azure
```

## License

This chart is licensed under the MIT License - see the LICENSE file for details.