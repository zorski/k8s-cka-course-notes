# MOCK EXAMS
## MOCK EXAM 1:

0. Set k alias and autocompletion (https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
`source <(kubectl completion bash) # set up autocomplete in bash into the current shell, bash-completion package should be installed first.
echo "source <(kubectl completion bash)" >> ~/.bashrc # add autocomplete permanently to your bash shell.`

`alias k=kubectl
complete -o default -F __start_kubectl k`

1. Deploy a pod named nginx-pod using the nginx:alpine image.
`kubectl run nginx-pod --image nginx:alpine`

2. Deploy a messaging pod using the redis:alpine image with the labels set to tier=msg.
`kubectl run messaging --image redis:alpine --labels='tier=msg'`

3. Create a namespace named apx-x9984574
`kubectl create ns apx-x9984574`

4. Get the list of nodes in JSON format and store it in a file at /opt/outputs/nodes-z3444kd9.json.
`kubectl get nodes -o json > /opt/outputs/nodes-z3444kd9.json`

5. Create a service messaging-service to expose the messaging application within the cluster on port 6379. (Use imperative commands.)
`kubectl expose pod messaging --port 6379 --name messaging-service`

6. Create a deployment named hr-web-app using the image kodekloud/webapp-color with 2 replicas.
`kubectl create deployment hr-web-app --image kodekloud/webapp-color --replicas=2`

7. Create a static pod named static-busybox on the controlplane node that uses the busybox image and the command sleep 1000.
`kubectl run static-busybox --image=busybox --dry-run=client -o yaml --command -- sleep 1000 > static-busybox.yaml`
`cp static-busybox.yaml /etc/kubernetes/manifests`

8. Create a POD in the finance namespace named temp-bus with the image redis:alpine.
`kubectl run temp-bus -n finance --image=redis:alpine`

9. A new application orange is deployed. There is something wrong with it. Identify and fix the issue.
Init:CrashLoopBackOff indicates that there is a init container
`kubectl logs orange init-myservice`
`kubectl describe pod orange`
`kubectl edit pod orange`
sleeep 2; -> sleep
`k replace --force -f /tmp/kubectl-edit-357876292.yaml`

10. Expose the hr-web-app as service hr-web-app-service application on port 30082 on the nodes on the cluster. The web application listens on port 8080.
`kubectl expose deployment hr-web-app --type='NodePort' --port 8080 --name=hr-web-app-service`
`kubectl edit svc hr-web-app` edit `- nodePort:`

11. Use JSON PATH query to retrieve the osImages of all the nodes and store it in a file /opt/outputs/nodes_os_x43kj56.txt. The osImages are under the nodeInfo section under status of each node.
`kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.osImage}'`

12. Create a Persistent Volume with the given specification.
no imperative command
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-analytics
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /pv/data-analytics`
```

`k create -f <yaml>`

# MOCK EXAM #2:
1. Take a backup of the etcd cluster and save it to /opt/etcd-backup.db.


`ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key snapshot save /opt/etcd-backup.db`

(source)[https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster]


use info from here to fill the parameters of etcdctl command (cacert,cert,key)
`cat /etc/kubernetes/manifests/etcd.yaml`
OR 
`k get pods -n kube-system etcd-controlplane -o yaml` 

2. Create a Pod called redis-storage with image: redis:alpine with a Volume of type emptyDir that lasts for the life of the Pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: redis-storage
  name: redis-storage
spec:
  containers:
  - image: redis:alpine
    name: redis-storage
    volumeMounts:
    - mountPath: /data/redis
      name: cache
  volumes:
  - name: cache
    emptyDir:
      sizeLimit: 100Mi
```

3. Create a new pod called super-user-pod with image busybox:1.28. Allow the pod to be able to set system_time. The container should sleep for 4800 seconds.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: super-user-pod
  name: super-user-pod
spec:
  containers:
  - command:
    - sleep
    - "4800"
    image: busybox:1.28
    name: super-user-pod
    securityContext:
      capabilities:
        add: ["SYS_TIME"]
```

4. A pod definition file is created at /root/CKA/use-pv.yaml. Make use of this manifest file and mount the persistent volume called pv-1. Ensure the pod is running and the PV is bound. mountPath: /data

first create PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
```

next use it in POD manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: use-pv
  name: use-pv
spec:
  containers:
  - image: nginx
    name: use-pv
    volumeMounts:
    - mountPath: "/data"
      name: mypd
  volumes:
  - name: mypd
    persistentVolumeClaim:
      claimName: my-pvc
```

5. Create a new deployment called nginx-deploy, with image nginx:1.16 and 1 replica. Next upgrade the deployment to version 1.17 using rolling update.

create deployment
`kubectl create deploy nginx-deploy --image=nginx:1.16 --replicas=1`

update
`kubectl set image deployments nginx-deploy nginx=nginx:1.17`
or
`kubect set image deployment/nginx-deploy nginx=nginx:1.17` 

nxing= is a pod name

6. Create a new user called john. Grant him access to the cluster. John should have permission to create, list, get, update and delete pods in the development namespace . The private key exists in the location: `/root/CKA/john.key` and csr at `/root/CKA/john.csr`. 
*Important Note: As of kubernetes 1.19, the CertificateSigningRequest object expects a signerName.*

* Convert CSR to base64 `cat john.csr | base64 | tr -d "\n"`
* Paste to CSR yaml file
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john-developer
spec:
  request: <base64 CSR>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
```
* Submit CSR `kubectl create -f <csr.yaml>`
* Approve `kubectl certificate approve john-developer` 
* Create role `k create role developer --verb=create,list,get,update,delete --resource=pods -n development`
* Check if user can do something `k auth can-i get pods -n development  --as john`
* Create role binding `k create rolebinding john-developer --role=developer --user=john -n development`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: john-developer
  namespace: development
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: john
```

7. Create a nginx pod called nginx-resolver using image nginx, expose it internally with a service called nginx-resolver-service. Test that you are able to look up the service and pod names from within the cluster. Use the image: busybox:1.28 for dns lookup. Record results in /root/CKA/nginx.svc and /root/CKA/nginx.pod

* create pod `k run nginx-resolver --image nginx`
* expose it as clusterIP `k expose pod nginx-resolver --name nginx-resolver-service --port=80`
* test with busybox:
`k run busybox --image busybox:1.28 --command -- sleep 40000`
`k exec busybox -- nslookup nginx-resolver-service`
`k exec busybox -- nslookup 10-244-192-4.default.pod.cluster.local`



## MOCK EXAM 3
1. Create a new service account with the name pvviewer. Grant this Service account access to list all PersistentVolumes in the cluster by creating an appropriate cluster role called pvviewer-role and ClusterRoleBinding called pvviewer-role-binding. Next, create a pod called pvviewer with the image: redis and serviceAccount: pvviewer in the default namespace.

* create service account:
`k create serviceaccount pvviewer`

* create cluster role:
`kubectl api-resources | grep persisten` - check naming for resource
`k create clusterrole pvviewer-role --verb=list --resource=persistentvolumes` - could be also `--resource=pv`

* create role binding:
`k create clusterrolebinding pvviewer-role-binding --clusterrole=pvviewer-role --serviceaccount=default:pvviewer` 

* create pod, newer kubectl is missing --serviceaccount flag
`k run nginx --image=nginx --overrides='{"spec":{"serviceAccount": "pvviewer"}}' -o yaml --dry-run=client > pod-pvviewer.yaml` - kubectl run is missing --serviceaccount flag

2. List the InternalIP of all nodes of the cluster. Save the result to a file /root/CKA/node_ips. Answer should be in the format: InternalIP of controlplane `space`InternalIP of node01 (in a single line)

`k get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}' > /root/CKA/node_ips`

5. We have deployed a new pod called np-test-1 and a service called np-test-service. Incoming connections to this service are not working. Troubleshoot and fix it.
Create NetworkPolicy, by the name ingress-to-nptest that allows incoming connections to the service over port 80.

Important: Don't delete any current objects deployed.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-to-nptest
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: np-test-1
  policyTypes:
    - Ingress
    - Egress
  ingress:
  - 
    ports:
    - protocol: TCP
      port: 80
```

6. Taint the worker node node01 to be Unschedulable. Once done, create a pod called dev-redis, image redis:alpine, to ensure workloads are not scheduled to this worker node. Finally, create a new pod called prod-redis and image: redis:alpine with toleration to be scheduled on node01.

key: env_type, value: production, operator: Equal and effect: NoSchedule

`kubectl taint node node01 env_type=production:NoSchedule`

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: prod-redis
  name: prod-redis
spec:
  tolerations:
  - key: "env_type"
    operator: "Equal"
    value: "production"
    effect: "NoSchedule"
  containers:
  - image: redis:alpine
    name: prod-redis
```

7. Create a pod called hr-pod in hr namespace belonging to the production environment and frontend tier .
image: redis:alpine

Use appropriate labels and create all the required objects if it does not exist in the system already.

k run hr-pod --image redis:alpine -n hr --labels=environment=production,tier=frontend

8. A kubeconfig file called super.kubeconfig has been created under /root/CKA. There is something wrong with the configuration. Troubleshoot and fix it.

edit server in kubeconfig


9. We have created a new deployment called nginx-deploy. scale the deployment to 3 replicas. Has the replica's increased? Troubleshoot the issue and fix it.

edit kube-controller-manager, it is a static pod