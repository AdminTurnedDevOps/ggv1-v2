# Gloo Gateway V1/V2 POC
<p align="center">
 <img src="images/1.png?raw=true" alt="Logo" width="50%" height="50%" />
</p>


The purpose of this POC is to showcase how to not only install and configure Gloo Gateway (v1 for portal and v2 for everything else), but to have comprehensive tests for:

- Resilience
- Reliability
- Failover
- Uptime

And overall traffic management needs

# Gloo Gateway V2 Configs

The following sections will showcase everything from a Gloo Gateway V2 perspective including installation, configuration, and resilience configs.

## Installation Gloo Gateway v2

### Helm
1. Configure product key env variables

```
export GLOO_GATEWAY_LICENSE_KEY=
```

2. Install Kubernetes Gateway API
You need the experimental version as Gloo Gateway v2 has a requirement of the `BackendConfigPolicy` object, which is an experimental feature in Kubernetes Gateway API.

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/experimental-install.yaml
```

3. Install Gloo Gateway v2 CRDs
```
helm upgrade -i gloo-gateway-crds oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/gloo-gateway-crds \
--namespace gloo-system \
--version 2.0.0-rc.2 \
--create-namespace
```

4. Install Gloo Gateway v2
```
helm upgrade -i gloo-gateway oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/gloo-gateway \
-n gloo-system \
--version 2.0.0-rc.2 \
--set licensing.glooGatewayLicenseKey=$GLOO_GATEWAY_LICENSE_KEY
```

5. Confirm that Gloo Gateway is running
```
kubectl get pods -n gloo-system
NAME                           READY   STATUS    RESTARTS   AGE
gloo-gateway-6dfbfb7d7-9kbhf   1/1     Running   0          2m38s
```


#### Argo/GitOps

ArgoCD installation is available as well: https://docs.solo.io/gateway/2.0.x/install/argocd/

## Sample App Deployment

1. Create the Namespace for the microapp (extensive decoupled app)
```
kubectl create ns microapp
```

2. Deploy the sample decoupled application stack
```
kubectl apply -f ggv2/sampleapp-microdemo/microservices-demo/release/kubernetes-manifests.yaml -n microapp
```

3. Confirm that the app stack is running
```
kubectl get pods -n microapp
```

You can also see the Services that are deployed, which is what you'll use to create the backend routes in the next step.

```
kubectl get svc -n microapp
```

4. Create a Gateway for the application

The `allowedroutes` portion means that you can create an `HTTPRoute` resource from all Namespaces

```
kubectl apply --context=$CLUSTER1 -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: frontend-gateway
  namespace: microapp
spec:
  gatewayClassName: gloo-gateway-v2
  listeners:
  - name: frontend
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend
  namespace: microapp
spec:
  parentRefs:
  - name: frontend-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
      - name: frontend
        port: 80
EOF
```

5. Check to see the gateway IP address.

```
kubectl get gateway -n microapp
```

Example output below:
```
NAME               CLASS             ADDRESS         PROGRAMMED   AGE
frontend-gateway   gloo-gateway-v2   x.x.x.x   True         36m
```

![](images/3.png)

You can now `curl` the gateway IP or use a tool like Postman.

## Gateway UI
1. Capture your cluster name as an environment variable for the UI installation in the coming steps.
```
export CLUSTER_NAME=

echo $CLUSTER_NAME
```

2. Add the Gloo Platform Chart (for the UI)
```
helm repo add gloo-platform https://storage.googleapis.com/gloo-platform/helm-charts
helm repo update
```

3. Install the Gloo Platform CRDs
```
helm upgrade -i gloo-platform-crds gloo-platform/gloo-platform-crds \
--namespace=gloo-system \
--version=2.10.1 \
--set installEnterpriseCrds=false
```

4. Deploy the UI Helm Chart

You'll see that the enterprise version of the chart gives you:
- The UI
- Gloo Insights that you can see from the portal
- Prometheus enabled for metrics collection
- A Telemetry collector

```
helm upgrade -i gloo-platform gloo-platform/gloo-platform \
--namespace gloo-system \
--version=2.10.1 \
-f - <<EOF
common:
  adminNamespace: "gloo-system"
  cluster: $CLUSTER_NAME
