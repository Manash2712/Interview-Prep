# Breakdown of a Kubernetes Ingress Resource File

An **Ingress Resource** file is a configuration file written in YAML that acts as a **traffic router config**. It does not run as a software process itself; instead, it provides rules that your running Ingress Controller reads and executes.

Here are the main structural components of the file and exactly what they do:

---

## 1. The API & Type Declarations
These fields define the schema version and tell Kubernetes exactly what kind of configuration object you are creating.

*   **`apiVersion: networking.k8s.io/v1`**: Specifies the stable networking API version required to interpret modern ingress structures.
*   **`kind: Ingress`**: Tells the cluster control plane to create an Ingress rule object rather than a standard deployment or pod.

---

## 2. Metadata & Configuration Annotations
Metadata uniquely identifies the resource within your cluster, while annotations act as advanced configuration switches for the controller.

*   **`metadata.name`**: The unique administrative name used to manage or delete this specific routing file via `kubectl`.
*   **`metadata.namespace`**: Dictates which cluster isolation boundary this file belongs to. An Ingress resource can *only* route traffic to backend services sitting within its same namespace.
*   **`annotations`**: Key-value pairs used to inject custom behavior directly into the routing engine without changing the core Kubernetes schema.
    *   `kubernetes.io/ingress.class: "nginx"`: Signals to the cluster that the NGINX controller should pick up and execute this file (crucial if you run multiple controllers like Traefik and NGINX side-by-side).
    *   `nginx.ingress.kubernetes.io/ssl-redirect: "true"`: Instructs the NGINX controller to dynamically intercept unencrypted HTTP traffic and force an HTTP `308` redirect to secure HTTPS.

---

## 3. The Spec (Routing Blueprints)
The `spec` block contains the actual operational blueprint for how incoming internet requests are parsed and directed.

*   **`rules`**: A list container holding all routing configurations. You can host multiple separate domains inside a single ingress file by adding multiple items under this block.
*   **`host: ://yourcompany.com`**: The exact domain name matched against the incoming HTTP `Host` header. If a user types a different domain that resolves to the same load balancer, the engine rejects the traffic.

---

## 4. The Path & Target Mapping
This section acts as the final traffic intersection, mapping URL endpoints directly to your internal cluster applications.

*   **`paths`**: A list of URL endpoints evaluated from top to bottom.
*   **`path: /`**: The literal URL string path string matched against the user's browser request.
*   **`pathType: Prefix`**: Defines the matching logic. `Prefix` means that if you match `/`, it will automatically match `/dashboard`, `/profile`, or any downstream sub-paths unless a more specific rule exists.
*   **`backend`**: The ultimate destination block.
    *   `service.name: frontend-service`: The precise internal Kubernetes `ClusterIP` service name where the traffic must be sent.
    *   `port.number: 80`: The exact target port configured on that internal service.

