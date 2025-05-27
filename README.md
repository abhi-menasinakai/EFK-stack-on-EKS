
# EFK Stack on EKS

This project shows the configurations of EFK stack onto the EKS cluster.
 


![Logo](https://imgs.search.brave.com/7HRiScvUDVmqEKCLYaHvAE0_PqH7vA039HNdcAYO8_M/rs:fit:500:0:0:0/g:ce/aHR0cHM6Ly9kZXZv/cHNjdWJlLmNvbS9j/b250ZW50L2ltYWdl/cy8yMDI1LzAzL2lt/YWdlLTctNTYucG5n)


## Documentation

This ReadMe file contains step by step procedure to provision EKS cluster and Install EFK stack on EKS cluster using helm charts.
So Let's begin,

  1. Install EKS cluster
  2. Configure ElasticSearch using helm charts using EBS volume as statefulset
  3. Configure FluentBit using helm
  4. Integrate FluentBit with ElasticSearch
  4. Configure Kibana Dashboard using helm
  5. Kibana Integration with ElasticSearch




## Install the EKS cluster 

Launch the cluster using eksctl utility

create cluster named my-cluster in us-east-1 zone without nodegroups

```bash
  eksctl create cluster --name=my-cluster \
--region=us-east-1 --zones=us-east-1a,us-east-1b --without-nodegroup
```

Create nodegroup for the cluster

```bash
  eksctl create nodegroup --cluster=my-cluster \
--region=us-east-1 \
--name=my-cluster-nodegroup \
--node-type=t3.medium \
--nodes-min=2 \
--nodes-max=3 \
--node-volume-size=10 \
--managed \
--asg-access \
--external-dns-access \
--full-ecr-access \
--appmesh-access \
--alb-ingress-access \
--node-private-networking
```

Create OIDC provider for your cluster

```bash
  eksctl utils associate-iam-oidc-provider \
--region us-east-1 \
--cluster my-cluster \
--approve
```

Once the cluster created, Update the kubeconfig file on your machine from where you want to access EKS cluster

```bash
  aws eks update-kubeconfig --name observability
```

Since we need EBS volume for our EKS cluster, The access for the EBS volume will be made through API call by EBS CSI driver.
So CSI driver runs as POD which requires a service account which should have permissions to make EBS API call.

Let's create IAM role with policy to make API calls for EBS provisioning


```bash
  aws iam create-role \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --assume-role-policy-document file://trust-policy.json
```

the contents of trust-policy.json shown below. I have added temporary OIDC and Account ID in the file. Make sure to replace it with valid one.


```bash
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": {
				"Federated": "arn:aws:iam::241099305681:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/FB347EGDJE8293ED274664B"
			},
			"Action": "sts:AssumeRoleWithWebIdentity",
			"Condition": {
				"StringEquals": {
					"oidc.eks.us-east-1.amazonaws.com/id/FB348277835BDRGW274664B:aud": "sts.amazonaws.com"
				}
			}
		}
	]
}
```

Once the role is created, make sure that created role has valid policy along with OIDC URL and ID. 



Now attach this role with service account which will be used in EBS CSI driver plugin pods, By creating IRSA.

```bash
eksctl create iamserviceaccount \
--name ebs-csi-controller-sa \
--namespace kube-system \
--cluster my-cluster \
--role-name AmazonEKS_EBS_CSI_DriverRole \
--attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
--approve
```

Set Amazon resource number as variable
```bash
  ARN=$(aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole --query 'Role.Arn' --output text)
```

Now deploy the CSI driver with IRSA being created.
```bash
  eksctl create addon --cluster observability --name aws-ebs-csi-driver --version latest \
--service-account-role-arn $ARN --force
```
Deubg: If your elastic search pod is not able to access the ebs and not able to run then verify that OIDC url and configuration done in the policy.

Go To IAM roles and look for EBS role and check trust policy with OIDC  & Validate the Role trust relationship is valid or not using below command

use below command to get OIDC URL

```bash
  aws eks describe-cluster --name your-cluster-name --query "cluster.identity.oidc.issuer" --output text
```

ElasticSearch Installation:
