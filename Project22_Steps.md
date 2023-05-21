# DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER
PROJECT 22

## UNDERSTANDING THE CONCEPT

- The service object in Kubernetes routes traffic to pods.
- The service's type can be ClusterIP, which acts as an internal load balancer.
- The service forwards requests to pods based on their respective selector labels.
- Pods have virtual IP addresses assigned by Kubernetes network plugins.
- Pods can have different IP addresses from the servers they are running on.

<!--
Self Side Task
- Build the Tooling app Dockerfile and push it to Dockerhub registry.
- Write a Pod and a Service manifests to ensure access to the Tooling app's frontend using port-forwarding.
-->

#### Expose a Service on a Server's Public IP Address & Static Port
- NodePort service type exposes the service on a static port on the node's IP address.
- NodePorts are in the range of 30000-32767 by default.
- NodePort allows direct access to the application using the node's public IP address and appended port.
- Inbound traffic to the NodePort range needs to be allowed in the EC2's Security Group.

#### Maintaining Desired Number of Pods
- ReplicaSet (RS) object ensures a stable set of pod replicas running.
- RS guarantees the availability of a specified number of identical pods.

*Note: ReplicaSets are recommended over the older ReplicationController (RC) object.*

## COMMON KUBERNETES OBJECTS  

1. **Pod**: A Pod is the smallest deployable unit in Kubernetes. It represents a single instance of a running process in a cluster. Pods can contain one or more containers that are tightly coupled and share resources.

2. **Namespace**: A Namespace provides a way to group and isolate resources within a cluster. It allows multiple teams or projects to share the same cluster while keeping their resources separate.

3. **ReplicaSet**: A ReplicaSet ensures that a specified number of Pod replicas are running at all times. It is used to scale and manage the lifecycle of Pods. ReplicaSets are typically managed by higher-level controllers like Deployments.

4. **Deployment**: A Deployment provides declarative updates for Pods and ReplicaSets. It manages the creation and scaling of ReplicaSets, making it easier to manage application deployments and updates.

5. **StatefulSet**: A StatefulSet is similar to a ReplicaSet but is used for managing stateful applications. It provides guarantees about the ordering and uniqueness of Pods, making it suitable for applications that require stable network identities and persistent storage.

6. **DaemonSet**: A DaemonSet ensures that a copy of a Pod is running on each node in the cluster. It is useful for running system daemons or agents that need to be present on every node.

7. **Service**: A Service is an abstraction that defines a logical set of Pods and a policy for accessing them. It provides a stable network endpoint to access the Pods, allowing for load balancing and service discovery.

8. **ConfigMap**: A ConfigMap is used to store configuration data as key-value pairs. It allows you to separate configuration from the container images and make it easier to manage and update configurations.

9. **Volume**: A Volume is used to provide persistent storage for containers in a Pod. It allows data to outlive the lifecycle of individual containers and can be shared between containers within the same Pod.

10. **Job/CronJob**: A Job represents a task or a batch job that runs to completion. It ensures that a specified number of Pods successfully complete their tasks before terminating. CronJob is a type of Job that runs on a schedule.

The common YAML fields for every Kubernetes object include:

- `kind`: Specifies the type of Kubernetes object being created, such as Pod, Deployment, or Service.
- `version`: Indicates the version of the Kubernetes API used to create the resource.
- `metadata`: Provides information about the resource, such as its name, labels, and annotations.
- `spec`: Contains the core information about the resource, defining its desired state. This includes details like container images, number of replicas, environment variables, and volumes.
- `status`: Represents the current status of the object and is updated by Kubernetes after creation. This field is not typically included in the YAML manifest provided by the user.

These fields help define the characteristics and behavior of the Kubernetes objects and guide Kubernetes in managing and maintaining the desired state of the cluster.



## Kubernetes on AWS (EKS)

So following [Getting started with Amazon EKS – AWS Management Console and AWS CLI - Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html)

