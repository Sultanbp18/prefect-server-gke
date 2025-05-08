# Prefect Server Deployment Guide with External PostgreSQL

This guide explains how to deploy Prefect Server on Google Kubernetes Engine (GKE) using an external PostgreSQL database with credentials stored in Kubernetes secrets.

## Prerequisites

- Google Kubernetes Engine (GKE) cluster
- Helm 3.x installed
- `kubectl` configured to access your GKE cluster
- External PostgreSQL database (Cloud SQL or any other PostgreSQL instance)
- Basic understanding of Kubernetes and Helm

## Step 1: Create a Kubernetes Namespace

```bash
kubectl create namespace prefect
```

## Step 2: Create a Secret for PostgreSQL Credentials

Create a Kubernetes secret to store your PostgreSQL credentials:

```bash
kubectl apply -f kubernetes/prefect-db-secret.yaml
```

The secret should contain the following keys:
- `username`: PostgreSQL username
- `password`: PostgreSQL password
- `host`: PostgreSQL host address
- `port`: PostgreSQL port (usually 5432)
- `database`: PostgreSQL database name

You can modify the `kubernetes/prefect-db-secret.yaml` file with your actual database credentials:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: prefect-db-credentials
  namespace: prefect
type: Opaque
stringData:
  username: "your_actual_username"
  password: "your_actual_password"
  host: "your-postgres-host.example.com"
  port: "5432"
  database: "prefect_db"
```

## Step 3: Add the Prefect Helm Repository

```bash
helm repo add prefecthq https://prefecthq.github.io/prefect-helm
helm repo update
```

## Step 4: Deploy Prefect Server

Deploy the Prefect server using the external database configuration:

```bash
helm install prefect-server prefecthq/prefect-server \
  -f helm-values/prefect-server-external-db.yaml \
  -n prefect
```

The `prefect-server-external-db.yaml` file is configured to:
- Disable the built-in PostgreSQL subchart
- Use the external PostgreSQL database via the `prefect-db-credentials` secret
- Configure ingress for external access (modify the hostname as needed)

## Step 5: Deploy Prefect Worker

Deploy the Prefect worker that will connect to the server:

```bash
helm install prefect-worker prefecthq/prefect-worker \
  -f helm-values/prefect-worker-external-db.yaml \
  -n prefect
```

The `prefect-worker-external-db.yaml` file is configured to:
- Connect to the Prefect server using the internal Kubernetes DNS
- Use a Kubernetes work pool
- Set appropriate RBAC permissions for creating and managing flow run jobs

## Step 6: Create a Work Pool

After the Prefect server is deployed, you need to create a work pool:

1. Access the Prefect UI through your ingress URL or by port-forwarding:
   ```bash
   kubectl port-forward svc/prefect-server 4200:4200 -n prefect
   ```

2. Open your browser at: http://localhost:4200

3. Navigate to "Work Pools" in the left sidebar

4. Click "Create Work Pool"

5. Select "Kubernetes" as the type

6. Enter "kubernetes-pool" as the name (matching the configuration in the worker values file)

7. Configure the base job template as needed or use the default

## Step 7: Verify the Deployment

Check that all components are running:

```bash
kubectl get pods -n prefect
```

You should see the Prefect server and worker pods running.

## Configuration Details

### External Database Configuration

The key configuration for using an external PostgreSQL database is in the `prefect-server-external-db.yaml` file:

```yaml
# Disable the built-in PostgreSQL subchart
postgresql:
  enabled: false

# Configure the external database connection
secret:
  create: false
  name: "prefect-db-credentials"
```

This tells the Prefect server to use the existing `prefect-db-credentials` secret for database connection information.

### Worker Configuration

The worker is configured to connect to the Prefect server in the `prefect-worker-external-db.yaml` file:

```yaml
worker:
  apiConfig: selfHostedServer
  
  selfHostedServerApiConfig:
    apiUrl: "http://prefect-server.prefect.svc.cluster.local:4200/api"
```

This uses the internal Kubernetes DNS to connect to the Prefect server. If you're accessing the server through an ingress, you can use the external URL instead.

## Troubleshooting

### Database Connection Issues

If the Prefect server can't connect to the database:

1. Verify the secret contains the correct credentials:
   ```bash
   kubectl get secret prefect-db-credentials -n prefect -o jsonpath='{.data}' | \
     jq 'map_values(@base64d)'
   ```

2. Check that the PostgreSQL instance is accessible from the GKE cluster:
   ```bash
   kubectl run postgres-client --rm --tty -i --restart='Never' \
     --image postgres:14 --namespace prefect \
     --env="PGPASSWORD=your_password" \
     -- psql -h your-postgres-host.example.com -U your_username -d prefect_db
   ```

### Worker Connection Issues

If the worker can't connect to the server:

1. Check the worker logs:
   ```bash
   kubectl logs -l app=prefect-worker -n prefect
   ```

2. Verify the server is accessible:
   ```bash
   kubectl exec -it $(kubectl get pod -l app=prefect-server -n prefect -o jsonpath='{.items[0].metadata.name}') -n prefect -- curl -s http://localhost:4200/api/health
   ```

## Upgrading

To upgrade your deployment with new configuration:

```bash
helm upgrade prefect-server prefecthq/prefect-server \
  -f helm-values/prefect-server-external-db.yaml \
  -n prefect

helm upgrade prefect-worker prefecthq/prefect-worker \
  -f helm-values/prefect-worker-external-db.yaml \
  -n prefect
```

## Uninstalling

To uninstall the deployment:

```bash
helm uninstall prefect-worker -n prefect
helm uninstall prefect-server -n prefect
```

Note that this will not delete the Kubernetes secret or any persistent volumes. To clean up completely:

```bash
kubectl delete secret prefect-db-credentials -n prefect
kubectl delete namespace prefect
```
