# 2. Core Concepts

## 11. Cluster Architecture

Master node is responsible for managing the cluster (schedule, plan, monitor, store metadata etc.). There are different controllers which run on master node, which ensure a proper state of cluster (replication-controller, node-controller etc.). Kube-apiserver exposes kubernetes API.

Worker node hosts containers. Kubelet is an agent which runs on each node in a cluster and listens for instructions from kube-apiserver (kube-apiserver periodically fetches reports from kubelet). Kube-proxy ensures communication between nodes

There is a container runtime which runs on each node (e.g. docker, containerd, rkt)

```
      master node             worker nodes
┌─────────────────────────┐  ┌──────────────┐
│                         │  │              │
│                         │  │  kubelet     │
│                         │  │              │
│ etcd cluster            │  │  kube-proxy  │
│                         │  │              │
│ kube-apiserver          │  └──────────────┘
│                         │
│ kube controller manager │  ┌──────────────┐
│                         │  │              │
│ kubec-scheduler         │  │ kubelet      │
│                         │  │              │
│                         │  │ kube-proxy   │
│                         │  │              │
└─────────────────────────┘  └──────────────┘
```

master:
- etcd cluster stores information about cluster
- kube-scheduler schedules applications (containers) on nodes
- kube-apiserver is responsible for orchestrating all operations on cluster
- kube-controller-manager contains different controllers which regulate the state of cluster, e.g.:
  - replication controller
  - endpoints controller
  - namespace controller
  - serviceaccounts controller.

worker:
- kubelet listens for instructions from kube-apiserver
- kube-proxy enables communication between nodes

references:
1. https://kubernetes.io/docs/reference/command-line-tools-reference/
1. https://kubernetes.io/docs/concepts/architecture/

## 12. Docker vs ContainerD

Kubernetes allows any container runtime as long as it adhere to OCI (Open Container Initiative) standards. There's imagespec and runtimespec. On version v1.24 Docker support was removed, however, it still supports containerd (which is a part docker, but different project) - ContainerD is Docker's internal runtime. 

Tools:
- `ctr` is containerd tool (not user-friendly from containerd community)
- `nerdctl` is docker-like cli for containerd (containerd community)
  - you can replace docker with `nerdctl` and run same command
- `crictl` is command-line too for CRI-compatible container runtimes (kubernetes community)
  - usually used for debugging purposes
  - is aware of pods (`crictl pods`)

Examples:
- `crictl pull busybox`
- `crticl images`
- `critcl ps -a`
- `crictl exec -i -t 1f73f2d81bf98 ls`
- `nerdctl --namespace=k8s.io ps -a`

Summary:
|              | ctr           | nerdctl         | crictl              |
| -------------| ------------- |-----------------| --------------------|
| purpose      | debugging     | general purpose | debugging           |
| community    | containerd    | containerd      | kubernetes          |
| works with   | containerd    | containerd      | all CRI compatible  |


references: 
1. https://earthly.dev/blog/containerd-vs-docker/
1. https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/
1. https://github.com/containerd/nerdctl
1. https://kubernetes.io/docs/reference/tools/map-crictl-dockercli/

## 13. ETCD for Begginers

ETCD is a distributed key-value store. Data is not stored in tabular format like in traditional relational databases, but in a form of documents, which contain keys and values.

```json
{
  "name": "John Doe",
  "age": 45,
  "location": "New York",
  "salary": 5000
}
```

Installing ETCD is relatively easy and consists of downloading pre-built binaries and runing ETCD service. ETCD listens on port `2379` by default. Default client CLI client for ETCD is `etcdctl`, it can read, write, delete keys and values and more:
- `etcdctl put key value`
- `etcdctl --version`
- `ectdctl del key`

There are two versions of etcdctl: version 2 and 3. To make `etcdctl` use version 3 API environment variable `ETCDCTL_API` should be set to `3` (e.g. `ETCDCTL_API=3 ./ectdctl version` or `export ETCDCTL_API=3`).