glooInsightsEngine:
  enabled: true
glooAnalyzer:
  enabled: true
glooUi:
  enabled: true
licensing:
  glooGatewayLicenseKey: $GLOO_GATEWAY_LICENSE_KEY
prometheus:
  enabled: true
telemetryCollector:
  enabled: true
  mode: deployment
  replicaCount: 1
EOF
```

5. Ensure that the UI it's running as expected (you should see 3 containers in the Pod)
```
kubectl get pods -n gloo-system
```

6. Access the UI
```
kubectl port-forward deployment/gloo-mesh-ui -n gloo-system 8090
```

![](images/4.png)

## Monitoring, Observability, & Telemetry

1. Add the Grafana Helm Chart
```
helm repo add grafana https://grafana.github.io/helm-charts
```

2. Install Grafana in the `monitoring` Namespace
```
helm install grafana grafana/grafana --namespace monitoring --create-namespace
```

3. Retrieve the default admin password from the Kubernetes Secret.

The default username is: `admin`

```
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

4. Access the Grafana UI
```
kubectl port-forward svc/grafana 3000:80 --namespace monitoring
```

5. Add a new metrics endpoint by going to: **Connections > Data Sources > Choose Prometheus

Under the Connection URL, add the following:
```
http://prometheus-server.gloo-system.svc.cluster.local:80
```

You'll now be able to see Metrics and create Dashboard in Grafana

![](images/6.png)

For a list of metrics exposed via the Control Plane:

![](images/7.png)


## Traffic Debugging

Debugging live traffic, ensuring health of traffic, and showcasing policies can be done from your monitoring & observability tool of choice, but also form the Gloo UI.

1. Port-forward to the deployed UI
```
kubectl port-forward deployment/gloo-mesh-ui -n gloo-system 8090
```

2. Access the UI in a browser

```
127.0.0.1:8090
```

From the dashboard, you can see requests per second, errors, and any latency that may be occurring, or has occurred, within your application.

![](images/8.png)

The Graph can give you information on how your Services are connected and various forms of telemetry data like requests, latency, and errors. 

![](images/9.png)
![](images/10.png)

Without Routes, you can see the HTTP Routes for your application including any hostnames, the Gateways the route is attached to, and the path/destination.

![](images/11.png)

## Rate Limiting

With Gloo Gateway v2, you will see that there is a Rate Limit server out of the box.

```
kubectl get svc -n gloo-system
NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
rate-limiter-gloo-gateway-v2       ClusterIP   LB-PULIC-IP   <none>        8083/TCP,8084/TCP,9091/TCP   6m1s
```

You can set the traffic policy for the `HTTPRoute` that you configured a few sections ago.

1. Apply the `TrafficPolicy` for local rate limiting
```
kubectl apply -f- <<EOF
apiVersion: gloo.solo.io/v1alpha1
kind: GlooTrafficPolicy
metadata:
  name: local-frontend
  namespace: microapp
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: frontend
  rateLimit:
    local:
      tokenBucket:
        maxTokens: 1
        tokensPerFill: 1
        fillInterval: 100s
EOF
```

2. Confirm that the traffic policy has been created
```
kubectl get glootrafficpolicy -n microapp

NAME             AGE
local-frontend   71s
```

3. Curl the demo app.
```
curl -n http://LB-PULIC-IP
```


4. Send the `curl` again and you'll see an output like the below:

```
local_rate_limited% 
```

The reason why is because the `GlooTrafficPolicy` configured only has one (1) token and it is refilled every 100 seconds.

5. Delete the `GlooTrafficPolicy` to avoid any rate limiting issues for the testing through the POC.

## Advanced Routing

In this section, you will find advanced routing and authentication implementations like JWT and Canary.

### JWT (Authentication)

1. Set a variable with the Gateway for the frontend app that you deployed in a previous section
```
export INGRESS_GW_ADDRESS=$(kubectl get svc -n microapp frontend-gateway -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
echo $INGRESS_GW_ADDRESS
```

