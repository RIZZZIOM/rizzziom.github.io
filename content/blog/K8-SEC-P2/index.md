---
title: "Kubernetes Security #2"
date: 2025-12-24
draft: false
summary: "Understanding how to perform reconnaissance & gain initial foothold on a cluster."
tags: ["kubernetes testing"]
categories: ["blog"]
series: ["kubernetes"]
showToc: true
---

This blog is the second of the Kubernetes Security series where I'll talk about cluster reconnaissance and gaining initial foothold. 
<!--more-->
**Throughout the blog/series, I use 3 tools:**
- [docker](https://www.docker.com/get-started/) - container runtime
- [minikube](https://minikube.sigs.k8s.io/docs/start/) - local single node cluster
- [kubectl](https://kubernetes.io/docs/reference/kubectl/) - tool to interact with cluster

## Cluster Reconnaissance

Before we touch controls or try to exploit misconfigurations, we first need to map the cluster - the same way a penetration tester maps a network. Reconnaissance in Kubernetes is a process where we understand what's running, where it's running and who might have access to it.

Here are some of the key components we look into:

| **Component**        | **Description**                                                                        |
| -------------------- | -------------------------------------------------------------------------------------- |
| Namespace            | a logical folder. Policies/quotas and RBACs can scope to it.                           |
| Pod                  | 1 or more container that are always scheduled together                                 |
| Deployment           | keeps N identical pods running and handles rollout + self healing                      |
| Service              | stable name or IP to reach pods.                                                       |
| Secret               | key/value store for sensitive data                                                     |
| ServiceAccount       | identity for pods when they call the API                                               |
| Role/ClusterRole     | permission sets (namespace scoped or cluster wide) and who gets them                   |
| Node                 | a worker machine (VM/physical) that schedules pods, runs Kubelet and container runtime |
| Context/Cluster info | which cluster and control plane addresses you're on                                    |
Reconnaissance is the foundation of Kubernetes testing. It's how we build our cluster map - what namespaces exist, what's exposed, where sensitive data might be hiding etc. Everything we do next - from initial access to privilege escalation depends on what we discover here.

### Practical 1 - Reconnaissance Lab

Let's create a simple lab to understand how we perform reconnaissance inside a cluster.

*Start a new cluster using minikube*
```shell
$ minikube start
😄  minikube v1.37.0 on Debian kali-rolling
✨  Automatically selected the docker driver
📌  Using Docker driver with root privileges
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.48 ...
🔥  Creating docker container (CPUs=2, Memory=3072MB) ...
🐳  Preparing Kubernetes v1.34.0 on Docker 28.4.0 ...
🔗  Configuring bridge CNI (Container Networking Interface) ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

$ minikube status 
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

*We can then ensure our `kubectl` points to minikube:*
- Check which cluster `kubectl` talks to (avoid recon on the wrong cluster)
```shell
$ kubectl config current-context
minikube
```

- view core endpoints (API server, DNS) to prove the control plane is reachable
```shell
$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

*Once, we have ensured we are pointing at the right cluster, we can create an intentionally vulnerable lab using the following `yaml`*
```yaml
# recon-lab.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
---
apiVersion: v1
kind: Namespace
metadata:
  name: team-b
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: team-a
  labels: {app: web}
spec:
  replicas: 3
  selector:
    matchLabels: {app: web}
  template:
    metadata:
      labels: {app: web}
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: team-a
  labels: {app: web}
spec:
  type: NodePort
  selector: {app: web}
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: default
  labels: {app: hello}
spec:
  replicas: 1
  selector:
    matchLabels: {app: hello}
  template:
    metadata:
      labels: {app: hello}
    spec:
      containers:
        - name: hello
          image: nginxdemos/hello:plain-text
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: default
  labels: {app: hello}
spec:
  type: ClusterIP
  selector: {app: hello}
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alpine-sleeper
  namespace: team-b
  labels: {app: alpine}
spec:
  replicas: 2
  selector:
    matchLabels: {app: alpine}
  template:
    metadata:
      labels: {app: alpine}
    spec:
      containers:
        - name: alpine
          image: alpine:3.20
          command: ["sh","-c","tail -f /dev/null"]
---
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
  namespace: team-b
type: Opaque
stringData:
  username: demo_user
  password: SuperSecretP@ssw0rd
---
# Grants cluster-admin to the *default* SA in team-a
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: team-a-default-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: default
    namespace: team-a
```

*apply the config by:*
```shell
$ kubectl apply -f recon-lab.yaml    
namespace/team-a created
namespace/team-b created
deployment.apps/web created
service/web created
deployment.apps/hello created
service/hello created
deployment.apps/alpine-sleeper created
secret/db-creds created
clusterrolebinding.rbac.authorization.k8s.io/team-a-default-admin created
```

Here's what this creates:
- Namespaces : `team-a` and `team-b`
- Deployments/Pods: `web` , `hello` and `alpine-sleeper`
- Services: `web`,  `hello`
- Secrets: `db-creds`
- RBAC binding: `team-a-default-admin`

*After creating our lab, we can start by listing namespaces. These are Kubernetes isolation boundaries. They group resources and set default policies/quotas. If everything is in `default`, that's a red flag. We can also check the labels as they often hint at the owners or environments.*
```shell
$ kubectl get ns
NAME              STATUS   AGE
default           Active   7m43s → default namespace
kube-node-lease   Active   7m41s → created by minikube
kube-public       Active   7m44s → created by minikube
kube-system       Active   7m46s → created by minikube
team-a            Active   2m
team-b            Active   2m

$ kubectl get ns --show-labels
NAME              STATUS   AGE     LABELS
default           Active   9m4s    kubernetes.io/metadata.name=default
kube-node-lease   Active   9m2s    kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   9m5s    kubernetes.io/metadata.name=kube-public
kube-system       Active   9m7s    kubernetes.io/metadata.name=kube-system
team-a            Active   3m21s   kubernetes.io/metadata.name=team-a
team-b            Active   3m21s   kubernetes.io/metadata.name=team-b
```

*Then, we can list all pods across namespaces; this shows what's deployed and which nodes they're running on. It's common to find leftover dev workloads or monitoring tools with broader privs than they should have.*
```shell
$ kubectl get pods -A        
NAMESPACE     NAME                               READY   STATUS    RESTARTS      AGE
default       hello-7fc9b7988b-stbbl             1/1     Running   0             9m53s
kube-system   coredns-66bc5c9577-8f6rr           1/1     Running   0             13m
kube-system   etcd-minikube                      1/1     Running   0             15m
kube-system   kube-apiserver-minikube            1/1     Running   0             15m
kube-system   kube-controller-manager-minikube   1/1     Running   2             15m
kube-system   kube-proxy-cnl4d                   1/1     Running   0             13m
kube-system   kube-scheduler-minikube            1/1     Running   0             15m
kube-system   storage-provisioner                1/1     Running   1 (12m ago)   14m
team-a        web-fd748c9d5-66twr                1/1     Running   0             9m57s
team-a        web-fd748c9d5-w5qsl                1/1     Running   0             9m56s
team-a        web-fd748c9d5-zxndl                1/1     Running   0             9m56s
team-b        alpine-sleeper-68df567b4c-g66mp    1/1     Running   0             9m50s
team-b        alpine-sleeper-68df567b4c-gw4hj    1/1     Running   0             9m49s

$ kubectl get deploy -A
NAMESPACE     NAME             READY   UP-TO-DATE   AVAILABLE   AGE
default       hello            1/1     1            1           12m
kube-system   coredns          1/1     1            1           17m
team-a        web              3/3     3            3           13m
team-b        alpine-sleeper   2/2     2            2           12m
```
- `-A` displays pods across all namespaces
- For additional information about the pods, we can run `kubectl get pods -A -o wide`. This will display:
	- The pod's internal IP
	- Node where the pod is scheduled
	- Nominated node (Used for preemption (usually `<none>`))
	- Readiness gates (Extra readiness conditions (usually `<none>`))
- Similarly, adding `-o wide` to any other kubectl `get` command displays additional information.

*Since we discovered a deployment in `team-a` as well as `team-b` and `default`, we can find more details about it using the `desribe` query*
```shell
$ kubectl -n team-a describe deploy web
Name:                   web
Namespace:              team-a
CreationTimestamp:      Mon, 22 Dec 2025 13:18:35 -0500
Labels:                 app=web
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=web
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=web
  Containers:
   nginx:
    Image:         nginx:1.25-alpine
    Port:          80/TCP
    Host Port:     0/TCP
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   web-fd748c9d5 (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  15m   deployment-controller  Scaled up replica set web-fd748c9d5 from 0 to 3
  
$ kubectl -n team-b describe deploy alpine-sleeper
Name:                   alpine-sleeper
Namespace:              team-b
CreationTimestamp:      Mon, 22 Dec 2025 13:18:42 -0500
Labels:                 app=alpine
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=alpine
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=alpine
  Containers:
   alpine:
    Image:      alpine:3.20
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
      -c
      tail -f /dev/null
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   alpine-sleeper-68df567b4c (2/2 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  15m   deployment-controller  Scaled up replica set alpine-sleeper-68df567b4c from 0 to 2
 
$ kubectl describe deployment hello                
Name:                   hello
Namespace:              default
CreationTimestamp:      Mon, 22 Dec 2025 13:18:37 -0500
Labels:                 app=hello
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=hello
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=hello
  Containers:
   hello:
    Image:         nginxdemos/hello:plain-text
    Port:          80/TCP
    Host Port:     0/TCP
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   hello-7fc9b7988b (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  15m   deployment-controller  Scaled up replica set hello-7fc9b7988b from 0 to 1
```
- `-n` is used to specify the namespace where the deployment is present. By default, kubectl will run all queries in the `default` namespace.

*We can then check for services. This will show how workloads are exposed. A `ClusterIP` is an internal only IP. `NodePort` is open for every node and `LoadBalancer` is a cloud facing virtual IP.*
```shell
$ kubectl get svc -A -o wide
NAMESPACE     NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE   SELECTOR
default       hello        ClusterIP   10.103.6.77    <none>        80/TCP                   18m   app=hello
default       kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP                  22m   <none>
kube-system   kube-dns     ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP,9153/TCP   22m   k8s-app=kube-dns
team-a        web          NodePort    10.99.185.45   <none>        80:30080/TCP             18m   app=web

# view service present in team-a and default namespace
$ kubectl -n team-a describe svc web
Name:                     web
Namespace:                team-a
Labels:                   app=web
Annotations:              <none>
Selector:                 app=web
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.99.185.45
IPs:                      10.99.185.45
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30080/TCP
Endpoints:                10.244.0.3:80,10.244.0.4:80,10.244.0.5:80
Session Affinity:         None
External Traffic Policy:  Cluster
Internal Traffic Policy:  Cluster
Events:                   <none>

$ kubectl -n default describe svc hello
Name:                     hello
Namespace:                default
Labels:                   app=hello
Annotations:              <none>
Selector:                 app=hello
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.103.6.77
IPs:                      10.103.6.77
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
Endpoints:                10.244.0.6:80
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

- The service on *team-a* namespace is a `Nodeport`, hence can be accessible outside the cluster. To test it,
```shell
$ minikube service -n team-a web --url
http://192.168.49.2:30080

$ curl http://192.168.49.2:30080
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
```

*Secrets can also be listed by*
```shell
$ kubectl get secrets -A
NAMESPACE     NAME                     TYPE                            DATA   AGE
kube-system   bootstrap-token-aokh2x   bootstrap.kubernetes.io/token   6      32m
team-b        db-creds                 Opaque                          2      27m
```

This will reveal a secret in the `team-b` namespace. To find more information about it,
```shell
$ kubectl -n team-b describe secret db-creds
Name:         db-creds
Namespace:    team-b
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  19 bytes
username:  9 bytes

$ kubectl -n team-b get secret db-creds -o yaml
apiVersion: v1
data:
  password: U3VwZXJTZWNyZXRQQHNzdzByZA==
  username: ZGVtb191c2Vy
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Secret","metadata":{"annotations":{},"name":"db-creds","namespace":"team-b"},"stringData":{"password":"SuperSecretP@ssw0rd","username":"demo_user"},"type":"Opaque"}
  creationTimestamp: "2025-12-22T18:18:42Z"
  name: db-creds
  namespace: team-b
  resourceVersion: "687"
  uid: dadee146-7424-465f-a802-0883b55f636a
type: Opaque

# to view keys/values, we can also filter the output by json
$ kubectl -n team-b get secret db-creds -o jsonpath='{.data.username}' | base64 -d; echo
demo_user

$ kubectl -n team-b get secret db-creds -o jsonpath='{.data.password}' | base64 -d; echo
SuperSecretP@ssw0rd
```

*Finally, we can look for interesting permissions and privileges by checking the `rolebindings` and `clusterrolebindings`*
```shell
$ kubectl get clusterrolebindings
NAME                                 ROLE                            AGE
cluster-admin                ClusterRole/cluster-admin               39m
kubeadm:cluster-admins       ClusterRole/cluster-admin               39m
....
....
team-a-default-admin         ClusterRole/cluster-admin               35m

$ kubectl get rolebindings -A
NAMESPACE     NAME                        ROLE                                AGE
kube-system   kube-proxy                  Role/kube-proxy                     41m
kube-system   kubeadm:kubelet-config      Role/kubeadm:kubelet-config         42m
....
....

# to view service account where roles are bound
$ kubectl get serviceaccounts -A
NAMESPACE         NAME                                          SECRETS   AGE
default           default                                       0         44m
kube-node-lease   default                                       0         44m
kube-public       default                                       0         44m
kube-system       attachdetach-controller                       0         44m
...
...
team-a            default                                       0         40m
team-b            default                                       0         40m
```

*`team-a-default-admin` Binds `ClusterRole/cluster-admin` to the **default service account in namespace `team-a`**. This means any pod running as `team-a:default` service account has **full cluster admin access***

This should reveal a cluster role binding on a service account in `team-a` namespace. To find more information about it,
```shell
$ kubectl get clusterrolebinding team-a-default-admin -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"ClusterRoleBinding","metadata":{"annotations":{},"name":"team-a-default-admin"},"roleRef":{"apiGroup":"rbac.authorization.k8s.io","kind":"ClusterRole","name":"cluster-admin"},"subjects":[{"kind":"ServiceAccount","name":"default","namespace":"team-a"}]}
  creationTimestamp: "2025-12-22T18:18:44Z"
  name: team-a-default-admin
  resourceVersion: "693"
  uid: 2df76ba9-3d4e-4c60-9727-f0c9ee99d212
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: default
  namespace: team-a
```

Feel free to explore other configs present in the lab. To delete it,
```shell
$ kubectl delete -f recon-lab.yaml
namespace "team-a" deleted
namespace "team-b" deleted
deployment.apps "web" deleted from team-a namespace
service "web" deleted from team-a namespace
deployment.apps "hello" deleted from default namespace
service "hello" deleted from default namespace
deployment.apps "alpine-sleeper" deleted from team-b namespace
secret "db-creds" deleted from team-b namespace
clusterrolebinding.rbac.authorization.k8s.io "team-a-default-admin" deleted
```

## Gaining Foothold

Now that we know what’s inside the cluster, we can begin exploring initial access vectors. When we talk about _initial access_ in Kubernetes, we mean the very first step in a compromise, i.e how someone gets their foot in the door. It might be an exposed dashboard, a leaked kubeconfig file, or a container image that's been tampered with. Because Kubernetes is so modular, there are many possible ways.

These are some of the most common vectors that I have seen or read about:
1. **Publicly exposed dashboards or APIs**: Accessible without authentication or using default creds.
2. **Insecure images or registries**: Pulling unverified or outdated container images.
3. **Leaked kubeconfig files or tokens**: Stored in repos or CI/CD logs.
4. **Secrets in version control**: Hardcoded tokens, API keys, passwords.
5. **Weak credentials or default admin accounts**: Common in dev clusters or early setups.

We can use tools like **[Trivy](https://trivy.dev/)** or **[Grype](https://github.com/anchore/grype)** to flag known CVE's and hardcoded secrets before deployment. Similarly, we can scan codebases for leaked credentials using tools like **[Gitleaks](https://github.com/gitleaks/gitleaks)** and **[Trufflehog](https://github.com/trufflesecurity/trufflehog)**. Registries and dashboards can be restricted to authenticated users only to prevent unauthorized access.

### Practical 2 - Scanning With Trivy

1. *We will use `debian:bookworm-slim` as the target. Download it using the below command*

```shell
$ docker pull debian:bookworm-slim                                                             
bookworm-slim: Pulling from library/debian
ae4ce04d0e1c: Pull complete 
Digest: sha256:e899040a73d36e2b36fa33216943539d9957cba8172b858097c2cabcdb20a3e2
Status: Downloaded newer image for debian:bookworm-slim
docker.io/library/debian:bookworm-slim
```

2. *Run Trivy against the target*

```shell
$ docker run --rm aquasec/trivy:latest image --scanners vuln -q --severity HIGH,CRITICAL --format table -q debian:bookworm-slim
  
Report Summary

┌─────────────────────────────────────┬────────┬─────────────────┐
│               Target                │  Type  │ Vulnerabilities │
├─────────────────────────────────────┼────────┼─────────────────┤
│ debian:bookworm-slim (debian 12.12) │ debian │        5        │
└─────────────────────────────────────┴────────┴─────────────────┘
Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)


debian:bookworm-slim (debian 12.12)
===================================
Total: 5 (HIGH: 4, CRITICAL: 1)

# table of details about the vulns found
```
> Trivy supports other formats aswell.

Hence, using it we scanned the layers that make up the image. This includes the OS packages, libraries etc.

**If this image (or its base OS) has known holes, attackers can chain them with a misconfig (e.g., exposed service) to run code, read files, or pivot. Using `:latest` is risky: you deploy whatever is newest today, which might include fresh CVEs without you noticing.**

### Practical 3 - Scanning With Gitleaks

Another common mechanism of gaining a foothold is by using creds left behind in CI/CD logs or git history. **Gitleaks** let's us find these sensitive credentials before it is pushed to the pipeline

Let's explore this practically:
- create repo with secrets
```shell
mkdir leaky-repo && cd leaky-repo
git init -q
```

- create `config.py` for secrets:
```python
AWS_ACCESS_KEY_ID = "AKIADEMOFAKEKEY1234"
AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYDEMOFAKEKEY"
SLACK_BOT_TOKEN = "xoxb-111111111111-222222222222-EXAMPLE"
print("hello world")
```

- commit this and then remove it and recommit the file
```shell
git add config.py
git commit -m "init: add project config"
# then remove the secret from the file
git add config.py
git commit -m "fix: remove junk"
```

- download **gitleaks** and run it on the repo
```shell
$ gitleaks detect --source=. -v       

    ○
    │╲
    │ ○
    ○ ░
    ░    gitleaks

Finding:     SLACK_BOT_TOKEN = "xoxb-111111111111-222222222222-EXAMPLE"
Secret:      xoxb-111111111111-222222222222-EXAMPLE
RuleID:      slack-bot-token
Entropy:     3.966666
File:        config.py
Line:        3
Commit:      47107127d5a434c1d56bdd814bd6472da9621c89
Author:      kali-user
Email:       kali@example.local
Date:        2025-10-07T19:07:34Z
Fingerprint: 47107127d5a434c1d56bdd814bd6472da9621c89:config.py:slack-bot-token:3

9:38AM INF 2 commits scanned.
9:38AM INF scanned ~298 bytes (298 bytes) in 612ms
9:38AM WRN leaks found: 1
```

We can also verify it using 
```shell
$ git show 47107127d5a434c1d56bdd814bd6472da9621c89
commit 47107127d5a434c1d56bdd814bd6472da9621c89
Author: kali-user <kali@example.local>
Date:   Tue Oct 7 15:07:34 2025 -0400

    init: add project config

diff --git a/config.py b/config.py
new file mode 100644
index 0000000..7dd982b
--- /dev/null
+++ b/config.py
@@ -0,0 +1,3 @@
+AWS_ACCESS_KEY_ID = "AKIADEMOFAKEKEY1234"
+AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYDEMOFAKEKEY"
+SLACK_BOT_TOKEN = "xoxb-111111111111-222222222222-EXAMPLE"
```

- to fully purge the secrets from git history, we can use tools like **filter-repo** or do it manually with git.

*lab cleanup*
```shell
cd ../ && rm -rf leaky-repo
docker system prune -f
```

### Managed Interface

When we're using managed Kubernetes service like **GKE**, **AKS**, **EKS**, our control plane runs in the cloud provider's infrastructure. This means new risks:
- API endpoints can sometimes be publicly reachable
- IAM roles can be over-privileged
- dashboards can be left open to the internet.

During testing, we always check whether the control plane endpoint is private and whether IAM roles are tightly scoped. A simple `gcloud` or `aws eks describe-cluster` command can reveal if our API endpoint is exposed. Managed platforms abstract some risks, but they also introduce new ones at the IAM and network boundary.

## Closure

Those were some of the ways we can map out the cluster and gain initial foothold. Usually, when we get the foothold, we are limited to a standard user level. Hence, the next logical step would be to elevate privileges before we perform other post exploitation tactics. In the next blog, we'll look into post exploitation tactics including privilege escalation, credential access, lateral movement and persistence in clusters.

Until next time :)

---