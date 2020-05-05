# Getting up and running with Calico Host Micro-segmentation Protection on AWS

## Agenda

- Calico Host Micro-segmentation Protection deployment considerations
- Deploy Calico Host Micro-segmentation Protection Control Plane
- Deploy Calico Host Micro-segmentation Protection
- Simple Calico Host Micro-segmentation Protection network policy example

## Notes

### Calico Host Micro-segmentation Protection deployment considerations

- [System Requirements](https://docs.projectcalico.org/getting-started/bare-metal/requirements)
  - All nodes (Ubuntu 18.04 LTS)
- [Installation method](https://docs.projectcalico.org/getting-started/bare-metal/installation/)
  - Binary Install
- [Choose a Datastore Type](https://docs.tigera.io/getting-started/bare-metal/installation/binary#step-3-create-environment-file)
  - Kubernetes Datastore (KDD)

### Deploy Calico Host Micro-segmentation Protection Control Plane

#### Launch an Amazon VPC infrastructure stack using CloudFormation

1. Choose an AWS region

```
export DEFAULT_AWS_REGION=us-east-1
```

2. Launch an Amazon VPC infrastructure stack using CloudFormation

```
aws cloudformation deploy \
  --no-fail-on-empty-changeset \
  --capabilities CAPABILITY_NAMED_IAM \
  --template-file cloudformation-infra.yaml \
  --tags StackType="infra" \
  --stack-name calico-demo \
  --parameter-overrides \
    VpcCidrBlock=10.0.0.0/16 \
    CreatePrivateNetworks=false
```

The base networking environment consists of a VPC with IPv4 and IPv6 addressing for subnets across three availability zones. The IPv4 network cidr is configurable and the IPv6 cidr is dynamically assigned by AWS.  Private subnets with a NAT gateway per availability zone can optionally be created.  Security groups and IAM roles used by EKS are also deployed as part of the base networking environment.

![infra](images/infra.png)

#### Launch an Amazon EKS stack into the Amazon VPC infrastructure using CloudFormation

1. Launch an Amazon EKS stack into the Amazon VPC infrastructure using CloudFormation

Ensure the Amazon EKS-optimized Linux AMI(https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html) for your worker nodes exist in the AWS region you've chosen and also make sure the `KeyName` for the AWS Key Pair exists.

```
aws cloudformation deploy \
  --no-fail-on-empty-changeset \
  --capabilities CAPABILITY_NAMED_IAM \
  --template-file cloudformation-eks.yaml \
  --tags StackType="eks" \
  --stack-name calico-demo-eks \
  --parameter-overrides \
    EnvironmentName=calico-demo \
    KeyName=calico-demo \
    ImageId=ami-011c865bf7da41a9d \
    InstanceType=t3.2xlarge \
    WorkerNodeCount=2 \
    KubernetesVersion=1.16
```

2. Update your local kubeconfig to talk to the EKS cluster

```
aws eks update-kubeconfig --name calico-demo-eks
kubectl get nodes
```

3. Update the `aws-auth-cm.yaml` with your account ID

Take a look at the `aws-auth-cm.yaml`.  This ConfigMap is used to allow your worker nodes to join the cluster. This ConfigMap is also used to add RBAC access to IAM users and roles. 

```
ACCOUNTID=$(aws sts get-caller-identity --output text --query 'Account')
sed -i "" "s/ACCOUNTID/$ACCOUNTID/g" aws-auth-cm.yaml
``` 

You may have a different `sed` syntax so make sure the ConfigMap has been updated with your account ID.  

4.  Apply the ConfigMap yaml and watch the cluster nodes come up

```
kubectl apply -f aws-auth-cm.yaml
kubectl get nodes --watch
```

#### Deploy Calico Host Micro-segmentation Protection Control Plane

1. Install Calico on Amazon EKS

```
kubectl get pods --all-namespaces
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/release-1.5/config/v1.5/calico.yaml
kubectl get pods --all-namespaces --watch
```

### Deploy Calico Host Micro-segmentation Protection

#### Launch a Calico Host Micro-segmentation stack into the Amazon VPC infrastructure using CloudFormation

1. Launch a Calico Host Micro-segmentation stack into the Amazon VPC infrastructure using CloudFormation

Let's stand up a simple Amazon Virtual Machine via an Autoscale Group of 1. 

```
aws cloudformation deploy \
    --no-fail-on-empty-changeset \
    --capabilities CAPABILITY_NAMED_IAM \
    --template-file cloudformation-hms.yaml \
    --tags StackType="hms" \
    --stack-name calico-demo-hms \
    --parameter-overrides \
      EnvironmentName=calico-demo \
      KeyName=calico-demo \
      ImageId=ami-085925f297f89fce1 \
      InstanceType=t3.2xlarge \
      NodeCount=1
```

#### Configure Calico Host Micro-segmentation 


### Simple Calico Host Micro-segmentation Protection network policy example

1.  Inspect the network policies

```
calicoctl get networkpolicies --all-namespaces
```

```
NAMESPACE   NAME

```

2.  Inspect the global network policies

```
calicoctl get globalnetworkpolicies
```

```
NAME

```

3. Deploy the example segmentation policies

```
calicoctl apply -f .yaml
```

4. Verify the segmentation policies

```
calicoctl get networkpolicies --all-namespaces
```

## References

* Calico Host Micro-segmentation: https://docs.projectcalico.org/getting-started/bare-metal/
* Managing users or IAM roles for your EKS cluster: https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html
* Installing Calico on Amazon EKS: https://docs.aws.amazon.com/eks/latest/userguide/calico.html
* Understanding Policy Enforcement Options with Calico: https://www.projectcalico.org/understanding-the-policy-enforcement-options-with-calico/