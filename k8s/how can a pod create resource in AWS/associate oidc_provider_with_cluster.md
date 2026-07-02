# AWS Commands for EKS Pod Identities vs. OIDC (IRSA)

The exact commands required to map AWS IAM permissions to your Kubernetes pods depend entirely on whether you are using the older **OIDC (IRSA)** workflow or the modern **EKS Pod Identities** feature.

---

## 1. The Modern Way: EKS Pod Identities (No OIDC Required)
This method entirely skips OIDC provider creation and global IAM trust modifications. Instead, it relies on a local agent running inside the cluster.

### Step A: Install the Pod Identity Agent Add-on
Run this command to install the required agent daemonset inside your EKS cluster:
```bash
aws eks create-addon \
  --cluster-name <your-cluster-name> \
  --addon-name eks-pod-identity-agent
```

### Step B: Create the Pod Identity Association
Run this command to map your Kubernetes `ServiceAccount` to your target AWS `IAM Role`. EKS handles the underlying credential token exchange via its own internal Auth APIs:
```bash
aws eks create-pod-identity-association \
  --cluster-name <your-cluster-name> \
  --namespace <kubernetes-namespace> \
  --service-account <kubernetes-service-account-name> \
  --role-arn arn:aws:iam::<your-account-id>:role/<your-iam-role-name>
```

---

## 2. The Legacy Way: OIDC Provider Association (IRSA)
If your architecture relies on the older IAM Roles for Service Accounts (IRSA) workflow, you must manually bridge the gap between the EKS OIDC URL and global AWS IAM using either `eksctl` or the native AWS CLI.

### Option A: Using `eksctl` (Simplest One-liner)
`eksctl` automatically fetches the unique OIDC URL from your cluster backend and registers it as a trusted identity provider in your AWS account in a single step:
```bash
eksctl utils associate-iam-oidc-provider \
  --cluster <your-cluster-name> \
  --approve
```

### Option B: Using Native AWS CLI (Manual Process)
If you do not use `eksctl`, you must find the URL first, and then explicitly create the provider via the IAM sub-command.

**Step 1: Extract your cluster's unique OIDC Issuer URL:**
```bash
aws eks describe-cluster \
  --name <your-cluster-name> \
  --query "cluster.identity.oidc.issuer" \
  --output text
```
*(This returns an HTTPS URL unique to your cluster, such as `https://amazonaws.com`)*

**Step 2: Bind that URL to AWS IAM:**
```bash
aws iam create-open-id-connect-provider \
  --url <paste-the-url-from-step-1-here> \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list <oidc-server-ssl-root-certificate-thumbprint>
```


# Choosing Between Modern (EKS Pod Identities) and Legacy (OIDC/IRSA)

The **modern way (EKS Pod Identities) is decisively the best choice** for almost all new AWS architectures. AWS introduced it specifically to eliminate the operational pain, security pitfalls, and complexity of the legacy OIDC (IRSA) method.

---

## 1. Why Modern (EKS Pod Identities) is Better

* **Significantly Simpler**: You completely bypass the step of creating and linking an OIDC provider for every single cluster. You simply install an EKS add-on, and AWS handles everything behind the scenes.
* **Easier Reusability**: With legacy IRSA, an IAM Role trust policy is strictly hardcoded to *one specific cluster's OIDC URL*. If you have three clusters (Dev, Stage, Prod), you have to create three separate IAM roles. With Pod Identities, the same IAM role can be easily reused across multiple clusters.
* **Scalable Tagging & Session Handling**: Pod Identities automatically tag your temporary AWS credentials with metadata like `eks-cluster-arn`, `kubernetes-namespace`, and `kubernetes-service-account`. This allows you to write highly advanced IAM policies based on tags rather than creating new roles for every deployment.
* **No OIDC Tag Management**: You do not have to worry about fetching, rotating, or troubleshooting SSL thumbprints for OIDC endpoints.

---

## 2. The Only Times You Must Use the Legacy Way (OIDC/IRSA)

While the modern way is superior, there are three specific scenarios where you are forced to use the legacy OIDC method:

1. **Multi-Cloud or Hybrid Clusters**: Pod Identities require the official AWS EKS backend infrastructure. If you are running an on-premises Kubernetes cluster (like Red Hat OpenShift, Rancher, or upstream Kubernetes on EC2), you *must* use the legacy OIDC/S3 trick.
2. **Cross-AWS-Account Access**: If your EKS cluster lives in AWS Account A, but your pods need to access an S3 bucket or DynamoDB database sitting in AWS Account B, EKS Pod Identities cannot natively cross that account boundary yet. IRSA handles cross-account IAM mapping much cleaner.
3. **Legacy EKS Versions**: EKS Pod Identities require EKS cluster versions `1.24` or higher. If you are maintaining ancient legacy clusters, you are restricted to IRSA.

---

## 3. Feature Matrix Comparison

| Feature | Modern (Pod Identities) | Legacy (OIDC / IRSA) |
| :--- | :--- | :--- |
| **Requires Cluster OIDC URL?** | No ❌ | Yes |
| **Setup Complexity** | Very Low (1 command) | High (Multiple linking steps) |
| **IAM Role Reusability** | Excellent (1 role for many clusters) | Poor (1 role per cluster) |
| **Performance Scale** | High (AWS API-driven) | Throttling limits (Large clusters hit IAM limits) |
| **Supports Non-EKS Cluster?** | No ❌ | Yes |

---

## 4. Summary Recommendation
If you are deploying your applications on standard **AWS managed Amazon EKS** and your infrastructure lives within the same AWS account, **always use EKS Pod Identities**. It will save you time and dramatically simplify your CI/CD automation pipelines.