## 14. ETCD in Kubernetes

ETCD is used in Kubernetes for storing information regarding cluster, like information on pods, nodes, secrets, roles, rolebindings etc. Every change done in cluster is stored in ETCD and only when stored successfully change could be considered complete. 

When Kubernetes cluster is deployed using `kubeadm` tool, ETCD is deployed as a Pod. All keys in ETCD could be retrieved using following command:
- `kubectl exec etcd-master -n kube-system ectdctl get / --prefix -keys-only - to get all keys`
- `kubectl exec etcd-master -n kube-system -- sh -c "ETCDCTL_API=3 etcdctl get / --prefix --keys-only --limit=10 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt  --key /etc/kubernetes/pki/etcd/server.key"`

Example commands:
- `etcdctl snapshot save`
- `etcdctl endpoint health`
- `etcdctl get`
- `etcdctl put`

additional links:
1. https://etcd.io/docs/v3.4/dev-guide/interacting_v3/


## 16. Kube-APIServer
Kube-APIServer is a primary management component in a Kubernetes cluster. When `kubectl` command is run, it is in fact reaching `kube-apiserver` for requested information. First, request is authenticated and validated by `kube-apiserver`, next respective data is retrieved from `etcd` cluster and presented back to the user. In fact, the same data could be also retrieved using `POST` requests (e.g. using POSTMAN).

For example, when Pod is created:
- request is first authenticted and validated (by `kube-apiserver`)
- Pod object is created (`curl -X POST /api/v1/namespaces/default/pods`) is created.
  - object is not assigned to any node
  - information is stored in `etcd`
  - sends back information to used that `Pod has been created`
- Scheduler (which continuously monitors api server) detects that there is Pod without node assigned.
  - identifies the right node to place a Pod on
  - communicates this back to `kube-apiserver`
- `kube-apiserver` updates information on assigned node in `etcd`
- `kube-apiserver` passes all the information on about to be created Pod to `kubelet`
- `kubelet` is responsible with implementing the actual Pod on node
  - it instructs container runtime to create a pod
  - reports back to `kube-apiserver`
- `kube-apiserver` makes a final update on `etcd`

In summary, `kube-apiserver` is responsible for:
- authenticating user
- validating requests
- retrieving data and updating data (to and from `etcd`)
  - it is the only component that interacts with `etcd`

To view parameters of running `kube-apiserver`:
- kubeadm setup `cat /etc/kubernetes/manifests/kube-apiserver.yaml`
- hard way `cat /etc/systemd/system/kube-apiserver.service` and `ps -aux | grep kube-apiserver`

## 17. Kube Controller Manager

Controller is a daemon which continously monitors and attempts to bring back a cluster to a desired state.
There are various controllers responsible for certain aspects of a running cluster, e.g.: 
- node-controller
- replication-controller

For example `node-controller` monitors nodes (through `kube-apiserver`). It checks on nodes every 5 seconds, if it doesn't receive a heartbeat from a node, it marks it as unreachable (it does so after 40 seconds grace period). After that, `node-controller` waits for 5 minutes for node to go back. If a node is still not reachable, pods are removed from an unreachable node and provisioned on a different healthy ones [^1].

All controllers are packaged in component called `kube-controller-manager` and it could be customized through parameters. To view `kube-controller-manager` options:
- `cat /etc/kubernetes/manifests/kube-controller-manager.yaml` (kubeadm) 
- `cat /etc/systemd/system/kube-controller-manager.serivce` (standalone deployment)
- `ps -aux | grep kube-controller`

[^1]: if pods are part of a replicaset

## 18. Kube Scheduler

The Kubernetes schedules is a control plane process which assigns Pods to Nodes. The scheduler determines the valid placement for each Pod based on contraints and availble resources. It is a `kubelet` which actually creates Pods, 
scheduler is responsible for decision-making. 

`kube-scheduler`:
1. filters nodes
2. ranks nodes 
  - priority function which assigns a score to each valid Node

