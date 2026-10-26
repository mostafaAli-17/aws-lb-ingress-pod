# Deploy a Web Application on Amazon EKS with AWS Application Load Balancer

A hands-on DevOps lab demonstrating how to deploy an NGINX web
application on Amazon EKS and expose it to the internet using the AWS
Load Balancer Controller and an Application Load Balancer (ALB).

> **Status:** Lab resources were created and the ALB was successfully
> provisioned. Confirm the browser test and clean up AWS resources
> before describing the lab as fully tested and removed.

## Architecture

``` text
User / Browser
      |
      v
AWS Application Load Balancer (ALB)
      |
      v
Kubernetes Ingress (web-ingress-alb)
      |
      v
Kubernetes Service (web-service:80)
      |
      +-------------------+
      |                   |
      v                   v
   NGINX Pod 1         NGINX Pod 2
```

The AWS Load Balancer Controller watches Kubernetes resources and
provisions/configures the ALB based on the Ingress resource. The ALB
used IP targets to route HTTP traffic to the application Pods.

## Technologies Used

-   Amazon EKS
-   Kubernetes
-   AWS Application Load Balancer (ALB)
-   AWS Load Balancer Controller
-   Helm
-   eksctl
-   AWS IAM and OIDC / IRSA
-   NGINX
-   AWS CLI

## What This Lab Covers

1.  Deploying an NGINX application with two replicas.
2.  Exposing the Pods internally through a Kubernetes `ClusterIP`
    Service.
3.  Installing the AWS Load Balancer Controller with Helm.
4.  Configuring IAM permissions and associating an OIDC provider for the
    EKS cluster.
5.  Creating an ALB Ingress to expose the application over HTTP.
6.  Verifying the ALB address and its registered Pod IP targets.
7.  Cleaning up resources to reduce ongoing AWS charges.

## Prerequisites

-   An AWS account and an existing EKS cluster.
-   AWS CLI configured with permissions appropriate for EKS, IAM, and
    load-balancer resources.
-   `kubectl`, `eksctl`, and Helm installed.
-   The Kubernetes context set to the intended cluster.
-   Awareness that AWS load balancers and related resources may incur
    charges.

The lab used cluster name `cluster1` in region `us-east-1`. Replace
these values if adapting the instructions to another environment. Never
commit AWS credentials, access keys, account-specific secrets, or
kubeconfig files to GitHub.

## 1. Set Environment Variables

``` bash
export CLUSTER_NAME=cluster1
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

Check that the cluster is active:

``` bash
aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'cluster.status' \
  --output text
```

## 2. Deploy the NGINX Application

Create a Deployment with two replicas:

``` bash
kubectl create deployment web-app --image=nginx --replicas=2
```

Create a Service to provide stable internal access:

``` bash
kubectl expose deployment web-app \
  --name=web-service \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP
```

Check the resources:

``` bash
kubectl get deployments,pods,services
```

## 3. Configure IAM and OIDC for the Controller

Associate an IAM OIDC provider with the cluster if one is not already
associated:

``` bash
eksctl utils associate-iam-oidc-provider \
  --cluster="$CLUSTER_NAME" \
  --region="$AWS_REGION" \
  --approve
```

The AWS Load Balancer Controller needs an IAM policy and an IAM role
associated with a Kubernetes ServiceAccount. Use the current AWS EKS
documentation to obtain the policy that matches the controller version
you intend to install. If a policy with the intended name already
exists, verify its contents and version rather than creating a duplicate
or assuming it is current.

Example ServiceAccount setup (the IAM policy must already exist):

``` bash
eksctl create iamserviceaccount \
  --cluster="$CLUSTER_NAME" \
  --region="$AWS_REGION" \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name=AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn="arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy" \
  --approve
```

If the ServiceAccount or role already exists, inspect the existing
configuration before retrying. Do not blindly overwrite or delete shared
IAM resources.

## 4. Install the AWS Load Balancer Controller with Helm

Add and update the AWS EKS Helm repository:

``` bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