Once again we will utilize the existing **AWS CLI** setup *(from [Project 15](https://github.com/hectorproko/AWS-CLOUD-SOLUTION-FOR-2-COMPANY-WEBSITES-USING-A-REVERSE-PROXY-TECHNOLOGY/blob/main/Project15_Steps.md))* using sub-account **DevOps**

<!--
Need to put the part of setup
54. Kubernetes on AWS (EKS)
-->

Create Stack
``` css
hector@hector-Laptop:~/Project22$ aws cloudformation create-stack \
> --region us-east-1 \
> --stack-name my-eks-vpc-stack \
> --template-url https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml
{
    "StackId": "arn:aws:cloudformation:us-east-1:199055125796:stack/my-eks-vpc-stack/1fbf1cc0-1842-11ed-b05c-0e1c47e12f6b"
}
hector@hector-Laptop:~/Project22$
```

<!--kube user is for EKS not creating Stack
-->

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/stacks.png)


As per [aws documentation](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html)

Create a cluster IAM role and attach the required Amazon EKS IAM managed policy to it. Kubernetes clusters managed by Amazon EKS make calls to other AWS services on your behalf to manage the resources that you use with the service.

1. We create file `cluster-role-trust-policy.json`.  
    ```css
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "Service": "eks.amazonaws.com"
          },
          "Action": "sts:AssumeRole"
        }
      ]
    }
    ```


2. Created the role **myAmazonEKSClusterRole**.  
  `--assume-role-policy-document` is used during role creation to define the trust policy document, which determines who can assume the role.  

    ```css
    hector@hector-Laptop:~$ aws iam create-role \
    >   --role-name myAmazonEKSClusterRole \
    >   --assume-role-policy-document file://"cluster-role-trust-policy.json"
    {
        "Role": {
            "Path": "/",
            "RoleName": "myAmazonEKSClusterRole",
            "RoleId": "AROAS4WE4FUSEZXBGIH4R",
            "Arn": "arn:aws:iam::199055125796:role/myAmazonEKSClusterRole",
            "CreateDate": "2022-07-26T13:26:57Z",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "eks.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            }
        }
    }
    ```

3. Attach the required Amazon EKS managed IAM policy (**AmazonEKSClusterPolicy**) to the role (**myAmazonEKSClusterRole**).

    `attach-role-policy` is used to attach an IAM policy to an existing role, granting permissions and actions to the role.  

    ```css
    hector@hector-Laptop:~$ aws iam attach-role-policy \
    >   --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy \
    >   --role-name myAmazonEKSClusterRole
    hector@hector-Laptop:~$
    ```


**Creating Cluster** 

<details close>
<summary>Using the Console (web-based graphical user interface):</summary>

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/clusters.png)

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/configure.png)

Notice how the role we previously created appears as an option  

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
</details>


<details close>
<summary>Using AWS CLI:</summary>

