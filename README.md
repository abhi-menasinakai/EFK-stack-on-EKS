
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

