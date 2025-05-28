
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

Now Install ElasticSearch using helm:
```bash
  helm repo add elastic https://helm.elastic.co
```

```bash
  helm install elasticsearch \
--set replicas=1 \
--set volumeClaimTemplate.storageClassName=gp2 \
--set persistence.labels.enabled=true elastic/elasticsearch -n logging
```

Now use below command to get user and password generated in elasticsearch pods.

```bash
  kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.username}' | base64 -d 
```

```bash
kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d
```

Testing of Elsatic search 
Check the ElasticSearch accessibility using it's service and ensure it's connectivity to store the logs in EBS. Make sure 
```bash
kubectl run curl-test --image=radial/busyboxplus:curl -it --rm --restart=Never -- sh
```
```bash
curl -u <elastic-user>:<elastic-passwd> -k https://elasticsearch-master.default.svc.cluster.local:9200
```


Installing FluentBit:
Install FluentBit using helm charts as shown
```bash
helm repo add fluent https://fluent.github.io/helm-charts
```

```bash
helm install fluent-bit fluent/fluent-bit -f fluentbit-values.yaml -n logging
```
contents of fluent-bit values are shown below.

```bash
apiVersion: v1
data:
  custom_parsers.conf: |
    [PARSER]
        Name docker_no_time
        Format json
        Time_Keep Off
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
  fluent-bit.conf: |
    [SERVICE]
        Daemon Off
        Flush 1
        Log_Level info
        Parsers_File /fluent-bit/etc/parsers.conf
        Parsers_File /fluent-bit/etc/conf/custom_parsers.conf
        HTTP_Server On
        HTTP_Listen 0.0.0.0
        HTTP_Port 2020
        Health_Check On

    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        multiline.parser docker, cri
        Tag kube.*
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On

    [INPUT]
        Name systemd
        Tag host.*
        Systemd_Filter _SYSTEMD_UNIT=kubelet.service
        Read_From_Tail On

    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Keep_Log Off
        K8S-Logging.Parser On
        K8S-Logging.Exclude On

    [OUTPUT]
        Name es
        Match kube.*
        Host elasticsearch-master.default.svc.cluster.local
        Port 9200
        TLS On
        TLS.Verify off
        HTTP_User elastic
        HTTP_Passwd 8uROQOSbIhMALKec
        Retry_Limit False
        Suppress_Type_Name On
        Replace_Dots On

    [OUTPUT]
        Name es
        Match host.*
        Host elasticsearch-master.default.svc.cluster.local
        Port 9200
        TLS On
        TLS.Verify off
        HTTP_User elastic
        HTTP_Passwd 8uROQOSbIhMALKec
        Logstash_Prefix node
        Retry_Limit False
        Suppress_Type_Name On
        Replace_Dots On
kind: ConfigMap
metadata:
  annotations:
    meta.helm.sh/release-name: my-release
    meta.helm.sh/release-namespace: default
  creationTimestamp: "2025-05-26T06:38:36Z"
  labels:
    app.kubernetes.io/instance: my-release
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: fluent-bit
    app.kubernetes.io/version: 4.0.1
    helm.sh/chart: fluent-bit-0.49.0
  name: my-release-fluent-bit
  namespace: default
  resourceVersion: "31536"
  uid: 5426e512-8f4f-4e49-85e3-0c8394165e22
  ```
  
  Now install kibana dashboard using helm as well

  ```bash
  helm install kibana --set service.type=LoadBalancer elastic/kibana -n logging
  ```

Once the Kibana dashboard is installed access the dashboard using access credentials fetched from Elastoc search sts secrets. 
Thanks for Reading...!