```css
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
</details><br>

**Configuring Computer to Communicate with Kubernetes Cluster**  

- Deleted the existing kubeconfig file located at `~/.kube/config` to make way for a new configuration.  

- This AWS CLI command is used to update the kubeconfig file with the necessary configuration for accessing the EKS cluster named "Project22" in the AWS region "us-east-1". The --profile option specifies the AWS profile to use for authentication and authorization.  
  ``` css
  hector@hector-Laptop:~/Project22$ aws eks update-kubeconfig --profile kube --region us-east-1 --name Project22
  Added new context arn:aws:eks:us-east-1:199055125796:cluster/Project22 to /home/hector/.kube/config
  ```

- This kubectl command is used to retrieve information about the Kubernetes cluster. It provides details such as the URL of the Kubernetes control plane and the URL of CoreDNS, which is responsible for DNS resolution within the cluster.
  ``` css
  hector@hector-Laptop:~/Project22$ kubectl cluster-info
  Kubernetes control plane is running at https://522B9ADEF131F42CC77EB11C3FB33A42.gr7.us-east-1.eks.amazonaws.com
  CoreDNS is running at https://522B9ADEF131F42CC77EB11C3FB33A42.gr7.us-east-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

  To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
  hector@hector-Laptop:~/Project22$
  ```
  ![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/clusterinfo.png)

- I attempted to create a Pod in the Kubernetes cluster, the Pod remained in the "Pending" state.   
*(In Kubernetes, a "Pending" status means that the Pod has been scheduled to run on a node but is waiting for the necessary resources, such as CPU and memory, to become available)*  

- The cause was that the cluster had no active worker nodes available to schedule  and run Pods.  
*(Worker nodes are responsible for executing and hosting Pods in a Kubernetes cluster)*

- To address the problem, I had to create a Node group.  
  (Node groups are a way to provision and manage worker nodes in an Amazon EKS cluster. By creating a Node group, we added worker nodes to the cluster, providing the necessary resources for Pods to be scheduled and run)

  ![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/configureNodeGroup.png)
  *(Everything else defaults)*

  ![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/nodegroups.png)



<details close>
<summary>Once you delete the Node group, whatever pod was running disappears</summary>

``` css
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
```
</details>



## ACCESSING THE APP FROM THE BROWSER

Let's create a Pod named **nginx-pod** by defining a YAML manifest on master node.  

```css
sudo cat <<EOF | sudo tee ./nginx-pod.yaml
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
EOF
```
We should have a YAML file named nginx-pod.yaml with the necessary specifications for the Pod. It defines a single container running the latest version of the nginx image, named **nginx-pod**. Port 80 is exposed using the TCP protocol.

To create the Pod we apply the manifest using `kubectl`  
```css
kubectl apply -f nginx-pod.yaml
pod/nginx-pod created <<< output
```

To verify the status of the running Pods in the cluster `kubectl get pods`
```css
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          19m
```


<!--
We use **kubectl** to create a new Pod named "curl" and allocate an interactive shell within the container to run the `curl` command inside the container to perform a `GET` request to verify that the nginx service is up and running properly

<details close>
<summary>Output</summary>
##An example of using the IP directly to access the pod
``` css
hector@hector-Laptop:~/Project22$ kubectl run curl --image=dareyregistry/curl -i --tty
If you don't see a command prompt, try pressing enter.
/ # curl -v 192.168.13.153:80
> GET / HTTP/1.1
> User-Agent: curl/7.35.0
> Host: 192.168.13.153
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx/1.23.1
< Date: Wed, 10 Aug 2022 03:00:26 GMT
< Content-Type: text/html
< Content-Length: 615
< Last-Modified: Tue, 19 Jul 2022 14:05:27 GMT
< Connection: keep-alive
< ETag: "62d6ba27-267"
< Accept-Ranges: bytes
<
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
/ #
```
*(In the provided output, the response indicates an HTTP status code of 200 OK)*
</details>
-->



*Assuming that the requirement is to access the Nginx Pod internally, using the Pod’s IP address directly is not a reliable choice because Pods are ephemeral. They are not designed to run forever. When they die and another Pod is brought back up, the IP address will change and any application that is using the previous IP address directly will break.*  
*To solve this problem, kubernetes uses **Service** – An object that abstracts the underlining IP addresses of Pods. A service can serve as a load balancer, and a reverse proxy which basically takes the request using a human readable DNS name, resolves to a Pod IP that is running and forwards the request to it. This way, you do not need to use an IP address. Rather, you can simply refer to the service name directly.*


Since we want to provide access to the **Nginx Pod** from the outside world, such as a web browser, we create a Service.

1. Create a Service `yaml` manifest file `nginx-service.yaml`


``` css
hector@hector-Laptop:~/Project22$ cat nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-pod
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply the manifest using the `kubectl apply -f nginx-service.yaml` command. This creates the Service in the Kubernetes cluster.
```
hector@hector-Laptop:~/Project22$ kubectl apply -f nginx-service.yaml
service/nginx-service created
```

To verify that the Service is created, run `kubectl get service` command. This will list the Services in your cluster, including the newly created nginx-service.
```
hector@hector-Laptop:~/Project22$ kubectl get service
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes      ClusterIP   10.100.0.1     <none>        443/TCP   56m
nginx-service   ClusterIP   10.100.15.31   <none>        80/TCP    64s
```
*(The output will show the CLUSTER-IP, PORT(S), and other information about the Service. Note that the EXTERNAL-IP is \<none> at this stage, indicating that the Service is not yet externally accessible)*  

Attempting to port forward using `kubectl port-forward` command to forward traffic from local port 8089 to port 80 of the nginx-service results in a timeout error. This error occurs because the Pod associated with the Service does not have the necessary labels for the Service to select it.
```
hector@hector-Laptop:~/Project22$ kubectl port-forward svc/nginx-service 8089:80
error: timed out waiting for the condition
```

To establish the required connection, you need to modify the Pod manifest to include **labels** that align with the **selectors** specified in the Service manifest.

1. Deleted the existing Pod to start fresh `kubectl delete pod nginx-pod`
2. Updated the YAML manifest file of the pod `nginx-pod.yaml` with the necessary changes (added labels)
    ``` css
    hector@hector-Laptop:~/Project22$ cat nginx-pod.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-pod
      labels:
        app: nginx-pod
    spec:
      containers:
      - image: nginx:latest
        name: nginx-pod
        ports:
        - containerPort: 80
          protocol: TCP
    ```

3. Applied the updated manifest file creating a new pod 
    ```css
    hector@hector-Laptop:~/Project22$ kubectl apply -f nginx-pod.yaml
    pod/nginx-pod created
    ```

