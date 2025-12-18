# Kubernetes Probe Support in Mainstream API Gateways
## Comprehensive Analysis and Implementation Guide

---

## Executive Summary

This document provides a comprehensive analysis of Kubernetes probe support across mainstream API Gateway implementations. The analysis focuses on liveness, readiness, and startup probes, as well as health check capabilities for RESTful and WebSocket protocols.

**Last Updated**: December 2025  
**Document Version**: 1.0

---

## Comparison Matrix

| API Gateway | Liveness Probe | Readiness Probe | Startup Probe | RESTful Health Check | WebSocket Health Check | Pros | Cons |
|-------------|---------------|-----------------|---------------|---------------------|----------------------|------|------|
| **Kong Gateway** | ✅ Native | ✅ Native | ✅ Supported | ✅ Active + Passive | ⚠️ TCP/HTTP Only | • Dedicated endpoints (`/status`, `/status/ready`)<br>• Database connection validation<br>• Configuration state checking<br>• Well-documented | • `/status` endpoint can be resource-intensive with large databases<br>• No specialized WebSocket probe<br>• Readiness behavior varies by deployment mode |
| **Apache APISIX** | ✅ Native | ✅ Native | ✅ Supported | ✅ Active + Passive | ⚠️ TCP/HTTP Only | • Control API for health status queries<br>• Lazy-loading health checks<br>• Custom headers support<br>• Granular health counters | • Requires explicit configuration<br>• Limited upstream WebSocket-specific health checks<br>• Documentation primarily in Chinese |
| **Traefik** | ✅ Native | ✅ Native | ✅ Supported | ⚠️ Relies on K8s | ⚠️ Relies on K8s | • Built-in `/ping` endpoint<br>• Automatic integration with K8s services<br>• Zero configuration required<br>• Graceful shutdown support | • No active upstream health checking in K8s mode<br>• Relies entirely on K8s service discovery<br>• Limited customization options |
| **Spring Cloud Gateway** | ✅ Native (Actuator) | ✅ Native (Actuator) | ✅ Auto-detected | ✅ Actuator-based | ⚠️ Standard HTTP | • Mature Actuator ecosystem<br>• Automatic Kubernetes detection<br>• Extensive customization<br>• Rich health indicator framework | • Requires Spring Boot runtime<br>• Higher resource consumption (JVM)<br>• Steeper learning curve for non-Spring developers |
| **Envoy Proxy** | ✅ Native (Admin API) | ✅ Native (Admin API) | ✅ Supported | ✅ Multi-protocol | ✅ TCP Socket Check | • High-performance C++ implementation<br>• Outlier detection for passive checks<br>• Multi-protocol support (HTTP/TCP/gRPC)<br>• Advanced load balancing | • Complex configuration (Protobuf)<br>• Steeper learning curve<br>• Limited built-in WebSocket-specific features |
| **Nginx Plus / OSS** | ✅ Custom Endpoint | ✅ Custom Endpoint | ✅ Supported | ✅ Active (Plus only) | ⚠️ TCP Only | • Mature and stable<br>• High performance<br>• Extensive ecosystem | • Active health checks require Nginx Plus (commercial)<br>• Limited native Kubernetes integration<br>• Requires external configuration |

### Legend
- ✅ **Native/Full Support**: Built-in, production-ready implementation
- ⚠️ **Partial/Indirect Support**: Available but requires additional configuration or workarounds
- ❌ **Not Supported**: Feature not available

---

## Detailed Implementation Analysis

### 1. Kong API Gateway

#### Overview
Kong provides dedicated health check endpoints that align with Kubernetes probe patterns, offering both liveness and readiness checks with different semantics based on deployment mode.

#### Key Endpoints
- **Liveness**: `/status` - Checks if Kong process is running
- **Readiness**: `/status/ready` - Validates configuration and dependencies

#### GitHub Repository
- **Main Repository**: https://github.com/Kong/kong
- **Health Check Implementation**: `kong/api/routes/health.lua`
- **Health Check Library**: https://github.com/Kong/lua-resty-healthcheck

#### Key Code Reference

**Location**: `kong/api/routes/health.lua`

