# Connectivity Link Sample Demo


## Prerequisites

- OpenShift Cluster where ConnectivityLink 1.5 or higher is available.

- Deploy small application: https://github.com/osa-ora/java-ocp-demo

```bash
oc new-project conn-link
oc new-app java:openjdk-17-ubi8~https://github.com/osa-ora/java-ocp-demo \
  --name=java-ocp-demo \
  -n conn-link

oc logs -f bc/java-ocp-demo -n conn-link
oc rollout status deployment/java-ocp-demo -n conn-link

oc expose svc/java-ocp-demo -n conn-link

```

This application will expose 2 end points: 

```
export APP_URL=$(oc get route java-ocp-demo -n conn-link -o jsonpath='{.spec.host}')

# Test Balance endpoint
curl "http://${APP_URL}/loyalty/v1/balance/1000"

# Test Transaction endpoint
curl "http://${APP_URL}/loyalty/v1/transaction/1000"

```

<img width="292" height="336" alt="Screenshot 2026-10-29 at 12 20 32 PM" src="https://github.com/user-attachments/assets/0a7cad30-ebed-458b-874d-df39e8ba8b6a" />

After you deploy the application and test the route, delete that route.

```
oc delete route java-ocp-demo -n conn-link

```

## 1. Infrastructure Setup

**Install the Istio Service Mesh Operator (Sail Operator)**

```bash
oc new-project istio-system

```

```yaml
apiVersion: sailoperator.io/v1alpha1
kind: Istio
metadata:
  name: default
  namespace: istio-system
spec:
  # The specific version of Istio deployed by the Sail Operator
  version: v1.22.1 
  namespace: istio-system

```

**Install the Connectivity Link Operator (Kuadrant)**

```yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant-sample
  namespace: conn-link
spec:
  mtls:
    limitador: false
  tokenRateLimiting:
    mode: Reservation

```

> **Note:** If you provisioned Kuadrant before the Service Mesh, restart the operator so it detects Istio:
> `kubectl delete pod -l control-plane=kuadrant-operator -n openshift-operators`

---

## 2. Gateway and Routing Configuration

**Create the Gateway**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: conn-link
spec:
  gatewayClassName: istio
  listeners:
    - name: api-listener
      port: 80
      protocol: HTTP
      allowedRoutes:
        kinds:
          - group: gateway.networking.k8s.io
            kind: HTTPRoute
        namespaces:
          from: All

```

**Create the HTTPRoutes**

*Loyalty Balance API Route:*

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-loyalty-balance-api
  namespace: conn-link
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: my-gateway
      port: 80
      sectionName: api-listener
  rules:
    - matches:
        - method: GET
          path:
            type: PathPrefix
            value: /loyalty/v1/balance
      backendRefs:
        - group: ''
          kind: Service
          name: java-ocp-demo
          port: 8080
          weight: 1

```

*Loyalty Transaction API Route:*

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-loyalty-transaction-api
  namespace: conn-link
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: my-gateway
      port: 80
      sectionName: api-listener
  rules:
    - matches:
        - method: GET
          path:
            type: PathPrefix
            value: /loyalty/v1/transaction
      backendRefs:
        - group: ''
          kind: Service
          name: java-ocp-demo
          port: 8080
          weight: 1

```

---

## 3. Testing the Gateway & Routes

Extract your Gateway's ELB address and save it as an environment variable:

```bash
# Get the gateway location
kubectl get gateway my-gateway -n conn-link -o jsonpath='{.status.addresses[0].value}'

# Export it as a variable for testing
export GATEWAY_URL=$(kubectl get gateway my-gateway -n conn-link -o jsonpath='{.status.addresses[0].value}')

```

Run test requests:

```bash
curl "http://${GATEWAY_URL}/loyalty/v1/balance/1000"
curl "http://${GATEWAY_URL}/loyalty/v1/transaction/1000"

```

---

## 4. Rate Limiting

**Global Gateway Rate Limit (5 requests / 30s)**

```yaml
apiVersion: kuadrant.io/v1
kind: RateLimitPolicy
metadata:
  name: gw-rate-limit
  namespace: conn-link
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: my-gateway
  limits:
    gateway-limit:
      rates:
        - limit: 5
          window: 30s

```

*Test the gateway limit by running the curl commands more than 5 times quickly.*

**Route-Level Rate Limit (10 requests / 30s for Balance API)**

```yaml
apiVersion: kuadrant.io/v1
kind: RateLimitPolicy
metadata:
  name: loyalty-balance-rate-limit
  namespace: conn-link
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: my-loyalty-balance-api
  limits:
    my-rate:
      rates:
        - limit: 10
          window: 30s

