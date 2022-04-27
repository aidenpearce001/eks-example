# eks-example

## 1. Setting up EKS Cluster on AWS

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

### Step 4: Launching Kubernetes worker nodes with CloudFormation

Launch Kubernetes worker nodes with CloudFormation

```
└───Setup EKS Cluster
    └───amazon-eks-nodegroup.yaml
```

- ClusterName – the name of your Kubernetes cluster (e.g. demo)

- ClusterControlPlaneSecurityGroup – the same security group you used for creating the cluster in previous step.

- NodeGroupName – a name for your node group.
NodeAutoScalingGroupMinSize – leave as-is. The minimum number of nodes that your worker node Auto Scaling group can scale to.

- NodeAutoScalingGroupDesiredCapacity – leave as-is. The desired number of nodes to scale to when your stack is created.

- NodeAutoScalingGroupMaxSize – leave as-is. The maximum number of nodes that your worker node Auto Scaling group can scale out to.
NodeInstanceType – leave as-is. The instance type used for the worker nodes.

- NodeImageId – the Amazon EKS worker node AMI ID for the region you’re using. For us-east-1, for example: ami-0c5b63ec54dd3fc38
KeyName – the name of an Amazon EC2 SSH key pair for connecting with the worker nodes once they launch.

- BootstrapArguments – leave empty. This field can be used to pass optional arguments to the worker nodes bootstrap script.

- VpcId – enter the ID of the VPC you created in Step 2 above.

- Subnets – select the three subnets you created in Step 2 above.

### Step 5: Install Amazon's authenticator for kubectl and IAM. 

Amazon EKS uses IAM for user management and access to clusters. Out of the box, kubectl does not support IAM. To bridge the gap, you must install a binary on your system called  aws-iam-authenticator

Open the file and replace the rolearn with the ARN of the NodeInstanceRole created above:

```
└───Setup EKS Cluster
    └───aws-auth-cm.yaml
```

Save the file and apply the configuration:

```
kubectl apply -f aws-auth-cm.yaml
```

To create an example application 

```
kubectl apply -f nginx-deployment-service.yaml
```

Check your pods' IPs:

```
kubectl get pods -l run=my-nginx -o yaml |  grep podIP 
```

You should be able to ssh into any node in your cluster and use a tool such as curl to make queries against both IPs.

## 2. Setup Nginx Ingress Controller

Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined on the Ingress resource.

A fanout configuration routes traffic from a single IP address to more than one Service, based on the HTTP URI being requested. An Ingress allows you to keep the number of load balancers down to a minimum. For example, a setup like:

![Ingress](https://k21academy.com/wp-content/uploads/2021/04/Ingress2.png)

Create nginx ingress on your cloud provider (azure/gcp/aws…)

```
kubectl apply -f nginx-ingress.yaml
```

Check the status of the pods to see if the ingress controller is online.

```
kubectl get pods --namespace ingress-nginx
```

Example:

```
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-hnlsq        0/1     Completed   0          3m6s
ingress-nginx-admission-patch-bds42         0/1     Completed   1          3m5s
ingress-nginx-controller-69d6546d6d-b6lvp   1/1     Running     0          3m18s
```

Now let's check to see if the service is online
```
kubectl get services --namespace ingress-nginx
```

Example:

```
NAME                                 TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.0.134.85   52.227.171.191   80:30697/TCP,443:30217/TCP   4m49s
ingress-nginx-controller-admission   ClusterIP      10.0.138.37   <none>           443/TCP                      4m50s
Create a deployment, scale it to 2 replicas and expose it as a service. This service will be ClusterIP and we'll expose this service via the Ingress.
kubectl create deployment hello-world-service-single --image=gcr.io/google-samples/hello-app:1.0
kubectl scale deployment hello-world-service-single --replicas=2
kubectl expose deployment hello-world-service-single --port=80 --target-port=8080 --type=ClusterIP
```

Create a single Ingress routing to the one backend service on the service port 80
```
kubectl apply -f ingress-single.yaml
```

Get the status of the ingress. It's routing for all host names on that public IP on port 80

```
kubectl get ingress
kubectl get services --namespace ingress-nginx
```

Example:

```
NAME                                 TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.0.134.85   52.227.171.191   80:30697/TCP,443:30217/TCP   10m
ingress-nginx-controller-admission   ClusterIP      10.0.138.37   <none>           443/TCP                      10m
```

Describe

```
kubectl describe ingress ingress-single
```

Access the application via the exposed ingress on the public IP
```
INGRESSIP=<EXTERNAL-IP>
curl http://$INGRESSIP
```

Create 2 additional services

```
kubectl create deployment hello-world-service-blue --image=gcr.io/google-samples/hello-app:1.0
kubectl create deployment hello-world-service-red  --image=gcr.io/google-samples/hello-app:1.0
kubectl expose deployment hello-world-service-blue --port=4343 --target-port=8080 --type=ClusterIP
kubectl expose deployment hello-world-service-red  --port=4242 --target-port=8080 --type=ClusterIP
```

Create an ingress with paths each routing to different backend services.
```
kubectl apply -f ingress-path.yaml
kubectl get ing
```

Example:

```
NAME             HOSTS              ADDRESS          PORTS   AGE
ingress-path     path.example.com   52.227.171.191   80      78s
ingress-single   *                  52.227.171.191   80      13m
```

Testing to see if it's work

Route traffic to the services using named based virtual hosts rather than paths

```
kubectl apply -f ingress-namebased.yaml
curl http://$INGRESSIP/ --header 'Host: red.example.com'
```
**Output:**

```
Hello, world!
Version: 1.0.0
Hostname: hello-world-service-red-56cc7b86b-dtprz
```

```
curl http://$INGRESSIP/ --header 'Host: blue.example.com'
```

**Output:**

```
Hello, world!
Version: 1.0.0
Hostname: hello-world-service-blue-7647475b7-hzcn9
```

