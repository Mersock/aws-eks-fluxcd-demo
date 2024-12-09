## Getting started

To make it easy for you to get started with GitLab, here's a list of recommended next steps.

Already a pro? Just edit this README.md and make it your own. Want to make it easy? [Use the template at the bottom](#editing-this-readme)!

## Prerequisite
 - Gitlab personal access token require permission scope to grants complete read/write access to the API.
 - [AWS CLI install](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#getting-started-install-instructions)
 - [eksctl CLI Install](https://eksctl.io/installation/) (Optional)

## Install
 - Create an EKS cluster using the following command (Optiontal)
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
mkdir -p  myfluxrepo/clusters/eks
```

- Boot Flux CD with the new EKS cluster
```
export GITLAB_TOKEN=<Gitlab personal token>

flux bootstrap gitlab --owner=abohmeed --repository=fluxrepotest --branch=main --path=clusters/eks --token-auth --personal
```
