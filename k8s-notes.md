## CORE CONCEPTS  
### 2.46. Solution - Imperative Commands:
- Deploy a pod named nginx-pod using the nginx:alpine image.

`kubectl run nginx-pod --image=nginx:alpine`

- Deploy a redis pod using the redis:alpine image with the labels set to tier=db.

`kubectl run redis --image=redis:alpine --labels="tier=db"`

- Create a service redis-service to expose the redis application within the cluster on port 6379.

  `kubectl expose pod redis --name="redis-service" --port=6379`

- Create a deployment named webapp using the image kodekloud/webapp-color with 3 replicas.

`kubectl create deployment webapp --image=kodekloud/webapp-color --replicas=3`

- Create a new pod called custom-nginx using the nginx image and expose it on container port 8080.

`kubectl run custom-nginx --image=nginx --port=8080`

- Create a new namespace called dev-ns
`kubectl create namespace dev-ns`

- Create a new deployment called redis-deploy in the dev-ns namespace with the redis image. It should have 2 replicas.

`kubectl create deploy redis-deploy --image=redis --namespace=dev-ns --replicas=2`

- Create a pod called httpd using the image httpd:alpine in the default namespace. Next, create a service of type ClusterIP by the same name (httpd). The target port for the service should be 80.

`kubectl run httpd --image=httpd:alpine`
`kubectl expose pod httpd --port=80`
OR
`kubectl run httpd --image=httpd:alpine --port=80 --expose=true`


## 3.58 Taints and Tolerations
`kubectl taint node node01 spray=mortein:NoSchedule`

## MISC
wszystkie pody do json i wyciagam nazwe obrazu
`kubectl get pod -o json | jq '.items[].spec.containers[].image'`

definicja replicasetu do pliku yaml i do pliku
`kubectl get replicasets new-replica-set -o yaml > new-replica-set.yaml`

ustawienia VIM:
  set smartindent
  set tabstop=2
  set expandtab
  set shiftwidth=2

jakies inne ustawienia VIM
    :set tabstop=2 shiftwidth=2 expandtab
    :retab

    tabstop   -> Indentation width in spaces
    shiftwidth  -> Autoindentation width in spaces
    expandtab   -> Use actual spaces instead of tabs
    retab     -> Convert existing tabs to spaces

