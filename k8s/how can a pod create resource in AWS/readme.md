# Kubernetes Pod Creates AWS Load Balancer with AWS Load Balancer Controller

A pod inside a Kubernetes cluster creates an AWS Load Balancer by using the AWS Load Balancer Controller [1]. The pod interacts with the Kubernetes API server to watch for resources, and then uses official AWS API calls to provision infrastructure in your AWS account.

For this process to work securely without hardcoding dangerous AWS access keys inside your cluster, Kubernetes relies on a feature called IRSA (IAM Roles for Service Accounts).

## Step-by-Step Provisioning Flow [DevOps Engineer]

```
       │ (1. Applies YAML)
       ▼
[ K8s API Server ] ◄──(2. Watches Events)── [ AWS Load Balancer Controller Pod ]
                                                    │
                                                    │ (3. Assumes IAM Role via OIDC)
                                                    ▼
                                            [ AWS API (ELBv2) ]
                                                    │
                                                    │ (4. Provisions)
                                                    ▼
                                          [ AWS Application/Network LB ]
```

### 1. The Trigger (User Action)
A DevOps engineer applies a Kubernetes manifest to the cluster. This manifest is either a Service of type: `LoadBalancer` or an `Ingress` resource with the class set to `alb` [1].

### 2. The Watcher (Controller Pod)
The AWS Load Balancer Controller runs as a standard pod inside your cluster [1]. It constantly listens to the Kubernetes API server for these specific resource events. When it detects your new Service or Ingress, it intercepts the request and parses your configuration annotations (like subnets, security groups, or SSL certificates).

### 3. The Authentication (OIDC & IRSA Handshake)
To create infrastructure on AWS, the controller pod needs AWS permissions. It securely proves its identity using an OIDC Provider link between AWS IAM and your EKS cluster:
- The pod is assigned a specific Kubernetes ServiceAccount.
- This ServiceAccount is mapped to a secure AWS IAM Role via an annotation.
- The pod presents a temporary Kubernetes token to AWS STS (Security Token Service).
- AWS verifies the token against the cluster's OIDC connector and returns short-lived, temporary AWS credentials to the pod.

### 4. The Creation (AWS API Call)
Now armed with valid, temporary AWS credentials, the controller pod makes an outbound HTTPS request directly to the AWS ELBv2 API endpoints. It instructs AWS to spin up a physical Classic, Network, or Application Load Balancer inside your VPC, configuring its listeners, target groups, and health checks to match your deployment.

### 5. The Binding (Status Update)
Once AWS finishes provisioning the load balancer, it passes the external DNS URL back to the controller pod. The pod updates the Kubernetes API server, filling in the `status.loadBalancer.ingress` field of your Service or Ingress so you can see the address via `kubectl get svc`.

## Essential Prerequisites to Make This Work
If you want to set this up in an **AWS EKS** cluster, you must configure three components:

1. **Enable the OIDC Provider:**
you must register your EKS cluster as an OIDC identity provider in your **AWS IAM console** so that **AWS trusts** your cluster's pods.
2. **Create an IAM Policy & Role:** Create an IAM role containing specific permissions to create ELBs, target groups, and security groups; establish a trust relationship with your controller's Kubernetes ServiceAccount.
3. **Deploy the Helm Chart:** Install the official `aws-load-balancer-controller` via Helm inside your cluster,
passing it your cluster name and the created ServiceAccount.

