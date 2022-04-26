# eks-example

## Setting up EKS Cluster on AWS

### Step 1: Creating an IAM role that Kubernetes can assume to create AWS resources

### Step 2: Creating a VPC for EKS with CloudFormation

Open up CloudFormation, and Create new stack. With template in Setup EKS Cluster

```
└───Setup EKS Cluster
    └───amazon-eks-vpc-sample.yaml
```

### Step 3: Use the AWS CLI to create the Kubernetes cluster.

```
aws eks --region <region> create-cluster --name <clusterName> 
--role-arn <EKS-role-ARN> --resources-vpc-config 
subnetIds=<subnet-id-1>,<subnet-id-2>,<subnet-id-3>,securityGroupIds=
<security-group-id>
```

It takes about 5 minutes before your cluster is created. You can ping the status of the command using this CLI command:
```
aws eks describe-cluster --name <clusterName> --query cluster.status
```
The output displayed will be:

"CREATING"