
We finally get to run some stuff on our newly installed Kubernentes cluseter. Let's see what Kubernetes can do for us!



### Pods

Pods are the smallest deployable units in Kubernetes that can be created, scheduled, and managed. They're often used to host a single instance of an application.

Let's get a better understanding of Kubernetes architecture by running this command:
```
kubectl get pods -n kube-system
```

```
kubectl get pods -n kube-system -o wide
```

### Deploy Pods

```
kubectl run nginx --image nginx:latest
```

```
kubectl get pods
```

```
kubectl get pods -o wide
```

Let's deploy another

```
kubectl run apache --image httpd:latest
```

  
What should we use to “expose” our apps?

  

Use a service (NodePort, Loadbalancer, ClusterIP, EndPoint, Ingress).


Let's create one more pod.
```
kubectl run redis --image redis:latest
```

As an industry practice you don't always want to use the latest images for applications you should be defining the specific version of what you're going to be using for your application and stay with that to avoid any stability issues.

```
kubectl get pods --field-selector=spec.nodeName=kube-node02
```

### Exposing Apps

You've mentioned some ways to expose apps. Here's a bit more detail:

- **NodePort**: Exposes the service on each Node’s IP at a static port.
- **LoadBalancer**: Exposes the service externally using a cloud provider's load balancer.
- **ClusterIP**: Exposes the service on a cluster-internal IP. Suitable for intra-cluster communication.
- **Ingress**: Manages external access to services in a cluster, typically HTTP.


### Namespaces
You cant have an object with the same name in the same namespace.

**Were namespaces can be particularly useful is when we're trying to divide between a production environment and a Dev or testing environment so what we're going to do next is give you a command to create a name space and then to show you that we can use an object with the same name in different namespaces we're going to recreate our Apache pod in our testing namespace**

```
kubectl create namespace testing
```

```
kubectl run apache --image httpd:latest -n testing
```


### YAML Files

**We're using yaml files we're going to be writing our configuration information declaratively within the yaml file and then we will be using a specific kubernetes command to use that yaml file to create our object.**

apache-pod.yml
```
---

apiVersion: v1

kind: Pod

metadata:

name: apache

labels:

app: webserver

spec:

containers:

- name: apache

image: httpd:latest

ports:

- containerPort: 80

```

```

apiVersion: v1

kind: Service

metadata:

name: web-service

spec:

selector:

app: webserver

ports:

- protocol: TCP

port: 80

targetPort: 80

```

Deploy pod and service

```
kubectl create -f apache-pod.yml

kubectl create -f apache-service.yml

```


View services

```
kubectl get services
```

Change service port of service

```
kubectl edit service/web-service
```

**So at this point we changed a service port from 80 to 80 80 but we might want to be able to do some troubleshooting on a pod or a service and so I'm going to give you some commands that are used for this the commands in questions are called the describe a commands and we can use them against pods nodes services so on and so forth**

```
kubectl describe pod apache | less
```

```
kubectl decribe node kube-node-1 | less
```


### Clean up

```
kubectl delete pod --all
```

```
kubectl delete namespace **name of namespace**
```

```
kubectl delete service web-service

```
Certainly! Below is the markdown document you requested. It covers topics such as scheduling, node selection, affinities, taints, tolerations, and deployments in Kubernetes.

---

## Scheduler

**Default schedule choose the node with the fewest pods.**

* **Does the number of pods really tell us how a node is being used?** No, it doesn't necessarily tell us.

## Manual Scheduling

So we have several kinds of scheduling, but at this point, we're going to go ahead and schedule a manual scheduling by modifying our pod configuration and we're going to use the YAML file below 👍

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
  nodeName: ub02
