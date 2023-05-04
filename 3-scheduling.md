## 52.Manual Scheduling
- Every pod has a filed `nodeName` usually not set
- Scheduler identifies fitting Node to run Pod on
- `nodeName` can be set manually
  - only at creation time
- `Binding` object can be defined and send to Kubernetes API:
  - `curl --header "Content-Type:application/json" --request POST --data '{"apiVersion": "v1", "kind":"Biding"...}' http://$SERVER/api/v1/namespaces/default/pods/$PODNAME/binding`


#### pod-binding-definition.yaml:
```yaml
  apiVersion: v1
  kind: Binding
  metadata:
    name: nginx
  target:
    apiVersion: v1
    kind: Node
    name: node02
```


## 55. Labels and selectors
Labels and selectors are used to target and filter desired objects in Kubernetes cluster
Labels are specified in metadata section of the definition.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx01
  labels:
    app: nginx
    tier: frontend
    team: devops
(...)
```

Next, selector could be used to:
- Retrieve Pods with certain labels, e.g. `kubectl get pods --selector app=nginx`
- Specify Pods which will be a part of a ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: simple-webapp
  labels:
    app: App1
    function: frontend
  annotations:
    buildversion: "1.34"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: App1
  template:
    metadata:
      app: App1
      function: frontend
  spec:
    containers:
      - name: simple-webapp
        image: simple-webapp
```

## 58. Taints and tolerations
Taints and tolerations are used to control which Pods can be schedules on Nodes

Taints - repellant (assinged to Nodes)
Tolerations - toleration for certain taint (assigned to Pods)

For exmaple:
- there are 3 nodes: node01, node02, node03
- 

taint a node:
`kubectl taint nodes <node-name> key=value:taint-effect`
  - key=value, e.g. app=blue
  - taint-effect:
    - NoSchedule - Pods will not be scheduled
    - PreferNoSchedule - Scheduler will try not to schedule 
    - NoExecute - new Pods will not be scheduled and existing evicted if not tolerating a taint
untaint a node:
- `kubectl taint node controlplane node-role.kubernetes.io/control-plane:NoSchedule-`

#### pod-toleration-definition.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: nginx-container
      image: nginx
  tolerations:
    - key: "app"
      operator: "Equal"
      value: "blue"
      effect: "NoSchedule"
```

## 61. Node Selectors
Node selectors allow to dedicate certain workloads on certain nodes. Default setup is any Pod can get scheduled on any Node.

#### pod-definition-node-selector.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: data-processor
      image: data-processor
  nodeSelector:
    size: Large
```

Prior to creating Node needs to be labeled:
`kubectl label nodes <node-name> <label-key>:<label-val>`
`kubectl label nodes node01 size=Large`

### Node Selector limitations
NodeSelectors cannot express more complex logical expressions, e.g. `size=Large OR size=Medium` 

## 62. Node Affinity
Node Affinity allows to control scheduling of Pods on certain Nodes (e.g. place a Pod on a "Large" Node).
It also allows for more advanded expressions than NodeSelectors.

nodeAffinity strategies:
- requiredDuringSchedulingIgnoredDuringExecution
- prefferedDuringSchedulingIgnoredDuringExecution

operators:
- In
- NotIn
- Exists

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: color
            operator: In
            values:
            - blue
            - red            
  containers:
  - name: nginx
    image: nginx
```

## 65. Taints and Tolerations vs Node Affinity

## 66. Resource Requirements and Limits
Each node has a limited capacity of resources (cpu, mem, disk).

default resource request:
- cpu 0.5
- mem 256Mi

Default values can be modified in Pod definitions (`spec.containers.resources.requests`)

#### pod-definition-resources.yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: simple-webapp
      image: simple-webapp
  resources:
    requests:
      memory: "1Gi"
      cpu: 1
    limits:
      memory: "2Gi"
      cpu: 2
```

- Limits and requests are set for each container in a Pod
- If there are not enought resources, Pod will stay in Pending state
- Pod which constantly uses more memory than its limit will get terminated


## DaemonSets
DaemonSets ensured that a copy of a certain Pod is always running on every Node. Typical use cases are monitoring solutions, log viewers, kube-proxy, networking solutions (e.g. weave-net).

#### deamon-set-definition.yaml:
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      app: monitoring-agent
    spec:
      containers:
        - name: monitoring-agent
          image: something-like-prometheus-xd
```

### How does it work?
- default scheduler
- Node Affinity rules

## 74. Static Pods
Static pods are pods managed directly by kubelet daemon no a specific node (without API server).
Static pods could be created by placing a pod definition in certain directory defined during kubelet instalation.
It is defined as an argument for parameters `--pod-manifest-path` or `--config` and could be found in kubelet service definition.

Often it is set to `/etc/kubernetes/manifests`

`kube-apiserver` is aware of static pods when node is a part of a cluster. However, static pods are read-only from `kubeapi-server` (and `kubectl`) perspective.

### Static Pods vs DaemonSets

| Static Pods                                   | Daemon Sets                                       |
|-----------------------------------------------|---------------------------------------------------|
| created by kubelet                            | created by `kube-apiserver` (deamonset controller)|
| deploy control plane components as Static Pods| deploy monitoring agent, logging agents etc.      |
| ignored by kube-scheduler                     | ignored by kube-scheduler                         |


## 77. Multiple schedulers
Kubernetes is highly extensible - different schedulers (custom ones) can be deployed. More than one scheduler can be active in a cluster at the same time.

#### custom scheduler could be deployed like below (as a service):
##### my-scheduler-2-config.yaml
```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: my-custom-scheduler
leaderElect:
  leaderElect: true
  resourceNamespace: kube-system
  resourceName: lock-object-my-scheduler
```

##### kube-scheduler.service
```
ExcexStart=/usr/local/bin/kube-scheduler \\
      --config=/etc/kubernetes/config/kube-scheduler.yaml
```

##### how to use a new custom scheduler?
This could defined in `spec.schedulerName`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx01
spec:
  containers:
    - image: nginx
      name: nginx01
  schedulerName: my-custom-scheduler
```

##### how to check which scheduler picked up a Pod?
This could be by issuing command `kubectl get events -o wide` and looking in `REASON` column for **Scheduled** and check what is specified in `SOURCE` column - it should be a scheduler's name. 

Alteratively, Pods logs could be viewed by running command `kubectl logs <scheduler pod> -n kube-system`

references:
1. https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/


## 80. Configuring Scheduler Profiles
Scheduler phases:
- scheduling queue (Priority Sort)
- filtering (NodeResourcesFit)
  - NodeName
  - NodeUnschedulable
- scoring (NodeResourcesFit)
  - ImageLocality - Nodes which already have an image
- binding (DefaultBinder)

ExtensionPoints can be used to extend above phases:
- queueSort
- preFilter
- filter
- postFilter
- preScore
- score
- reserve
- permit
- preBind
- bind
- postBind

Scheduler profiles allow to specify more than one profile for one scheduler process.

```yaml
apiVersion: kubescheduler.config.k8s.io/vi1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: my-scheduler01
    plugins:
      score:
        disabled:
          - name: TaintToleration
        enabled:
          - name: CustomPlugin01
          - name: CustomPlugin02

  - schedulerName: my-scheduler02
    plugins:
      preScore:
        disabled:
          - name: '*'
      score:
        - name:
          disabled:
            - name: '*'
```





