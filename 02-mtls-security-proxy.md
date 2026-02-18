# Decision 2: Implementing a Service Mesh Proxy Sidecar for Network Abstraction (MFU Digital Campus Scenario)

## Context

In a distributed microservices architecture, core business logic typically represents only a fraction of the necessary operational codebase. Every individual service, regardless of its primary domain function, must integrate auxiliary capabilities to ensure reliability and security. Authentication and authorization mechanisms, such as **Open Policy Agent (OPA)** policy enforcement, must be uniformly applied. Resiliency patterns, including advanced **rate limiting** and **circuit breaking**, are mandatory for maintaining overall system stability. Furthermore, security mandates strict encryption of data in transit, often requiring **mutual TLS (mTLS)** between all intra-cluster communications to satisfy stringent enterprise compliance and regulatory frameworks.

Historically, tightly coupling infrastructure code (like cryptographic libraries) with business logic drastically compromises independent deployability. An urgent security patch discovered in an embedded cryptographic library necessitates the recompilation, comprehensive testing, and simultaneous redeployment of the entire application suite. Alternatively, routing all traffic through a single, monolithic API gateway introduces severe network latency penalties and creates single points of failure.

Specifically, within our current distributed ecosystem, we are observing a critical friction point: **the unreliability and insecurity of inter-service network communication**. In the MFU Digital Campus scenario, services such as the student portal, digital library, attendance tracking, and finance billing systems are implemented by different teams and frequently fail to consistently implement mutual TLS (mTLS), retry logic, and circuit breaking. This leads to cascading failures during transient network partitions and violations of internal zero-trust compliance mandates.

---

## Decision

To resolve this, the adopted architectural solution is the implementation of a **Service Mesh Proxy Sidecar for Network Abstraction**. The sidecar pattern provides strict operational isolation and functional encapsulation by deploying a helper container alongside the primary application. Because all containers within a single Kubernetes Pod share the same execution environment and local network namespace, the sidecar and the primary application can communicate via the local loopback interface (`localhost`).

To enforce zero-trust security and guarantee network resiliency, we are deploying an advanced **Layer 7 network proxy** (specifically **Envoy**, orchestrated via the **Istio** service mesh) as a sidecar alongside every microservice. All inbound and outbound network traffic destined for the primary application will be transparently intercepted by this proxy sidecar. The primary application will issue unencrypted, standard HTTP requests directed simply at `localhost`. The Envoy sidecar intercepts this traffic, applies sophisticated routing rules, encrypts the payload using automatically rotated mutual TLS (mTLS) certificates, implements dynamic rate limiting and circuit breaking, and forwards the secure transmission to the destination service's corresponding sidecar proxy. This fundamentally removes all network reliability and cryptographic logic from the domain of the application developer.

---

## Rationale

Platform architects evaluated the **embedded library approach** and the **remote service approach** against the **sidecar model**.

### Evaluating Alternative Approaches

Attempting to provide resilient network retries or mTLS via centralized libraries creates brittle interdependence and mandates technological homogeneity. Furthermore, offloading these cross-cutting concerns to centralized remote services introduces severe network latency. If every microservice must make a synchronous network hop to a remote authorization server and another remote hop to an encryption service, the cumulative latency renders the system unusable for high-throughput business transactions. The Sidecar pattern elegantly resolves this. Physical proximity ensures that the latency overhead is measured in low single-digit milliseconds or microseconds, practically negating the performance penalty of out-of-process execution.

### Architectural Comparison Synthesis Matrix

| Architectural Dimension | Embedded Library Model | Remote Service Model | Sidecar Pattern Model |
|---|---|---|---|
| **Fault Isolation** | Poor. Co-mingled process space means library crashes or memory leaks will fatally terminate the parent application. | High. Complete physical and network isolation prevents cascading application crashes. | High. A separate container isolates runtime failures, preventing sidecar crashes from downing the main app. |
| **Language Dependency** | Dependent. Requires rewriting, testing, and maintaining tools for every supported programming language. | Independent. Communicates via standard network protocols irrespective of application language. | Independent. Operates alongside any application runtime, allowing platform teams to use optimized languages (e.g., C++/Rust). |
| **Execution Latency** | Minimal. In-memory function calls incur almost zero overhead. | High. Requires traversing the external network stack, susceptible to network congestion. | Low. IPC or local loopback interaction bypasses external networks, keeping latency extremely marginal. |
| **Lifecycle Management** | Tightly coupled. Updates require application recompilation, testing, and full redeployment. | Decoupled. Services update entirely independently of the application workloads. | Shared lifecycle. Scales seamlessly and deploys 1:1 with the application instance automatically. |