```lua
-- Liveness check - verifies Kong process is running
local function status_handler(self)
  return kong.response.exit(200, {
    database = {
      reachable = kong.db:is_ready()
    },
    server = {
      connections_accepted = ngx.var.connections_accepted,
      connections_active = ngx.var.connections_active,
      connections_handled = ngx.var.connections_handled,
      connections_reading = ngx.var.connections_reading,
      connections_waiting = ngx.var.connections_waiting,
      connections_writing = ngx.var.connections_writing,
      total_requests = ngx.var.connections_handled,
    }
  })
end

-- Readiness check - validates configuration is loaded and ready
local function ready_handler(self)
  local ok, err = kong.db:is_ready()
  if not ok then
    return kong.response.exit(503, {
      message = "Database connection failed: " .. err
    })
  end
  
  if not kong.configuration.loaded then
    return kong.response.exit(503, {
      message = "Configuration not loaded"
    })
  end
  
  return kong.response.exit(200, { status = "ready" })
end
```

#### Kubernetes Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kong-gateway
  namespace: kong
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kong-gateway
  template:
    metadata:
      labels:
        app: kong-gateway
    spec:
      containers:
      - name: kong
        image: kong/kong-gateway:3.8
        ports:
        - name: proxy
          containerPort: 8000
        - name: admin
          containerPort: 8001
        
        env:
        - name: KONG_DATABASE
          value: "postgres"
        - name: KONG_PG_HOST
          value: "postgres.kong.svc.cluster.local"
        - name: KONG_ADMIN_LISTEN
          value: "0.0.0.0:8001"
        
        # Startup Probe - allows time for database connection
        startupProbe:
          httpGet:
            path: /status
            port: 8001
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 30  # 150 seconds total
          timeoutSeconds: 3
        
        # Liveness Probe - checks if Kong process is alive
        livenessProbe:
          httpGet:
            path: /status
            port: 8001
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
          timeoutSeconds: 5
        
        # Readiness Probe - checks if Kong is ready to proxy traffic
        readinessProbe:
          httpGet:
            path: /status/ready
            port: 8001
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
          successThreshold: 1
          timeoutSeconds: 3
        
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
```

#### Upstream Health Check Configuration

```yaml
# Kong declarative configuration (kong.yaml)
_format_version: "3.0"

upstreams:
- name: user-service
  algorithm: round-robin
  slots: 10000
  
  # Active health checks (HTTP)
  healthchecks:
    active:
      type: http
      http_path: /health
      timeout: 3
      concurrency: 10
      healthy:
        interval: 5
        successes: 2
        http_statuses: [200, 302]
      unhealthy:
        interval: 3
        http_failures: 2
        tcp_failures: 1
        timeouts: 1
        http_statuses: [429, 500, 501, 502, 503, 504]
    
    # Passive health checks (Circuit Breaker)
    passive:
      type: http
      healthy:
        successes: 3
        http_statuses: [200, 201, 202, 203, 204, 205, 206, 207, 208, 226]
      unhealthy:
        http_failures: 5
        tcp_failures: 5
        timeouts: 5
        http_statuses: [429, 500, 503]
  
  targets:
  - target: user-service-1.default.svc.cluster.local:8080
    weight: 100
  - target: user-service-2.default.svc.cluster.local:8080
    weight: 100
```

**Documentation References**:
- Official Docs: https://docs.konghq.com/gateway/latest/production/monitoring/healthcheck-probes/
- Health Checks Guide: https://docs.konghq.com/gateway/latest/how-kong-works/health-checks/

---

### 2. Apache APISIX

#### Overview
APISIX provides health check endpoints and a Control API for querying health status. It implements both active and passive health checks with lazy-loading optimization.

#### Key Endpoints
- **Liveness**: `/healthz` (port 9080)
- **Readiness**: `/readyz` (port 9080)
- **Control API**: `/v1/healthcheck` (port 9090)

#### GitHub Repository
- **Main Repository**: https://github.com/apache/apisix
- **Health Check Documentation**: `docs/en/latest/tutorials/health-check.md`
- **Control API**: `apisix/control/v1.lua`

#### Key Code Reference

**Location**: `apisix/control/v1.lua`

```lua
-- Health check status query
local function get_health(conf)
    local healthcheck = require("resty.healthcheck")
    local upstreams = core.table.new(0, 4)
    
    for key, value in pairs(health_check.get_health_checkers()) do
        local upstream_name, checker_type = key:match("^(.+)#(.+)$")
        
        local nodes = {}
        for _, node in ipairs(checker.get_target_status()) do
            core.table.insert(nodes, {
                ip = node.ip,
                port = node.port,
                hostname = node.hostname,
                status = node.health and "healthy" or "unhealthy",
                counter = {
                    http_failure = node.counter.http_failures,
                    tcp_failure = node.counter.tcp_failures,
                    timeout_failure = node.counter.timeouts,
                    success = node.counter.successes
                }
            })
        end
        
        core.table.insert(upstreams, {
            name = upstream_name,
            type = checker_type,
            nodes = nodes
        })
    end
    
    return 200, upstreams