More:
- resource requirements and limts
- taints and tolerations
- node selectors / afinity

View `kube-scheduler` options:
- `cat /etc/kubernetes/manifests/kube-scheduler.yaml` (kubeadm)
- `cat /etc/systemd/system/kube-scheduler.service` (standalone)
- `ps -aux | grep kube-scheduler`

references:
1. https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/


## 19. Kubelet

Kubelet leads all activities on Node. It interacts with container runtime. It also monitors Pods and report back their status to Master Node.

Kubelet has to be manually installed (even when Kubernetes cluster is deployed via `kubeadm`). To view a running `kubelet` process and its options run `ps -aux | grep kubelet`

references:
1. https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/

## 20. Kube Proxy

Kube proxy is a process which runs on each node. It is a network proxy responsible for reflecting Services on each node. For example, thanks to `kube-proxy` Pods can be reached via Services, so it doesn't matter if Pods changes - it is still reachable via Services ip and hostname. Kube proxy can use various ways to achieve that (iptables, docker links etc.) 

## Recap - Pods

Pod is a single instance of an application and a smallest which can be crated in a cluster. 

Single Pod can consist of more than one container. It usually is not same app. Helper containers. init containers

## Pods in YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: frontend
spec:
  containers:
    - name: nginx-container
      image: nginx
    - name: busybox
      image: busybox
```

| Kind          | Version         |
| ------------- |-----------------|
| Pod           | v1              |
| Service       | v1              |
| ReplicaSet    | apps/v1         |
| Deployment    | apps/v1         |

## 29. Recap - ReplicaSets

repl-controller 
- helps to run more than one instance of pod
- works with single pod as well 
- ensured that specifed no runs all the time
- load balancing & scaling - helps balance the load and scale

Replication Controller vs Replica Set

Replication Controller - older
Replica Set - new way

`rc-definiton.yaml`:
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp-rc
  labels:
    app: myapp
    type: frontend
spec:
  # here the pod spec is provided here in under "template"
  template:
    # start pod spec
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: frontend
    spec:
      containers:
      - name: nginx-container
        image: nginx
    # end pod spec
  replicas: 3
```

Next:
- `kubectl create -f rc-definiton.yaml`
  - to create replication controller
- `kubectl get replicationcontroller`
  - to list rc in the cluster

### ReplicaSets

selector is a difference in replicaset. it's required in RS and not required in RC

`replitaset-definition.yaml`
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
    type: frontend
spec:
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: frontend
    spec:
      containers:
      - name: nginx-container
        image: nginx
  replicas: 3
  selector:
    matchLabels:
      type: frontend
```

Labels and Selectors:
- used to filter out pods

Scale `ReplicaSet`:
- update definition file and `kubectl replace -f replicaset-definition.yaml`
- `kubectl scale --replicas=6 -f replicaset-definition.yaml`
  - definition file won't be updated
- `kubectl scale --replicas=6 replicaset myapp-replicaset`
  - definition file won't be updated



## Deployments

- rolling updates - upgrade one after another
- rollback changes
- multiple changes 

Deployment > ReplicaSet > Pods > Container

`deployment-definition.yml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
    type: frontend
spec:
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: frontend
    spec:
      containers:
      - name: nginx-container
        image: nginx
  replicas: 3
  selector:
    matchLabels:
      type: frontend
