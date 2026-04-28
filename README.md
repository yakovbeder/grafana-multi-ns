# Per-Namespace Grafana on OpenShift with Thanos 9092 Isolation

Deploy isolated Grafana instances on OpenShift -- one per namespace. Each instance can only query metrics from its own namespace, enforced by the Thanos tenancy endpoint (port 9092) and OpenShift RBAC. Users who are namespace admins get full Grafana admin on their instance.

&nbsp;

## Table of Contents

- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Procedure (Manual Step-by-Step)](#procedure-manual-step-by-step)
  - [Step 1: Create the namespace](#step-1-create-the-namespace)
  - [Step 2: Create RBAC resources](#step-2-create-rbac-resources)
  - [Step 3: Create secrets and ConfigMaps](#step-3-create-secrets-and-configmaps)
  - [Step 4: Create the nginx namespace-proxy ConfigMap](#step-4-create-the-nginx-namespace-proxy-configmap)
  - [Step 5: Deploy the Grafana instance](#step-5-deploy-the-grafana-instance)
  - [Step 6: Create the SA token secret](#step-6-create-the-sa-token-secret)
  - [Step 7: Deploy the datasource](#step-7-deploy-the-datasource)
  - [Step 8: Grant namespace admin to a user](#step-8-grant-namespace-admin-to-a-user)
  - [Step 9: Access Grafana](#step-9-access-grafana)
- [Kustomize Deployment (Alternative)](#kustomize-deployment-alternative)
- [Namespace Isolation -- How It Works](#namespace-isolation----how-it-works)
- [RBAC Summary](#rbac-summary)
- [Cleanup](#cleanup)

&nbsp;

## Architecture

### High-Level Overview

```mermaid
flowchart LR
    subgraph users [Users]
        nsAdmin["Namespace Admin"]
        clusterAdmin["Cluster Admin"]
    end

    subgraph ocpCluster [OpenShift Cluster]
        subgraph ns1 [grafana-user1 namespace]
            subgraph pod1 [Grafana Pod]
                oauthProxy1["oauth-proxy\n:9091 HTTPS"]
                grafana1["Grafana\n:3000"]
                nginx1["nginx sidecar\n:8080"]
            end
            sm1["ServiceMonitors\n+ Apps"]
        end
        subgraph ns2 [grafana-user2 namespace]
            subgraph pod2 [Grafana Pod]
                oauthProxy2["oauth-proxy\n:9091 HTTPS"]
                grafana2["Grafana\n:3000"]
                nginx2["nginx sidecar\n:8080"]
            end
            sm2["ServiceMonitors\n+ Apps"]
        end
        subgraph monitoring [openshift-monitoring]
            thanos["Thanos Querier\n:9092 tenancy"]
            uwProm["User Workload\nPrometheus"]
        end
    end

    nsAdmin -->|OpenShift OAuth| oauthProxy1
    clusterAdmin -->|OpenShift OAuth| oauthProxy1
    clusterAdmin -->|OpenShift OAuth| oauthProxy2
    oauthProxy1 -->|X-Forwarded-User| grafana1
    oauthProxy2 -->|X-Forwarded-User| grafana2
    grafana1 -->|"query"| nginx1
    grafana2 -->|"query"| nginx2
    nginx1 -->|"+ namespace=grafana-user1\n+ Bearer token"| thanos
    nginx2 -->|"+ namespace=grafana-user2\n+ Bearer token"| thanos
    thanos --- uwProm
    uwProm -.->|scrape| sm1
    uwProm -.->|scrape| sm2
```

### Pod Internal Flow

```mermaid
flowchart LR
    browser["Browser"] -->|HTTPS :9091| oauth["oauth-proxy\n(OpenShift auth +\nnamespace SAR check)"]
    oauth -->|"HTTP :3000\nX-Forwarded-User"| grafana["Grafana\n(auth.proxy mode)"]
    grafana -->|"HTTP :8080\nGET /api/v1/query?query=..."| nginx["nginx sidecar"]
    nginx -->|"HTTPS :9092\n+ namespace=my-ns\n+ Authorization: Bearer"| thanos["Thanos Querier\n(tenancy-enforced)"]
```

### Three containers per Grafana pod

| Container | Port | Role |
|---|---|---|
| **oauth-proxy** | 9091 (HTTPS) | Authenticates users via OpenShift OAuth. Namespace-scoped SAR check ensures only users with access to the namespace can log in. |
| **grafana** | 3000 (HTTP) | Grafana application using `auth.proxy` to trust the `X-Forwarded-User` header from oauth-proxy. All authenticated users get Grafana `Admin` role. |
| **nginx sidecar** | 8080 (HTTP) | Injects `namespace=<ns>` query parameter and `Authorization: Bearer <SA-token>` header into every Prometheus request before forwarding to Thanos 9092. |

&nbsp;

## Repository Structure

```
common/base/
├── core/              # Grafana CR, datasource, CA bundle, session secret
├── rbac/              # ClusterRole, ClusterRoleBindings, edit RoleBinding, SA token
└── thanos-proxy/      # nginx ConfigMap with namespace injection template
overlays/
├── grafana-user1/     # Overlay for the grafana-user1 namespace
└── grafana-user2/     # Overlay for the grafana-user2 namespace
```

Base files use `NAMESPACE` as a placeholder. Each overlay replaces it with the actual namespace name via kustomize patches.

&nbsp;

## Prerequisites

- User with the **cluster-admin** cluster role
- OpenShift 4.x
- Grafana Operator v5.x installed from OperatorHub (all namespaces mode)
- User Workload Monitoring enabled

&nbsp;

## Procedure (Manual Step-by-Step)

> Replace `my-grafana` with your desired namespace name throughout.

&nbsp;

### Step 1: Create the namespace

```shell
oc create namespace my-grafana
```

&nbsp;

### Step 2: Create RBAC resources

- Create the shared ClusterRole for the OAuth proxy (only needed once per cluster, idempotent):

```shell
cat <<EOF | oc apply -f -
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: grafana-oauth-proxy
rules:
  - apiGroups: [authentication.k8s.io]
    resources: [tokenreviews]
    verbs: [create]
  - apiGroups: [authorization.k8s.io]
    resources: [subjectaccessreviews]
    verbs: [create]
EOF
```

- Create the ClusterRoleBindings to allow the Grafana service account to perform OAuth token reviews and authentication delegation:

```shell
cat <<EOF | oc apply -f -
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: grafana-oauth-proxy-my-grafana
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: grafana-oauth-proxy
subjects:
  - kind: ServiceAccount
    name: grafana-sa
    namespace: my-grafana
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auth-delegator-my-grafana
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: grafana-sa
    namespace: my-grafana
EOF
```

- Grant the Grafana service account `edit` permissions in the namespace. This is required for the Thanos 9092 tenancy authorization check (`verb=create, resource=pods`):

```shell
cat <<EOF | oc apply -f -
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: grafana-sa-edit
  namespace: my-grafana
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
  - kind: ServiceAccount
    name: grafana-sa
    namespace: my-grafana
EOF
```

&nbsp;

### Step 3: Create secrets and ConfigMaps

- Create the OAuth proxy session secret and the trusted CA bundle ConfigMap:

```shell
cat <<EOF | oc apply -f -
---
apiVersion: v1
kind: Secret
metadata:
  name: grafana-proxy
  namespace: my-grafana
type: Opaque
data:
  session_secret: $(openssl rand -base64 43 | base64 -w 0)
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ocp-injected-certs
  namespace: my-grafana
  labels:
    config.openshift.io/inject-trusted-cabundle: "true"
EOF
```

&nbsp;

### Step 4: Create the nginx namespace-proxy ConfigMap

- This ConfigMap contains the nginx configuration that injects the fixed `namespace=` parameter and the service account Bearer token into every Prometheus query forwarded to Thanos 9092:

```shell
cat <<'EOF' | oc apply -f -
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: thanos-namespace-proxy
  namespace: my-grafana
data:
  nginx.conf.template: |
    worker_processes 1;
    error_log /tmp/error.log warn;
    pid /tmp/nginx.pid;
    events { worker_connections 64; }
    http {
      server {
        listen 8080;
        location / {
          set $args "$args&namespace=my-grafana";
          proxy_set_header Authorization "Bearer SA_TOKEN_PLACEHOLDER";
          proxy_pass https://thanos-querier.openshift-monitoring.svc.cluster.local:9092;
          proxy_ssl_verify off;
        }
      }
    }
EOF
```

> **Important:** Replace both occurrences of `my-grafana` in the ConfigMap (metadata.namespace and the `set $args` line) with your namespace name.

&nbsp;

### Step 5: Deploy the Grafana instance

- Create the Grafana CR with the OpenShift OAuth proxy sidecar and the nginx thanos-proxy sidecar. The oauth-proxy performs a namespace-scoped SAR check, ensuring only users with access to this namespace can log in. The nginx sidecar handles Thanos authentication and namespace injection.

> **Important:** Replace the three occurrences of `my-grafana` in the `openshift-sar`, `openshift-delegate-urls` args and the `dashboards` label with your namespace name.

```shell
cat <<'EOF' | oc apply -f -
---
apiVersion: grafana.integreatly.org/v1beta1
kind: Grafana
metadata:
  name: grafana
  namespace: my-grafana
  labels:
    dashboards: my-grafana
spec:
  persistentVolumeClaim:
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 1Gi
  client:
    preferIngress: false
  serviceAccount:
    metadata:
      annotations:
        serviceaccounts.openshift.io/oauth-redirectreference.primary: '{"kind":"OAuthRedirectReference","apiVersion":"v1","reference":{"kind":"Route","name":"grafana-route"}}'
  route:
    spec:
      port:
        targetPort: https
      tls:
        termination: reencrypt
      to:
        kind: Service
        name: grafana-service
        weight: 100
      wildcardPolicy: None
  service:
    metadata:
      annotations:
        service.beta.openshift.io/serving-cert-secret-name: grafana-tls
    spec:
      ports:
        - name: https
          port: 9091
          protocol: TCP
          targetPort: https
  deployment:
    spec:
      template:
        spec:
          volumes:
            - name: grafana-tls
              secret:
                secretName: grafana-tls
            - name: grafana-proxy
              secret:
                secretName: grafana-proxy
            - name: ocp-injected-certs
              configMap:
                name: ocp-injected-certs
            - name: grafana-data
              persistentVolumeClaim:
                claimName: grafana-pvc
            - name: thanos-proxy-config
              configMap:
                name: thanos-namespace-proxy
          containers:
            - name: grafana-proxy
              image: registry.redhat.io/openshift4/ose-oauth-proxy-rhel9:v4.20
              args:
                - -provider=openshift
                - -pass-basic-auth=false
                - -https-address=:9091
                - -http-address=
                - -email-domain=*
                - -upstream=http://localhost:3000
                - '-openshift-sar={"namespace": "my-grafana", "resource": "pods", "verb": "get"}'
                - '-openshift-delegate-urls={"/": {"namespace": "my-grafana", "resource": "pods", "verb": "get"}}'
                - -tls-cert=/etc/tls/private/tls.crt
                - -tls-key=/etc/tls/private/tls.key
                - -client-secret-file=/var/run/secrets/kubernetes.io/serviceaccount/token
                - -cookie-secret-file=/etc/proxy/secrets/session_secret
                - -openshift-service-account=grafana-sa
                - -openshift-ca=/etc/pki/tls/cert.pem
                - -openshift-ca=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
                - -openshift-ca=/etc/proxy/certs/ca-bundle.crt
                - -skip-auth-regex=^/metrics
              ports:
                - containerPort: 9091
                  name: https
              resources:
                requests:
                  cpu: 100m
                  memory: 128Mi
                limits:
                  cpu: 500m
                  memory: 256Mi
              volumeMounts:
                - mountPath: /etc/tls/private
                  name: grafana-tls
                  readOnly: true
                - mountPath: /etc/proxy/secrets
                  name: grafana-proxy
                  readOnly: true
                - mountPath: /etc/proxy/certs
                  name: ocp-injected-certs
                  readOnly: true
            - name: thanos-proxy
              image: registry.access.redhat.com/ubi9/nginx-124:latest
              command: ["/bin/sh", "-c"]
              args:
                - |
                  TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
                  sed "s|SA_TOKEN_PLACEHOLDER|${TOKEN}|g" /etc/nginx-custom/nginx.conf.template > /tmp/nginx.conf
                  exec nginx -c /tmp/nginx.conf -g 'daemon off;'
              ports:
                - containerPort: 8080
                  name: thanos-proxy
              resources:
                requests:
                  cpu: 10m
                  memory: 32Mi
                limits:
                  cpu: 50m
                  memory: 64Mi
              volumeMounts:
                - mountPath: /etc/nginx-custom
                  name: thanos-proxy-config
                  readOnly: true
  config:
    log:
      mode: console
    auth:
      disable_login_form: "False"
      disable_signout_menu: "True"
    auth.anonymous:
      enabled: "True"
    auth.basic:
      enabled: "True"
    auth.proxy:
      enabled: "True"
      enable_login_token: "True"
      header_name: X-Forwarded-User
      header_property: username
    users:
      auto_assign_org: "true"
      auto_assign_org_id: "1"
      auto_assign_org_role: Admin
EOF
```

- Wait for the Grafana Operator to reconcile:

```shell
oc get grafana grafana -n my-grafana -w
```

Wait until `STAGE STATUS` shows `success`.

&nbsp;

### Step 6: Create the SA token secret

- The Grafana Operator creates the `grafana-sa` service account during Step 5. Now create a long-lived token secret for the nginx sidecar to use when authenticating to Thanos:

```shell
cat <<EOF | oc apply -f -
---
apiVersion: v1
kind: Secret
metadata:
  name: grafana-ds-token
  namespace: my-grafana
  annotations:
    kubernetes.io/service-account.name: grafana-sa
type: kubernetes.io/service-account-token
EOF
```

- Wait a few seconds for the token to be populated, then re-sync the datasource by deleting and re-applying:

```shell
oc delete grafanadatasource prometheus-thanos -n my-grafana 2>/dev/null
```

&nbsp;

### Step 7: Deploy the datasource

- Create the GrafanaDatasource pointing to the local nginx sidecar (`http://localhost:8080`). The nginx sidecar handles namespace injection and authentication -- no auth headers needed in the datasource itself:

> **Important:** Replace `my-grafana` in the `instanceSelector.matchLabels.dashboards` field with your namespace name.

```shell
cat <<EOF | oc apply -f -
---
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDatasource
metadata:
  name: prometheus-thanos
  namespace: my-grafana
spec:
  datasource:
    access: proxy
    editable: false
    isDefault: true
    jsonData:
      httpMethod: GET
      timeInterval: 5s
    name: Prometheus
    type: prometheus
    url: "http://localhost:8080"
  instanceSelector:
    matchLabels:
      dashboards: my-grafana
EOF
```

- Verify the datasource was applied:

```shell
oc get grafanadatasource -n my-grafana
```

&nbsp;

### Step 8: Grant namespace admin to a user

```shell
oc adm policy add-role-to-user admin <username> -n my-grafana
```

&nbsp;

### Step 9: Access Grafana

- Get the Grafana route URL:

```shell
oc get route grafana-route -n my-grafana -o jsonpath='https://{.spec.host}{"\n"}'
```

- Open the URL in a browser. The user will be authenticated via OpenShift OAuth.
- The user has full Grafana admin on this instance and can create/upload dashboards.
- All Prometheus queries are automatically scoped to the `my-grafana` namespace by the nginx sidecar.

&nbsp;

## Kustomize Deployment (Alternative)

Instead of applying each resource manually, use the provided kustomize overlays:

```shell
# 1. Create the namespace
oc create namespace my-grafana

# 2. Create a new overlay (copy an example and replace namespace)
cp -r overlays/grafana-user1 overlays/my-grafana
# Edit overlays/my-grafana/kustomization.yaml:
#   Replace every occurrence of "grafana-user1" with "my-grafana"

# 3. Apply the overlay
oc apply -k overlays/my-grafana/

# 4. Wait for Grafana to reconcile
oc get grafana grafana -n my-grafana -w

# 5. Create the SA token secret (after the SA exists)
oc apply -f common/base/rbac/ds-token-secret.yaml -n my-grafana

# 6. Re-sync the datasource
oc delete grafanadatasource prometheus-thanos -n my-grafana
oc apply -k overlays/my-grafana/

# 7. Grant a user admin access
oc adm policy add-role-to-user admin <username> -n my-grafana
```

&nbsp;

## Namespace Isolation -- How It Works

- The **nginx sidecar** hardcodes `namespace=<ns>` into every request to Thanos 9092. This value is baked into the ConfigMap and cannot be changed by the Grafana user.
- The **SA token** used by nginx only has `edit` permissions in its own namespace via a RoleBinding.
- **Thanos 9092** performs a SubjectAccessReview to verify the SA can access the requested namespace.
- Even if a user uploads a dashboard with a namespace variable or selector, **all queries are forced through the nginx sidecar** which always appends the fixed namespace.
- Cross-namespace queries are **rejected by Thanos with `403 Forbidden`**.

&nbsp;

## RBAC Summary

| Resource | Scope | Purpose |
|---|---|---|
| ClusterRole `grafana-oauth-proxy` | Cluster (shared) | Allows oauth-proxy to perform token reviews and SAR checks |
| ClusterRoleBinding `grafana-oauth-proxy-<ns>` | Cluster | Binds the ClusterRole to the namespace's `grafana-sa` |
| ClusterRoleBinding `auth-delegator-<ns>` | Cluster | Allows `grafana-sa` to delegate authentication |
| RoleBinding `grafana-sa-edit` | Namespace | Grants `grafana-sa` the `edit` role (required for Thanos 9092 tenancy check) |
| Secret `grafana-ds-token` | Namespace | Long-lived SA token used by the nginx sidecar for Thanos authentication |

&nbsp;

## Cleanup

Remove a single instance:

```shell
# If deployed with kustomize:
oc delete -k overlays/my-grafana/

# Delete the namespace:
oc delete namespace my-grafana

# Delete cluster-scoped bindings:
oc delete clusterrolebinding grafana-oauth-proxy-my-grafana auth-delegator-my-grafana
```

Remove the shared ClusterRole (only if no other instances remain):

```shell
oc delete clusterrole grafana-oauth-proxy
```
