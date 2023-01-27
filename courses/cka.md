# CKA study notes

This document contains my CKA (V1.25.0) study notes.

## Table of Contents

* [CKA study notes](#cka-study-notes)
  * [Table of Contents](#table-of-contents)
  * [Kubernetes](#kubernetes)
  * [Primary components](#primary-components)
    * [Control plane](#control-plane)
    * [K8s nodes](#k8s-nodes)
  * [K8s management tools](#k8s-management-tools)
  * [Basic cluster management](#basic-cluster-management)
    * [Setup cluster](#setup-cluster)
    * [Upgrading K8s](#upgrading-k8s)
      * [Upgrade control plane nodes](#upgrade-control-plane-nodes)
      * [Upgrade worker nodes](#upgrade-worker-nodes)
    * [Cluster logs](#cluster-logs)
  * [K8s namespaces](#k8s-namespaces)
  * [Role based access control and users](#role-based-access-control-and-users)
  * [Service accounts](#service-accounts)
  * [Storage in K8s](#storage-in-k8s)
    * [Storage mechanisms](#storage-mechanisms)
    * [Standard Volumes](#standard-volumes)
    * [Persistent volumes](#persistent-volumes)
      * [Reclaim policies](#reclaim-policies)
      * [Storage classes](#storage-classes)
      * [Persistent volume claims](#persistent-volume-claims)
      * [Persistent volumes in a pod](#persistent-volumes-in-a-pod)
  * [Networking in K8s](#networking-in-k8s)
    * [Ingress and Egress traffic](#ingress-and-egress-traffic)
    * [K8s DNS](#k8s-dns)
    * [Network policies](#network-policies)
  * [K8s services](#k8s-services)
    * [Service types](#service-types)
    * [Service DNS](#service-dns)
    * [Accessing services from outside the cluster](#accessing-services-from-outside-the-cluster)
  * [Pods and containers](#pods-and-containers)
    * [Static pods](#static-pods)
    * [Multi-container pods](#multi-container-pods)
    * [Init containers](#init-containers)
    * [Container resources](#container-resources)
    * [Assign containers to specific nodes](#assign-containers-to-specific-nodes)
    * [Container monitoring](#container-monitoring)
    * [Container logs](#container-logs)
    * [Container restart policies](#container-restart-policies)
    * [Automatically replicate pods](#automatically-replicate-pods)
    * [Managing and passing configuration data](#managing-and-passing-configuration-data)
  * [K8s deployments](#k8s-deployments)
    * [Rolling updates](#rolling-updates)

## Kubernetes

Kubernetes (K8s) is a platform for managing containers in a cluster. In a nutshell it:

* Allows to dynamically manage containers across multiple hosts
* Provides reliability through self-healing and scalability
* Allows the autotomation of container management

## Primary components

K8s clusters are build up by various components which will run on the control plane or the worker nodes.

### Control plane

The K8s control plane controls the cluster. It is possible to setup a high availability platform by running multiple controller nodes behind a loadbalancer. Usually the control plane components are run on dedicated controller machines. These components are:

* kube-api-server
  * The primary interface to the control plane and the cluster
  * kubectl depends on kube-api-server
  * CRI and kubelet services need to be running
* Etcd
  * The backend data store
  * Provides high-availability storage for cluster related data and information (e.g. configuration)
  * Can also be configured on dedicated hosts (external etcd) contrary to the default (stacked etcd)
  * Can be backed up and restored through `etcdctl` snapshots
* kube-scheduler
  * Handles container scheduling
  * Selects an available node on which to run a container
* kube-controller-manager
  * Runs a collection of controller utilities
  * Controller utilities carry out automation-related tasks
* cloud-controller-manager
  * Provides an interface between K8s and cloud platforms

These components usually run as pods in the kube-system namespace when a cluster has been setup with kubeadm. It is possible to get information on them by using:

```bash
kubectl get pods -n kube-system
kubectl describe pod [POD_NAME] -n kube-system
```

### K8s nodes

K8s nodes are the machines where the containers managed by the cluster run. A cluster can have any number of nodes.

Node components manage containers on the machine and communicate with the control plane. These components are:

* kubelet
  * K8s agent that runs on each node
  * Communicates with the control plane
  * Ensures container state as instructed by the control plane
  * Reports container status back to the control plane
* container runtime
  * Container software that can actually run the containers (e.g. docker and podman)
* kube-proxy
  * Network proxy
  * Provides networking between containers and services in the cluster

Information about nodes can be retrieved by using `kubectl`:

```bash
kubectl get nodes
kubectl describe node [NODE_NAME]
```

## K8s management tools

The following tools are used to setup, configure and manage the cluster and its various components:

* kubectl
  * Official K8s CLI
  * Communicate with the clusters control plane
  * Add the --record to add executed actions to annotations
* minikube
  * A tool that helps creating and managing a single node cluster
  * Useful for setting up development / testing environments
* kubeadm
  * A tool that helps creating and managing a K8s cluster
* helm
  * Package management / templating solution
  * Helps deploying and managing K8s applications
* kompose
  * Translate docker compose files into K8s objects
* kustomize
  * Configuration management tool for K8s object configurations

## Basic cluster management

### Setup cluster

* Add setting up external etcd
* Add setting up loadbalancer in front of control servers

Kubeadm allows setting up a basic K8s cluster. In this example there will be three Ubuntu 20.04 VM's running K8s 1.25.0 with containerd as container runtime and with one control server and two worker nodes.

To setup the cluster:

On each VM:

Enable required kernel modules:

```bash
vi /etc/modules-load.d/containerd.conf
  br_netfilter
  overlay
modprobe br_netfilter; modprobe overlay
```

Allow packet forwarding:

```bash
vi /etc/sysctl.d/99-kubernetes-cri.conf
  net.bridge.bridge-nf-call-iptables = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.ipv4.ip_forward = 1
sysctl --system
```

Containers should not be allowed to use swap. It probably is better to configure containers not to use swap and leave swap on, but for now it will be just disabled globally:

```bash
swapoff -a
vi /etc/fstab
  # Remove swap mounts
```

Install APT dependencies:

```bash
apt-get update
apt-get install -y apt-transport-https curl
```

Setup containerd:

```bash
apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
systemctl restart containerd
```

Install K8s packages:

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
vi /etc/apt/sources.list.d/kubernetes.list
  deb https://apt.kubernetes.io/ kubernetes-xenial main
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

Setup bash autocompletion for K8s:

```bash
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
# Logout and log back in
```

On the control server:

Setup the K8s cluster using kubeadm:

```bash
kubeadm init --pod-network-cidr [PRIVATE_NETWORK_RANGE] --kubernetes-version [K8S_VERSION]
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Setup a networking plugin, in this example Calico:

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Check if pods are running:

```bash
kubectl get pods --all-namespaces
```

Create join command for worker nodes:

```bash
kubeadm token create --print-join-command
```

Execute the join command on the worker nodes to connect them to the cluster.

Run `kubectl get nodes` on the controller nodes to check if the worker nodes have been connected correctly.

### Upgrading K8s

In order to upgrade K8s to a newer version it is important to first upgrade the components on the control plane node, before upgrading the worker nodes.

#### Upgrade control plane nodes

Drain node:

```bash
kubectl drain [NODE_HOSTNAME] --ignore-daemonsets
```

Upgrade kubeadm software:

```bash
apt-get update && apt-get install -y --allow-change-held-packages kubeadm=[NEW_VERSION]
kubeadm version
```

Plan and execute the cluster upgrade

```bash
kubeadm upgrade plan [NEW_VERSION]
kubeadm upgrade apply [NEW_VERSION]
```

Upgrade kubelet and kubectl:

```bash
apt-get install -y --allow-change-held-packages kubelet=[NEW_VERSION] kubectl=[NEW_VERSION]
systemctl daemon-reload
systemctl restart kubelet
```

Uncordon the node:

```bash
kubectl uncordon [NODE_HOSTNAME]
```

Check if node is fully back online:

```bash
kubectl get nodes
```

If there are multiple K8s control plane nodes in a High Availability cluster these steps can be repeated for every node in the cluster.

#### Upgrade worker nodes

On the controller node, drain the worker node:

```bash
kubectl drain [NODE_HOSTNAME] --ignore-daemonsets --force
```

On the worker node:

Upgrade kubeadm:

```bash
apt-get update && apt-get install -y --allow-change-held-packages kubeadm=[NEW_VERSION]
kubeadm version
```

Upgrade kubeadm configuration:

```bash
kubeadm upgrade node
```

Upgrade kubelet and kubectl:

```bash
apt-get install -y --allow-change-held-packages kubelet=[NEW_VERSION] kubectl=[NEW_VERSION]
systemctl daemon-reload
systemctl restart kubelet
```

From controller node:

Uncordon the worker node:

```bash
kubectl uncordon [NODE_HOSTNAME]
```

Check if the worker node is working correctly again:

```bash
kubectl get nodes
```

Repeat these steps for other worker nodes in the cluster.

### Cluster logs

K8s related services (like kubelet and the CRI like docker) run through systemd. The logs of these services can be viewed with journalctl:

```bash
journalctl -u kubelet
journalctl -u docker
```

Cluster components log output in `/var/log/`, for example:

* /var/log/kube-apiserver.log
* /var/log/kube-scheduler.log
* /var/log/kube-controller-manager.log

When using kubeadm the components run as pods and their logs can be accessed by using `kubectl`:

```bash
kubectl logs -n kube-system [POD_NAME]
```

## K8s namespaces

Namespaces are virtual clusters backed by the same physical cluster. Namespaces are a way to separate and organize objects (like pods and containers) in your cluster. kubeadm creates a kube-system namespace which holds system components to control the cluster. When executing commands and no other namespace is specified, the default is used.

Create a namespace:

```bash
kubectl create namespace [NAMESPACE]
```

List existing namespaces:

```bash
kubectl get namespaces
```

List pods from specific namespace:

```bash
kubectl get pods --namespace [NAMESPACE]
```

List pods in all namespaces:

```bash
kubectl get pods --all-namespaces
```

## Role based access control and users

Role based access control allows control over what users are allowed to do and access within a cluster. For example you can use it to allow users to read logs, but not manage the pods.

In K8s roles are used to define permissions in a particular namespace and clusterroles are used to define cluster wide permissions. Rolebindings are used in order to connect (cluster)roles with users. These definitions are made in YAML files.

To setup a pod-reader role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "watch", "list"]
```

```bash
kubectl apply -f pod-reader-role.yml
```

And to connect it with a user `dev`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader
  namespace: default
subjects:
- kind: User
  name: dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

kubectl apply -f pod-reader-binding.yml

## Service accounts

Service accounts are used to communicate with the K8s API from within pods.

To setup a service account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-serviceaccount
  namespace: default
```

```bash
kubectl create -f my-serviceaccount.yml
```

Access is managed using RBAC objects:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sa-pod-reader
  namespace: default
subjects:
- kind: ServiceAccounts
  name: my-serviceaccount
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create -f sa-podreader.yml
```

## Storage in K8s

By default the container file system is `ephemeral`. The files on the container exist only as long as the container exists.

When a persistent method of data storage is needed volumes and persistent volumes can be used.

### Storage mechanisms

With volume types the underlying storage mechanism is defined. There are multiple types available, like:

* Directories on the K8s node
  * Specified with `hostPath`
* Temporarily volumes
  * Specified with `emptyDir: {}`
  * Useful to allow containers exchanging data
* Network (like NFS) mounts
* Cloud storage mechanisms
* ConfigMaps and Secrets

### Standard Volumes

Standard volumes are created together with the pod. As example to mount a directory called `/data` as `/output` in a container and `/input` for a second container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: busybox1
    image: busybox
    volumeMounts:
      - name: my-volume
        mountPath: /output
  - name: busybox2
    image: busybox
    volumeMounts:
      - name: my-volume
        mountPath: /input
  volumes:
  - name: my-volume
    hostPath:
      path: /data 
```

### Persistent volumes

Contrary to standard volumes persistent volumes are an object on their own, allowing storage to be treated as an abstract resource to be used by pods.

To setup a persistent volume and use it in a pod it is necessary that a storage class, a persistent volume object and a persistent volume claim has been setup.

To define a persisent volume on "/var/output" that uses a "localdisk" `storage class` and has its `reclaim policy` set to "recycle":

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: my-pv
spec:
  storageClassName: localdisk
  persistentVolumeReclaimPolicy: Recycle
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /var/output
```

After creating `kubectl get pv` will show information on Persistent Volumes.

#### Reclaim policies

Setting `persistentVolumeReclaimPolicy` defines if the volumes can be reused when associated `persistentVolumeClaims` are deleted.

* Retain
  * Keeps all data
  * Requires manual cleaning up and prepare resource for reuse
* Delete
  * Deletes all data
  * Only works for cloud storage resources
* Recycle
  * Automatically deletes all data
  * Allows for automatically reuse

#### Storage classes

A `StorageClass` allows to specify the types of storage services that are offered on the platform. They basically connect the `PersistentVolume` to a storage source.

A simple example to store data locally and allow resizing of the volumes that use it:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: localdisk
provisioner: kubernetes.io/no-provisioner
allowVolumeExpansion: true
```

#### Persistent volume claims

A `PersistentVolumeClaim` represent a user's request for storage resources, allowing defined `PersistentVolumes` to be used in a pod definition.

When using a `PersistentVolumeClaim` it will automatically lookup and bound to a corresponding `PersistentVolume`. `PersistentVolumeClaims` will need to have the same `storageClassName`, `accessModes` and require less then the amount of available space in the `PersistentVolume`.

When the `StorageClass` has `allowVolumeExpansion` set to `true` it is possible to expand the `PersistentVolumeClaim` by simply editing the `spec.resources.requests.storage` attribute and saving it.

To setup a persistent volume claim using the same localdisk `storageClass` and with 100MB of storage:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: localdisk
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```

After creating `kubectl get pvc` will show information on Persistent Volume Claims.

#### Persistent volumes in a pod

To use a Persistent Volume in a pod it is necessary to refer to the PersistentVolumeClaim in the volumes definition, after setting up the storage class, persistent volume and the persistent volume claim itself:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'echo Success! > /output/success.txt']
    volumeMounts:
      - name: pv-storage
        mountPath: /output
  volumes:
  - name: pv-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

## Networking in K8s

The K8s network model is a set of standards that define how networking between pods behave, regardless of which node the pods are running on. Each pod in the cluster has its own unique IP adress within this virtual network.

Container Network Interface (CNI) plugins, like Calico, implement this model. There are various plugins available.

It is possible to troubleshoot network issues from within a container. A great image for troubleshooting network is `nicolaka/netshoot`.

### Ingress and Egress traffic

* Ingress:
  * Applies to traffic coming into the pod from another source
* Egress:
  * Applies to outgoing traffic leaving the pod for another destination

### K8s DNS

The K8s virtual network includes a DNS system, running as a service pod within the kube-system namespace in the cluster. This DNS system automatically gives every pod a domainname, for example: `192-168-10-100.default.pod.cluster.local`.

DNS will mostly start getting interesting when setting up and using K8s services.

### Network policies

By default pods are non-isolated and open to all communication. With network policies the pods can be isolated and only open to traffic that is allowed by these policies.

Network policies consists of:

* From and to selectors:
  * From selects ingress (incoming) traffic that will be allowed
  * To selects egress (outgoing) traffic that will be allowed
* PodSelector:
  * Applies policies to pods using pod labels
* NamespaceSelectors:
  * Select namespaces to allow traffic from and to
* ipBlock:
  * Select an IP range to allow traffic from and to
* Ports:
  * Specify one or more ports that will be allowed

To setup a network policy allowing port 80 traffic from all pods in the np-test namespace into the pods with the nginx label:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-networkpolicy
  namespace: np-test
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
      matchLabels:
        team: np-test
    ports:
    - protocol: TCP
    - port: 80
```

## K8s services

K8s services provide a way to expose an application running as a set of pods. This way clients can access applications without needing to be aware of application's pods. Requests to a service get routed to its pods in a load-balanced manner. Each pod will represent an endpoint for the service.

### Service types

Each service has a type. The type determines how and where the service will expose the application:

* ClusterIP
  * Expose applications inside the cluster network
  * Useful for when clients will be other pods within the cluster
* NodePort
  * Expose applications outside the cluster network
  * Useful for when applications or users will access the application from outside the cluster
* LoadBalancer
  * Expose applications outside the cluster network as well
  * Uses an external cloud load balancer from cloud platforms
  * Useful for when using K8s within cloud platforms or when using a cloud platform load balancer
* ExternalName

To setup a `ClusterIP` service that routes to pods with the label `svc-example` on port 80 to port 80:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-clusterip
spec:
  type: ClusterIP
  selector:
    app: svc-example
  ports:
    - name: web
      protocol: TCP
      port: 80
      targetPort: 80
```

It will then be possible to access the service from within the cluster by using the service name, like by using a busybox pod:

```bash
kubectl get service svc-clusterip
kubectl exec [POD_NAME] -- curl svc-clusterip:80
```

In a smilar way it is possible to setup a `NodePort` service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-nodeport
spec:
  type: NodePort
  selector:
    app: svc-example
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

It will then be able to access the service from outside of the cluster by using the public IP address from either the controller or worker nodes and the specified port.

### Service DNS

The K8s DNS service assigns DNS names to services, allowing applications within the cluster to easily locate them. The fully qualified domain name will be built up like `service-name.namespace-name.svc.cluster-domain.example` / `svc-clusterip.default.svc.cluster.local`. Pods with the same namespace can also use the service-name, like `svc-clusterip`.

To lookup the DNS of a service using a pod running busybox:

```bash
kubectl get service [SERVICE_NAME]
kubectl exec [POD_NAME] -- nslookup [CLUSTER_IP]
```

### Accessing services from outside the cluster

It is possible to allow and route external clients / traffic to services by using an `ingress controller`. Such an ingress controller is capable of much more functionality, like SSL termination and advanced load balancing.

To define an ingress object that routes traffic on `http://public_ip/some_path` to the `svc-clusterip` service on port 80 (with name `web` as defined in the service definition):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - http:
    paths:
    - path: /somepath
      pathType: Prefix
      backend:
        service:
          name: svc-clusterip
          port:
            name: web
```

To show information on the created ingress object:

```bash
kubectl describe ingress [INGRESS_NAME]
```

## Pods and containers

Containers are isolated Linux environments which allow any application and its dependencies to be bundled up in a single file. K8s does not run containers directly, but wraps them in a higher-level structure called pods. Any containers in the same pod will share the same resources and local network.

It is possible to execute commands within the container, like:

```bash
kubectl exec -n [NAMESPACE] [POD_NAME] -c [CONTAINER_NAME] -- [COMMAND]
```

When K8s metric server has been installed, it is possible to lookup metrics from pods with `kubectl`:

```bash
kubectl top
```

### Static pods

Static pods run directly on a node and are managed by the kubelet installation on the node, not by the K8s API server. Kubelet will however create a mirror pod that you can not manage through the API, but allows you to see the status of the static pod using the K8s API.

Good use cases for static pods are control plane pods or load balancers for the control plane.

To define a static pod on a worker node:

```bash
vi /etc/kubernetes/manifests/my-static-pod.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-static-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.19.1
```

```bash
systemctl restart kubelet
```

### Multi-container pods

It is best practice to only bundle multiple containers in the same pod if they absolutely need to share resources. Containers in the same pod will have access to the same (internal) network and these containers can communicate with each other on any port, even if that port is not exposed to the cluster. Containers in the same pod will also be able to share storage volumes.

An example defining one container that writes output to a shared volume and a second container reading this output on the shared volume:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
spec:
  containers:
  - name: busy_write
    image: busybox
    command: ['sh', '-c', 'while true; do echo logs data > /output/output.log; sleep 5; done']
    volumeMounts:
      - name: shared_volume
        mountPath: /output
  - name: 
    image: busy_read
    command: ['sh', '-c', 'tail -f /input/output.log']
    volumeMounts:
      - name: shared_volume
        mountPath: /input
  volumes:
  - name: shared_volume
    emptyDir: {}
```

### Init containers

Init containers run once during the startup process of a pod and can be used to perform startup tasks. They can contain and use software and setup scripts that are not needed by the main containers, for example they can:

* Cause a pod to wait for another K8s resource to be created
* Perform sensitive startup steps securely outside of application containers
* Populate data into a shared volume at startup
* Communicate with another service at startup

Setup a init container that delays the startup of the main container with 30 seconds:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.19.1
  initContainers:
  - name: delay
    image: busybox
    command: ['sleep', '30']
```

### Container resources

K8s provides two ways of defining resources (CPU and memory) for containers. By using `resource requests` the amount of resources a container is expected to use is defined. This will be used by the scheduler to avoid scheduling pods on nodes that do not have enough available resources. It is possible the pod uses more than the given resource request.

With `resource limits` the pod is limited on resources. This will actually limit the amount of resources a pod can use and exceeding these limits will affect the pod. Some container runtimes might stop the container when limits are exceeded.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'while true; do sleep 3600; done']
    resources:
      requests:
        cpu: "250m"
        memory: "128mi"
      limits:
        cpu: "500m"
        memory: "256mi"
```

### Assign containers to specific nodes

It is possible to provide `nodeName` to assign a pod to a specific node:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-specific-pod
spec:
  containers:
  - name: busybox
    image: busybox
  nodeName: k8s-worker1
```

With `nodeSelector` it is possible to assign a pod to nodes with a specific given label:

```bash
kubectl label nodes [NODE_HOSTNAME] [LABEL]=true
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-specific-pod
spec:
  containers:
  - name: busybox
    image: busybox
  nodeSelector:
    [LABEL]: "true"
```

### Container monitoring

By default K8s will consider a container down when the main process has stopped. It is possible to customize this detection mechanism and improve the monitoring of containers by using the following features of K8s:

* liveness probes
  * Continously monitor a specific process
* startup probes
  * Detect when the application has successfully started up
  * Useful when applications have long startup times
* readiness probes
  * Determine when a container is ready to accept requests
  * Prevent user traffic being sent to pods that are still starting up

These probes can be configured in the container YAML:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probes-http-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.19.1
    startupProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 45
      periodSeconds: 5
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 60
      periodSeconds: 5
  ```

### Container logs

Containers log everything that is written to stdout and stderr by the container process. Using `kubectl` these logs can be viewed:

```bash
kubectl logs [POD_NAME] -c [CONTAINER_NAME]
```

### Container restart policies

Restart policies make it possible to define if a container needs to be restarted in certain situations:

* Never
  * The container will not be restarted, even when it crashed
* OnFailure
  * The container will only be restarted when it crashed
* Always
  * The container will always be restarted, even when it is stopped on purpose

These policies can be defined in the container YAML:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-policy-pod
spec:
  restartPolicy: Always
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'while true; do sleep 3600; done']
```

### Automatically replicate pods

Using `DaemonSet` K8s will automatically run a copy of a pod on each node and will create new copies when  new nodes are added to the cluster. These daemonsets respect normal scheduling rules around node labels, taints and tolerations.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-daemonset
spec:
  selector:
    matchLabels:
      app: my-daemonset
  template:
    metadata:
      labels:
        app: my-daemonset
    spec:
      containers:
      - name: nginx
        image: nginx:1.19.1
```

### Managing and passing configuration data

To configure applications that run in containers it is possible to pass on configuration data. Configuration data can be stored in various ways, like:

* ConfigMaps which hold key-value mappings of the configuration data:

```yaml
apiVersion: v1
kind: ConfigMap
metadata: my-configmap
data:
  key1: value1
  key2: value2
  key3:
    subkey:
      sub-subkey: data
      sub-subkey2: more data
  key4: |
    Multiline data
    is also a possibility.
```

* Secrets which use similar mapping als ConfigMaps, but store encoded (secret) data like passwords:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: user
  password: base64_encoded_password
```

These configuration files and their data can be passed to the container using:

* Environment variables, which expose the data as variables that will be visible to the container process at runtime:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'echo "configmap: $CONFIGMAPVAR secret: $SECRETVAR"']
  env:
    - name: CONFIGMAPVAR
      valueFrom:
        configMapKeyRef:
          name: my-configmap
          key: key1
    - name: SECRETVAR
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: secretkey1
```

* Configuration volumes, which will make the configuration data accessible in files available to the container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'while true; do sleep 3600; done']
  volumeMounts:
  - name: configmap-volume
    mountPath: /etc/config/configmap
  - name: secret-volume
    mountPath: /etc/config/secret
  volumes:
  - name: configmap-volume
    configMap:
      name: my-configmap
  - name: secret-volume
    secret:
      secretName: my-secret
```

## K8s deployments

`Deployments` in K8s define desired states for a `ReplicaSet`. Pods in a replica set are all running the same containers and configuration. The deployment controller is responsible for maintaing the desired state by creating, deleting and replacing Pods with new configurations.

With deployments it is easy to scale an application up or down, perform rolling updates and roll back to a previous software version.

Important configuration fields to use in deployments are:

* Replicas
  * The number of replica pods the deployment will try to maintain
* Selector
  * Identify the replica pods managed by the deployment
* Template
  * Defines the pods that needs to be created

An example deployment running one NGINX container with three replicas:

```bash
vi my-deployment.yml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-deployment
  template:
    metadata:
      labels:
        app: my-deployment
    spec:
      containers:
        - name: nginx
          image: nginx:1.19.1
          ports:
            - containerPort: 80
```

```bash
kubectl create -f my-deployment.yml
kubectl get deployments
kubectl get pods
```

It is possible to generate an example YAML file by using the `--dry-run` flag of `kubectl`:

```bash
kubectl create deployment [NAME] --image=[IMAGE_NAME] --dry-run -o yaml
```

When changing the replicas to `5` in the YAML file for example, we can use `kubectl apply -f my-deployment.yml` to apply this change. It is also possible to edit using `kubectl edit` or even directly with `kubectl scale`:

```bash
kubectl scale deployment [NAME] --replicas [REPLICA_NUMBER]
```

### Rolling updates

Using deployments makes it possible to rollout updates of software:

```bash
kubectl edit deployment my-deployment
  # Edit the software version and save and exit the editor
```

K8s will then immediately execute the rolling update. You can check the status:

```bash
kubectl rollout status deployment.v1.apps/my-deployment
```

It is also possible to rollback to a previous version:

```bash
kubectl rollout undo deployment.v1.apps/my-deployment --to-revision=[REVISION_NUMBER]
```

Without specifying the revision number K8s will automatically rollback to the latest before the current version.