This output signifies that now the port forwarding is functioning correctly. Requests made to `127.0.0.1:8089` or `[::1]:8089` on your local machine will be redirected to port 80 of the Nginx service.  
```css
hector@hector-Laptop:~/Project22$ kubectl  port-forward svc/nginx-service 8089:80
Forwarding from 127.0.0.1:8089 -> 80
Forwarding from [::1]:8089 -> 80
Handling connection for 8089
Handling connection for 8089
```



When we execute the command `lynx 127.0.0.1:8089`, the Nginx web page will appear in the lynx text-based web browser. This confirms that we can now access the Nginx service running on the Pod through the forwarded port.  

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/testingNginxPod.gif)






## CREATE A REPLICA SET

Let us create a **rs.yaml** manifest for a ReplicaSet object. ReplicaSet (RS) object ensures a stable set of pod replicas running

```css
# Part 1
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
# Part 2
  template:
    metadata:
      name: nginx-pod
      labels:
        app: nginx-pod
    spec:
      containers:
      - image: nginx:latest
        name: nginx-pod
        ports:
        - containerPort: 80
          protocol: TCP

```

The ReplicaSet named **nginx-rs** was created successfully.  
```css
hector@hector-Laptop:~/Project22$ kubectl apply -f rs.yaml
replicaset.apps/nginx-rs created
```
Listed the pods using `kubectl get pods` command. It shows the status of the existing pods, including nginx-pod and the pods managed by the ReplicaSet (**nginx-rs-6qshv** and **nginx-rs-ch9tp**).  
``` css
hector@hector-Laptop:~/Project22$ kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
nginx-pod        1/1     Running   0          49m
nginx-rs-6qshv   1/1     Running   0          17m
nginx-rs-ch9tp   1/1     Running   0          17m
```

Deleting one of the ReplicaSet pods **nginx-rs-ch9tp** 
```
hector@hector-Laptop:~/Project22$ kubectl delete pod nginx-rs-ch9tp
pod "nginx-rs-ch9tp" deleted
```

Listing the pods it shows that a new ReplicaSet pod **nginx-rs-tqvs8** was created to maintain the desired number of replicas.  
```
hector@hector-Laptop:~/Project22$ kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
nginx-pod        1/1     Running   0          50m
nginx-rs-6qshv   1/1     Running   0          18m
nginx-rs-tqvs8   1/1     Running   0          21s
```

Listing the ReplicaSets shows **nginx-rs** with the desired, current, and ready replicas set to 3. 
```
hector@hector-Laptop:~/Project22$ kubectl get rs -o wide
NAME       DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES         SELECTOR
nginx-rs   3         3         3       19m   nginx-pod    nginx:latest   app=nginx-pod
```

```
hector@hector-Laptop:~/Project22$ kubectl describe rs nginx-rs
Name:         nginx-rs
Namespace:    default
Selector:     app=nginx-pod
Labels:       <none>
Annotations:  <none>
Replicas:     3 current / 3 desired
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=nginx-pod
  Containers:
   nginx-pod:
    Image:        nginx:latest
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  19m   replicaset-controller  Created pod: nginx-rs-6qshv
  Normal  SuccessfulCreate  19m   replicaset-controller  Created pod: nginx-rs-ch9tp
  Normal  SuccessfulCreate  106s  replicaset-controller  Created pod: nginx-rs-tqvs8
```

**Scaling the ReplicaSet:** We can use the **imperative** command `kubectl scale` to scale up the ReplicaSet named **nginx-rs** to have 5 replicas.  

``` css
hector@hector-Laptop:~/Project22$ kubectl scale rs nginx-rs --replicas=5
replicaset.apps/nginx-rs scaled
```

There are now a total of 5 pods running, including the original nginx-pod and the newly created pods by the ReplicaSet.
```
hector@hector-Laptop:~/Project22$ kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
nginx-pod        1/1     Running   0          54m
nginx-rs-6qshv   1/1     Running   0          23m
nginx-rs-gzn6q   1/1     Running   0          26s
nginx-rs-hkpfm   1/1     Running   0          26s
nginx-rs-tqvs8   1/1     Running   0          5m6s
hector@hector-Laptop:~/Project22$
```
The ReplicaSet nginx-rs has been scaled to have 5 replicas.
``` css
hector@hector-Laptop:~/Project22$ kubectl get rs
NAME       DESIRED   CURRENT   READY   AGE
nginx-rs   5         5         5       29m
```

