# eks-game-2048

Deploying the classic 2048 game to Amazon EKS (Kubernetes on AWS), exposed to the internet through an Application Load Balancer (ALB) — a hands-on way to learn EKS networking end to end.

## eks-game-2048

"2048_full.yaml"    # all Kubernetes resources for the app, in one manifest

"iam_policy.json"    # AWS IAM permissions needed by the ALB controller

## Step-by-step: what `2048_full.yaml` creates

The file contains four Kubernetes resources, applied together:

1. **Namespace `game-2048`** — isolates all the app's resources from the rest of the cluster.
2. **Deployment `deployment-2048`**
   - Runs **5 replicas** of the public image `public.ecr.aws/l6m2t8p7/docker-2048:latest`.
   - Each pod exposes container port `80`.
   - `imagePullPolicy: Always` ensures the latest image is pulled each time a pod starts.
3. **Service `service-2048`**
   - Type `NodePort`, listening on port `80` and forwarding to `targetPort: 80` on matching pods (selected via the `app.kubernetes.io/name: app-2048` label).
   - Groups the 5 replica pods behind one stable internal address.
4. **Ingress `ingress-2048`**
   - Annotated for the **AWS Load Balancer Controller**: `scheme: internet-facing` (public ALB) and `target-type: ip` (ALB routes directly to pod IPs, not node ports).
   - Uses `ingressClassName: alb` to tell Kubernetes which controller should handle it.
   - Routes all paths (`/`) to `service-2048` on port `80`.

## 🔄 Step-by-step: what `iam_policy.json` is for

This is a standard **AWS IAM policy** (247 lines) granting the permissions the AWS Load Balancer Controller needs to manage load balancers on your behalf, e.g.:
- Describing VPCs, subnets, security groups, and instances
- Creating/modifying/deleting Application and Network Load Balancers
- Managing target groups and registering/deregistering targets
- Managing listeners, rules, and tags on those resources

Without this policy attached to the controller's IAM role, the Ingress in step 4 above would never actually provision a real ALB.

## ⚙️ Requirements

- An EKS cluster (e.g. via `eksctl create cluster`)
- `kubectl` and `aws-cli` configured for that cluster
- The **AWS Load Balancer Controller** installed on the cluster, using the permissions from `iam_policy.json`

## How to deploy, step by step

# 1. Create the IAM policy the Load Balancer Controller needs
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# 2. Install the AWS Load Balancer Controller on your cluster,
#    attaching the policy above to its service account (via IRSA)
#    — see AWS's official install guide for the full command set

# 3. Deploy the game — creates the namespace, Deployment, Service, and Ingress
kubectl apply -f 2048_full.yaml

# 4. Confirm the pods are running
kubectl get pods -n game-2048

# 5. Get the ALB's public URL once it's provisioned 
kubectl get ingress -n game-2048

Open the ADDRESS shown by the last command in a browser to play the game.