```

`kubectl get all`


## 36. Services
In Kubernetes, a Service is a method of exposing a network application in a cluster. Service makes a Pod or set of Pods available on the network so it can be interacted with. 

In a cluster a number of Pods is usually ephemeral (especially when Deployments are used), hence Pods shouldn't be considered as reliable and durable. Each Pod gets it own IP address and for a given Deployment, the set of Pods running in one moment could be different a moment later. 

This state introduces a problem in a cluster: how set of Pods (e.g. "frontends") which need other set of Pods (e.g. "backends") keep track of "frontends" IP addresses? Services solve this problem (by allowing loose coupling between microservices in an application). The set of Pods targeted by a Service is usually determined by a selector that you define.

There are following Service types:
- `NodePort` which makes Pods (Endpoints) accessible via port on exposed on Node(s)
  - assigns ports from a range of 30000-32767
- `ClusterIP` exposed a cluster-internal IP. Makes Service reachable only from within a cluster
  - it is a default Service type that is used, when type is not explicitly specified
- `LoadBalancer` exposed a Service externally (cloudProvider, metalLb)
- `ExternalName` maps a Service to the contents of `externalName` field (dns records)


`service-definition.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  ports:
  - targetPort: 80
    port: 80
    nodePort: 30008
  selector:
    app: myapp
    type: frontend
```

Following ports could be defined in a Service:
- `targetPort` which represents a port on a Pod 
  - if it is not provided, it is assumed the same as `port`
- `port` which represents a port on a Service
- `nodePort` which is a exposed port on Node
  - if not provided automatic port is picked from range from `30000` to `32767`


```
      ┌────────────────────────────────────────────────┐
      │                                                │
      ├─────┐          ┌─────────────┐                 │
      │30008├──────────┤             │                 │
      ├─────┘          │          ┌──┤port             │
      │nodePort        │          │80├─────┐           │
      │                │          └──┤     │           │
      │                │SERVICE      │     │           │
      │                └─────────────┘     │           │
      │                                    │           │
      │                                    │targetPort │
      │                              ┌──┬──┴─┬──┐      │
      │                              │  │ 80 │  │      │
      │                              │  └────┘  │      │
      │                              │POD       │      │
      │                              └──────────┘      │
      │NODE                                            │
      └────────────────────────────────────────────────┘
```


When Service points to more than one Pod, traffic is *evenly* distributed between the Enpoints (Pods). Services distribute traffic between Pods in a simple round-robin fashion. 

Session stickines could be configured using `sessionAffinity` spec. Session stickiness means ability to bind specific client to specific Pod. 

references:
1. https://kubernetes.io/docs/concepts/services-networking/service/
1. https://kubernetes.io/docs/reference/networking/virtual-ips/

## 37. Services - ClusterIP

Services of type `clusterIP` allow to assign a static ip to a group of pods. IP address is for internal use only.
It is a default type of Service.


```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  ports:
    - targetPort: 80
      port: 80
  selector:
    app: myapp
    type: backend 
```

## 38. Services - Load Balancer

Service of type `LoadBalancer` are used usually in supported environments (cloud providers).

dopisac tu wiecej

## 41. Namespaces

Namespace is a mechanism which allows to isolate groups of resources within a single cluster.

- uniqueness of resource names
- default namespace
- system namespaces (kube-system, kube-public)
  - kube-system
  - kube-public
- quota can be assigned
- permissions can be defined
- resources within a namespace can refer to shorter names
  - for another namespaces fqdn(?) should be used 

```
   db-service.dev.svc.cluster.local
   │        │ │ │ │ │ │           │
   └───┬────┘ └┬┘ └┬┘ └─────┬─────┘
       │       │   │        │
       ▼       │   ▼        ▼
  service name │ service  domain
               │ (type)
               ▼
           namespace
```
db-service.dev.svc.cluster.local:
- cluster.local
- svc
- dev
- db-service

Commands/Operational:
- `kubectl get pods --namespace=kube-system` - list Pods in a kube-system namespace
- namespace field in metadata
- `kubectl create namespace <name>`
- `kubectl config set-context $(kubectl config current-context) --namespace=dev` - sets `dev` namespace as a default namespace
- `kubectl get pods --all-namespaces`


`compute-quota.yaml`
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 5Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```


refs:
1. https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/
1. https://kubernetes.io/docs/concepts/policy/resource-quotas/


## 48. Kubectl Apply Command
Command takes three sources of information into consideration:
- local file
- last applied config (json)
- kubernetes (live config object)
- do **NOT** mix imperative vs declarative commands 















