# Chapter 4

`adapter container` - modify data, ingress

storage / memory limit will cause container to evict, cpu can be overused.

containers are trasient

LAB answer
- one per pod
- one per pod
- one per pod
- 1
- dns, label, loopback, shared file system, ipc
- monitoring / logging, eg sidecar container

# Chapter 5

`CSI` - container storage interface

`persistent volume` - volume plugins, independant lifecycle to pod, provides an API object on top of NFS, iSCSI or cloud storage

`PersistentVolumeClaim` - request for storage by a user like a pod

`persistentVolumeReclaimPolicy` - Retain, Recycle (deprecated), Delete, dynamic claimed by default is Delete, important data persistency can use Retain

encoded data are passed using a Secret - ssh keys, passwords

non-encoded data are passed as ConfigMap - /etc/hosts file

volume can be accessible for multi-container pods or multiple pods. There is no concurrency checking, data corruption is possible

`emptyDir:{}` is a type of storage created inside consider and gets destroyed with the container lifecycle

`StorageClass API` - allows an admin to define a persistent volume provisioner of a certain type, dynamic provisioning

```
volumes:
- name: test
  emptyDir: {}
  containers:
- name: busy
  image: busybox
  volumeMounts:
  - mountPath: /busy
    name: test
- name: box
  image: busybox
  volumeMounts:
  - mountPath: /box
    name: test
```
`rbd` volume persists the data when pod is destroyed

`Secrets`
- can be used as environment variable in a Pod, 1MB size
- can also be mounted as volume

```
kubectl get secret
```

```
kubectl create secret generic
```

```
spec:
    containers:
    - image: mysql
    env
    - name: MY_PASSWORD
      valueFrom:
        secretKeyRef:
            name: mysql
            key: password

```

`ConfigMaps` decouples config artifacts from images
- use as env vars
- used in pod commands
- populate volumes
- add configMap data to path in Volume

`Config Status`
- availableReplicas: how many were configured by the replicaset
- observedGeneration: how often the deployment has been updated

```
kubectl rollout history
```

`Scaling vs rolling updates`

- scaling changes the immutable values such as replicatset, when replicatset = 0, no containers
- rolling changes the no-immutable values such as version of the container and triggers rolling udpates

annotate a rollout

```
$ kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.91 --record=true
```

check history of rollouts

```
$ kubectl rollout history deployment.v1.apps/nginx-deployment
```

rollback to previous version

```
$ kubectl rollout undo deployment.v1.apps/nginx-deployment
```

rollback to previous version

```
$ kubectl rollout history deployment.v1.apps/nginx-deployment --revision=2
```

pause and resume deployments
```
kubectl pause
kubectl resume
```

`Job`

starts a new job when the first pod fails or deleted

```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

# Chapter 6

`Authentication`
- certificates, tokens or basic auth
- Users are managed external to kubernetes
- API access are processed by System accounts
- Webhooks
- OpenID

Process

```
*Request => API server => 401 => Authorisation => admission control*
```

`ABAC, RBAC, Webhook`

```--authorization-mode=ABAC
--authorization-mode=RBAC
--authorization-mode=Webhook
--authorization-mode=AlwaysDeny
--authoization-mode=AlwaysAllow
(user,group, namespace, verb)
```

### `ABAC` ###

```
{
  "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
  "kind": "Policy",
  "spec": {
    "user": "warren",
    "namespace": "default",
    "resource": "pods",
    "readonly": "false"
  }
}
```

###  `RBAC` ###

API Groups - core, apps\
Roles - group of rules \
Cluster Roles - scope of entire cluster

Each operation can be bound to 3 objects - `User Accounts, Service Accounts, Groups`, deny or allow based

<span style="color:pink">User is not an object of API server</span> - OpenSSL certificartes is used for cert authorisation. context is used for reference

`` apiGroups, resources, verbs``

check or create namespace => create cert credentitals => set context for user to the namespace<sup>[1](#f1)</sup> => create role for the task => bind user to the role => verify user access<sup>[2](#f2)</sup>

<a name="f1">1</a>: authentication\
<a name="f2">2</a>: authorization

### `admission controls` ###

modify the request content or validate it

>*Initializer* controller allow dynamic modification of the API request


>*ResourceQuota* controller validate quota limitation


### ``` Security Context ``` ###
security policies for processes in the container

<pre>
apiVersion: v1
kind: Pod
metadata: 
  name: nginx
spec:<span style="color: red">
  securityContext:
    runAsNonRoot: true</span>
  containers:
  - image: nginx:latest
    name: nginx
</pre>


### ``` Pod Security Policies (PSP) ``` ###

 cluster level security rules

defined via kube manifest

<pre>
apiVersion: extension/v1beta1
kind: PodSecurityPolicy
metadata: 
  name: restricted
spec:
  seLinux:
    rule: RunAsAny
  supplenmentalGroups:
    rule: RunAsAny
  <span style="color: red">runAsUser:
    rule: MustRunAsNonRoot</span>
  fsGroup:
    rule: RunAsAny
</pre>

PSP + RBAC = fine tuned privileges

### ``` Network Security Policies ``` ###

By default all pods can reach each other\
all ingress and egress traffic is allowed

Network policies are used to restrict accesses

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: 
  name: ingress-egress-policy
  namespace: default
spec:
- podSelector:
    matchedLabels:
      role: db
  policyTypes:
  - ingress
  - egress
  ingress:
  - from: 
    - ipBlock: 
        cidr: 172.17.0.0/16
        except: 
        - 172.17.1.0/24
    - nameSpaceSelector:
        matchedLabels:
          project: myproject
    - podSelector:
        matchedLabels:
          role: frontend
    ports:
    - protocols: TCP
      port: 8080
  egress:
  - to:
    - ipBlock:
      cidr: 10.0.0.0/24
    ports: 
    - protocol: TCP
      port: 8081
```

 any pods labeled *role: frontend* with ip ranges 172.17.0.0 to 172.255.255 (except for  172.17.1.\*)  are allowed for outbound traffic on TCP port 8080. 
 ### `namespaceselector??` ###

Chapter 7
===
### ``Service Type`` ###
- ClusterIP
  - default
  - inter-cluster communcation
- NodePort 
  - static IP
  - generate a clusterIP automatically
  - only high-ports (30000-32767) can be used map to ClusterIP via iptables / ipvs
  - 2<sup>15</sup>  
  - can be called via *NodeIP:NodePort*
- LoadBalancer
  - generate a *NodePort*
  - send async call to external LB
  - pass request 
- ExternalName 
  - no selector
  - no ports
  - no endpoints
  - CName record,
  - Redirection happens at DNS level
  - e.g. point to external integration point
  - ```
    spec: 
      Type: ExternalName
      externalName: wap.db.pickles.com
    ```

```
kubectl proxy - local service to access Cluster IP
```

**kube-proxy** watches API service resource.

api server -> kubeproxy -> seviceIP (ipvs/iptables) -> backend pods

```
mode=iptables
mode=IPVS  // fastest, more scalable
mode=userspace
```

### `` Labels `` ###
determines which Pods should receive traffic from service, e.g. in rolling updates, update the pods label first then change the label in the service to match to the pods like version number

```
kubectl expose deployment/nginx --port=80 --type=NodePort
```

### `` Ingress `` ###

**Ingress Resource** is API object with a list of HTTP rules matched against all incoming requests for both hosts and path.


**Ingress Controller** - fan out to services, name-based hosting, TLS, or LB. Combining L4 and L7 ingress and requesting name or IP via claims. 

```
Ingress Controller -> (Services) x n
```