### Specific Rationale for the Service Mesh Proxy Sidecar

The rationale for adopting the Envoy proxy sidecar is driven by the absolute necessity for **transparent, zero-trust security**. Attempting to enforce mTLS manually via libraries requires developers to correctly manage, rotate, and securely store cryptographic certificates within their application code—a highly error-prone practice. The Envoy sidecar offloads this completely; the proxy automatically provisions and rotates identities via the Istio control plane. Because the sidecar runs with elevated network privileges, it transparently hijacks traffic, executing retries on transient failures and applying intelligent load-balancing algorithms without any developer intervention.

---

## Consequences

### Pros – What becomes easier?

- **Transparent Network Security and Zero-Trust Compliance:** The primary application sends plain-text HTTP traffic to the local loopback interface. The sidecar proxy encrypts it using automatically rotated cryptographic identities and routes it to the destination sidecar, which verifies the policy and decrypts the payload. This architecture guarantees strict policy enforcement, instantly bringing legacy applications into compliance.

- **Advanced Traffic Management and Resiliency:** Features such as automatic retries with exponential backoff, timeout management, circuit breaking, and rate limiting are handled entirely by the sidecar. Platform operators can execute dynamic configuration changes (e.g., shifting traffic for canary deployments) without disrupting primary application containers.

### Cons – What becomes more difficult?

- **Massive Resource Footprint:** The memory and CPU overhead scales linearly with the number of application pods. Empirical benchmarking for an Envoy proxy processing 1000 HTTP requests per second indicates an overhead of **~0.20 vCPU and 60 MB of memory per sidecar**. In densely packed clusters, this becomes a massive financial burden.

- **Latency Accumulation in "Chatty" Architectures:** Every network request traverses two additional out-of-process network hops. Each proxy adds CPU cycles for TLS termination and routing evaluations, typically introducing **1 to 5 milliseconds per hop**. In deep microservice call chains, this latency aggregates exponentially, potentially rendering the sidecar pattern unsuitable for sub-millisecond real-time platforms.

- **The Ambient Mesh Evolution:** As a direct consequence of the severe resource overhead (0.20 vCPU and 60 MB per pod), the industry is evolving toward **"Sidecar-less" or Ambient Service Meshes**. Ambient mode utilizes a lightweight node-level "ztunnel" (consuming only ~0.06 vCPU and 12 MB memory) strictly for Layer 4 routing and mTLS, decoupling heavy Layer 7 processing. While Ambient Mesh resolves cost bottlenecks, the traditional sidecar remains the most secure solution for highly regulated environments requiring absolute cryptographic isolation.

---

## Sample Code

**Implementing a Service Mesh Proxy Sidecar (Envoy)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-profile-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-profile
  template:
    metadata:
      labels:
        app: user-profile
    spec:
      initContainers:
        # Step 1: Network Traffic Interception Setup
        # This standard init container runs once to completion. It uses iptables to 
        # transparently hijack all pod traffic and redirect it to the Envoy sidecar.
        - name: traffic-interception-setup
          image: alpine:latest
          command: ["/bin/sh", "-c"]
          args:
            - >
              apk add iptables;
              iptables -t nat -A OUTPUT -p tcp -j REDIRECT --to-port 15001;
              iptables -t nat -A INPUT -p tcp -j REDIRECT --to-port 15006
          securityContext:
            capabilities:
              add:
                - NET_ADMIN

      containers:
        # Step 2: Core Application Service
        # The developer deploys this service completely agnostic of mTLS or retries.
        - name: user-profile-app
          image: enterprise/user-profile-api:v1.0
          ports:
            - containerPort: 3000

        # Step 3: Envoy Proxy Sidecar
        # This container shares the network namespace, receives the intercepted
        # traffic, applies L7 rules, and manages mTLS encryption.
        - name: envoy-proxy-sidecar
          image: envoyproxy/envoy:v1.24.0
          ports:
            - containerPort: 15001  # Outbound interception listener
            - containerPort: 15006  # Inbound interception listener
            - containerPort: 8080   # Application traffic listener
            - containerPort: 9901   # Envoy Admin and telemetry interface
          volumeMounts:
            - name: envoy-config-volume
              mountPath: /etc/envoy

      volumes:
        - name: envoy-config-volume
          configMap:
            name: envoy-sidecar-config
```
