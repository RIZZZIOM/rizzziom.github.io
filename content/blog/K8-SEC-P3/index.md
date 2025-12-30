---
title: "Kubernetes Security #3"
date: 2025-12-30
draft: false
summary: "Exploring privesc and post exploitation mechanisms in Kubernetes cluster."
tags: ["kubernetes testing"]
categories: ["blog"]
series: ["kubernetes"]
showToc: true
---

This blog is the third in the Kubernetes Security series, where I’ll talk about privilege escalation and other post-exploitation tactics within a cluster.
<!--more-->
**Throughout the blog/series, I use 3 tools:**

- [docker](https://www.docker.com/get-started/) - container runtime
- [minikube](https://minikube.sigs.k8s.io/docs/start/) - local single node cluster
- [kubectl](https://kubernetes.io/docs/reference/kubectl/) - tool to interact with cluster

## Understanding Privilege Escalation

In the previous [blog](https://ziomsec.com/blog/k8-sec-p2/), we looked into how we could gain initial foothold inside a cluster. Usually, the user we get access to at this time isn't high privileged. So, the next logical step would be to gain higher privileges. In Kubernetes, when someone with limited access, like a pod running in one namespace, manages to gain higher privileges than intended, that is termed **Privilege Escalation**.

This could mean moving from a single pod to controlling the node, or even the cluster. It's one of the most important things to test because once an attacker reaches `cluster-admin` (equivalent to root), every other control becomes irrelevant.

Privilege escalation has 2 main forms:
- **Vertical** - moving up the hierarchy ( pod → node → cluster )
- **Horizontal** - moving sideways into other namespaces or roles ( user account → another user account in the same / different namespace )

![](https://cdn.ziomsec.com/k8-sec-p3/1.webp)

Some of the usual suspects for privilege escalation are:
- **Privileged Containers** - When you run a pod with `privileged: true` or with host mounts, that container can literally see and change the host filesystem.
- **Cluster-Admin bindings** - These are often leftovers from debugging or CI/CD setups. If your service account is bound to that role, you own the cluster.
- **Service account token abuse** - every pod gets a token by default, and if you use it outside the pod, you can often escalate by querying the API server.
- **Hostpath mounts / hostpid** - these namespaces can allow direct node access.

### Practical 1 - Using Privileged Pod For Escalation

*Let's take a look at how a pod running with dangerous attributes like `privileged: true`, `hostPID: true` etc can lead to privilege escalation.*

- The following `yaml` creates 2 namespaces - 1 hardened and 1 permissive.

```yaml
# privesc-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: psa-restricted
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
---
apiVersion: v1
kind: Namespace
metadata:
  name: psa-privileged
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: latest
```

```shell
# apply it
kubectl apply -f privesc-ns.yaml

# verify the namespaces have been created
kubectl get ns --show-labels | grep psa-
```

**Kubernetes Pod Security Admission** lets us label namespaces with policy levels. These policy levels control what kinds of pods are allowed to run in a namespace and help enforce security best practices at the cluster level.
- `restricted` - this is the safest. It enforces very strict security controls, such as preventing privileged containers and limiting access to the host system. This level is designed for most workloads and follows Kubernetes security best practices by default.
- `privileged` - this allows most host level powers. It permits pods to use advanced capabilities like host networking, host paths, and privileged containers. This level is typically used only for system components or trusted workloads that need deep access to the node.

Now that the namespaces are deployed, we can create a privileged pod.

```yaml
# priv-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: priv-pod
  namespace: psa-restricted # change to psa-privileged for no restriction
spec:
  hostPID: true
  hostNetwork: true
  containers:
    - name: demo
      image: alpine:3.20
      command: ["sh","-c","sleep infinity"]
      securityContext:
        privileged: true
        allowPrivilegeEscalation: true
        capabilities:
          add: ["SYS_ADMIN","NET_ADMIN"]
      volumeMounts:
        - name: host-etc
          mountPath: /host-etc
          readOnly: true
  volumes:
    - name: host-etc
      hostPath:
        path: /etc
        type: Directory
```

The above configuration will try deploying this privileged pod in the namespace with restricted PSA. This pod is allowed to see and interact with the host's process ID, use the host's network namespace instead of its own, gives the container full access to the host, allows processes inside the container to gain higher privileges and add specific linux capabilities to the container that grants admin control over system and network settings.

```shell
# deploy the pod
$ kubectl apply -f priv-pod.yaml 
Error from server (Forbidden): error when creating "priv-pod.yaml": pods "priv-pod" is forbidden: violates PodSecurity "restricted:latest": host namespaces (hostNetwork=true, hostPID=true), privileged (container "demo" must not set securityContext.privileged=true), allowPrivilegeEscalation != false (container "demo" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "demo" must set securityContext.capabilities.drop=["ALL"]; container "demo" must not include "NET_ADMIN", "SYS_ADMIN" in securityContext.capabilities.add), restricted volume types (volume "host-etc" uses restricted volume type "hostPath"), runAsNonRoot != true (pod or container "demo" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "demo" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

As seen above, this POD wont be deployed in a restricted environment because of the level of privs it required. However, when the same pod is deployed in a non-restricted environment, it can be used to escalate privs.

> change the `namespace` to `psa-privileged` in the pod config yaml file.

```
$ kubectl apply -f priv-pod.yaml
pod/priv-pod created
```

Let's escalate our priv from inside this pod.
- spawn a shell inside the pod
```
$ kubectl -n psa-privileged exec -it priv-pod -- sh
/ # whoami
root
/ # ls
bin       etc       host-etc  media     opt       root      sbin      sys       usr
dev       home      lib       mnt       proc      run       srv       tmp       var
/ # 
```

- As seen above, we can read the host `etc` from within the pod. We can read its contents:
```
/ # cd host-etc/
/host-etc # ls
X11            environment  ...........
/host-etc # cat shadow
root:*:20319:0:99999:7:::
daemon:*:20319:0:99999:7:::
bin:*:20319:0:99999:7:::
sys:*:20319:0:99999:7:::
sync:*:20319:0:99999:7:::
games:*:20319:0:99999:7:::
man:*:20319:0:99999:7:::
lp:*:20319:0:99999:7:::
mail:*:20319:0:99999:7:::
news:*:20319:0:99999:7:::
uucp:*:20319:0:99999:7:::
proxy:*:20319:0:99999:7:::
www-data:*:20319:0:99999:7:::
backup:*:20319:0:99999:7:::
list:*:20319:0:99999:7:::
irc:*:20319:0:99999:7:::
gnats:*:20319:0:99999:7:::
nobody:*:20319:0:99999:7:::
_apt:*:20319:0:99999:7:::
_rpc:*:20340:0:99999:7:::
systemd-network:*:20340:0:99999:7:::
systemd-resolve:*:20340:0:99999:7:::
statd:*:20340:0:99999:7:::
sshd:*:20340:0:99999:7:::
docker:*:20340:0:99999:7:::
```

- we can also read configuration files inside `kubernetes` directory found in the host's `etc`
```
/host-etc/kubernetes # ls
addons                   controller-manager.conf  manifests                super-admin.conf
admin.conf               kubelet.conf             scheduler.conf
```
> This gives us certificates and keys that can be used to query the Kubernetes API for more information.

Here's all the vulnerabilities in this pod that can be exploited.
1. `privileged: true` removes most kernel restrictions normally applied to the container.
2. `hostPID: true` exposes all host processes to the container.
3. `hostNetwork: true` places the pod directly on the node’s network stack.
4. `SYS_ADMIN` is a capability that enables many sensitive kernel operations. `NET_ADMIN` allows full control over network interfaces and routing.
5. Mounting `/etc` exposes host configuration files.
6. `allowPrivilegeEscalation: true` permits processes to gain higher privileges.

### Practical 2 - The Power Of `Cluster-Admin` Role Bindings

*A regular service account is not permitted to read cluster secrets. However, when it has `cluster-admin` rights, this changes.*

The following config yaml creates a secret and namespace with a low privileged service account.
```
# setup.yaml
apiVersion: v1
kind: Secret
metadata:
  name: demo-secret
  namespace: kube-system
type: Opaque
stringData:
  password: kubeR00t!
---
apiVersion: v1
kind: Namespace
metadata:
  name: bindings-demo
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: viewer
  namespace: bindings-demo
```

```
kubectl apply -f setup.yaml
```

This creates a secret in the `kube-system` namespace and a service account in another new namespace called `bindings-demo`.

If, we try checking if we are authorized to get secrets, we will be forbidden.
```shell
$ kubectl auth can-i --as=system:serviceaccount:bindings-demo:viewer -A get secrets
no

$ kubectl --as=system:serviceaccount:bindings-demo:viewer -n kube-system get secret demo-secret
Error from server (Forbidden): secrets "demo-secret" is forbidden: User "system:serviceaccount:bindings-demo:viewer" cannot get resource "secrets" in API group "" in the namespace "kube-system"
```

Hence, we cannot read the secrets. If I bind this account to `Clusteradmin` role, I would be privileged to read the secret present in the other namespace. The following yaml can be used to bind the service account to `clusteradmin` role.
```
# cluster-admin.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bindings_demo-viewer-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: viewer
    namespace: bindings-demo
```

```shell
kubectl apply -f cluster-admin.yaml
```

With `Clusteradmin` role, I can list the secret.

```shell
$ kubectl --as=system:serviceaccount:bindings-demo:viewer -n kube-system get secret demo-secret
NAME          TYPE     DATA   AGE
demo-secret   Opaque   1      6m6s

$ kubectl --as=system:serviceaccount:bindings-demo:viewer -n kube-system get secret demo-secret -o yaml
apiVersion: v1
data:
  password: a3ViZVIwMHQh
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Secret","metadata":{"annotations":{},"name":"demo-secret","namespace":"kube-system"},"stringData":{"password":"kubeR00t!"},"type":"Opaque"}
  creationTimestamp: "2025-12-29T17:28:51Z"
  name: demo-secret
  namespace: kube-system
  resourceVersion: "6024"
  uid: 9e61ecc0-e06f-49ac-ad0a-150bf2a37dc8
type: Opaque

$ echo 'a3ViZVIwMHQh' | base64 -d
kubeR00t!    
```

Usually, when we find these bindings, we create a backdoor for persistence so that we can have privileged access later aswell.

- The following config yaml can be used to create another service account with clusteradmin privs
```
# backdoor.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: backdoor-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: default
    namespace: kube-system
```

```shell
kubectl apply -f backdoor.yaml
```

After applying it, I can verify that this new account indeed has cluster admin privileges:
```shell
$ kubectl auth can-i --as=system:serviceaccount:kube-system:default get secrets -A
yes
```

- cleanup
```shell
kubectl delete -f backdoor.yaml --ignore-not-found
kubectl delete -f cluster-admin.yaml --ignore-not-found
kubectl delete -f setup.yaml
```

## Credential Access

After gaining elevated privileges, we can extract credentials. Kubernetes store data in `Secrets`. These secrets are base64 encoded and not encrypted unless encryption is enabled at rest.

Every pod also has a service account token automatically mounted inside `/var/run/secrets/...`. If our RBAC is too loose, that token might give admin-level access. Even environment variables and cloud metadata endpoints can leak credentials if not locked down.

![](https://cdn.ziomsec.com/k8-sec-p3/2.webp)

### Practical 3 - Using Pod Token For Privilege Escalation

This lab will show how a pod's built in token can access the Kubernetes API with whatever permissions that pod's Service Account has.

The following configuration yaml creates:
- namespace → `token-impact` 
- pod → `tokenpod`
- clusterrole → `secret-reader` (get and list secrets)
- service account → `default` 
- secret → `db`
```
# token-impact-lab.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: token-impact
---
apiVersion: v1
kind: Secret
metadata:
  name: db
  namespace: token-impact
type: Opaque
stringData:
  password: "P@ssw0rd!"
---
# Allow ONLY get/list of secrets in this namespace to the default SA
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: token-impact
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default-read-secrets
  namespace: token-impact
subjects:
- kind: ServiceAccount
  name: default
  namespace: token-impact
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: secret-reader
---
# Simple pod; token auto-mounts under /var/run/secrets/...
apiVersion: v1
kind: Pod
metadata:
  name: tokenpod
  namespace: token-impact
spec:
  serviceAccountName: default
  containers:
  - name: curl
    image: curlimages/curl
    command: ["/bin/sh","-c","sleep infinity"]
```

```shell
# deploy the lab
kubectl apply -f token-impact-lab.yaml
kubectl -n token-impact wait --for=condition=Ready pod/tokenpod --timeout=60s
```

Because the service account is bound to a role that allows secret access, the mounted token can be used to authenticate to the Kubernetes API and list secrets.

> **Note: this is possible because the configuration file defines a role of retrieval and listing namespace wide secrets and binds this to the service account on the namespace. Using this token, our pod will communicate as the service account with the Kubernetes API.**

```shell
$ kubectl -n token-impact exec tokenpod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token

eyJhbGciOiJSUzI1NiIsImtpZCI6ImNNRllpOFR5cW5lTWtmUDlxalF1VENMaHFBT1B5MElldTN3ZWEwOE9kVjgifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzk4NjA4MzMyLCJpYXQiOjE3NjcwNzIzMzIsImlzcyI6Imh0dHBzOi8va3V....
```

This revealed the JWT token. This token is signed by the API server and hence cannot be modified as of now.

```shell
$ kubectl -n token-impact exec tokenpod -- sh -lc 'curl -sSk -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" "https://kubernetes.default.svc/api/v1/namespaces/token-impact/secrets"'

{
  "kind": "SecretList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "1240"
  },
  "items": [
    {
      "metadata": {
        "name": "db",
        "namespace": "token-impact",
        "uid": "dc5c68b7-e2b3-4f87-9f83-5ca04e42a26d",
        "resourceVersion": "664",
        "creationTimestamp": "2025-12-30T05:25:28Z",
        "annotations": {
          "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Secret\",\"metadata\":{\"annotations\":{},\"name\":\"db\",\"namespace\":\"token-impact\"},\"stringData\":{\"password\":\"P@ssw0rd!\"},\"type\":\"Opaque\"}\n"
        },
        "managedFields": [
          {
            "manager": "kubectl-client-side-apply",
            "operation": "Update",
            "apiVersion": "v1",
            "time": "2025-12-30T05:25:28Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {
              "f:data": {
                ".": {},
                "f:password": {}
              },
              "f:metadata": {
                "f:annotations": {
                  ".": {},
                  "f:kubectl.kubernetes.io/last-applied-configuration": {}
                }
              },
              "f:type": {}
            }
          }
        ]
      },
      "data": {
        "password": "UEBzc3cwcmQh"
      },
      "type": "Opaque"
    }
  ]
}             

# decoding the secret (remember, they are base64 encoded)
$ echo 'UEBzc3cwcmQh' | base64 -d
P@ssw0rd!
```

- cleanup
```
kubectl delete -f token-impact-lab.yaml
```

## Lateral Movement

Once the credentials are stolen, the next step is lateral movement. Attackers look for reused service accounts, open network paths or exposed Kubelet APIs. By default, pods can talk to any other pod, unless a `NetworkPolicy` is defined. This means that without a network policy, it is possible to access pods in other namespaces. The entire cluster would act like 1 big flat LAN.

### Practical 4 - Cross Namespace Communication

*This lab shows how easy it is for a pod running in 1 namespace to communicate with a pod running in another namespace without appropriate network policies.*

The following configuration `yaml` creates 2 namespace and a small web server and client pod in them.
- Namespace → `server` and `client`
- Pod → `web` and `curlpod`
- service → `web` (for the `web` pod)
```
# cross-ns-demo.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: server
---
apiVersion: v1
kind: Namespace
metadata:
  name: client
---
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: server
  labels:
    app: web
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
  namespace: server
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: curlpod
  namespace: client
spec:
  containers:
    - name: curl
      image: curlimages/curl:latest
      command: ["/bin/sh","-c","sleep infinity"]
```

```shell
# deploy it
kubectl apply -f cross-ns-demo.yaml
# wait for it to be running
kubectl -n server  wait --for=condition=Ready pod/web --timeout=90s
kubectl -n client  wait --for=condition=Ready pod/curlpod --timeout=60s
```

Once both the pods are up and running, we can execute a `curl` command from inside the client pod (`curlpod`) to access the pod running in another namespace.

```shell
$ kubectl -n client exec curlpod -- curl -m 5 -s http://web.server.svc.cluster.local

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

*This is why every namespace should begin with a default-deny `NetworkPolicy`, and only known flows should be explicitly allowed.*

- cleanup

```shell
kubectl delete -f cross-ns-demo.yaml
```

## Persistence In Kubernetes

Persistence is about making sure our access sticks around. There are multiple ways of gaining persistence:
- Deploying a Cronjob that respawns our pod.
- Modifying a static pod on a node
- Registering a malicious admission controller that injects sidecars into new pods.

> **[Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)** are code within the Kubernetes API server that check the data arriving in a request to modify a resource. They are applied to requests that create, delete or modify objects.
> 
> **[Sidecars](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)** are secondary containers that run along with the main application container within the same pod.


### Practical 5 - Exploring Ways Of Maintaining Persistence

*This lab will help grasp the concept of establishing persistence inside a cluster. For simplicity, we will create a cronjob that prints some text inside a file.*

The following configuration `yaml` creates a namespace called `jobs-allowed` and a role called `cron-writer` that allows the service account of the namespace to create, list, get and delete cronjobs.

```
# cron-setup.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jobs-allowed
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cron-writer
  namespace: jobs-allowed
rules:
- apiGroups: ["batch"]
  resources: ["cronjobs"]
  verbs: ["create","get","list","delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default-can-cron
  namespace: jobs-allowed
subjects:
- kind: ServiceAccount
  name: default
  namespace: jobs-allowed
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cron-writer
```

```shell
# deploy the config
kubectl apply -f cron-setup.yaml
```

The following configuration `yaml` creates a cronjob that echoes some text every minute

```
# cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: minute-ticker
  namespace: jobs-allowed
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: ticker
            image: busybox
            command: ["sh","-c","date; echo hello from cron"]
```

Now, we can create a cronjob as the namespace's default SA (since we gave it privs in `cron-setup.yaml`) to prove the RBAC works

```shell
$ kubectl --as=system:serviceaccount:jobs-allowed:default -f cronjob.yaml apply
cronjob.batch/minute-ticker created
```

This will now create a  task that runs persistently every minute. We can verify this by

```shell
$ kubectl -n jobs-allowed get cronjob
NAME            SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
minute-ticker   */1 * * * *   <none>     False     0        54s             73s

$ kubectl -n jobs-allowed get jobs,pods
NAME                               STATUS     COMPLETIONS   DURATION   AGE
job.batch/minute-ticker-29451320   Complete   1/1           14s        70s
job.batch/minute-ticker-29451321   Complete   1/1           8s         10s

NAME                               READY   STATUS      RESTARTS   AGE
pod/minute-ticker-29451320-k48sq   0/1     Completed   0          70s
pod/minute-ticker-29451321-6qw2h   0/1     Completed   0          10s

$ kubectl -n jobs-allowed logs -l job-name
Tue Dec 30 07:22:05 UTC 2025
hello from cron
Tue Dec 30 07:20:10 UTC 2025
hello from cron
Tue Dec 30 07:21:05 UTC 2025
hello from cron
```

*Cronjob runs every minute → proof of a simple persistence.*

In hardened clusters, this should only be allowed in specific namespaces and should trigger alerts in audit logs.

- cleanup
```shell
kubectl delete -f cronjob.yaml
kubectl delete -f cron-setup.yaml
```

## Closure

This brings us to the end of the offensive part of Kubernetes Security. In the next blog, we will look at cluster hardening techniques and recommendations.

Until next time `^_^`

---