2. Create the Gloo Traffic Policy that specifies the need for a JWT token
```
kubectl apply -f- <<EOF
apiVersion: gloo.solo.io/v1alpha1
kind: GlooTrafficPolicy
metadata:
  name: jwt
  namespace: microapp
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: frontend-gateway
  glooJWT:
    beforeExtAuth:
      providers:
        selfminted:
          issuer: solo.io
          jwks:
            local:
              key: '{"keys":[{"kty":"RSA","kid":"solo-public-key-001","use":"sig","alg":"RS256","n":"AOfIaJMUm7564sWWNHaXt_hS8H0O1Ew59-nRqruMQosfQqa7tWne5lL3m9sMAkfa3Twx0LMN_7QqRDoztvV3Wa_JwbMzb9afWE-IfKIuDqkvog6s-xGIFNhtDGBTuL8YAQYtwCF7l49SMv-GqyLe-nO9yJW-6wIGoOqImZrCxjxXFzF6mTMOBpIODFj0LUZ54QQuDcD1Nue2LMLsUvGa7V1ZHsYuGvUqzvXFBXMmMS2OzGir9ckpUhrUeHDCGFpEM4IQnu-9U8TbAJxKE5Zp8Nikefr2ISIG2Hk1K2rBAc_HwoPeWAcAWUAR5tWHAxx-UXClSZQ9TMFK850gQGenUp8","e":"AQAB"}]}'
EOF
```

3. Curl the Gateway address
```
curl -vik http://$INGRESS_GW_ADDRESS
```

You should see a failure similar to the below:

```
* Connection #0 to host LB-PULIC-IP left intact
Jwt is missing%                                                                                                                                           
```

4. Export the token for authentication to the frontend app
```
export BOB_TOKEN=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InNvbG8tcHVibGljLWtleS0wMDEifQ.eyJpc3MiOiJzb2xvLmlvIiwib3JnIjoic29sby5pbyIsInN1YiI6ImJvYiIsInRlYW0iOiJvcHMiLCJleHAiOjIwNzQyNzQ5NTQsImxsbXMiOnsibWlzdHJhbGFpIjpbIm1pc3RyYWwtbGFyZ2UtbGF0ZXN0Il19fQ.GF_uyLpZSTT1DIvJeO_eish1WDjMaS4BQSifGQhqPRLjzu3nXtPkaBRjceAmJi9gKZYAzkT25MIrT42ZIe3bHilrd1yqittTPWrrM4sWDDeldnGsfU07DWJHyboNapYR-KZGImSmOYshJlzm1tT_Bjt3-RK3OBzYi90_wl0dyAl9D7wwDCzOD4MRGFpoMrws_OgVrcZQKcadvIsH8figPwN4mK1U_1mxuL08RWTu92xBcezEO4CdBaFTUbkYN66Y2vKSTyPCxg3fLtg1mvlzU1-Wgm2xZIiPiarQHt6Uq7v9ftgzwdUBQM1AYLvUVhCN6XkkR9OU3p0OXiqEDjAxcg
```

5. Run the curl again
```
curl -vik http://$INGRESS_GW_ADDRESS \
--header "Authorization: Bearer $BOB_TOKEN"
```

You should see an output similar to the below, indicating that the token was accepted

```
* Connection #0 to host 35.231.233.180 left intact
```

6. Delete the JWT policy

```
kubectl delete -f- <<EOF
apiVersion: gloo.solo.io/v1alpha1
kind: GlooTrafficPolicy
metadata:
  name: jwt
  namespace: microapp
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: frontend-gateway
  glooJWT:
    beforeExtAuth:
      providers:
        selfminted:
          issuer: solo.io
          jwks:
            local:
              key: '{"keys":[{"kty":"RSA","kid":"solo-public-key-001","use":"sig","alg":"RS256","n":"AOfIaJMUm7564sWWNHaXt_hS8H0O1Ew59-nRqruMQosfQqa7tWne5lL3m9sMAkfa3Twx0LMN_7QqRDoztvV3Wa_JwbMzb9afWE-IfKIuDqkvog6s-xGIFNhtDGBTuL8YAQYtwCF7l49SMv-GqyLe-nO9yJW-6wIGoOqImZrCxjxXFzF6mTMOBpIODFj0LUZ54QQuDcD1Nue2LMLsUvGa7V1ZHsYuGvUqzvXFBXMmMS2OzGir9ckpUhrUeHDCGFpEM4IQnu-9U8TbAJxKE5Zp8Nikefr2ISIG2Hk1K2rBAc_HwoPeWAcAWUAR5tWHAxx-UXClSZQ9TMFK850gQGenUp8","e":"AQAB"}]}'
EOF
```