end
```

#### Kubernetes Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apisix
  namespace: apisix
spec:
  replicas: 3
  selector:
    matchLabels:
      app: apisix
  template:
    metadata:
      labels:
        app: apisix
    spec:
      containers:
      - name: apisix
        image: apache/apisix:3.11.0-debian
        ports:
        - name: http
          containerPort: 9080
        - name: https
          containerPort: 9443
        - name: control
          containerPort: 9090
        
        # Startup Probe
        startupProbe:
          httpGet:
            path: /healthz
            port: 9080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 30
          timeoutSeconds: 3
        
        # Liveness Probe
        livenessProbe:
          httpGet:
            path: /healthz
            port: 9080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
          timeoutSeconds: 5
        
        # Readiness Probe
        readinessProbe:
          httpGet:
            path: /readyz
            port: 9080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
          successThreshold: 1
          timeoutSeconds: 3
        
        volumeMounts:
        - name: config
          mountPath: /usr/local/apisix/conf/config.yaml
          subPath: config.yaml
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
      
      volumes:
      - name: config
        configMap:
          name: apisix-config
```

#### Upstream Health Check Configuration

```yaml
# APISIX Route with Health Check
routes:
  - uri: /api/users/*
    upstream:
      type: roundrobin
      nodes:
        "user-service-1.default.svc.cluster.local:8080": 1
        "user-service-2.default.svc.cluster.local:8080": 1
      
      # Active health checks
      checks:
        active:
          type: http
          http_path: /health
          timeout: 3
          healthy:
            interval: 2
            successes: 1
          unhealthy:
            interval: 1
            http_failures: 2
          req_headers: ["User-Agent: APISIX-HealthChecker"]
        
        # Passive health checks
        passive:
          type: http
          healthy:
            http_statuses: [200, 201]
            successes: 3
          unhealthy:
            http_statuses: [500]
            http_failures: 3
            tcp_failures: 3
```

**Documentation References**:
- Official Docs: https://apisix.apache.org/docs/apisix/tutorials/health-check/
- Control API: https://apisix.apache.org/docs/apisix/control-api/

---

### 3. Traefik

#### Overview
Traefik provides a built-in `/ping` endpoint for health checks and relies on Kubernetes native service discovery for upstream health checking.

#### Key Endpoints
- **Liveness/Readiness**: `/ping` (configurable port)

#### GitHub Repository
- **Main Repository**: https://github.com/traefik/traefik
- **Ping Handler**: `pkg/ping/handler.go`

#### Key Code Reference

**Location**: `pkg/ping/handler.go`

```go
package ping

import (
    "net/http"
)

// Handler is a simple health check handler that responds
// with 200 OK when Traefik is running normally, and 503
// during graceful shutdown.
type Handler struct {
    terminating bool
}

// ServeHTTP implements http.Handler
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if h.terminating {
        w.WriteHeader(http.StatusServiceUnavailable)
        return
    }
    
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("OK"))
}

// SetTerminating marks the handler as terminating
func (h *Handler) SetTerminating() {
    h.terminating = true
}
```

#### Kubernetes Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traefik
  namespace: traefik
spec:
  replicas: 3
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      containers:
      - name: traefik
        image: traefik:v3.2
        ports:
        - name: web
          containerPort: 80
        - name: websecure
          containerPort: 443
        - name: admin
          containerPort: 8080
        
        args:
        - --api.insecure=true
        - --ping=true
        - --ping.entrypoint=web
        - --providers.kubernetesingress
        - --entrypoints.web.address=:80
        - --entrypoints.websecure.address=:443
        
        # Startup Probe
        startupProbe:
          httpGet:
            path: /ping
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 30
          timeoutSeconds: 2
        
        # Liveness Probe
        livenessProbe:
          httpGet:
            path: /ping
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
          failureThreshold: 3
          timeoutSeconds: 2
        
        # Readiness Probe
        readinessProbe:
          httpGet:
            path: /ping
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
          successThreshold: 1
          timeoutSeconds: 2
        
        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 20"]
        
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      
      terminationGracePeriodSeconds: 60