Deleting previous ReplicaSet
```
hector@hector-Laptop:~/Project22$ kubectl delete rs nginx-rs
replicaset.apps "nginx-rs" deleted
hector@hector-Laptop:~/Project22$
```

The new ReplicaSet manifest `rs2.yaml` introduces more advanced label selection and customization options.
``` css
hector@hector-Laptop:~/Project22$ cat rs2.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      env: prod
    matchExpressions:
    - { key: tier, operator: In, values: [frontend] }
  template:
    metadata:
      name: nginx
      labels:
        env: prod
        tier: frontend
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
          protocol: TCP
```

The new ReplicaSet **nginx-rs** with the advanced label selection and customization options has been created.  
```
hector@hector-Laptop:~/Project22$ kubectl apply -f rs2.yaml
replicaset.apps/nginx-rs created
```

The ReplicaSet `nginx-rs` now has a desired replica count of 3, and all replicas are running and ready. The selector specifies that the replicas should have labels matching `env=prod` and `tier=frontend`.  
```
hector@hector-Laptop:~/Project22$ kubectl get rs nginx-rs -o wide
NAME       DESIRED   CURRENT   READY   AGE   CONTAINERS        IMAGES         SELECTOR
nginx-rs   3         3         3       15s   nginx-container   nginx:latest   env=prod,tier in (frontend)
```


## USING AWS LOAD BALANCER TO ACCESS YOUR SERVICE IN KUBERNETES.

We previously used the ClusterIP service type to access the Nginx service internally. Now, we'll switch to the LoadBalancer service type, which creates an actual load balancer in AWS. This allows us to expose the Nginx service to the external world and benefit from load balancing capabilities provided by the external load balancer. <br>

New service manifest using the LoadBalancer type:  
``` css
hector@hector-Laptop:~/Project22$ cat nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    tier: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Applying the manifest to create the service:  
```
hector@hector-Laptop:~/Project22$ kubectl apply -f nginx-service.yaml
service/nginx-service configured
```

Retrieving information about the service shows it is of type LoadBalancer:  
```
hector@hector-Laptop:~/Project22$ kubectl get service nginx-service
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP                                                              PORT(S)        AGE
nginx-service   LoadBalancer   10.100.15.31   a0e08a526ccb04426acb64895c03dc0d-651336585.us-east-1.elb.amazonaws.com   80:30466/TCP   95m
```



![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/createLB.png)

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/description.png)

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/tags.png)  

<details close>
<summary>kubectl get service nginx-service -o yaml</summary>

``` css
hector@hector-Laptop:~/Project22$ kubectl get service nginx-service -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"nginx-service","namespace":"default"},"spec":{"ports":[{"port":80,"protocol":"TCP","targetPort":80}],"selector":{"tier":"frontend"},"type":"LoadBalancer"}}
  creationTimestamp: "2022-08-10T03:04:53Z"
  finalizers:
  - service.kubernetes.io/load-balancer-cleanup
  name: nginx-service
  namespace: default
  resourceVersion: "25770"
  uid: 0e08a526-ccb0-4426-acb6-4895c03dc0da
spec:
  allocateLoadBalancerNodePorts: true
  clusterIP: 10.100.15.31
  clusterIPs:
  - 10.100.15.31
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 30466
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    tier: frontend
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - hostname: a0e08a526ccb04426acb64895c03dc0d-651336585.us-east-1.elb.amazonaws.com
hector@hector-Laptop:~/Project22$
```
</details>

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/testingNginxLoadBalancer.gif)  




## USING DEPLOYMENT CONTROLLERS

If I scale to 15 with the name of the replicate set, it is brought down to 3, because the replicaset was a result of a deployment and the demployment is set to 3, so terminates untils it goes down to 3

<details close>
<summary>Multiple-Output</summary>

``` css
hector@hector-Laptop:~/Project22$ kubectl delete rs nginx-rs
replicaset.apps "nginx-rs" deleted

hector@hector-Laptop:~/Project22$ nano deployment.yaml
hector@hector-Laptop:~/Project22$ cat deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 8

hector@hector-Laptop:~/Project22$ kubectl apply -f deployment.yaml
deployment.apps/nginx-deployment created

hector@hector-Laptop:~/Project22$ kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           20s

hector@hector-Laptop:~/Project22$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-5cb44ffccf   3         3         3       32s