### Mirroring (Canary Deployments)

To test Mirroring (Canary), you'll need an application that has two versions readily available. Because of that, you can follow two instruction sets (they're short) to ensure you have the proper environment for Mirroring.

1. Deploy the http app, which has v1 and v2 available out of the box: https://docs.solo.io/gateway/2.0.x/operations/sample-app/
2. Use these instructions after you deploy the http app: https://docs.solo.io/gateway/2.0.x/resiliency/mirroring/

### Load Balancing

There are several methods of load balancing:
1. Least Requests
2. Round robin
3. Random

The goal with load balancing is for the LB policy to hit hosts (Pods) that have the least amount of load. That way, performance can occur as expected.

The following Round Robin policy will:
- Set slow start more to progressively increase the amount of traffic that is routed to it
- Set the duration of the slow start
- Increase the rate of traffic to the host (Pod)
- Set a minimum percentage of weight that an endpoint needs to calcuate aggression

```
kubectl apply -f- <<EOF
kind: BackendConfigPolicy
apiVersion: gateway.kgateway.dev/v1alpha1
metadata:
  name: microapp-rr-policy
  namespace: microapp
spec:
  targetRefs:
    - name: frontend
      group: ""
      kind: Service
  loadBalancer:
    roundRobin:
      slowStart:
        window: 10s
        aggression: "1.5"
        minWeightPercent: 10
EOF
```

### Redirection

1. Set a variable with the Gateway for the frontend app that you deployed in a previous section
```
export INGRESS_GW_ADDRESS=$(kubectl get svc -n microapp frontend-gateway -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
echo $INGRESS_GW_ADDRESS
```

2. Curl the Gateway address to confirm it works
```
curl -vik http://$INGRESS_GW_ADDRESS
```

3. Create an HTTP Route that configures a redirect
```
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-redirect
  namespace: microapp
spec:
  parentRefs:
    - name: frontend-gateway
      namespace: microapp
  hostnames:
    - host.redirect.example
  rules:
    - filters:
      - type: RequestRedirect
        requestRedirect:
          hostname: "www.example.com"
          statusCode: 302
EOF
```

4. Curl the Gateway with the redirect host
```
curl -vi http://$INGRESS_GW_ADDRESS -H "host: host.redirect.example:8080"
```

5. You should see a `302` similar to the below
```
*   Trying 35.231.233.180:80...
* Connected to 35.231.233.180 (35.231.233.180) port 80
> GET / HTTP/1.1
> Host: host.redirect.example:8080
> User-Agent: curl/8.7.1
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 302 Found
HTTP/1.1 302 Found
```

## Resiliency/Circuit Breaking

In this section, you will find three subsections that showcase resilience configurations and circuit breaking (outlier detection and connection settings)

### Outlier Detection

Outlier Detection in Circuit Breaking is all about removing an unhealthy host (Pod) from the load balancing pool.

The test configuration below specifies that if the frontend host returns one 5XX HTTP response code, it'll eject the unhealthy host (Pod) for one hour.

The unhealthy host (Pod) will then be brought back into the load balancing pool after the one hour is up

```
kubectl apply -f- <<EOF
apiVersion: gateway.kgateway.dev/v1alpha1
kind: BackendConfigPolicy
metadata:
  name: microapp-dead-app-protection
  namespace: microapp
spec:
  targetRefs:
    - name: frontend
      group: ""
      kind: Service
  outlierDetection:
    interval: 2s
    consecutive5xx: 1
    baseEjectionTime: 1h
    maxEjectionPercent: 80
EOF
```

### HTTP Connecting Settings

1. Timeout and read/write buffer limits for connections to the Service.
```
kubectl apply -f - <<EOF
apiVersion: gateway.kgateway.dev/v1alpha1
kind: BackendConfigPolicy
metadata:
  name: microapp-buffer
  namespace: microapp
spec:
  targetRefs:
    - name: httpbin
      group: ""
      kind: Service
  connectTimeout: 5s
  perConnectionBufferLimitBytes: 1024
EOF
```