```

#### Upstream Health Check Configuration

```yaml
# Note: Traefik relies on Kubernetes service endpoints
# for upstream health. Configure health checks on the
# backend pods themselves:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-service
spec:
  template:
    spec:
      containers:
      - name: backend
        image: backend:latest
        
        # Backend pod health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5

---
# Traefik IngressRoute (no additional health check needed)
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: backend-route
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`example.com`) && PathPrefix(`/api`)
    kind: Rule
    services:
    - name: backend-service
      port: 8080
```

**Documentation References**:
- Health Check Docs: https://doc.traefik.io/traefik/reference/install-configuration/observability/healthcheck/
- Kubernetes Provider: https://doc.traefik.io/traefik/providers/kubernetes-ingress/

---

### 4. Spring Cloud Gateway

#### Overview
Spring Cloud Gateway leverages Spring Boot Actuator to provide comprehensive health checks with automatic Kubernetes environment detection.

#### Key Endpoints
- **Liveness**: `/actuator/health/liveness`
- **Readiness**: `/actuator/health/readiness`
- **Health Overview**: `/actuator/health`

#### GitHub Repository
- **Main Repository**: https://github.com/spring-cloud/spring-cloud-gateway
- **Spring Boot Actuator**: https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator
- **Health Indicators**: `spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/`

#### Key Code Reference

**Location**: `LivenessStateHealthIndicator.java`

```java
package org.springframework.boot.actuate.health;

import org.springframework.boot.availability.ApplicationAvailability;
import org.springframework.boot.availability.LivenessState;

/**
 * A {@link HealthIndicator} that checks the {@link LivenessState} of the application.
 */
public class LivenessStateHealthIndicator extends AvailabilityStateHealthIndicator {

    public LivenessStateHealthIndicator(ApplicationAvailability availability) {
        super(availability, LivenessState.class, 
              (statusMappings) -> {
                  statusMappings.add(LivenessState.CORRECT, Status.UP);
                  statusMappings.add(LivenessState.BROKEN, Status.DOWN);
              });
    }
}
```

**Location**: `ReadinessStateHealthIndicator.java`

```java
package org.springframework.boot.actuate.health;

import org.springframework.boot.availability.ApplicationAvailability;
import org.springframework.boot.availability.ReadinessState;

/**
 * A {@link HealthIndicator} that checks the {@link ReadinessState} of the application.
 */
public class ReadinessStateHealthIndicator extends AvailabilityStateHealthIndicator {

    public ReadinessStateHealthIndicator(ApplicationAvailability availability) {
        super(availability, ReadinessState.class,
              (statusMappings) -> {
                  statusMappings.add(ReadinessState.ACCEPTING_TRAFFIC, Status.UP);
                  statusMappings.add(ReadinessState.REFUSING_TRAFFIC, Status.OUT_OF_SERVICE);
              });
    }
}
```

#### Kubernetes Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-cloud-gateway
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-cloud-gateway
  template:
    metadata:
      labels:
        app: spring-cloud-gateway
    spec:
      containers:
      - name: gateway
        image: spring-cloud-gateway:latest
        ports:
        - name: http
          containerPort: 8080
        - name: actuator
          containerPort: 8081
        
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "kubernetes"
        - name: MANAGEMENT_SERVER_PORT
          value: "8081"
        - name: MANAGEMENT_ENDPOINT_HEALTH_PROBES_ENABLED
          value: "true"
        
        # Startup Probe
        startupProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8081
          initialDelaySeconds: 20
          periodSeconds: 10
          failureThreshold: 30
          timeoutSeconds: 5
        
        # Liveness Probe
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8081
          initialDelaySeconds: 30
          periodSeconds: 15
          failureThreshold: 3
          timeoutSeconds: 5
        
        # Readiness Probe
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8081
          initialDelaySeconds: 10
          periodSeconds: 10
          failureThreshold: 3
          successThreshold: 1
          timeoutSeconds: 5
        
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
```