hector@hector-Laptop:~/Project22$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5cb44ffccf-4m86n   1/1     Running   0          44s
nginx-deployment-5cb44ffccf-8rrkf   1/1     Running   0          44s
nginx-deployment-5cb44ffccf-p8w6q   1/1     Running   0          44s

hector@hector-Laptop:~/Project22$ kubectl scale rs nginx-deployment-5cb44ffccf --replicas=15
replicaset.apps/nginx-deployment-5cb44ffccf scaled

hector@hector-Laptop:~/Project22$ kubectl get pods
NAME                                READY   STATUS        RESTARTS   AGE
nginx-deployment-5cb44ffccf-4m86n   1/1     Running       0          2m16s
nginx-deployment-5cb44ffccf-87bqs   1/1     Terminating   0          6s
nginx-deployment-5cb44ffccf-8rrkf   1/1     Running       0          2m16s
nginx-deployment-5cb44ffccf-lsgcc   1/1     Terminating   0          6s
nginx-deployment-5cb44ffccf-mcwjr   1/1     Terminating   0          6s
nginx-deployment-5cb44ffccf-p8w6q   1/1     Running       0          2m16s
nginx-deployment-5cb44ffccf-pr2lf   1/1     Terminating   0          6s
nginx-deployment-5cb44ffccf-qjhrl   1/1     Terminating   0          6s
nginx-deployment-5cb44ffccf-wvlzn   1/1     Terminating   0          6s

hector@hector-Laptop:~/Project22$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5cb44ffccf-4m86n   1/1     Running   0          2m41s
nginx-deployment-5cb44ffccf-8rrkf   1/1     Running   0          2m41s
nginx-deployment-5cb44ffccf-p8w6q   1/1     Running   0          2m41s

hector@hector-Laptop:~/Project22$ kubectl scale rs nginx-deployment-5cb44ffccf --replicas=15
replicaset.apps/nginx-deployment-5cb44ffccf scaled

hector@hector-Laptop:~/Project22$ kubectl get pods
NAME                                READY   STATUS        RESTARTS   AGE
nginx-deployment-5cb44ffccf-4m86n   1/1     Running       0          3m49s
nginx-deployment-5cb44ffccf-7rst5   1/1     Terminating   0          6s
nginx-deployment-5cb44ffccf-8rrkf   1/1     Running       0          3m49s
nginx-deployment-5cb44ffccf-cqjvx   0/1     Terminating   0          5s
nginx-deployment-5cb44ffccf-jzs7p   1/1     Terminating   0          6s
nginx-deployment-5cb44ffccf-lxt8f   0/1     Terminating   0          5s
nginx-deployment-5cb44ffccf-p8w6q   1/1     Running       0          3m49s
nginx-deployment-5cb44ffccf-t4g4j   1/1     Terminating   0          5s
nginx-deployment-5cb44ffccf-vllkl   1/1     Terminating   0          6s
nginx-deployment-5cb44ffccf-xk4kh   1/1     Terminating   0          5s
nginx-deployment-5cb44ffccf-z4s2s   1/1     Terminating   0          5s
hector@hector-Laptop:~/Project22$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-5cb44ffccf   3         3         3       4m11s

hector@hector-Laptop:~/Project22$ kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           5m52s
hector@hector-Laptop:~/Project22$ kubectl scale rs nginx-deployment --replicas=15
Error from server (NotFound): replicasets.apps "nginx-deployment" not found
hector@hector-Laptop:~/Project22$
```
</details>

<details close>
<summary>Exec into one of the Pod’s container to run Linux commands</summary>

``` css
hector@hector-Laptop:~/Project22$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5cb44ffccf-4m86n   1/1     Running   0          7m23s
nginx-deployment-5cb44ffccf-8rrkf   1/1     Running   0          7m23s
nginx-deployment-5cb44ffccf-p8w6q   1/1     Running   0          7m23s

hector@hector-Laptop:~/Project22$ kubectl exec -it nginx-deployment-5cb44ffccf-4m86n bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
```
</details>

<details close>
<summary>nginx-deployment-5cb44ffccf-4m86n</summary>

```bash
root@nginx-deployment-5cb44ffccf-4m86n:/# ls -ltr /etc/nginx/
total 24
-rw-r--r-- 1 root root  664 Jul 19 14:05 uwsgi_params
-rw-r--r-- 1 root root  636 Jul 19 14:05 scgi_params
-rw-r--r-- 1 root root 5349 Jul 19 14:05 mime.types
-rw-r--r-- 1 root root 1007 Jul 19 14:05 fastcgi_params
-rw-r--r-- 1 root root  648 Jul 19 15:06 nginx.conf
lrwxrwxrwx 1 root root   22 Jul 19 15:06 modules -> /usr/lib/nginx/modules
drwxr-xr-x 1 root root   26 Aug 10 04:53 conf.d
```

```bash
root@nginx-deployment-5cb44ffccf-4m86n:/# cat  /etc/nginx/conf.d/default.conf
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}

