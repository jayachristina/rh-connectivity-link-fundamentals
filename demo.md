## HTTPRoute

```yaml
---
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: globex-mobile-gateway
  namespace: globex-apim-user1
  creationTimestamp: null
  labels:
    deployment: globex-mobile-gateway
    service: globex-mobile-gateway
spec:
  parentRefs:
    - kind: Gateway
      namespace: ingress-gateway
      name: prod-web
  hostnames:
    - globex-mobile.managed.sandbox2662.opentlc.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: "/mobile/services/product/category/"
          method: GET
      backendRefs:
        - name: globex-mobile-gateway
          namespace: globex-apim-user1
          port: 8080
    - matches:
        - path:
            type: Exact
            value: "/mobile/services/category/list"
          method: GET
      backendRefs:
        - name: globex-mobile-gateway
          namespace: globex-apim-user1
          port: 8080
```

```bash
oc apply -f httproute.yaml
```

## Auth Policy

```yaml
apiVersion: kuadrant.io/v1beta2
kind: AuthPolicy
metadata:
  name: globex-mobile-gateway
  namespace: globex-apim-user1
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: globex-mobile-gateway
    namespace: globex-apim-user1
  rules:
    authentication:
      "keycloak-users":
        jwt:
          issuerUrl: https://sso.apps.cl.cllt-us.sandbox2662.opentlc.com/realms/globex-user1
    response:
      success:
        dynamicMetadata:
          identity:
            json:
              properties:
                userid:
                  selector: auth.identity.sub
  routeSelectors:
    - matches: []
```

```bash
oc apply -f authpolicy-oidc.yaml
```

## Rate Limiting

```yaml
apiVersion: kuadrant.io/v1beta2
kind: RateLimitPolicy
metadata:
  name: globex-mobile-gateway
  namespace: globex-apim-user1
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: globex-mobile-gateway
    namespace: globex-apim-user1
  limits:
    "per-user":
      rates:
        - limit: 5
          duration: 10
          unit: second
      counters:
        - metadata.filter_metadata.envoy\.filters\.http\.ext_authz.identity.userid
```

```bash
oc apply -f ratelimiting-oidc.yaml
```