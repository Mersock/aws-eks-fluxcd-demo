## Getting started

This repository demonstrates how to install and configure FluxCD to deploy an application on an EKS cluster by pulling images from ECR.

!Important If you want to practice by follow these steps please create new repository.

## Prerequisite
 - Gitlab personal access token require permission scope to grants complete read/write access to the API.
 - [Kubectl CLI install](https://kubernetes.io/docs/tasks/tools/)
 - [Helm CLI install](https://helm.sh/docs/intro/install/)
 - [Flux CLI install](https://fluxcd.io/flux/installation/#install-the-flux-cli)
 - [AWS CLI install](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#getting-started-install-instructions)
 - [eksctl CLI install](https://eksctl.io/installation/) (Optional)

## Steps
- [Install FluxCD in AWS EKS cluster](https://gitlab.com/true-dc-exist/poc/aws-eks-fluxcd#install-fluxcd-in-aws-eks-cluster)
- [Setup FluxCD](https://gitlab.com/true-dc-exist/poc/aws-eks-fluxcd#setup-fluxcd)
- [Push chart to ECR](https://gitlab.com/true-dc-exist/poc/aws-eks-fluxcd#push-chart-to-ecr)
- [Deploy nginx from ECR to EKS cluster using Helm repository](https://gitlab.com/true-dc-exist/poc/aws-eks-fluxcd#deploy-nginx-from-ecr-to-eks-cluster-using-helm-repository)

## Install FluxCD in AWS EKS cluster
 - Create an EKS cluster using the following command and then setup kubeconfig (fluxcd-demo-cluster.conf) (Optiontal)
```
# Create EKS cluster
eksctl create cluster \
--name fluxcd-demo-cluster \
--version auto \
--region ap-southeast-1 \
--nodegroup-name fluxcd-demo-group \
--node-type t3.large \
--nodes 1 \
--nodes-min 1 \
--nodes-max 2 \
--managed \
--with-oidc

# Setup Kubeconfig
aws eks update-kubeconfig --name fluxcd-demo-cluster --region ap-southeast-1  --kubeconfig ~/.kube/fluxcd-demo-cluster.conf
```


- Create a trust.json policy that contains the following (replace accountID and clusterID with the respective values: your account ID, and the clusterID from the EKS page.)

![eks-console](https://gitlab.com/true-dc-exist/poc/aws-eks-fluxcd/-/raw/main/eks-console.jpg)
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<accountID>:oidc-provider/oidc.eks.eu-west-1.amazonaws.com/id/<clusterID>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.eu-west-1.amazonaws.com/id/<clusterID>:sub": "system:serviceaccount:flux-system:source-controller",
                    "oidc.eks.eu-west-1.amazonaws.com/id/<clusterID>:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```
- Create the role, attach the trust policy and the ECR readonly policy
```
aws iam create-role --role-name FluxCDECR --assume-role-policy-document file://trust.json

aws iam attach-role-policy --role-name FluxCDECR --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

- IAM console and and copy role's ARN

![iam-role](https://gitlab.com/true-dc-exist/poc/aws-eks-fluxcd/-/raw/main/iam-role.jpg)

- Create a directory for the EKS cluster

```
mkdir -p  clusters/eks
```

- Boot Flux CD with the new EKS cluster
```
export GITLAB_TOKEN=<Gitlab personal token>

# Pattern command to bootstrap fluxcd
flux bootstrap gitlab --owner=<gitlab group/subgroup if thers no group using username or email> --repository=<repository name> --branch=main --path=clusters/eks --token-auth --personal
# This command using in this repository
flux bootstrap gitlab --owner=true-dc-exist/poc --repository=aws-eks-fluxcd --branch=main --path=clusters/eks --token-auth --personal
# Result should be as follows
.
.
✔ Kustomization reconciled successfully
► confirming components are healthy
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```

- Pull the changes that Flux CD did to the cluster
```
git pull
```
## Setup FluxCD
- In the `clusters/eks/flux-system` directory, modify the Kustomization to be as follows then push to main branch.

PS. Don't forget to change ARN from the IAM role create in the previous step. And if put wrong ARN then EKS well be able to pull chart from ECR.
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- gotk-components.yaml
- gotk-sync.yaml
patches:
  - patch: |
      apiVersion: v1
      kind: ServiceAccount
      metadata:
        name: source-controller
        namespace: flux-system
        annotations:
          eks.amazonaws.com/role-arn: "arn:aws:iam::accountID:role/FluxCDECR" 
      target:
        kind: ServiceAccount
        name: source-controller
        namespace: flux-system
```

- Reconcile Flux CD (no waiting for interval time to auto reconcile)
```
flux reconcile kustomization flux-system --with-source
► annotating GitRepository flux-system in flux-system namespace
✔ GitRepository annotated
◎ waiting for GitRepository reconciliation
✔ fetched revision main@sha1:e77b15f9f462c7c721d1f294fd24e4ee9e054af2
► annotating Kustomization flux-system in flux-system namespace
✔ Kustomization annotated
◎ waiting for Kustomization reconciliation
✔ applied revision main@sha1:e77b15f9f462c7c721d1f294fd24e4ee9e054af2
```

## Push chart to ECR
- Create a new Helm package for Nginx and deploy to ECR
```
#At root directory
mkdir charts
cd charts
helm create nginx
```

- Package chart and push chart to ECR repository

Important! Before package and push uppdate name in `Chart.yaml` to the same name wit ECR repository 

```
helm package .
aws ecr get-login-password --region ap-southeast-1 | helm registry login --username AWS --password-stdin accountID.dkr.ecr.ap-southeast-1.amazonaws.com
# aws-eks-fluxcd is also the name of ECR repository
helm push aws-eks-fluxcd-0.1.0.tgz oci://accountID.dkr.ecr.ap-southeast-1.amazonaws.com
```

## Deploy nginx from ECR to EKS cluster using Helm repository

- Create a new Helm repository file called `ecr.yaml` in `clusters/eks` directory with the following contents
```
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: ecr
  namespace: default
spec:
  type: oci
  interval: 5m0s
  url: oci://accoundID.dkr.ecr.eu-west-1.amazonaws.com
  provider: aws
```
- Create a new Helm Release that install Apache and called it `nginx-helm-release.yaml`.
```
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: nginx
  namespace: default
spec:
  interval: 5m
  chart:
    spec:
      chart: aws-eks-fluxcd
      version: '0.1.0'
      sourceRef:
        kind: HelmRepository
        name: ecr
        namespace: default
      interval: 1m
```

- Push to main branch and check the OCI repository, the Helm release, and the pods.
```
git push 
kubectl get helmrepository,helmrelease,pods
NAME                                          URL                                                       AGE    READY   STATUS
helmrepository.source.toolkit.fluxcd.io/ecr   oci://799067542302.dkr.ecr.ap-southeast-1.amazonaws.com   8m4s           
NAME                                       AGE    READY   STATUS
helmrelease.helm.toolkit.fluxcd.io/nginx   4m2s   True    Helm install succeeded for release default/nginx.v1 with chart aws-eks-fluxcd@0.1.0
NAME                                       READY   STATUS    RESTARTS   AGE
pod/nginx-aws-eks-fluxcd-96c9675b6-lkrv9   1/1     Running   0          91s
```