root@nginx-deployment-5cb44ffccf-4m86n:/#
```
</details>

## PERSISTING DATA FOR PODS

<details close>
<summary>Multiple-Output</summary>

``` css
hector@hector-Laptop:~/Project22$ kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           20m


hector@hector-Laptop:~/Project22$ kubectl autoscale deployment nginx-deployment --max=1 --min=1
horizontalpodautoscaler.autoscaling/nginx-deployment autoscaled

hector@hector-Laptop:~/Project22$ kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           21m

hector@hector-Laptop:~/Project22$ kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           21m

hector@hector-Laptop:~/Project22$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5cb44ffccf-ws8b5   1/1     Running   0          9m9s
hector@hector-Laptop:~/Project22$
```
</details>

How I figured out to scale https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/

`kubectl exec -it nginx-deployment-5cb44ffccf-4m86n bash`


<details close>
<summary>kubectl exec -it nginx-deployment-5cb44ffccf-ws8b5 bash</summary>

``` css
hector@hector-Laptop:~/Project22$ kubectl exec -it nginx-deployment-5cb44ffccf-ws8b5 bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@nginx-deployment-5cb44ffccf-ws8b5:/# apt-get update
Get:1 http://deb.debian.org/debian bullseye InRelease [116 kB]
Get:2 http://deb.debian.org/debian-security bullseye-security InRelease [48.4 kB]
Get:3 http://deb.debian.org/debian bullseye-updates InRelease [44.1 kB]
Get:4 http://deb.debian.org/debian bullseye/main amd64 Packages [8182 kB]
Get:5 http://deb.debian.org/debian-security bullseye-security/main amd64 Packages [175 kB]
Get:6 http://deb.debian.org/debian bullseye-updates/main amd64 Packages [2592 B]
Fetched 8567 kB in 2s (5092 kB/s)
Reading package lists... Done
root@nginx-deployment-5cb44ffccf-ws8b5:/# apt-get install vim
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  libgpm2 vim-common vim-runtime xxd
Suggested packages:
  gpm ctags vim-doc vim-scripts
The following NEW packages will be installed:
  libgpm2 vim vim-common vim-runtime xxd
