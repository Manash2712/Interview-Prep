# Deploying an NGINX Ingress Controller and Routing Rules

To expose your applications to the outside world using an Ingress Controller, you must deploy two separate components: the **Ingress Controller Deployment** (the routing engine) and an **Ingress Resource** (the routing rules for your specific applications).

---

## 1. Deploy the Ingress Controller Engine

Instead of writing a massive deployment file from scratch, you can use the production-ready installation manifest maintained directly by the Kubernetes community.

Run the following command to deploy the official NGINX Ingress Controller pods, configurations, and its required `type: LoadBalancer` service:

```bash
kubectl apply -f https://githubusercontent.com
```

### What This Creates Behind the Scenes:
* A dedicated namespace called `ingress-nginx`.
* The **Ingress Controller Deployment** containing the NGINX software pods.
* A Kubernetes **Service of `type: LoadBalancer`** that instructs AWS, Azure, or GCP to provision exactly *one* external, public-facing cloud load balancer to act as your cluster's entry point.

---

## 2. Write Your Ingress Resource (The Routing Rules)

Once the engine is running inside your cluster, you write a small `Ingress` rule file. This file tells the NGINX controller how to parse internet domain names and URL paths, and where to drop that traffic inside your internal cluster.

Save the following file as `app-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  namespace: default
  annotations:
    # Signals to the cluster that the NGINX controller should execute this file
    kubernetes.io/ingress.class: "nginx"
    
    # Automatically intercepts unencrypted HTTP traffic and forces an HTTPS redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
  # 1. Match the incoming domain name typed by the user
  - host: ://yourcompany.com
    http:
      paths:
      # 2. Match the root URL path context
      - path: /
        pathType: Prefix
        backend:
          service:
            # 3. Forward the traffic directly to your internal ClusterIP service name
            name: frontend-service
            port:
              # 4. Target the specific port exposed by that internal service
              number: 80
```

---

## 3. Apply, Verify, and Route Your Traffic

### Step A: Apply the Configuration
Send your newly written routing rules to the Kubernetes API server:
```bash
kubectl apply -f app-ingress.yaml
```

### Step B: Fetch the External Address
Watch your resource status to see when the cloud provider finishes setting up your external load balancer connection:
```bash
kubectl get ingress frontend-ingress -w
```
After roughly 60 seconds, the `ADDRESS` column will populate with a public IPv4 address or an AWS Elastic Load Balancer (ELB) DNS string (e.g., `://amazonaws.com`).

### Step C: Update Your DNS Records
To make the setup fully functional, log into your DNS registrar (like Route 53, Cloudflare, or GoDaddy) and create a **CNAME record** mapping your domain name (`://yourcompany.com`) directly to the address fetched in Step B. 

Once the DNS propagates, all public traffic to your URL will seamlessly flow through your cloud load balancer, hit your internal Ingress Controller pod, and arrive safely at your frontend application pods.