```

*Run both routes more than 10 times to see which one succeeds and which one fails first.*

---

## 5. Security & Authorization

**Create a Global "Deny All" AuthPolicy**

```yaml
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: deny-all-authpolicy
  namespace: conn-link
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: my-gateway
  rules:
    authorization:
      denyAll:
        metrics: false
        opa:
          allValues: false
          rego: allow = false
          version: v0
        priority: 0
    response:
      unauthorized:
        body:
          value: |-
            {
              "error": "Forbidden",
              "message": "Access denied by default. Create a specific auth policy for the route."
            }
        headers:
          content-type:
            value: application/json

```

*Test the Deny All policy by running your curl commands. Both should now be blocked.*

```bash
curl "http://${GATEWAY_URL}/loyalty/v1/balance/1000"
curl "http://${GATEWAY_URL}/loyalty/v1/transaction/1000"

```

**Enable API Key Authentication for the Balance API**

```yaml
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: loyalty-api-auth
  namespace: conn-link
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: my-loyalty-balance-api
  rules:
    authentication:
      api-key-users:
        apiKey:
          allNamespaces: false
          selector:
            matchLabels:
              api-access: loyalty
        credentials:
          queryString:
            name: api_key
        metrics: false
        priority: 0

```

*Test without a key to confirm you receive a 401 Unauthorized error.*

```bash
curl "http://${GATEWAY_URL}/loyalty/v1/balance/1000"
curl "http://${GATEWAY_URL}/loyalty/v1/transaction/1000"

```

---

## 6. Developer Portal (API Discovery and Request Access)

**Create a PlanPolicy (Gold Tier)**

```yaml
apiVersion: extensions.kuadrant.io/v1alpha1
kind: PlanPolicy
metadata:
  name: loyalty-api-plans
  namespace: conn-link
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: my-loyalty-balance-api
  plans:
    - tier: Gold
      predicate: "has(auth.identity) && auth.identity.metadata.annotations['secret.kuadrant.io/plan-id'] == 'Gold'"
      limits:
        daily: 100
        weekly: 1000

```

**Publish the API Product to the Dev Portal**

```yaml
apiVersion: devportal.kuadrant.io/v1alpha1
kind: APIProduct
metadata:
  name: balance-api-3654
  namespace: conn-link
spec:
  displayName: Balance_API
  description: Get the balance
  tags:
    - v1
  publishStatus: Published
  approvalMode: manual
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: my-loyalty-balance-api

```

**Generate the API Key Secret**

You can either request the API key via the Kuadrant UI and approve it, or manually create this secret to test immediately:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-valid-api-key
  namespace: conn-link
  labels:
    authorino.kuadrant.io/managed-by: authorino
    api-access: loyalty
stringData:
  api_key: "super-secret-key-123"
type: Opaque

```

Or Request Key for the API product and specific plan:

<img width="616" height="740" alt="Screenshot 2026-10-29 at 12 23 33 PM" src="https://github.com/user-attachments/assets/6428d27c-68ba-461d-868a-5c9c963f8c51" />

Approve it:

<img width="1385" height="396" alt="Screenshot 2026-10-29 at 12 24 21 PM" src="https://github.com/user-attachments/assets/d24fa600-6d80-470a-bce7-978f9e9ea772" />


Copy the key to use it:

<img width="1419" height="354" alt="Screenshot 2026-10-29 at 12 24 54 PM" src="https://github.com/user-attachments/assets/b0234688-79fa-4a7b-a0ca-864cf715f1f3" />


**Final Verification Tests**

** Note: Replace super-secret-key-123 with the actual key they copied from the UI if you used the console not the secret approach.

```bash
# Balance API with valid key (Should Succeed)
curl "http://${GATEWAY_URL}/loyalty/v1/balance/1000?api_key=super-secret-key-123"

# Transaction API without key (Should Fail - Deny All)
curl "http://${GATEWAY_URL}/loyalty/v1/transaction/1000"

# Transaction API with key (Should Fail - Route not mapped for Auth)
curl "http://${GATEWAY_URL}/loyalty/v1/transaction/1000?api_key=super-secret-key-123"

```

Topology View:

<img width="1309" height="623" alt="Screenshot 2026-10-29 at 12 27 07 PM" src="https://github.com/user-attachments/assets/2b55c09d-c4e7-412a-a4c5-7dd5c9a7aab8" />


Overview:

<img width="1705" height="835" alt="Screenshot 2026-10-29 at 12 27 43 PM" src="https://github.com/user-attachments/assets/4d135614-79e7-4433-b866-813496cc68f0" />


## 7. Clean Everything


```bash

oc delete project conn-link --wait=true
oc delete project istio-system --wait=true

```
---

