# Kubectl

```bash
kubectl version
kubectl get all --all-namespaces
kubectl get namespaces
kubectl get ns
kubectl get replicasets
kubectl get rs
kubectl get services
kubectl get svc
kubectl get nodes -o wide
kubectl get pods -o wide
kubectl get pod --namespace=account-management  -o wide
kubectl get pods --all-namespaces -o wide
kubectl get pods -A -o wide
kubectl get replicationcontroller
kubectl get rc

kubectl create -f ./<filename>.yml

kubectl delete all --all
```

* Deployments create replicasets. Replicasets create pods.
* Supported formats are: `yaml`, `wide`, `json`, `name`.
* To create a resource in a different namespace, specify the namespace in the YAML file below `name` or append `-n` when creating the resource.



YAML (https://en.wikipedia.org/wiki/YAML):


## Pods

Lifecycle: Pending (waiting for being scheduled), ContainerCreating, Running, Terminated.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: nginx
    image: nginx
```

Create and delete the pod with the following commands:

```bash
kubectl create -f pod-definition.yml
kubectl delete pod myapp-pod
kubectl run --help
kubectl run nginx --image=nginx
kubectl run httpd --image=httpd:alpine --port=80 --expose
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml
kubectl run ubuntu --image=ubuntu -it
kubectl create service clusterip redis-service --tcp=6379:6379
kubectl edit pod nginx
kubectl describe pod nginx
kubectl create -f pod-definition.yml
kubectl apply -f pod-definition.yml
kubectl get pod <pod_name> -o yaml > my-new-pod.yaml
```

### Pod Volumes

Pods can has a `volumes` section were volumes of different kind are defined. These can be pod local, node local or cloud provider persistent storage. The container then has `volumeMountPoints` where the volumes are referenced and mounted.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-volume
spec:
  containers:
  - image: mroc/example-app
    name: example-app
    volumeMounts:
    - mountPath: /log
      name: log-volume
  volumes:
  - name: log-volume
    hostPath:
      path: /var/log/webapp
      type: Directory
```



### Pod Resources 

Resource requests for CPU and memory with request and limits. Without limit and request it can happen that a pod takes all resources. Note: CPU can be throttled, memory not. With memory the pod will be killed. More information can be found here https://kubernetes.io/docs/concepts/policy/limit-range/.

A pod can request or limit the resources per container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-limits
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "1Gi"
        cpu: 0.5
      limits:
        memory: "2Gi"
        cpu: 1
```

A LimitRange can be for CPU or memory and are applied namesoace-wide:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range-cpu
spec:
  limits:
  - default:
      cpu: 500m
    defaultRequest:
      cpu: 500m
    max:
      cpu: "1"
    min:
      cpu: 100m
    type: Container
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range-memory
spec:
  limits:
  - default:
      memory: 1Gi
    defaultRequest:
      memory: 1Gi
    max:
      memory: 1Gi
    min:
      memory: 500Mi
    type: Container
```

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: resource-quota
spec:
  hard:
    requests.cpu: 4
    requests.memory: 4Gi
    limits.cpu: 10
    limits.memory: 10Gi
```

```bash
kubectl delete limitranges --all
kubectl delete resourcequotas --all
```

### Pod Initialization

If there is a initialization task on a pod, put it in a `initContainers` section.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-init
spec:
  containers:
  - name: nginx
    image: nginx
  initContainers:
  - name: init
    image: ubuntu
    command:
    - "sleep"
    - "10"
```

### Readiness and liveness Probe

If the container is started, the service can still be in its warm-up phase. For that, use the `readinessProbe` property to let Kubernetes ask, when a pod is ready for accepting traffic. There are different kind of probes: `httpGet`, `tcpSocket`, and `exec`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-readiness-probe
spec:
  containers:
  - name: nginx
    image: nginx
    readinessProbe:
      httpGet:
        path: /api/ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 1
      failureThreshold: 10
    livenessProbe:
      httpGet:
        path: /api/live
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 1
      failureThreshold: 10
```

```yaml
      tcpSocket:
        port: 3306
```

```yaml
      exec:
        command:
          - cat
          - /tmp/healthy
```

```sh
for i in {1..20}; do
   kubectl exec --namespace=kube-public curl -- sh -c 'test=`wget -qO- -T 2  http://webapp-service.default.svc.cluster.local:8080/ready 2>&1` && echo "$test OK" || echo "Failed"';
   echo ""
done

```


## Replication Controller

Use ReplicaSet instead, see below.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```


## Replica Set

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replica-set
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
      - name: example-app
        image: mroc/example-app
```

```bash
kubectl get replicaset
kubectl get rs
kubectl create -f rs-definition.yml
kubectl replace -f rs-definition.yml
kubectl scale -replicas=6 -f rs-definition.yml
kubectl scale rs <rs-name> --replicas=4
kubectl delete rs <rs-name>
kubectl edit rs <rs-name>

```

## Deployments

A deployment triggers a roll out. A new roll out creates a new deployment revision. A deployment first creates a new replica set and then fades out the old replica set. Deployments can be created imperatively:

```bash
kubectl get deployments
kubectl create deployment nginx --image=nginx --replicas=4 --dry-run=client -o yaml > depl.yaml
kubectl create deployment nginx --image=k8s.gcr.io/echoserver:1.4
kubectl expose deployment nginx --type=NodePort --port=8080
kubectl create deployment nginx --image=httpd:2.4-alpine --replicas=3 --dry-run=client -o yaml > depl.yaml
kubectl create deployment nginx --image=nginx --replicas=4 --dry-run=client -o yaml > depl.yaml
```

To create a deployment and see its history, use the following commands:

```bash
kubectl get deployments
kubectl create -f deployment.yml
kubectl apply -f deployment.yml --record
kubectl edit deployment my-deployment --record --record
kubectl rollout status deployment my-deployment
kubectl rollout history deployment my-deployment
kubectl rollout undo deployment my-deployment --record
kubectl rollout undo deployment my-deployment --to-revision=1
```

The `--record` saves the command in the history.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  template:
    metadata:
      name: my-pod
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.24.0
  replicas: 3
  selector: 
    matchLabels:
      tier: frontend
```

To update the image of an existing deployment, use the following command:

```bash
kubectl set image deployments/<deployment> <container>=<image>
```

Updates can be rolled out with differnent strategies: Recreate, Rolling Update (one by one), Blue/Green (All with 100% switch), Canary. Recreate and rolling update are built in.

For blue/green deployment we deploy a complete separate instance with a alternating names (blue/green) and a new label (v1/v2/v3...). Then update the service the using the label selector to match the new version. Once switched, delete the old deployment.

For canary, we create a new deployment canary side-by-side with only a single replica. Both deployments have same label that is used by the service label selector.


## Services

Services connect pods with inside and outside the cluster. Services are bound to pods via label selectors.

If a pod should be available from outside the cluster, a `NodePort` is required. Its `nodePort` is the port outside the node ranging between 30000 - 32767. The `port` is the service's port inside the node. And the `targetPort` is the port of the pod we want to make accessible:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-node-port
spec:
  type: NodePort
  ports:
  - targetPort: 80
    port: 80
    nodePort: 30008
  selector:
    app: myapp
    type: front-end
```

If a pod should be available from inside the cluster, a `ClusterIP` is required. To address services, use `service.namespace.svc.cluster.local`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-cluster-ip
spec:
  type: ClusterIP
  ports:
  - targetPort: 80
    port: 80
  selector:
    app: myapp
    type: backend-end
```

A service can also be created with the expose command:

```bash
kubectl expose deployment <deployment-name> --type=NodePort --port=80 --name=<service-name>
```


A `LoadBalancer` that provisions a load balancer by cloud providers. It comes with a separate external IP through which the cluster can be accessed from the internet. Usually a load balancer has some cloud provider specific settings:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-load-balancer
spec:
  type: LoadBalancer
  ports:
  - targetPort: 80
    port: 80
  selector:
    app: myapp
    type: backend-end
```


## Ingress

An ingress controller is required to configure the routing rules between load balancers and nodes. Kubernetes has no ingress controller by default. This must be explicitely deployed. A common one is `nginx`. In its simplest frorm it requires a Deployment, Service, ConfigMap and a Service Account (Auth).

```bash
kubectl get ingress
kubectl create ingress <ingress-name> --rule="host/path=service:port"
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-srv
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
        - path: /api/users/?(.*)
          pathType: ImplementationSpecific
          backend:
            service:
              name: auth-clusterip-srv
              port:
                number: 3000
        - path: /(.*)
          pathType: ImplementationSpecific
          backend:
            service:
              name: client-clusterip-srv
              port:
                number: 3000
```

More options to be aware of:

```yaml
nginx.ingress.kubernetes.io/rewrite-target: /
nginx.ingress.kubernetes.io/ssl-redirect: "false"
```

Ingress is when traffic reaches an service. Egress is when it leaves to another service. Pods on different nodes are in a virtual network so they can talk to each other.


## Network policy

With a network policy we can control the traffic that can reach a pod. The policy is attached to the pod with a pod selector. More here https://kubernetes.io/docs/concepts/services-networking/network-policies/.

For example, when we want to limit traffic from and to the DB only for a specific pod.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api-pod
      # and
      namespaceSelector:
        matchLabels:
          name: prod
    # or
    - namespaceSelector:
        matchLabels:
          name: prod
    ports:
    - protocol: TCP
      port: 3306
  egress:
  - {}
```

```bash
kubectl get netpol
```


## Namespaces

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

```bash
kubectl get ns
kubectl create ns <namespace-name>
kubectl get all --all-namespaces
```

A resource is either namespace-scoped (pods, replicaset, roles, role-bindings...) or cluster-scoped (nodes)! To see all types that are in a namespace, run `kubectl api-resources --namespaced=true`. To see all cluster-based types run `kubectl api-resources --namespaced=false`.

## Environment variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-env-variable
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: APP_COLOR_2
      value: "green"
```


## ConfigMaps

```bash
kubectl get cm
kubectl describe cm app-config
kubectl create cm <config-name> --from-literal=<key>=<value>
kubectl create cm app-config --from-literal=APP_COLOR=blue
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  VARIABLE_1: hello
  VARIABLe_2: world
```

```
kubectl create -f ./ConfigMap.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-config-map
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: APP_COLOR
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_COLOR
    envFrom:
      - configMapRef:
          name: app-config
```


## Secrets

```bash
kubectl get secrets
kubectl describe <secret_name>
kubectl create secret generic <secret_name> --from-literal=<key>=<value>
kubectl create secret generic db-secret --from-literal=DB_Host=sql01 --from-literal=DB_User=root --from-literal=DB_Password=password123
```


### Secret.propertis

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  VARIABLE: dmFsdWUtMg0KDQo=
```

```bash
kubectl create -f ./Secret.yaml
```

```yaml
envFrom:
  - secretRef:
      name: app-secret
```


### Encrypt secret

Note: Secrets are not encrypted. Everyone with access to the cluster can see the secrets. To encrypt secrets, a `EncryptionConfiguration` is required as described in https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/. There is also a video about https://www.youtube.com/watch?v=MTnQW9MxnRI.

https://www.udemy.com/course/certified-kubernetes-application-developer/learn/lecture/34549240#overview

```bash
kubectl create secret generic -h
kubectl create secret generic my-secret --from-literal=key1=supersecret
echo "c3VwZXJzZWNyZXQ=" | base64 --decode
head -c 32 /dev/urandom | base64
```


## Security Contexts

With security contexts, you can set the user, group, and system capabilities permissions of a container or pod. See https://kubernetes.io/docs/tasks/configure-pod-container/security-context/. Run `kubectl exec <pod-name. -- whoami` to see which user runs a container. See which user runs a container by running `ps -a`. To change a user, add a security context to the pod definition.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: sec-ctx-demo
    image: busybox:1.28
    command: [ "sh", "-c", "sleep 1h" ]
    volumeMounts:
    - name: sec-ctx-vol
      mountPath: /data/demo
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        add: ["SYS_TIME"]
```

## Service Accounts

When a pod requires to communicate with Kubernetes API server, it needs a service account.

You can create a service account and short lived 1h token like following. However it is advised to use the TokenRequestAPI! See https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_token/.:

```bash
kubectl get sa
kubectl create sa my-service-account
kubectl create token my-service-account
kubectl describe sa my-service-account
```

When starting a pod, the standard service account is mounted into it:

```bash
kubectl run ubuntu --image=ubuntu -it
kubectl describe pod ubuntu
ls /var/run/secrets/kubernetes.io/serviceaccount
kubectl get pod ubuntu -o yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-sa-sample
  annotations:
    kubernetes.io/service-account.name: "<SERVICEACCOUNT>"
type: kubernetes.io/service-account-token
data:
  extra: YmFyCg==
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-dashboard
  namespace: default
spec:
  template:
    ...
    spec:
      serviceAccountName: <SERVICEACCOUNT>
      containers:
      - image: ubuntu
        ports:
        - containerPort: 8080
          protocol: TCP
```

## Taint and tolerations

With taints we can prefer certain pods on certain nodes. Taints are set on nodes and tolerations are set on pods. When placing a taint on a node, there are three effects: `NoSchedule`, `PreferNoSchedule`, and `NoExecute`.

```bash
kubectl taint node <node-name> key=value:<taint-effect>
kubectl taint nodes node1 app=blue:NoSchedule
```

To remove a taint, end with minus:

```bash
kubectl taint node <node-name> key=value:<taint-effect>-
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-tolerations
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
```

## Node selector

With a node selector we can specify which kind of node a pod should run on by using labels, both on the node and the pod.

```bash
kubectl get nodes --show-labels
kubectl label nodes <node-name> <label-key>=<label-value>
kubectl label nodes node1 size=Large
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-node-selector
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    size: Large
```

## Node affinity

With node affinty we can specify which kind of node a pod should run on by using labels, both on the node and the pod. Similar to node selector but more powerful. See https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity and https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/.


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-node-affinity
spec:
  containers:
  - name: nginx
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - Large
            - Medium
```

Beside `requiredDuringSchedulingIgnoredDuringExecution` there is also `preferredDuringSchedulingIgnoredDuringExecution`. The first has a hard requirment and if not met, the pod is not scheduled. The second tries to fulfill the requirement but if not met, the pod is scheduled anyway.

## Multi-container pods

The spec section in a pod or deployment allows to specify more than one container. There are three type of multi-containers design-patterns:

* Sidecar: For example, deploying a logging agent alognside the main container.
* Adapter: For example, adapting the output of a container to a different format.
* Ambassador: For example, a container that acts as a proxy to another container.

## Persistent Volumes

By default, pod storage gets deleted when the pod gets deleted. To have persistent storage, volumes are used. More here https://kubernetes.io/docs/concepts/storage/persistent-volumes/.

Here an example for a persistent storage that maps to the host path (which is actually also not very persistent):

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: persistent-volume-host-path
spec:
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/data
```

Here an example for a persistent storage that maps to a AWS storage:

```yaml
apiVersion: v1
kind: persistent-volume-aws
metadata:
  name: pv-vol
spec:
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  capacity:
    storage: 1Gi
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ext4
```

```bash
kubectl get pv
```

Using claims, persistent volumes can be bound to pods. More information can be found here: https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: persistent-volume-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

```bash
kubectl get pvc
```

To bind a pod to a claim, use the following:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: pv-claim
```

To bind a volume to a claim, use `claimRef` like the following:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-app-pv
spec:
  storageClassName: local-storage
  claimRef:
    name: example-app-pv-claim
    namespace: default
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /run/desktop/mnt/host/wsl/data/example-app-pv
```


### Static Provisioning

With static privisioning, a volume needs to be created in the cloud environment like this:

```bash
gcloud compute disks create my-disk --size 1GB
```

And then mounted like this:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol
spec:
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  capacity:
    storage: 1Gi
  gcePersistentDisk:
    volumeID: my-disk
    fsType: ext4
```

### Dynamic Privisioning

With dynamic provisioning, the cloud provider automatically creates storage when a claim is made. This first requires a storage class:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: none
```

```bash
kubectl get sc
```

Than a `PersistentVolumeClaim` can references the storage class by name. For Google Cloud Platform:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-claim
spec:
  storageClassName: google-storage
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

For Digital Ocean, the `storageClassName` changes to `do-block-storage`.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-claim
spec:
  storageClassName: do-block-storage
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

## Stateful set

When pods require a certain name and order when they are created, a stateful set is used. Ordered graceful deployment:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  labels:
    app: mysql
spec:
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql-h
  podManagementPolicy: OrderedReady 
```

```bash
kubectl get sts
```

To reach a specific pod from the ordered pods created by a `StatefulSet`, we need a headless service that does not load balance the request over all pods but allows to address a specific one. It can be created like this and has `clusterIp` set to `None`. The headless service is referenced by the `StatefulSet` with the `serviceName` above:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-h
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
```

To get a pod always get the same storage, the `volumeClaimTemplates` can be used:

```yaml
...
spec:
  ...
  volumeClaimTemplates:
  - metadata
    name: data-volume
  spec:
    accessModes:
    - ReadWriteOnce
    storageClassName: google-storage
    resources:
      requests:
        storage: 500Mi
```

```
podname.headless-servicename.namespace.svc-cluster-domain.example
mysql-0.mysql-h.default.svc.cluster.local
```

## Logging

With `kubectl logs <pod-name>` the logs of a pod can be displayed. More information here `kubectl logs -h`. With `-f` the logs can be streamed live. If a pod has multiple containers, the container needs to be explicitely specified:

```bash
kubectl logs -f <pod-name>
kubectl logs -f <pod-name> <container-name>
```

## Monitoring

For monitoring, different solutions exist like `Metrics Server`, `Prometheus`, `Elastic Stack`, `Datadog`, `dynatrace`.

Example: You can have one `Metrics Server` per cluster in memory. It gets sent data from the kubelet through the `cAdvisor`. To install, execute `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`. Then `kubectl top node` and `kubectl top pod` can be used.


## Labels and selectors

Group and filter using labels. For example when filtering by type, functionality or application. See https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/ for more information.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-limits
  labels:
    app: myapp
    type: front-end
spec:
  containers:
  - name: nginx
    image: nginx
```

The filter with applying a selector on the query like this:

```bash
kubectl get pods --selector app=myapp
kubectl get all --selector env=prod && bu=finance &&  tier=frontend
```

You can also print the labels of all pods like this:
  
```bash
kubectl get pods --show-labels
```

Useful labels could be `env` for environment, `bu` for business unit.

Annotations are used to record other things like build number, release number, etc.:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-limits
  annotations:
    buildversion: 3.14
spec:
  containers:
  - name: nginx
    image: nginx
```

## Jobs

Jobs start containers and run them to completion. They are used for batch processing. Jobs can be parallel or sequential. Jobs can be used for one-time tasks like database migrations, backups, etc. Jobs can be used for parallel tasks like processing multiple files or for equential tasks like processing files in a certain order.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: math-job
spec:
  completions: 6
  parallelism: 2
  template:
    spec:
      containers:
      - name: math-add
        image: ubuntu
        command: ["expr", "3", "+", "2"]
      restartPolicy: Never
```

```bash
kubectl create -f ./job.yaml
kubectl get jobs
kubectl get pods
kubectl logs math-job-<id>
kubectl delete job math-job
```

A cron job runs automatically at certain times. See https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/ for mor information.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cron-job
spec:
  # https://linuxhandbook.com/crontab/
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cron-job-pod
            image: ubuntu
            command: ["expr", "3", "+", "2"]
          restartPolicy: Never
```


```bash
kubectl create -f ./cron-job.yaml
kubectl get cronjobs
kubectl get pods
kubectl logs cron-job-pod-<id>
kubectl delete cronjobs cron-job
```

## Security

### Basics

Infrastructure must be secured: Root access disabled, password based authentication disabled, SSH key based authentication enabled. Who can access the cluster (authentication)? And what can they do (authorization)?

### Authentication

Users and service accounts. User are authenticated by `kube-apiserver` which supports static password or token files, certificates and external providers like LDAP. When running `kubectl`, the config file usually has the certificates to authenticater the user at the `kube-apiserver`. Then the request goes through an authorization process.


### Configuration

Kubernetes stores configuration in a file located at `~/.kube/config`. The config file has three sections: Clusters, Contexts and Users. Contexts bind a user to a cluster.

```
kubectl config -h
kubectl config view
kubectl config current-context
kubectl config use-context <context-name>
kubectl config --kubeconfig=/root/my-kube-config use-context research
```

A context can also specify a default namespace like `namespace: finance`.

### API

All resources are grouped into different API groups. At top there is the core API and then named API groups. Under an API group there are different resources and below a resource there are the verbs, e.g. `/apis/apps/v1/deployments/list`.

The API is versioned in order: `v1alpha`, `v1beta`, `v1`, `v2alpha`, `v2beta`, `v2`. An API can also support multiple versions with one being the preferred version. Sometimes APIs are also deprecated and later removed. For different version tiers, different deprecation rules apply. This means for example, once a feature is in `v1` it needs to be supported for at leasts 12 months or three releases, whichever is longer.

When an old API is removed, the YAML files need to be upgraded. For this we can use the `kubectl convert -f <old-file> --output-version <new-api>` command. The convert tool is a plug in that needs to be installed first.


Start a proxy and then connect to the API server like this:

```bash
kubectl proxy
curl localhost:8001 -k
curl localhost:8001/apis/authorization.k8s.io
```

### Authorization

Authorization mechanisms are: Node, ABAC, RBAC, Webhook.

* `Node` Authorizer authorize the calls made from a kubelet (node) to the Kube API.
* `ABAC` is attribute based access controls. For example `dev` user can create pods. This is done by editing the policy file.
* `RBAC` is rule based access contol is defining roles with access and then assinging users to roles. This is the standard approach.
* `Webhook` can be used to outsource to 3rd parties like *Open Policy Agent*.

When running `kubectl describe pod kube-apiserver-docker-desktop -n kube-system`, the enabled authorization schemes are listed as command arguments to the kube API server. In this example, `Node` and `RBAC` is enabled:

```Yaml
Containers:
  kube-apiserver:
    Container ID: ...
    Command:
      kube-apiserver
      --advertise-address=192.168.65.3
      --allow-privileged=true
      --authorization-mode=Node,RBAC
 ```

### RBAC

First we need to create a role. Note that the role only works in the namespace where it is created. If no namespace is specified it is `default`:

```Yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMaps"]
  verbs: ["create"]  
```

```bash
kubectl get roles
kubectl describe role developer
kubectl create role developer --resource=pods --verbs=list,create,delete

```

Then we need to create a role binding to bind the user to a role:

```Yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-role-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl get rolebindings
kubectl describe rolebinding dev-user-role-binding
```

To check if you as a user can do something you can use the `can-i` command:

```bash
 kubectl auth can-i create deployments
 kubectl auth can-i create deployments --as dev-user
```

### Cluster Roles

Cluser roles and bindings allow access to all objects in a cluster, not limited to a particular namespace.

```Yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
```

```Yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create clusterrole nodes-access --resource=persistentvolumes,nodes --verb=get,list,create,update,delete
kubectl create clusterrolebinding nodes-access-binding --clusterrole=nodes-access --user=michael
```

### Admission Controllers

Admission controllers are plugins for certain functionality that go beyond roles and clusterroles. There are two types also performed in this order:

* *Mutating admission controller* can change a request, for example the `DefaultStorageClass`
* *Validating admission controller* validates a request and can reject it.

Examples for built-in admission controllers are: AllwaysPullImages, DefaultStorageClass, EventRateLimit, NamespaceLifecycle.
 
Admission controllers can be turned on and off in the kubernetes control plane configuration. To call the api server you can exec commands on the kubebernetes controlplane pod:

```
kubectl exec kube-apiserver-controlplane -n kube-system -- kube-apiserver -h
kubectl describe pod kube-apiserver-controlplane -n kube-system
```

From the kubernetes host itself the settings can be found in:

```
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Additional admission controller can be integrated via MitatingAdmissionWebhook and ValidatingAdmissionWebhook. For example:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "pod-policy.example.com"
webhooks:
- name: pod-policy.example.com
  clientConfig:
    service:
      name: "webhook-service"
      namespace: "webhook-namespace"
    caBundle: "ASD...xclkjcs"
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
```

## Custom Resource Definition

With custom resource definitions, it is possible to add own type of objects to a kubernetes cluster. It is also possible to add a custom controller in the cluster that observes the resources and executes changes depending on them.

https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: "flighttickets.flights.com"
spec:
  names:
    kind: FlightTicket
    singular: flightticket
    plural: flighttickets
    shortNames:
    - ft
  group: flights.com
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              from:
                type: string
              to:
                type: string
              number:
                type: integer
                minimum: 1
                maximum: 10
```

```yaml
kind: FlightTicket
apiVersion: flights.com/v1
metadata:
  name: flight-space
spec:
  from: "Berlin"
  to: "Paris"
  number: 2
```

## Operator Framework

A Kubernetes operator is a method of packaging, deploying, and managing a Kubernetes application. A Kubernetes application is both deployed on Kubernetes and managed using the Kubernetes API (application programming interface) and kubectl tooling. There are many operators available at https://operatorhub.io/. usually operators bring applications that are based on Kubernetes Custom Resources and controllers.

## Helm

Help is a tool that helps installing applications that usually consists out of many YAML files. It's similar to an installer on a traditional operating system where you specify the install location and the installer takes care of copying all the files. Help does this for Kubernetes resources. It has a `values.yaml` file where you can edit the values for the installation. Installation instructions can be found at https://helm.sh/docs/intro/install/.

```bash
helm version
```

The YAML files can be saved in a folder named `templates` and then variables can be used like `{{ .Values.image }}` which can reference `values.yaml` like `image: wordpress:4.8-apache`. Templates with values gives final definition files and are named *Helm Chart*. Usually a chart also has a `Chart.yaml` that hase some more informations.

```bash
helm search hub wordpress
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo wordpress
helm install <release-name> <chart-name>
helm list
helm pull --untar bitnami/wordpress
```

## How to debug and troubleshoot

Source: https://medium.com/@ManagedKube/kubernetes-troubleshooting-ingress-and-services-traffic-flows-547ea867b120

- Check thew pod is running `kubectl get pods -o wide`.
- Check the log of a pod `kubectl logs <pod_name>`.
- Check list of services `kubectl get service`.
- Check service in particular `kubectl describe service <service_name>`. Endpoints must point to the correct pods. Check the IP is accessible from another pod within the cluster as the service is a ClusterIP.
- Check the ingress `kubectl get ingress`. Here the hosts should be listed.
- Check `kubectl describe ingress <ingress_name>`.
- Find ingress pod `kubectl get pods -n ingress-nginx`.
- Connect to ingress pod `kubectl exec -it <ingress_pod_name> -n ingress-nginx sh` and run `curl -H "HOST: echo1.example.com" localhost`.
- Check ingress logs `kubectl logs <ingress_pod_name> -n ingress-nginx`.


Source: https://kubernetes.github.io/ingress-nginx/troubleshooting/

- Check `kubectl get ing -n <namespace-of-ingress-resource>`
- `kubectl get pods -n <namespace-of-ingress-controller>`
- `kubectl exec -it -n <namespace-of-ingress-controller> <ingress_controller> -- cat /etc/nginx/nginx.conf`


## Abbrevations

```bash
kubectl api-resources --namespaced=true
```

```
NAME                        SHORTNAMES   APIVERSION                     KIND
bindings                                 v1                             Binding
configmaps                  cm           v1                             ConfigMap
endpoints                   ep           v1                             Endpoints
events                      ev           v1                             Event
limitranges                 limits       v1                             LimitRange
persistentvolumeclaims      pvc          v1                             PersistentVolumeClaim
pods                        po           v1                             Pod
podtemplates                             v1                             PodTemplate
replicationcontrollers      rc           v1                             ReplicationController
resourcequotas              quota        v1                             ResourceQuota
secrets                                  v1                             Secret
serviceaccounts             sa           v1                             ServiceAccount
services                    svc          v1                             Service
controllerrevisions                      apps/v1                        ControllerRevision
daemonsets                  ds           apps/v1                        DaemonSet
deployments                 deploy       apps/v1                        Deployment
replicasets                 rs           apps/v1                        ReplicaSet
statefulsets                sts          apps/v1                        StatefulSet
localsubjectaccessreviews                authorization.k8s.io/v1        LocalSubjectAccessReview
horizontalpodautoscalers    hpa          autoscaling/v2                 HorizontalPodAutoscaler
cronjobs                    cj           batch/v1                       CronJob
jobs                                     batch/v1                       Job
leases                                   coordination.k8s.io/v1         Lease
endpointslices                           discovery.k8s.io/v1            EndpointSlice
events                      ev           events.k8s.io/v1               Event
ingresses                   ing          networking.k8s.io/v1           Ingress
networkpolicies             netpol       networking.k8s.io/v1           NetworkPolicy
poddisruptionbudgets        pdb          policy/v1                      PodDisruptionBudget
rolebindings                             rbac.authorization.k8s.io/v1   RoleBinding
roles                                    rbac.authorization.k8s.io/v1   Role
csistoragecapacities                     storage.k8s.io/v1              CSIStorageCapacity

```

## Examples

```bash
kubectl run example-app --image=crccheck/hello-world --labels="app=example-app"

kubectl create service nodeport example-app --tcp=8000:8000 --node-port=30080
http://localhost:30080/

kubectl create service clusterip example-app --tcp=8000:8000
kubectl run ubuntu -n monitoring --image=ubuntu -it
apt update
apt install wget
wget clusterip:8000
```