```

### Exercise: Trying to assign this pod to worker node ub03

## Node Selector 

When we try to configure or schedule a pod without configuring the label on the Node first we're going to be stuck in this pending State forever, and the only way we can really troubleshoot it is to use the `kubectl describe` command for the pod in question.

### Troubleshooting:

```bash
kubectl describe pods apache
```

That kubelet is pulling our image and it's now going to assign

```bash
kubectl label node ub02 type=webserver
```

So after we assign this label to the node we see that kubelet Is pulling our image and it is now going to assign it to a node that has the matching label.

### Exercise: Configure a nodeSelector yaml for ub03

## Advanced Node Selection & Affinities

So I'm going to provide you with a sample node Affinity yaml file.

### YAML File requiredDuringSchedulingIgnoredDuringExecution:

```yaml
apiVersion: v1
kind: Pod
metadata:
   name: nginx
   labels:
     env: dev
spec:
   containers:
     - name: nginx
       image: nginx:latest
   affinity:
     nodeAffinity:
       requiredDuringSchedulingIgnoredDuringExecution:
         nodeSelectorTerms:
           - matchExpressions:
             - key: size
               operator: In
               values:
               - large
               - small
```

### YAML File preferredDuringSchedulingIgnoredDuringExecution:

```yaml
apiVersion: v1
kind: Pod
Metadata:
  name: nginx
  labels:
    env: dev
spec:
  containers:
    - name: nginx
      image: nginx:latest
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
            - key: type
              operator: In
              values:
               - large
               - small
```

### Affinity types in Kubernetes v1.28

- requiredDuringSchedulingRequiredDuringExecution
- preferredDuringSchedulingIgnoredDuringExecution
- requiredDuringSchedulingRequiredDuringExecution
- preferredDuringSchedulingRequiredDuringExecution

## Service Troubleshooting with BRD

### Deployment File (YAML): api-gateway-deployment.yml

```yaml
apiVersion: apps/v1
kind: Deployment
# Labels for deployment
metadata:
  name: buckapp
  labels:
    app: apigateway
spec:
  replicas: 1
# Labels for Pod (App/Container)
  selector:
    matchLabels:
      app: apigateway
  template:
    metadata:
# Labels for exposing app (Service)
      labels:
        app: apigateway
    spec:
      containers:
      - name: apigateway
# Harbor image
        image: gateway
        ports:
        - containerPort: 80
```

### Service file (YAML): api-gateway-service.yml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-gateway-service
spec:
  selector:
    app: apigateway
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```


## Taints and Tolerations

So here is a pod YAML configuration file where we're configuring

 this toleration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoExecute"
```

This toleration allows the pod to run on a node that has a taint with the key "key", the value "value", and the effect "NoExecute". If the taint effect is "NoExecute", it means that the pod will continue to run if it is already running on the node when the taint is added.


Certainly! Below, you'll find the markdown document detailing the process and configuration for handling taints and tolerations, including scenarios with different GPU types and deployments in Kubernetes.

---

### Tainting Nodes

Tainting a node means marking it so that only specific pods with a matching toleration can be scheduled on it. This can be useful for controlling where certain workloads run in the cluster.

**Example:**

```bash
kubectl taint nodes ub02 size=small:NoSchedule
```

### Understanding Taints and Tolerations

A taint and toleration must match for a pod to be scheduled on a tainted node.

**Example of Non-Matching Taint and Toleration:**

- **Toleration:** load=low:NoSchedule
- **Taint:** size=small:NoSchedule

These don't match, so the pod will never be scheduled on ub02.

**Example of Matching Taint and Toleration:**

- **Toleration:** size=small:NoSchedule
- **Taint:** size=small:NoSchedule

These match, so the effect will not be applied, and the pod will be scheduled.

### Use Case: GPU Configuration

In a mixed hardware cluster, taints can be used to ensure that workloads are scheduled on nodes with the correct GPUs.

#### Tainting Nodes

```bash
kubectl taint nodes ub02 gpu=amd:NoSchedule
kubectl taint nodes ub03 gpu=intel:NoSchedule
kubectl taint nodes ub04 gpu=nvidia:NoSchedule
kubectl taint nodes ub05 gpu=no_gpu_compute:NoSchedule
```

These commands taint the nodes to ensure that only pods with matching tolerations are scheduled on them.

#### AMD GPU Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apache
  labels:
    env: dev
spec:
  containers:
  - name: apache
    image: httpd:latest
    tolerations:
    - key: "gpu"
      value: "amd"
      effect: "NoSchedule"
      operator: "Equal"
```