2. Additional connection options when handling upstream HTTP requests
```
kubectl apply -f - <<EOF
apiVersion: gateway.kgateway.dev/v1alpha1
kind: BackendConfigPolicy
metadata:
  name: microapp-connections
  namespace: microapp
spec:
  targetRefs:
    - name: frontend
      group: ""
      kind: Service
  commonHttpProtocolOptions:
    idleTimeout: 10s
    maxHeadersCount: 15
    maxStreamDuration: 30s
    maxRequestsPerConnection: 100
EOF
```

### Retries

Retries enhances an app’s availability by making sure that calls don’t fail permanently because of transient problems, such as a temporarily overloaded service or network.

1. Delete the `HTTPRoute` that you created during the sample app deployment
```
kubectl delete httproute frontend -n microapp
```

2. Capture the Gateway address
```
export INGRESS_GW_ADDRESS=$(kubectl get svc -n microapp frontend-gateway -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
echo $INGRESS_GW_ADDRESS
```

3. Set up an access policy that tracks the number of retries.
```
kubectl apply -f- <<EOF
apiVersion: gateway.kgateway.dev/v1alpha1
kind: HTTPListenerPolicy
metadata:
  name: access-logs
  namespace: microapp
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: frontend-gateway
  accessLog:
  - fileSink:
      path: /dev/stdout
      jsonFormat:
        start_time: "%START_TIME%"
        method: "%REQ(:METHOD)%"
        path: "%REQ(:PATH)%"
        response_code: "%RESPONSE_CODE%"
        response_flags: "%RESPONSE_FLAGS%"
        upstream_host: "%UPSTREAM_HOST%"
        upstream_cluster: "%UPSTREAM_CLUSTER%"
EOF
```

4. Create a new `HTTPRoute` to test against. This `HTTPRoute` can be used within the `GlooTrafficPolicy` that applies a retry policy.
```
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: retry
  namespace: microapp
spec:
  hostnames:
  - retry.example
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: frontend-gateway
    namespace: microapp
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - group: ""
      kind: Service
      name: frontend
      port: 80
    name: timeout
EOF
```

5. Create the traffic policy for retries
```
kubectl apply -f- <<EOF
apiVersion: gloo.solo.io/v1alpha1
kind: GlooTrafficPolicy
metadata:
  name: retry
  namespace: microapp
spec:
  targetRefs:
  - kind: HTTPRoute
    group: gateway.networking.k8s.io
    name: retry
    sectionName: timeout
  retry:
    attempts: 3
    backoffBaseInterval: 1s
    retryOn:
    - 5xx
    - unavailable
  timeouts:
    request: 20s
EOF
```

6. Get the gateway address and send a request
```
curl -vi http://$INGRESS_GW_ADDRESS/cart -H "host: retry.example:8080"
```

7. Check that no retry occurred
```
kubectl logs -n microapp -l gateway.networking.k8s.io/gateway-name=frontend-gateway | tail -1 | jq
```

8. Scale the app down to `0`
```
kubectl scale deployment cartservice -n microapp --replicas=0
```

9. Curl again
```
curl -vi http://$INGRESS_GW_ADDRESS/cart -H "host: retry.example:8080"
```

10. Open a new tab while the `curl` is running and look at the logs
```
kubectl logs -n microapp -l gateway.networking.k8s.io/gateway-name=frontend-gateway | tail -1 | jq
```

You should see

```
{
  "method": "GET",
  "path": "/cart",
  "response_code": 500,
  "response_flags": "URX",
  "start_time": "2025-10-09T17:38:28.530Z",
  "upstream_cluster": "kube_microapp_frontend_80",
  "upstream_host": "10.68.3.29:8080"
}
```

URX = UpstreamRetryLimitExceeded (retries happened!)

11. Scale back up
```
kubectl scale deployment cartservice -n microapp --replicas=1
```


## Cleanup
To prepare your environment for the next part of the demo, which will be on Gloo Gateway v1 with Portal, destroy your cluster.

If you use the GKE config within this repo:

1. `cd` into the **ggv2/gke1** directory.

2. Destroy the cluster
```
terraform destroy --auto-approve
```

# Gloo Gateway V1 (Portal)
In this section, you will find the full configuration for setting up Gloo Gateway v1. The reason why v1 will be used is because Portal will not be GA until Gloo Gateway v2.2.

