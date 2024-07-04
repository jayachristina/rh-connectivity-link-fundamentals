# Prerequisite: Create Managed Zone on AWS

Refer to the [Prerequisites](https://developers.redhat.com/articles/2024/06/12/getting-started-red-hat-connectivity-link-openshift#prerequisites) section to correctly create a Managed Zone on AWS.



# Configure  Red Hat Connectivity Link - Advanced for Globex

Refer: [Kuadrant tutorial](https://docs.kuadrant.io/0.7.0/kuadrant-operator/doc/user-guides/secure-protect-connect-single-multi-cluster/#prerequisites) 

Setup prerequisites
```sh
export zid=change-this-to-your-zone-id
export rootDomain=example.com
export gatewayNS=api-gateway
export gatewayName=external
export devNS=toystore
export AWS_ACCESS_KEY_ID=xxxx
export AWS_SECRET_ACCESS_KEY=xxxx
export AWS_REGION=us-east-1
export clusterIssuerName=lets-encrypt
export EMAIL=foo@example.com
```

## As a Platform Engineer

### 1. Create ManagedZone


Create the zone credentials as follows:
```sh
oc create ns ${gatewayNS}

oc -n ${gatewayNS} create secret generic aws-credentials \
  --type=kuadrant.io/aws \
  --from-literal=AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  --from-literal=AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
```

Then create a ManagedZone as follows:

```sh
kubectl apply -f - <<EOF
apiVersion: kuadrant.io/v1alpha1
kind: ManagedZone
metadata:
  name: managedzone
  namespace: ${gatewayNS}
spec:
  id: ${zid}
  domainName: ${rootDomain}
  description: "Kuadrant managed zone"
  dnsProviderSecretRef:
    name: aws-credentials
EOF
```

### 2. Add a TLS issuer

already created

### 3. Set up a Gateway
```sh
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ${gatewayName}
  namespace: ${gatewayNS}
  labels:
    kuadrant.io/gateway: "true"
spec:
    gatewayClassName: istio
    listeners:

    - allowedRoutes:
        namespaces:
          from: Same
      hostname: "*.${rootDomain}"
      name: api
      port: 443
      protocol: HTTPS
      tls:
        certificateRefs:
        - group: ""
          kind: Secret
          name: api-${gatewayName}-tls
        mode: Terminate
EOF
```
<hr>

### 4. Setup Default Policies: Auth, TLS, rate limit, and DNS policies 

#### 4.1 Set Default AuthPolicy
```sh
kubectl apply -f - <<EOF
apiVersion: kuadrant.io/v1beta2
kind: AuthPolicy
metadata:
  name: ${gatewayName}-auth
  namespace: ${gatewayNS}
spec:
  rules:
    authorization:
      deny-all:
        metrics: false
        opa:
          allValues: false
          rego: allow = true
        priority: 0
    response:
      unauthorized:
        body:
          value: |
            {
              "error": "Forbidden",
              "message": "Access denied by default by the gateway operator. If you are the administrator of the service, create a specific auth policy for the route."
            }
        headers:
          content-type:
            value: application/json
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: ${gatewayName}
EOF
```

Check that your auth policy was accepted by the controller as follows:

```sh
kubectl get authpolicy ${gatewayName}-auth -n ${gatewayNS} -o=jsonpath='{.status.conditions[?(@.type=="Accepted")].message}'
```
<hr>

#### 4.2 Set Default TLSPolicy

Refer to [this section](https://docs.kuadrant.io/0.7.0/kuadrant-operator/doc/user-guides/secure-protect-connect-single-multi-cluster/#set-the-tls-policy)

```sh
kubectl apply -f - <<EOF
apiVersion: kuadrant.io/v1alpha1
kind: TLSPolicy
metadata:
  name: ${gatewayName}-tls
  namespace: ${gatewayNS}
spec:
  targetRef:
    name: ${gatewayName}
    group: gateway.networking.k8s.io
    kind: Gateway
  issuerRef:
    group: cert-manager.io
    kind: ClusterIssuer
    name: ${clusterIssuerName}
EOF
```

Check that your TLS policy was accepted by the controller as follows:

```sh
kubectl get tlspolicy ${gatewayName}-tls -n ${gatewayNS} -o=jsonpath='{.status.conditions[?(@.type=="Accepted")].message}'
```

<hr>

#### 4.3 Set Default RateLimitPolicy
```sh
kubectl apply -f  - <<EOF
apiVersion: kuadrant.io/v1beta2
kind: RateLimitPolicy
metadata:
  name: ${gatewayName}-rlp
  namespace: ${gatewayNS}
spec:
  defaults:
    limits:
      low-limit:
        rates:
          - duration: 10
            limit: 2
            unit: second
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: mobile-gateway
EOF
```

To check your rate limits have been accepted, enter the following command:
```sh
kubectl get ratelimitpolicy ${gatewayName}-rlp -n ${gatewayNS} -o=jsonpath='{.status.conditions[?(@.type=="Accepted")].message}'
```

<hr>

#### 4.4 Set Default DNSPolicy

```sh
kubectl apply -f - <<EOF
apiVersion: kuadrant.io/v1alpha1
kind: DNSPolicy
metadata:
  name: ${gatewayName}-dnspolicy
  namespace: ${gatewayNS}
spec:
  routingStrategy: loadbalanced
  loadBalancing:
    geo: 
      defaultGeo: EU 
    weighted:
      defaultWeight: 120 
  targetRef:
    name: ${gatewayName}
    group: gateway.networking.k8s.io
    kind: Gateway
EOF
```

### 5.Test connectivity and deny all auth

* You can use curl to hit your endpoint. Because this example uses Let's Encrypt staging, you can pass the -k flag:
```sh
curl -k -w "%{http_code}" https://$(kubectl get httproute test -n ${gatewayNS} -o=jsonpath='{.spec.hostnames[0]}')
```

* You should see a 403.

### Opening up the Gateway for other namespaces

```sh
kubectl patch gateway ${gatewayName} -n ${gatewayNS} --type='json' -p='[{"op": "replace", "path": "/spec/listeners/0/allowedRoutes/namespaces/from", "value":"All"}]'
```

<hr>


## As a Developer

### 1. Globex Mobile is already created



<hr>

### 2. Set up HTTPRoute and backend

```sh
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: globex-mobile
  namespace: ${gatewayNS} 
spec:
  hostnames:
    - mobile.${rootDomain}
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: ${gatewayName}
      namespace: ${gatewayNS} 
  rules:
    - backendRefs:
        - group: ''
          kind: Service
          namespace: globex-apim-user1
          name: globex-mobile-gateway
          port: 8080
          weight: 1
      matches:
        - path:
            type: PathPrefix
            value: /
# ---
# apiVersion: gateway.networking.k8s.io/v1beta1
# kind: ReferenceGrant
# metadata:
#   name: globex-ref-grant
#   namespace: globex-apim-user1
# spec:
#   from:
#   - group: gateway.networking.k8s.io
#     kind: HTTPRoute
#     namespace: globex-gateway
#   to:
#   - group: ""
#     kind: Service
EOF
```

<br>

### 3. Test connectivity and deny-all auth

You can use curl to hit an endpoint in the toystore app. Because you are using Let's Encrypt staging in this example, you can pass the -k flag as follows:

```sh
curl -s -k -o /dev/null -w "%{http_code}" "https://$(kubectl get httproute globex-mobile -n ${gatewayNS} -o=jsonpath='{.spec.hostnames[0]}')/v1/toys"
```

You are getting a 403 because of the existing default, deny-all AuthPolicy applied at the Gateway. You can override this for your HTTPRoute.


<br>

### 3. Setup overrides: Auth and Rate limit  policies using Auth Key


#### 3.1 Override AuthPolicy
Ref: [on Kuadrant](https://docs.kuadrant.io/0.7.0/kuadrant-operator/doc/user-guides/secure-protect-connect/#override-the-gateways-deny-all-authpolicy)

##### Now, we will override the AuthPolicy to start accepting the API keys:

```sh
kubectl apply -f - <<EOF
apiVersion: kuadrant.io/v1beta2
kind: AuthPolicy
metadata:
  name: globex-mobile
  namespace: ${gatewayNS} 
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: globex-mobile
  rules:
    authentication:
      "keycloak-kuadrant-realm":
        jwt:
          issuerUrl: https://sso.apps.rhcl.sandbox940.opentlc.com/realms/globex-user1
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
EOF
```

#### Test if the API key works for ALICE

```sh
curl -k -w "%{http_code}" https://$(kubectl get httproute toystore -n ${gatewayNS} -o=jsonpath='{.spec.hostnames[0]}') -H 'Authorization: APIKEY IAMALICE'
```

#### 3.1 Override RateLimitPolicy
```sh
kubectl apply -f - <<EOF
apiVersion: kuadrant.io/v1beta2
kind: RateLimitPolicy
metadata:
  name: globex-mobile
  namespace: ${gatewayNS} 
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: globex-mobile
  limits:
    "all-users":
      rates:
      - limit: 5
        duration: 10
        unit: second
      counters:
        - metadata.filter_metadata.envoy\.filters\.http\.ext_authz.identity.userid
EOF
```


### Let's test this new setup.

#### By sending requests as alice:

```sh
while :; do curl -k -w "%{http_code}" https://$(kubectl get httproute toystore -n ${gatewayNS} -o=jsonpath='{.spec.hostnames[0]}')  --silent --output /dev/null -H 'Authorization: APIKEY IAMALICE' | grep -E --color "\b(429)\b|$"; sleep 1; done
```

#### By sending requests as bob:

```sh
while :; do curl -k -w "%{http_code}" https://$(kubectl get httproute toystore -n ${gatewayNS} -o=jsonpath='{.spec.hostnames[0]}')  --silent --output /dev/null -H 'Authorization: APIKEY IAMBOB' | grep -E --color "\b(429)\b|$"; sleep 1; done
```