#### Intel GPU Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apache
  labels:
    env: dev
spec:
  containers:
  - name: apache
    image: httpd:latest
    tolerations:
    - key: "gpu"
      value: "intel"
      effect: "NoSchedule"
      operator: "Equal"
```

#### NVIDIA GPU Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apache
  labels:
    env: dev
spec:
  containers:
  - name: apache
    image: httpd:latest
    tolerations:
    - key: "gpu"
      value: "nvidia"
      effect: "NoSchedule"
      operator: "Equal"
```

#### No GPU Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apache
  labels:
    env: dev
spec:
  containers:
  - name: apache
    image: httpd:latest
    tolerations:
    - key: "gpu"
      value: "no_gpu_compute"
      effect: "NoSchedule"
      operator: "Equal"
```

### Removing Taints

Here's how you remove a taint:

```bash
kubectl taint nodes ub02 gpu=amd:NoSchedule-
kubectl taint nodes ub03 gpu=intel:NoSchedule-
kubectl taint nodes ub04 gpu=nvidia:NoSchedule-
kubectl taint nodes ub05 gpu=no_gpu_compute:NoSchedule-
```

### Kubernetes Default Taints

Kubernetes uses taints to keep application pods off the master.

```bash
kubectl describe node ub01 | grep -i taints
kubectl describe node ub02 | grep -i taints
kubectl describe node ub03 | grep -i taints
```

### Deployments

Simple YAML for Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        App: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

---

You can use this document to manage the scheduling of pods across different types of nodes in your cluster, especially if you're working with various types of hardware such as GPUs.


Certainly! Here's the markdown document based on the details you've provided:

# Storage

The first yaml file that I'm going to provide to you is going to be the storage class definition yaml file.

## Storage Class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: demo-storage-class
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

Up to associate our applications with pods that have specific requirements for storage maybe it's going to be nvme or SSD or maybe this is an archival type of application where we don't need as much disk performance.

## Persistent Volume (PV)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: demo-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: demo-storage-class
  local:
    path: /mnt/pv-volume
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker1
```

So this persistent volume is Created from the perspective of the host the node that you're running on. I'm going to provide you a configuration of a PVC that will be used by a pod a little bit later.

So Kubernetes is not going to create this local path for you that you see in the configuration file (/mnt/pv-volume). You need to create it before you create your persistent volume or the persistent volume will not get into a ready state.

## Persistent volume claim (PVC)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pv-claim
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: demo-storage-class
  resources:
    requests:
      storage: 5Gi
```

## POD Binding a PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pvc
  labels:
    name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
      - containerPort: 80
        name: nginx
    volumeMounts:
      - name: demo-pv-mount
        mountPath: /usr/share/nginx/html
  volumes:
    - name: demo-pv-mount
      persistentVolumeClaim:
        claimName: demo-pv-claim
```

### Pod

Let's say that I decide to write a text file to this path `/usr/share/nginx/html` → `/mnt/pv-volume`. So I write my text file within the container in my pod to the path I have described above.

From within our container in our Pod:

```bash
echo "Hello, world from our Pod!" >> /usr/share/nginx/html/helloworld.txt
ls -la /usr/share/nginx/html
cat /usr/share/nginx/html/helloworld.txt
```

What's interesting is that you're going to switch back over to your host and you're going to check inside `/mnt/pv-volume` to see if your file has been created there.

```bash
ls -la /mnt/pv-volume
cat /mnt/pv-volume/helloworld.txt
```
```

This markdown document captures the details of the storage class, persistent volume, persistent volume claim, and pod configuration. It also includes some example commands you might run within your pod and on the host. Feel free to modify as needed!


