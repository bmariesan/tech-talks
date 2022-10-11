## Prerequisites
- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [helm v3](https://helm.sh/)

## Slides
- [Securing K8s APIs with Emissary Ingress](Securing%20K8s%20APIs%20with%20Emissary%20Ingress.pdf)

## Minikube setup
### Startup
``` shell
minikube start --driver hyperkit
```

### Dashboard
``` shell
minikube dashboard
```

### Setup Minikube Ingress
``` shell
minikube addons enable ingress
```

## Kind, K3S
If you want you can surely use any other local cluster K8s tooling

## Installing Emissary Ingress
### Add the Repo:
``` shell
helm repo add datawire https://app.getambassador.io
helm repo update
```

### Create Namespace and Install:
``` shell
kubectl create namespace emissary && \
kubectl apply -f https://app.getambassador.io/yaml/emissary/3.2.0/emissary-crds.yaml
kubectl wait --timeout=90s --for=condition=available deployment emissary-apiext -n emissary-system
helm install emissary-ingress --namespace emissary datawire/emissary-ingress && \
kubectl -n emissary wait --for condition=available --timeout=90s deploy -lapp.kubernetes.io/instance=emissary-ingress
```

### Enable Http traffic for Emissary
``` shell
kubectl apply -f emissary-config/listener.yaml
```

If you want to access Emissary directly, you'd have to either use `minikube tunnel` if you
leave the default service type of `LoadBalancer` or if you switch them to `NodePort` you
can get individual tunnels for each service by running:
``` shell
minikube service emissary-ingress-admin --url -n emissary
```
which outputs something similar to
http://192.168.64.6:30494

after which in your browser you'll be able to interact with the basic admin console 
at http://192.168.64.6:30494/ambassador/v0/diag

## Emissary example mappings

### Table of contents

1. [Simple mapping](#1-simple-mapping)
2. [CQRS](#2-cqrs)
3. [Circuit breakers](#3-circuit-breakers)
4. [Canary Releases](#4-canary-releases)
5. [Automatic retries](#5-automatic-retries)
6. [Traffic Shadowing](#6-traffic-shadowing)
7. [Authentication](#7-authentication)
8. [Rate Limiting](#8-rate-limiting)
9. [Distributed Tracing](#9-distributed-tracing)
10. [Metrics](#10-metrics)
11. [Other protocols](#11-grpc-websockets-http3-etc)
12. [Query param based routing](#12-query-param-based-routing)

### 1. Simple mapping

Before exposing any mappings we must tell Emissary to listen to traffic using a `Listener` resource

``` shell
kubectl apply -f emissary-config/1_listener.yaml 

---
apiVersion: getambassador.io/v3alpha1
kind: Listener
metadata:
  name: emissary-ingress-listener-8080
  namespace: emissary
spec:
  port: 8080
  protocol: HTTP
  securityModel: XFP
  hostBinding:
    namespace:
      from: ALL
```

Next step we deploy a dummy service to demo the functionalities:

``` shell
kubectl apply -f emissary-config/1_sample-service.yaml 
```

After which we configure a `Mapping` resource to expose the service via the Api Gateway:
``` shell
kubectl apply -f emissary-config/1_sample-service-mapping.yaml 

---
apiVersion: getambassador.io/v3alpha1
kind: Mapping
metadata:
  name: quote-backend
spec:
  hostname: "*"
  prefix: /backend/
  service: quote
  docs:
    path: "/.ambassador-internal/openapi-docs"
```

If everything is configured correctly you should see a response similar to:
```shell
curl -v --request GET http://192.168.64.6:30344/backend/                                                                             ok | 12:11:24 

*   Trying 192.168.64.6:30344...
* Connected to 192.168.64.6 (192.168.64.6) port 30344 (#0)
> GET /backend/ HTTP/1.1
> Host: 192.168.64.6:30344
> User-Agent: curl/7.79.1
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-type: application/json
< date: Wed, 05 Oct 2022 09:11:40 GMT
< content-length: 149
< x-envoy-upstream-service-time: 0
< server: envoy
< 
{
    "server": "voluminous-tangerine-n8hzsbvk",
    "quote": "A late night does not make any sense.",
    "time": "2022-10-05T09:11:40.682910512Z"
* Connection #0 to host 192.168.64.6 left intact
}%                                                            
```

Looking at the response headers `server: envoy` tells us that indeed this 
request was served by Envoy (in this case as wrapped by Emissary-Ingress)

### 2. CQRS

Besides simple mappings, it is important to control access to routes/services
at a more granular level. For example on the same route we could simulate an
implementation of the [CQRS pattern](https://martinfowler.com/bliki/CQRS.html) having `GET`
requests forwarded to a service, while `PUT` going to a totally different one.

First step let's apply the mappings:
``` shell
kubectl apply -f emissary-config/2_cqrs-mapping.yaml

---
apiVersion: getambassador.io/v3alpha1
kind:  Mapping
metadata:
  name:  cqrs-get
spec:
  hostname: "*"
  prefix: /cqrs/
  method: GET
  service: quote
---
apiVersion: getambassador.io/v3alpha1
kind:  Mapping
metadata:
  name:  cqrs-put
spec:
  hostname: "*"
  prefix: /cqrs/
  rewrite: /put
  method: PUT
  service: https://httpbin.org
```

Looking at the mappings above, any `GET` requests on the `/cqrs/` path will hit the `quote` service
while `PUT` requests on the same path will reach `httpbin.org`

`GET` result:
``` shell
curl -v --request GET http://192.168.64.6:30344/cqrs/
                                                                                 ok | 12:19:26 
*   Trying 192.168.64.6:30344...
* Connected to 192.168.64.6 (192.168.64.6) port 30344 (#0)
> GET /cqrs/ HTTP/1.1
> Host: 192.168.64.6:30344
> User-Agent: curl/7.79.1
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-type: application/json
< date: Wed, 05 Oct 2022 09:19:28 GMT
< content-length: 143
< x-envoy-upstream-service-time: 0
< server: envoy
< 
{
    "server": "voluminous-tangerine-n8hzsbvk",
    "quote": "668: The Neighbor of the Beast.",
    "time": "2022-10-05T09:19:28.382702546Z"
* Connection #0 to host 192.168.64.6 left intact
}%                                                                     
```

And our `PUT`:
``` shell
curl -v --request PUT http://192.168.64.6:30344/cqrs/       
                                                                         ok | 12:21:07 
*   Trying 192.168.64.6:30344...
* Connected to 192.168.64.6 (192.168.64.6) port 30344 (#0)
> PUT /cqrs/ HTTP/1.1
> Host: 192.168.64.6:30344
> User-Agent: curl/7.79.1
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< date: Wed, 05 Oct 2022 09:21:32 GMT
< content-type: application/json
< content-length: 480
< server: envoy
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 473
< 
{
  "args": {}, 
  "data": "", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept": "*/*", 
    "Content-Length": "0", 
    "Host": "192.168.64.6", 
    "User-Agent": "curl/7.79.1", 
    "X-Amzn-Trace-Id": "Root=1-633d4c9c-097f062911a41da67ce9d4b7", 
    "X-Envoy-Expected-Rq-Timeout-Ms": "3000", 
    "X-Envoy-Internal": "true", 
    "X-Envoy-Original-Path": "/cqrs/"
  }, 
  "json": null, 
  "origin": "172.17.0.1, 188.24.11.166", 
  "url": "https://192.168.64.6/put"
}
* Connection #0 to host 192.168.64.6 left intact
```

### 3. Circuit breakers

Another cool trick we could do is to prevent request overflow to a specific service
by using [Circuit Breakers](https://martinfowler.com/bliki/CircuitBreaker.html)

What does that mean exactly? Let's say we know we have a service that supports only `1 request in parallel`.
Circuit breakers allow us to define a `capacity` for a service, without considering any
extra information => like what a rate limiter might use to determine if traffic should stop.

Furthermore, circuit breakers can stop traffic from hitting a specific service even though the 
rate limiter might say you are allowed to.

Again first step, let's define a mapping with circuit breakers:
``` shell
kubectl apply -f emissary-config/3_circuit-breaker-mapping.yaml

---
apiVersion: getambassador.io/v3alpha1
kind:  Mapping
metadata:
  name:  circuit-breaker
spec:
  hostname: "*"
  prefix: /breaker/
  method: GET
  service: quote
  circuit_breakers:
    - priority: default
      max_connections: 1
      max_pending_requests: 1
      max_requests: 1
      max_retries: 5 
```

And now let's be cheesy and simulate more traffic than the capacity of our service:
``` shell
 xargs -I % -P 5  curl -I --request GET http://CLUSTER_IP:NODE_PORT/breaker/ < <(printf '%s\n' {1..10}) 
```

Expected result is a mix and match between 200 OK and 503 Service Unavailable

### 4. Canary Releases

How about if we have a shinny new service but even though we've done a ton of testing on it,
we're not yet 100% sure if it will behave correctly. Or perhaps we have a new release for our service
and want to make sure that we've configured it correctly.

The `Mapping` resource has a solution for this as it allows weighted traffic on the same path prefix.
Let's look at an example to make things clearer.

Starting with the mapping:
``` shell
kubectl apply -f emissary-config/4_canary-release-mapping.yaml

---
apiVersion: getambassador.io/v3alpha1
kind:  Mapping
metadata:
  name:  main-backend
spec:
  hostname: "*"
  prefix: /canary/
  service: quote
---
apiVersion: getambassador.io/v3alpha1
kind:  Mapping
metadata:
  name:  canary-backend
spec:
  hostname: "*"
  prefix: /canary/
  service: https://httpbin.org
  rewrite: /status/200
  weight: 50
```

In the mapping above we've allowed `50%` of the traffic to be routed to `httbin.org`

```shell
curl -v --request GET http://192.168.64.6:30344/canary/ 

# or more in parallel
xargs -I % -P 5  curl -I --request GET curl -v --request GET http://192.168.64.6:30344/canary/ < <(printf '%s\n' {1..10}) 
```

### 5. Automatic Retries

Ok we know how to define capacities for our services, let's think of a different scenario now.
How about if we have a legacy service that we know for sure is prone to fail randomly, we 
don't have the expertise to fix it, but we know that most of the time retrying will fix the problem.

`Mapping` to the rescue:
``` shell
kubectl apply -f emissary-config/5_automatic-retries-mapping.yaml

---
apiVersion: getambassador.io/v3alpha1
kind:  Mapping
metadata:
  name:  automatic-retries
spec:
  hostname: "*"
  prefix: /auto-retries/
  service: https://httpbin.org
  rewrite: /status/
  retry_policy:
    retry_on: "5xx"
    num_retries: 2
    per_try_timeout: 5s
```

A simple test we could do is to curl the mapping above and compare the response time
to what we get from `httpbin.org`:
``` shell
curl -o /dev/null -s -w 'Total: %{time_total}s\n' http://httpbin.org/status/500                                                      ok | 12:45:45 

Total: 0.233284s

# By comparison Emissary has to retry 2 times

 curl -o /dev/null -s -w 'Total: %{time_total}s\n' http://192.168.64.6:30344/auto-retries/500                                         ok | 12:46:08 

Total: 1.520014s
```

### 6. Traffic shadowing

We have a monolith, we build a microservice, we have no idea if they work the same,
can you help us `Mapping`?

Let's go:
``` shell
kubectl apply -f emissary-config/6_traffic-shadowing-mapping.yaml

---
apiVersion: getambassador.io/v3alpha1
kind:  Mapping
metadata:
  name:  monolith
spec:
  hostname: '*'
  prefix: /shadow/
  service: quote

---
apiVersion: getambassador.io/v3alpha1
kind:  Mapping
metadata:
  name:  microservice-shadow
spec:
  hostname: '*'
  prefix: /shadow/
  service: quote-cqrs
  shadow: true
```

### 7. Authentication

Security wise, the Api Gateways should excel at enforcing at the edge policies.
In not so fancy wording, their purpose is to deny any unwanted request long before
traffic reaches the actual service.

Let's apply the magic:
``` shell
kubectl apply -f emissary-config/7_auth_service.yaml

# Take into account that the ^ above will enforce auth for all services
# thus if we want to use services without authentication we need to 
# configure the Mapping resource slightly different

kubectl apply -f emissary-config/7_mapping_without_aut.yaml

---
apiVersion: getambassador.io/v3alpha1
kind:  Mapping
metadata:
  name:  bypass-auth-mapping
spec:
  hostname: '*'
  prefix: /no-auth/
  service: quote
  bypass_auth: true
```

The `AuthService` itself can be implemented in whatever programming language you desire.
The only limitation is that you have to adhere strictly to the `gRPC` defined interfaces
that are bundled with Envoy and configured in Emissary-Ingress.

At the time of writing this material `V2` is the current gRPC Api version, however Emissary
is in the middle of transitioning to `V3`.

### 8. Rate limiting

We now have a microservice that can scale up to `100000 req/sec`
(for sure it's written in Go or Rust, right?).

As you can imagine we could simply use circuit breakers to define the capacity of the service
to what our load tests tell us, right? 

Correct, but how about if we have a really greedy API user that wants to keep all that traffic on his plate.
You could say it's good for business as long as he pays, but having just 1 user gobble up everything? Come on...

Similarly to the `AuthService`, Envoy has an API capable of enforce rate limits based on whatever
criteria we decide.

``` shell
kubectl apply -f emissary-config/8_rate_limit.yaml

---
apiVersion: getambassador.io/v3alpha1
kind: RateLimitService
metadata:
  name: ratelimit
spec:
  service: 'example-rate-limit.default:5000'
  protocol_version: v3
  failure_mode_deny: false # when set to true envoy will return 500 error when unable to communicate with RateLimitService
```

Again it can be built in whatever programming language you choose to as long as it adheres to the gRPC Api for rate limiting.

Additionally, you can add custom labels or even pass on headers from the upstream to help you decide:

``` shell
kubectl apply -f emissary-config/8_rate_limit_mapping.yaml

---
apiVersion: getambassador.io/v3alpha1
kind:  Mapping
metadata:
  name:  rate-limited-mapping
spec:
  hostname: '*'
  prefix: /rate-limited/
  service: quote
  labels:
    emissary:
      - request_label_header:
        - path:
            header: ":path"
            omit_if_not_present: false
      - request_label_header:
        - api_key:
            header: "Authorization"
            omit_if_not_present: true
```

### 9. Distributed tracing

`Mapping` has nothing to do with tracing, but debugging microservices has.
Not much to say on this topic, apply the `Zipkin` config, run a few requests and
visualise the traces.

``` shell
kubectl apply -f emissary-config/9_distributed_tracing.yaml
```

### 10. Metrics

I love metrics, and I'm sure you do too and yes we have them `Prometheus` compatible!

First let's expose the `/metrics` path as otherwise it won't be visible outside the cluster:
``` shell
kubectl apply -f emissary-config/10_metrics.yaml

---
apiVersion: getambassador.io/v3alpha1
kind: Mapping
metadata:
  name: metrics
spec:
  hostname: "*"
  prefix: /metrics
  rewrite: ""
  service: localhost:8877
```

### 11. gRPC, Websockets, Http3 etc.

gRPC? Sure thing we can:
``` shell
apiVersion: getambassador.io/v3alpha1
kind: Mapping
metadata:
  name: grpc-py
spec:
  hostname: "*"
  grpc: True
  prefix: /helloworld.Greeter/
  rewrite: /helloworld.Greeter/
  service: grpc-example
```

Websockets here we go:
``` shell
---
apiVersion: getambassador.io/v3alpha1
kind: Mapping
metadata:
  name: my-service-mapping
spec:
  hostname: "*"
  prefix: /my-service/
  service: my-service
  allow_upgrade:
    - websocket
```

Http/3? => [Http/3 docs](https://www.getambassador.io/docs/emissary/latest/topics/running/http3/)

### 12. Query param based routing

Another neat trick we could do with `Mapping` is to also route based on query params as part of the path. 

But you might ask - "what's so cool about doing this?" - so let's imagine a scenario:
*Given an analytics platform, we want to ensure high throughput for `GET` type calls to retrieve data for the last `24 hours`.
All data older than that should be retrieved paginated in queries capable of returning up to `1000 items per page`.*

Looking at the scenario above its clear that any of the `1000 item pe page` queries will for sure take longer time to run
when compared to what we consider as `high throughput queries to get aggregated data for hte last 24 hours`. Traditionally
in some frameworks or programming languages we could have different data sources and pools for the two and split the traffic,
however they'd still share some resources.

This is where using `query param based routing` could help us split traffic before it reaches our service, meaning that we
could literally have throughput and long-running query deployments and split traffic at the edge in `Emissary-Ingress`:

``` shell
---
apiVersion: getambassador.io/v3alpha1
kind:  Mapping
metadata:
  name:  quote-mode
spec:
  prefix: /query-param-routing/
  service: quote-throughput-service

---
apiVersion: getambassador.io/v3alpha1
kind:  Mapping
metadata:
  name:  quote-mode
spec:
  prefix: /query-param-routing/
  service: quote-slow-queries-service
  query_parameters:
    page_size: 1000
```

The split can be done on one or multiple params but the result is the same.

## Topics not covered
- [TLS Termination](https://www.getambassador.io/docs/edge-stack/latest/howtos/tls-termination/#tls-termination-and-enabling-https)
- [L7/L4 load balancing](https://www.getambassador.io/docs/emissary/latest/howtos/configure-communications/)
- [Remote debugging with Telepresence](https://www.telepresence.io/docs/v1/discussion/overview/)

## References
- [Emissary-Ingress official docs](https://www.getambassador.io/docs/emissary/latest/tutorials/getting-started/)
