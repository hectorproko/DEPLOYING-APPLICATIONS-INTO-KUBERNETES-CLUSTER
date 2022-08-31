# DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER
PROJECT 22
<!--
Need to put the part of setup
54. Kubernetes on AWS (EKS)
-->

### DEPLOYING APPLICATIONS INTO KUBERNETES CLUSTER
### UNDERSTANDING THE CONCEPT
### COMMON KUBERNETES OBJECTS

``` bash
hector@hector-Laptop:~/Project22$ aws cloudformation create-stack \
> --region us-east-1 \
> --stack-name my-eks-vpc-stack \
> --template-url https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml
{
    "StackId": "arn:aws:cloudformation:us-east-1:199055125796:stack/my-eks-vpc-stack/1fbf1cc0-1842-11ed-b05c-0e1c47e12f6b"
}
hector@hector-Laptop:~/Project22$
```

<!--kube user is for EKS not creatin Stack
-->
![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/stacks.png)

Creating Cluster From Console

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/clusters.png)

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/configure.png)

Keep in mind we created this service roel already therefore It appears there

Click **Next** to **Specify networking**

Make sure I pick the VPC created by the stack leave everything else default

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/specifynetworking.png)

Click **Next** to  **Configure logging (leave defaults)**

Click **Next** to **Review and create**

Click **Create**

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/project22cluster.png)

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/project22cluster2.png)

We need to specify the IDs of the VPC we want when creating cluster from **AWS CLI**
![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/subnets.png)

Create Cluster:
``` bash
aws eks create-cluster --profile kube --region us-east-1 --name Project22 --kubernetes-version 1.22 \
   --role-arn arn:aws:iam::199055125796:role/myAmazonEKSClusterRole \
   --resources-vpc-config subnetIds=subnet-039252ecb19e19d4e,subnet-09d3ea8fadca3b869,subnet-0c015424187074885,subnet-040dadfc9ad38ed59
```

Output:
```bash
hector@hector-Laptop:~/Project22$ aws eks create-cluster --profile kube --region us-east-1 --name Project22 --kubernetes-version 1.22 \
>    --role-arn arn:aws:iam::199055125796:role/myAmazonEKSClusterRole \
>    --resources-vpc-config subnetIds=subnet-039252ecb19e19d4e,subnet-09d3ea8fadca3b869,subnet-0c015424187074885,subnet-040dadfc9ad38ed59
CLUSTER arn:aws:eks:us-east-1:199055125796:cluster/Project22    2022-08-09T22:03:06.047000-04:00        Project22       eks.5   arn:aws:iam::199055125796:role/myAmazonEKSClusterRole   CREATING        1.22
KUBERNETESNETWORKCONFIG ipv4    10.100.0.0/16
CLUSTERLOGGING  False
TYPES   api
TYPES   audit
TYPES   authenticator
TYPES   controllerManager
TYPES   scheduler
RESOURCESVPCCONFIG      False   True    vpc-0b531a7a1ca65e1c8
PUBLICACCESSCIDRS       0.0.0.0/0
SUBNETIDS       subnet-039252ecb19e19d4e
SUBNETIDS       subnet-09d3ea8fadca3b869
SUBNETIDS       subnet-0c015424187074885
SUBNETIDS       subnet-040dadfc9ad38ed59
hector@hector-Laptop:~/Project22$
```

Configuring Computer to communicate with cluster 

Deleting current `~/.kube/config` since we already backed it 

``` bash
hector@hector-Laptop:~/Project22$ aws eks update-kubeconfig --profile kube --region us-east-1 --name Project22
Added new context arn:aws:eks:us-east-1:199055125796:cluster/Project22 to /home/hector/.kube/config
```
``` bash
hector@hector-Laptop:~/Project22$ kubectl cluster-info
Kubernetes control plane is running at https://522B9ADEF131F42CC77EB11C3FB33A42.gr7.us-east-1.eks.amazonaws.com
CoreDNS is running at https://522B9ADEF131F42CC77EB11C3FB33A42.gr7.us-east-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
hector@hector-Laptop:~/Project22$
```
![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/clusterinfo.png)

The creation of the POD said pending, we still had no workers in the cluster, that might be it, **confirmed**.
Created Node groups and it worked
![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/nodegroups.png)

Trying with just node, cant, only Node Groups


Redoing the Node Group

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/configureNodeGroup.png)

Everything else defaults

Once you delete the Node group, whatever pod was running disappears

``` bash
hector@hector-Laptop:~/Project22$ cat nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - image: nginx:latest
      name: nginx-pod
      ports:
      - containerPort: 80
        protocol: TCP
hector@hector-Laptop:~/Project22$ kubectl apply -f nginx-pod.yaml
pod/nginx-pod created
hector@hector-Laptop:~/Project22$ kubectl get pods -o wide
NAME        READY   STATUS              RESTARTS   AGE   IP       NODE                            NOMINATED NODE   READINESS GATES
nginx-pod   0/1     ContainerCreating   0          7s    <none>   ip-192-168-10-26.ec2.internal   <none>           <none>
hector@hector-Laptop:~/Project22$ kubectl get pods -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP               NODE                            NOMINATED NODE   READINESS GATES
nginx-pod   1/1     Running   0          31s   192.168.13.153   ip-192-168-10-26.ec2.internal   <none>           <none>
hector@hector-Laptop:~/Project22$
```



### ACCESSING THE APP FROM THE BROWSER
### CREATE A REPLICA SET
### USING AWS LOAD BALANCER TO ACCESS YOUR SERVICE IN KUBERNETES.
### USING DEPLOYMENT CONTROLLERS
### PERSISTING DATA FOR PODS
