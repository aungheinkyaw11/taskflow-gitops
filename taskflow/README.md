# TaskFlow Helm chart

This chart deploys the TaskFlow backend and frontend and connects traffic through Gateway API and an ALB Ingress. Argo CD renders it for normal deployments.

## Resources rendered

- Backend `Deployment` and `ClusterIP` `Service`
- Frontend `Deployment` and `ClusterIP` `Service`
- Backend `ConfigMap` and `Secret`
- Schema `ConfigMap` and Argo CD migration `Job`
- Optional `GatewayClass`
- Optional cross-namespace `Gateway`
- Optional `HTTPRoute` routing `/api` to the backend and `/` to the frontend
- Optional ALB `Ingress` in the gateway namespace

The migration Job applies the included schema to the configured PostgreSQL database before the backend sync wave. The chart does not create RDS or ECR, install controllers, or create DNS records.

## Values layering

Always render with the shared file first and the environment file second:

```bash
helm lint . -f values.yaml -f values-dev.yaml
helm template taskflow-dev . \
  --namespace taskflow \
  -f values.yaml \
  -f values-dev.yaml
```

`values-dev.yaml` currently overrides replicas, image tags, and `NODE_ENV`. `values-prod.yaml` provides production overrides. Helm merges these onto `values.yaml`.

## Values reference

| Values path | Purpose |
| --- | --- |
| `backend.replicaCount` | Number of API pods |
| `backend.image.repository` / `tag` | Backend ECR image |
| `backend.service` | Internal API service name and ports |
| `backend.resources` | API CPU and memory requests/limits |
| `frontend.replicaCount` | Number of UI pods |
| `frontend.image.repository` / `tag` | Frontend ECR image |
| `frontend.service` | Internal UI service name and ports |
| `frontend.resources` | UI CPU and memory requests/limits |
| `configMap.data` | Non-secret backend environment variables, including RDS and CORS settings |
| `secret.data` | Legacy plaintext secret inputs; replace with an external secret workflow |
| `migration` | PostgreSQL client image, schema ConfigMap, and migration Job settings |
| `gatewayClass` | Optional controller-owned cluster-level class |
| `gateway` | NGINX Gateway listener in `nginx-gateway` |
| `httpRoute` | `/api` and `/` routing to TaskFlow services |
| `ingress` | Internet-facing ALB pointing to the NGINX Gateway Service |

Repository URLs, tags, hostnames, service names, and namespaces must match resources that actually exist in the target account and EKS cluster. The current chart uses Docker Hub images; update both repository values if ECR is the intended registry.

## Traffic path

```text
Internet
  -> AWS ALB created from Ingress (nginx-gateway namespace)
  -> NGINX Gateway Fabric Service
  -> Gateway (nginx-gateway namespace)
  -> HTTPRoute (taskflow namespace)
       /api -> taskflow-backend-service:5001
       /    -> taskflow-frontend-service:80
```

The Gateway allows routes from all namespaces. Tighten `gateway.allowedRoutes` if multiple teams share the cluster.

## Direct Helm test installation

Argo CD is the owner in shared environments. Use a direct Helm install only in a disposable cluster or namespace where Argo CD is not managing the same resources:

```bash
helm upgrade --install taskflow-dev . \
  --namespace taskflow \
  --create-namespace \
  -f values.yaml \
  -f values-dev.yaml
```

Because the chart also creates objects in `nginx-gateway`, that namespace and the required controllers must already exist.

## Before committing a change

```bash
helm lint . -f values.yaml -f values-dev.yaml
helm template taskflow-dev . \
  --namespace taskflow \
  -f values.yaml \
  -f values-dev.yaml > /tmp/taskflow-rendered.yaml
kubectl apply --dry-run=client -f /tmp/taskflow-rendered.yaml
```

Check the rendered image names, namespaces, routes, resource limits, and environment values. Do not commit rendered YAML or plaintext credentials.

## Known gaps

- Plaintext secrets are still present in `values.yaml` and must be rotated and externalized.
- Deployments do not yet define readiness or liveness probes.
- No `PodDisruptionBudget`, autoscaling, network policy, or service account is defined.
- Schema changes are embedded in the chart rather than generated from `backend/schema.sql`; keep both copies synchronized or adopt a dedicated migration tool.
- The chart assumes both NGINX Gateway Fabric and AWS Load Balancer Controller already exist.
