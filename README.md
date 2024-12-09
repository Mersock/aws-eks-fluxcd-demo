## Getting started

To make it easy for you to get started with GitLab, here's a list of recommended next steps.

Already a pro? Just edit this README.md and make it your own. Want to make it easy? [Use the template at the bottom](#editing-this-readme)!

## Prerequisite
 - Gitlab personal access token require permission scope to grants complete read/write access to the API.
 - [Flux CLI install](https://fluxcd.io/flux/installation/#install-the-flux-cli)
 - [AWS CLI install](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#getting-started-install-instructions)
 - [eksctl CLI install](https://eksctl.io/installation/) (Optional)

## Install
 - Create an EKS cluster using the following command and then setup kubeconfig (fluxcd-demo-cluster.conf) (Optiontal)
```
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

![eks-console](https://gitlab.com/true-dc-exist/poc/aws-eks-fluxcd/-/raw/main/iam-role.jpg)

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

- In the `clusters/eks/flux-system` directory, modify the Kustomization to be as follows 
PS. Don't forget to change ARN from the IAM role create in the previous step
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
        annotations:
          eks.amazonaws.com/role-arn: "arn:aws:iam::accountID:role/FluxCDECR" 
      target:
        kind: ServiceAccount
        name: source-controller
```