0 upgraded, 5 newly installed, 0 to remove and 3 not upgraded.
Need to get 8174 kB of archives.
After this operation, 36.9 MB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://deb.debian.org/debian bullseye/main amd64 xxd amd64 2:8.2.2434-3+deb11u1 [192 kB]
Get:2 http://deb.debian.org/debian bullseye/main amd64 vim-common all 2:8.2.2434-3+deb11u1 [226 kB]
Get:3 http://deb.debian.org/debian bullseye/main amd64 libgpm2 amd64 1.20.7-8 [35.6 kB]
Get:4 http://deb.debian.org/debian bullseye/main amd64 vim-runtime all 2:8.2.2434-3+deb11u1 [6226 kB]
Get:5 http://deb.debian.org/debian bullseye/main amd64 vim amd64 2:8.2.2434-3+deb11u1 [1494 kB]
Fetched 8174 kB in 0s (91.8 MB/s)
debconf: delaying package configuration, since apt-utils is not installed
Selecting previously unselected package xxd.
(Reading database ... 7823 files and directories currently installed.)
Preparing to unpack .../xxd_2%3a8.2.2434-3+deb11u1_amd64.deb ...
Unpacking xxd (2:8.2.2434-3+deb11u1) ...
Selecting previously unselected package vim-common.
Preparing to unpack .../vim-common_2%3a8.2.2434-3+deb11u1_all.deb ...
Unpacking vim-common (2:8.2.2434-3+deb11u1) ...
Selecting previously unselected package libgpm2:amd64.
Preparing to unpack .../libgpm2_1.20.7-8_amd64.deb ...
Unpacking libgpm2:amd64 (1.20.7-8) ...
Selecting previously unselected package vim-runtime.
Preparing to unpack .../vim-runtime_2%3a8.2.2434-3+deb11u1_all.deb ...
Adding 'diversion of /usr/share/vim/vim82/doc/help.txt to /usr/share/vim/vim82/doc/help.txt.vim-tiny by vim-runtime'
Adding 'diversion of /usr/share/vim/vim82/doc/tags to /usr/share/vim/vim82/doc/tags.vim-tiny by vim-runtime'
Unpacking vim-runtime (2:8.2.2434-3+deb11u1) ...
Selecting previously unselected package vim.
Preparing to unpack .../vim_2%3a8.2.2434-3+deb11u1_amd64.deb ...
Unpacking vim (2:8.2.2434-3+deb11u1) ...
Setting up libgpm2:amd64 (1.20.7-8) ...
Setting up xxd (2:8.2.2434-3+deb11u1) ...
Setting up vim-common (2:8.2.2434-3+deb11u1) ...
Setting up vim-runtime (2:8.2.2434-3+deb11u1) ...
Setting up vim (2:8.2.2434-3+deb11u1) ...
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vim (vim) in auto mode
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vimdiff (vimdiff) in auto mode
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/rvim (rvim) in auto mode
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/rview (rview) in auto mode
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vi (vi) in auto mode
update-alternatives: warning: skip creation of /usr/share/man/da/man1/vi.1.gz because associated file /usr/share/man/da/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/de/man1/vi.1.gz because associated file /usr/share/man/de/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/fr/man1/vi.1.gz because associated file /usr/share/man/fr/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/it/man1/vi.1.gz because associated file /usr/share/man/it/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ja/man1/vi.1.gz because associated file /usr/share/man/ja/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/pl/man1/vi.1.gz because associated file /usr/share/man/pl/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ru/man1/vi.1.gz because associated file /usr/share/man/ru/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/man1/vi.1.gz because associated file /usr/share/man/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/view (view) in auto mode
update-alternatives: warning: skip creation of /usr/share/man/da/man1/view.1.gz because associated file /usr/share/man/da/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/de/man1/view.1.gz because associated file /usr/share/man/de/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/fr/man1/view.1.gz because associated file /usr/share/man/fr/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/it/man1/view.1.gz because associated file /usr/share/man/it/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ja/man1/view.1.gz because associated file /usr/share/man/ja/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/pl/man1/view.1.gz because associated file /usr/share/man/pl/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ru/man1/view.1.gz because associated file /usr/share/man/ru/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/man1/view.1.gz because associated file /usr/share/man/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/ex (ex) in auto mode
update-alternatives: warning: skip creation of /usr/share/man/da/man1/ex.1.gz because associated file /usr/share/man/da/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/de/man1/ex.1.gz because associated file /usr/share/man/de/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/fr/man1/ex.1.gz because associated file /usr/share/man/fr/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/it/man1/ex.1.gz because associated file /usr/share/man/it/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ja/man1/ex.1.gz because associated file /usr/share/man/ja/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/pl/man1/ex.1.gz because associated file /usr/share/man/pl/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ru/man1/ex.1.gz because associated file /usr/share/man/ru/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/man1/ex.1.gz because associated file /usr/share/man/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/editor (editor) in auto mode
update-alternatives: warning: skip creation of /usr/share/man/da/man1/editor.1.gz because associated file /usr/share/man/da/man1/vim.1.gz (of link group editor) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/de/man1/editor.1.gz because associated file /usr/share/man/de/man1/vim.1.gz (of link group editor) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/fr/man1/editor.1.gz because associated file /usr/share/man/fr/man1/vim.1.gz (of link group editor) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/it/man1/editor.1.gz because associated file /usr/share/man/it/man1/vim.1.gz (of link group editor) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ja/man1/editor.1.gz because associated file /usr/share/man/ja/man1/vim.1.gz (of link group editor) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/pl/man1/editor.1.gz because associated file /usr/share/man/pl/man1/vim.1.gz (of link group editor) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ru/man1/editor.1.gz because associated file /usr/share/man/ru/man1/vim.1.gz (of link group editor) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/man1/editor.1.gz because associated file /usr/share/man/man1/vim.1.gz (of link group editor) doesn't exist
Processing triggers for libc-bin (2.31-13+deb11u3) ...
```
</details>

<details close>
<summary>cat /usr/share/nginx/html/index.html</summary>

``` css
root@nginx-deployment-5cb44ffccf-ws8b5:/# cat /usr/share/nginx/html/index.html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to an EDITED PAGE!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
root@nginx-deployment-5cb44ffccf-ws8b5:/#
```
</details>

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/editedPage.png
)

![logo](https://raw.githubusercontent.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/main/images/editedNginxPage.gif)