Get the VPC ID for the cluster:

``` bash
VPC_ID=$(aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'cluster.resourcesVpcConfig.vpcId' \
  --output text)

echo "$VPC_ID"
```

Install the chart, using a version compatible with the IAM policy and
cluster configuration:

``` bash
helm upgrade --install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName="$CLUSTER_NAME" \
  --set region="$AWS_REGION" \
  --set vpcId="$VPC_ID" \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

Verify that the controller is ready:

``` bash
kubectl rollout status deployment/aws-load-balancer-controller \
  --namespace kube-system
```

## 5. Create an ALB Ingress

The following Ingress sends HTTP traffic on port 80 to `web-service`. It
uses IP targets so the ALB can route directly to the application Pods.

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress-alb
  namespace: default
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

Save this as `ingress.yaml` and apply it:

``` bash
kubectl apply -f ingress.yaml
```

## 6. Verify the Deployment

Check the Ingress and wait for the ALB address to appear:

``` bash
kubectl get ingress web-ingress-alb
kubectl describe ingress web-ingress-alb
```

Check the controller logs if provisioning fails:

``` bash
kubectl logs deployment/aws-load-balancer-controller \
  --namespace kube-system \
  --tail=100
```

When the ALB address appears, open `http://<ALB-DNS-NAME>` in a browser.
The expected page for this sample is the NGINX welcome page. The Ingress
events and registered target IPs help confirm configuration, but a
browser or HTTP request test is still needed to verify end-to-end
connectivity.

## Security and Cost Notes

-   This example uses plain HTTP on port 80 for a lab. For production,
    configure HTTPS with an appropriate certificate and review access
    controls.
-   `internet-facing` exposes the ALB publicly. Do not use it for
    private applications without reviewing the security requirements.
-   AWS load balancers and associated networking resources may incur
    charges. Check the pricing and Free Tier terms for your account.
-   Delete lab resources when finished and verify in the AWS Console
    that the ALB has been removed.
-   Use least-privilege IAM permissions and the policy version required
    by the controller version you install.
-   Do not commit credentials, private keys, kubeconfig files, or
    sensitive account data to the repository.

## Cleanup

Delete the Ingress first and wait for the ALB to be deleted by the
controller:

``` bash
kubectl delete ingress web-ingress-alb
```

If you also created the original NGINX Ingress resource and no longer
need it:

``` bash
kubectl delete ingress web-ingress
```

Delete the sample application:

``` bash
kubectl delete deployment web-app
kubectl delete service web-service
```

If this lab's AWS Load Balancer Controller is no longer needed and is
not shared with other workloads, uninstall it:

``` bash
helm uninstall aws-load-balancer-controller --namespace kube-system
```

Then remove the ServiceAccount and associated role only if they were
created exclusively for this lab:

``` bash
eksctl delete iamserviceaccount \
  --cluster="$CLUSTER_NAME" \
  --region="$AWS_REGION" \
  --namespace=kube-system \
  --name=aws-load-balancer-controller
```

If you installed the NGINX Ingress Controller solely for this lab,
remove its namespace only after confirming that no other workloads
depend on it:

``` bash
kubectl delete namespace ingress-nginx
```

Do not delete an existing IAM policy, OIDC provider, EKS cluster, VPC,
subnets, or EC2 instances unless you have verified they are dedicated to
this lab and you intentionally want them removed.

## Useful Commands

``` bash
kubectl get pods -A
kubectl get svc -A
kubectl get ingress -A
kubectl get events -A --sort-by=.lastTimestamp
helm list -A
```

## References

-   [Amazon EKS
    documentation](https://docs.aws.amazon.com/eks/latest/userguide/)
-   [AWS Load Balancer Controller
    documentation](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)
-   [AWS EKS Helm charts](https://github.com/aws/eks-charts)
-   [Helm documentation](https://helm.sh/docs/)
-   [eksctl documentation](https://eksctl.io/)

------------------------------------------------------------------------

Created as a hands-on learning lab for Kubernetes networking and AWS
load balancing.