## Installation Gateway v1

### Helm

1. Set your Gloo License Key variable
```
export GLOO_GATEWAY_LICENSE_KEY=
```

2. Install Kubernetes Gateway API
```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
```

3. Add the Gloo Gateway v1 Enterprise Helm repo
```
helm repo add glooe https://storage.googleapis.com/gloo-ee-helm
helm repo update
```

4. Install GGv1 Enterprise

The installation also includes:
- Grafana
- Prometheus

```
helm upgrade --install -n gloo-system gloo glooe/gloo-ee \
--create-namespace \
--version 1.20.1 \
--set-string license_key=$GLOO_GATEWAY_LICENSE_KEY \
-f - <<EOF
gloo:
  gatewayProxies:
    gatewayProxy:
      disabled: true
  kubeGateway:
    enabled: true
    portal:
      enabled: true
  gloo:
    disableLeaderElection: true
gloo-fed:
  enabled: false
  glooFedApiserver:
    enable: false
grafana:
  defaultInstallationEnabled: false
observability:
  enabled: false
prometheus:
  enabled: false
EOF
```

5. Ensure that Gloo Gateway v1 is running as expected
```
kubectl get pods -n gloo-system
```

6. Retrieve the Gateway Class 
```
kubectl get gatewayclass gloo-gateway
```

## Portal

In this previous section, you installed Gloo Gateway v1. If you take a look at the Helm config, the `portal: true` was already added in, so you won't need to do any extra configuration for the installation of Portal.

### Installation Of Portal

1. Ensure Portal is up and operational
```
kubectl get pods -n gloo-system -l app=gateway-portal-web-server
```

You should see an output similar to the below:
```
NAME                                        READY   STATUS    RESTARTS   AGE
gateway-portal-web-server-c9c78db5b-dfpfm   1/1     Running   0          2m1s
```

If you have `glooctl` installed, you can also check to confirm that all implementations (xDS, Gateways, Proxies, Rate Limiting Server, etc.) is ready to go.

```
glooctl check

glooctl binary version (1.19.6) differs from server components (v1.20.1) by at least a minor version.
Consider running:
glooctl upgrade --release=v1.20.1
----------

Checking Deployments... OK
Checking Pods... OK
Checking Upstreams... OK
Checking UpstreamGroups... OK
Checking AuthConfigs... OK
Checking RateLimitConfigs... OK
Checking VirtualHostOptions... OK
Checking RouteOptions... OK
Checking Secrets... OK
Checking VirtualServices... OK
Checking Gateways... OK
Checking Proxies... OK
No active gateway-proxy pods exist in cluster
Checking xds metrics... OK
Checking rate limit server... OK

Detected Kubernetes Gateway integration!
Checking Kubernetes GatewayClasses... OK
Checking Kubernetes Gateways... OK
Checking Kubernetes HTTPRoutes... OK

Skipping Gloo Instance check -- Gloo Federation not detected.
No problems detected.
```

### Deploy Sample Apps

For this section, please use the docs below and deploy the **Tracks** sample app

https://docs.solo.io/gateway/latest/portal/tutorials/setup/#apps

### Create API Products

For this section, please use the **Tracks** section as that's the demo app you deployed in the previous section

https://docs.solo.io/gateway/latest/portal/tutorials/portal/

### Create A Portal

https://docs.solo.io/gateway/latest/portal/tutorials/apis/


## Helpful Docs

Based on the Evaluation Criteria doc, the following links will be of help for knowing that Gloo Gateway supports what you need.

1. JWT/Payload support: https://docs.solo.io/gateway/2.0.x/security/jwt/overview/
2. Load balancing support: https://docs.solo.io/gateway/2.0.x/traffic-management/session-affinity/loadbalancing/
3. Kubernetes CRD (API/object/kind) support: https://docs.solo.io/gateway/2.0.x/reference/api/gloo-gateway/
4. Gloo Mesh/Gloo Gateway working together in multi-cluster support: https://github.com/AdminTurnedDevOps/ambient-mesh-lite-demo/blob/main/multi-cluster/sampleapp-microdemo/setup.md
5. Canary deployments with Mirroring: https://docs.solo.io/gateway/2.0.x/resiliency/mirroring/