#### Application Configuration

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
      - id: user-service
        uri: lb://user-service
        predicates:
        - Path=/api/users/**
      
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true

# Enable health probes
management:
  server:
    port: 8081
  endpoints:
    web:
      exposure:
        include: health,info,metrics
      base-path: /actuator
  endpoint:
    health:
      probes:
        enabled: true
      show-details: always
      group:
        liveness:
          include: livenessState,ping
        readiness:
          include: readinessState,db,redis
  
  # Custom health indicators
  health:
    circuitbreakers:
      enabled: true
    ratelimiters:
      enabled: true

# Graceful shutdown
server:
  shutdown: graceful
  
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

**Documentation References**:
- Spring Boot Actuator: https://docs.spring.io/spring-boot/reference/actuator/endpoints.html
- Kubernetes Probes: https://www.baeldung.com/spring-liveness-readiness-probes
- Spring Cloud Gateway: https://docs.spring.io/spring-cloud-gateway/reference/

---

### 5. Envoy Proxy

#### Overview
Envoy provides comprehensive health checking capabilities through its admin interface and supports multiple protocols including HTTP, TCP, and gRPC.

#### Key Endpoints
- **Admin API**: `/server_info`, `/ready`, `/stats/prometheus`
- **Health Check Filter**: Configurable HTTP endpoints

#### GitHub Repository
- **Main Repository**: https://github.com/envoyproxy/envoy
- **Health Checker Implementation**: `source/common/upstream/health_checker_impl.cc`
- **Admin Interface**: `source/server/admin/`
- **Protobuf Definitions**: `api/envoy/config/core/v3/health_check.proto`

#### Key Code Reference

**Location**: `health_check.proto`

```protobuf
syntax = "proto3";

package envoy.config.core.v3;

message HealthCheck {
  // Interval between health checks
  google.protobuf.Duration interval = 1;
  
  // Timeout for health check
  google.protobuf.Duration timeout = 2;
  
  // Unhealthy threshold
  google.protobuf.UInt32Value unhealthy_threshold = 3;
  
  // Healthy threshold
  google.protobuf.UInt32Value healthy_threshold = 4;
  
  // Health check types
  oneof health_checker {
    HttpHealthCheck http_health_check = 8;
    TcpHealthCheck tcp_health_check = 9;
    GrpcHealthCheck grpc_health_check = 11;
  }
  
  // Interval for unhealthy hosts
  google.protobuf.Duration unhealthy_interval = 14;
  
  // Interval for healthy hosts
  google.protobuf.Duration unhealthy_edge_interval = 15;
  google.protobuf.Duration healthy_edge_interval = 16;
}

message HttpHealthCheck {
  // Path to check
  string path = 1;
  
  // Expected status codes
  repeated envoy.type.v3.Int64Range expected_statuses = 9;
  
  // Request headers to add
  repeated HeaderValueOption request_headers_to_add = 6;
}
```

#### Kubernetes Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: envoy-proxy
  namespace: envoy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: envoy-proxy
  template:
    metadata:
      labels:
        app: envoy-proxy
    spec:
      containers:
      - name: envoy
        image: envoyproxy/envoy:v1.31-latest
        ports:
        - name: http
          containerPort: 10000
        - name: admin
          containerPort: 9901
        
        # Startup Probe
        startupProbe:
          httpGet:
            path: /ready
            port: 9901
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 30
          timeoutSeconds: 3
        
        # Liveness Probe
        livenessProbe:
          httpGet:
            path: /server_info
            port: 9901
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
          timeoutSeconds: 5
        
        # Readiness Probe
        readinessProbe:
          httpGet:
            path: /ready
            port: 9901
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
          successThreshold: 1
          timeoutSeconds: 3
        
        volumeMounts:
        - name: config
          mountPath: /etc/envoy
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
      
      volumes:
      - name: config
        configMap:
          name: envoy-config
```

#### Envoy Configuration with Health Checks

```yaml
# envoy-config.yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 10000
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          http_filters:
          # Health check filter for Kubernetes probes
          - name: envoy.filters.http.health_check
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.health_check.v3.HealthCheck
              pass_through_mode: false
              headers:
              - name: ":path"
                string_match:
                  exact: "/healthz"
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: backend_service
  
  clusters:
  - name: backend_service
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    
    # HTTP Health Checks
    health_checks:
    - timeout: 1s
      interval: 5s
      unhealthy_threshold: 3
      healthy_threshold: 2
      http_health_check:
        path: /health
        expected_statuses:
        - start: 200
          end: 399
        request_headers_to_add:
        - header:
            key: "x-envoy-health-check"
            value: "true"
      
      # Different intervals for unhealthy hosts
      unhealthy_interval: 2s
      unhealthy_edge_interval: 1s
      healthy_edge_interval: 10s
    
    # Outlier Detection (Passive Health Check)
    outlier_detection:
      consecutive_5xx: 5
      interval: 10s
      base_ejection_time: 30s
      max_ejection_percent: 50
      enforcing_consecutive_5xx: 100
      
      # Local origin failures
      split_external_local_origin_errors: true
      consecutive_local_origin_failure: 5
      enforcing_consecutive_local_origin_failure: 100
      
      # Success rate outlier detection
      success_rate_minimum_hosts: 5
      success_rate_request_volume: 100
      success_rate_stdev_factor: 1900
      enforcing_success_rate: 100
    
    load_assignment:
      cluster_name: backend_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: backend-service.default.svc.cluster.local
                port_value: 8080

admin:
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9901
```

**Documentation References**:
- Health Checking: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/health_checking
- Health Check Proto: https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/health_check.proto
- Admin Interface: https://www.envoyproxy.io/docs/envoy/latest/operations/admin

---

## Key Observations and Recommendations

### WebSocket Health Check Considerations

**Important Finding**: None of the mainstream API Gateways implement specialized WebSocket health check probes. All use standard approaches:

1. **Gateway Self-Health**: Standard HTTP probes (liveness/readiness)
2. **Upstream WebSocket Services**: HTTP or TCP health checks
3. **Connection Management**: Application-level heartbeat mechanisms (Ping/Pong frames)

**Recommendation**: For WebSocket support:
- Use standard HTTP liveness/readiness probes for the gateway itself
- Implement HTTP health check endpoints on WebSocket backend services
- Use TCP socket checks as fallback for WebSocket-only services
- Implement application-level heartbeat (30-second Ping/Pong recommended)
- Configure appropriate idle timeouts (5-10 minutes typical)

### Startup Probe Best Practices

All modern API Gateways support startup probes, which should be configured with:
- Higher `failureThreshold` (20-30) to allow adequate startup time
- Shorter `periodSeconds` (5-10s) for faster detection once started
- Appropriate `initialDelaySeconds` based on application warm-up time

### Liveness vs Readiness Guidelines

| Probe Type | Should Check | Should NOT Check |
|------------|-------------|------------------|
| **Liveness** | • Process running<br>• Internal state valid<br>• No deadlocks | • External dependencies<br>• Database connections<br>• Downstream services |
| **Readiness** | • Configuration loaded<br>• Critical dependencies<br>• Ready to serve | • Non-critical dependencies<br>• Optional features |

### Performance Considerations

1. **Kong**: The `/status` endpoint can be resource-intensive with large databases. Consider using `/status/ready` for frequent checks.

2. **Spring Cloud Gateway**: JVM-based, higher resource consumption. Ensure adequate memory allocation.

3. **Envoy**: C++ implementation offers best performance but requires more complex configuration.

4. **Traefik**: Lightweight but relies on Kubernetes for upstream health, limiting control.

### Production Deployment Checklist

- [ ] Liveness probe configured (checks process health only)
- [ ] Readiness probe configured (checks critical dependencies)
- [ ] Startup probe configured (for slow-starting applications)
- [ ] Appropriate `initialDelaySeconds` set
- [ ] `periodSeconds` optimized (5-10s for readiness, 10-30s for liveness)
- [ ] `failureThreshold` prevents flapping (3 for liveness, 3 for readiness)
- [ ] `terminationGracePeriodSeconds` > longest request timeout
- [ ] `preStop` hook configured for graceful shutdown
- [ ] Upstream health checks configured (active + passive recommended)
- [ ] Monitoring and alerting on probe failures

---

## Conclusion

All mainstream API Gateways provide robust support for Kubernetes health check probes, with varying levels of sophistication:

- **Kong and APISIX** offer the most comprehensive upstream health checking with active and passive mechanisms
- **Spring Cloud Gateway** provides the richest health indicator ecosystem through Actuator
- **Envoy** delivers high-performance health checking with advanced outlier detection
- **Traefik** offers simplicity by leveraging Kubernetes native service discovery

The choice depends on your specific requirements:
- **Complex routing and health check requirements**: Kong or Envoy
- **Spring ecosystem integration**: Spring Cloud Gateway
- **Simplicity and cloud-native focus**: Traefik or APISIX
- **High performance and advanced features**: Envoy

For WebSocket applications, all solutions use standard HTTP/TCP health checks rather than specialized WebSocket probes, with application-level heartbeat mechanisms recommended for connection management.

---

**Document Maintained By**: Platform Engineering Team  
**Next Review Date**: March 2026  
**Feedback**: Please submit improvement suggestions via Confluence comments
