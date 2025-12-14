---
title: "Kubernetes Security #1"
date: 2025-12-14
draft: false
summary: "Understanding the fundamentals of Kubernetes & why testing it is important."
tags: ["kubernetes testing"]
categories: ["blog"]
series: ["kubernetes"]
showToc: true
---

This blog is the first of the Kubernetes Security series where I'll talk about what Kubernetes is and why is it important for us to test it.
<!--more-->
**Throughout the blog/series, I use 3 tools:**
- [docker](https://www.docker.com/get-started/) - container runtime
- [minikube](https://minikube.sigs.k8s.io/docs/start/) - local single node cluster
- [kubectl](https://kubernetes.io/docs/reference/kubectl/) - tool to interact with cluster

## Introduction To Kubernetes

So, before diving into the technicality of security testing, it is important for us to understand what **Kubernetes** is in the first place.....

"*Kubernetes is an open source container orchestration framework that was originally developed by Google.*"

The easiest way to think about it is like an operating system for containerized applications. It decides where things run, keeps them healthy and exposes them to the network automatically.

![source: https://www.atlassian.com/microservices/microservices-architecture/kubernetes-vs-docker](https://cdn.ziomsec.com/k8-sec-p1/1.webp)

This might bring a question in our mind, why is it required? Isn't docker enough? Doesn't it allow us to pack applications along with their dependencies and ensure better management and scalability?

Well, that is partly true....As we shifted from [monolithic](https://www.geeksforgeeks.org/system-design/monolithic-architecture-system-design/) to [microservice](https://www.geeksforgeeks.org/system-design/microservices/) architecture, we started relying more and more on container technologies. This meant that now an application comprised of multiple containers - tens and hundreds if not thousands which made its management quite difficult and tedious. This led to the creation of container orchestration technology.

Tools like Kubernetes offer the following features:
- High availability and almost no downtime
- Scalability or high performance
- Disaster recovery or backup and restore

### Practical 1 - Experiencing The Power Of K8s

**Scenario**: *Spin up 5 docker containers running `alpine 3.20`.*

- Traditional way of doing this using `docker` would be:

```shell
docker run -d --name alp1 alpine:3.20 sleep infinity
docker run -d --name alp2 alpine:3.20 sleep infinity
docker run -d --name alp3 alpine:3.20 sleep infinity
docker run -d --name alp4 alpine:3.20 sleep infinity
docker run -d --name alp5 alpine:3.20 sleep infinity
```

- To add more containers, I would have to repeat the above commands...

```'shell
docker run -d --name alp6 alpine:3.20 sleep infinity
docker run -d --name alp7 alpine:3.20 sleep infinity
```

- If wanted to delete a particular container, or a set of containers,

```shell
docker rm -f alp1 alp2 alp3
```

- With the power of **Kubernetes**, these operations can be done with ease..... Let's use a simple `yaml` file to define our requirements.
> We can interact with the cluster in 2 ways: by providing a `yaml` file with configurations or by command line.

```yaml
# alpine-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alpine-demo
spec:
  replicas: 5
  selector:
    matchLabels:
      app: alpine-demo
  template:
    metadata:
      labels:
        app: alpine-demo
    spec:
      containers:
        - name: alpine
          image: alpine:3.20
          command: ["sh","-c","tail -f /dev/null"]
```

Using the above `yaml` file, I can deploy 5 containers running the `alpine` image:

```shell
$ kubectl apply -f alpine-deploy.yaml 
deployment.apps/alpine-demo created
```

- I can list the containers by

```shell
$ kubectl get pods -l app=alpine-demo
NAME                           READY   STATUS    RESTARTS   AGE
alpine-demo-65f4f95f8f-8fwlv   1/1     Running   0          28s
alpine-demo-65f4f95f8f-lrwxv   1/1     Running   0          28s
alpine-demo-65f4f95f8f-mgqnw   1/1     Running   0          28s
alpine-demo-65f4f95f8f-rzr4f   1/1     Running   0          28s
alpine-demo-65f4f95f8f-zjz7s   1/1     Running   0          28s
```

- Interacting with these containers is also pretty straightforward...

```shell
$ kubectl exec -it alpine-demo-65f4f95f8f-8fwlv -- sh -c 'echo "$(whoami) from $(hostname)"'
root from alpine-demo-65f4f95f8f-8fwlv
```

- let's say I wanted to add 2 more containers to this group/deployment

```shell
$ kubectl scale deployment/alpine-demo --replicas=7
deployment.apps/alpine-demo scaled

$ kubectl get pods -l app=alpine-demo         
NAME                           READY   STATUS    RESTARTS   AGE
alpine-demo-65f4f95f8f-8fwlv   1/1     Running   0          4m53s
alpine-demo-65f4f95f8f-lrwxv   1/1     Running   0          4m53s
alpine-demo-65f4f95f8f-ltjv2   1/1     Running   0          9s
alpine-demo-65f4f95f8f-mgqnw   1/1     Running   0          4m53s
alpine-demo-65f4f95f8f-rzr4f   1/1     Running   0          4m53s
alpine-demo-65f4f95f8f-z6rdh   1/1     Running   0          9s
alpine-demo-65f4f95f8f-zjz7s   1/1     Running   0          4m53s
```

- Even deleting them is super simple

```shell
$ kubectl delete deployment/alpine-demo
deployment.apps "alpine-demo" deleted from default namespace

$ kubectl get pods -l app=alpine-demo
No resources found in default namespace.
```

> *The goal of this practical demonstration was to understand how Kubernetes simplifies container management. We will explore various components involved in this demonstration as we proceed further.*

### Key Components Of Kubernetes

There are various components that make up a Kubernetes cluster. These are objects that are present when we are working with Kubernetes. Do note that these aren't what make up the cluster but are the ones that are created to keep our application running. How a cluster is structured will be covered when we talk about the [Architecture](#Architecture).

1. **POD**
	- This is the smallest building block in Kubernetes. You can think of it as a wrapper around a container (like a Docker container). It creates a running environment above the container.
	- With Kubernetes, we dont directly deal with containers, we deal with pods.
	- Usually, a pod runs just one main application container, though it can have extra side containers for helper tasks.
	- Each pod lives inside a **node** and gets its own virtual network with a unique IP address.
	- Pods can talk to each other using these internal IPs.
	- But pods can **die** easily-for example, if an app crashes or the server is restarted. When that happens, Kubernetes will spin up a new pod to replace it. But the new one will get a **new IP address**.

2. **NODE**
	- This is basically a machine - can be a physical server or a virtual one. Pods run inside nodes.

3. **SERVICE**
	- A service gives a **fixed IP** or name to a pod. Since pods can come and go, services help us keep a constant way to talk to them. An internal service is only available within the cluster. An external service allows traffic from outside.
	- Even though services are stable, when we access our app, we usually hit the **node IP**, not the service directly (unless you use something like ingress).
	- This also works as a load balancer.

> **A Service is a way to expose those pods** to other pods, external clients, or both. It provides a **stable endpoint** (IP address, DNS) to communicate with a dynamic set of pods. The service ensures that even if pods are added or removed, the endpoint to access them stays the same.

4. **INGRESS**
	- Think of this as the traffic manager. It handles things like domain routing and port forwarding. It sends incoming requests (from a browser or external source) to the right service/pod inside your cluster.

5. **CONFIGMAP**
	- These are external configuration files for our pods. They store values like service URLs or other settings. Each pod reads from this config.

6. **SECRET**
	- This is similar to a ConfigMap but used for sensitive stuff like usernames, passwords, API keys, etc.
	- The data is stored in base64-encoded format. Not encrypted by default, but it's better than plain text.

> Note: Kubernetes’ built-in security features are not enabled automatically. You have to set them up.

7. **VOLUMES**
	- Volumes are used to store data. Unlike pods, which can disappear and be replaced, volumes stick around.
	- This means even if a pod crashes or is restarted, the data stays safe.
	- Storage can be local (on the machine) or remote (like cloud storage).
	- Think of it like an external hard drive plugged into our system.

8. **DEPLOYMENT**
	- A deployment is like a **blueprint** for pods. It defines how many copies (replicas) of the pod you want running at the same time. These pods all connect to the same service.
	- Services act like a smart gateway; they route traffic to whichever pod is less busy.
	- This setup ensures **high availability**. If one pod dies, another one takes over automatically.

> **A Deployment is a set of pods** running the same containerized application. So, if you have an Ubuntu-based pod, that would be part of a deployment.

9. **STATEFULSET**
	- For databases, we use something called a `StatefulSet`. This helps manage multiple DB pods that need to stay in sync.
	- Reads and writes are coordinated to avoid conflicts or data loss. It’s made for systems that require data consistency, like databases.
	- `StatefulSets` enable stateful applications to run on Kubernetes, but unlike pods in a deployment, they cannot be created in any order and will have a unique ID (which is persistent, meaning if a pod fails, it will be brought back up and keep this ID) associated with each pod.
	- `StatefulSets` will have one pod that can read/write to the database referred to as the master pod. The other pods, referred to as slave pods, can only read and have their own replication of the storage, which is continuously synchronized to ensure any changes made by the master node are reflected.

10. **REPLICASET**
	- maintains a set of replica pods and can guarantee the availability of x number of identical pods (identical pods are helpful when a workload needs to be distributed between multiple pods). `ReplicaSets` usually aren't defined directly (neither are pods, for that matter) but are instead managed by a deployment

11. **JOB**
	- A Kubernetes Job is a controller that creates one or more Pods and ensures that a specified number of them successfully complete their tasks. Unlike deployments that run continuous, long-running services, Jobs are designed for finite tasks that run to completion and then stop.

### Architecture

![source: https://kubernetes.io/docs/concepts/architecture/](https://cdn.ziomsec.com/k8-sec-p1/2.webp)

Now that we have an idea of what Kubernetes is and why is it required, let's understand how it operates. As shown above, a Kubernetes cluster can be divided into 2 main components:
1. **Control Plane** : this is the brain of the cluster. It exposes the API server, runs tasks and stores information related to the cluster.
2. **Worker Nodes** : These are muscles. They run our containers and handle networking.

The **Control Plane** consists of 4 components:
- **API Server** → Acts as a gateway to the cluster. Handles all initial requests (like deploying apps, updates, or queries). It is the **only entry point** into the cluster.
- **Scheduler** → Decides **which worker node** should run an app. Chooses based on available resources and app requirements. It actively monitors the cluster.
- **Controller Manager** → Monitors the cluster state. Detects when a pod fails or when there's a change in state. Works with the scheduler to restore desired state (by telling `kubelet` to restart pods, etc).
- **Etcd** → A **key-value store** of the cluster state. Stores **only cluster-related data** (not application data). All decisions by scheduler, controller manager, etc., are based on data from etcd. 

**Worker Nodes** are made up of 3 components:
- **Kubelet** → Runs the containers (e.g., Docker). Must be installed to run containers inside pods.
- **Kube-Proxy** → Responsible for starting pods and the containers inside them. Reads the configuration and ensures the containers run properly. Talks to both the node and the container runtime.
- **Container Runtime** → Manages **network communication** between pods and services. Works like a load balancer, routes traffic efficiently. Sends requests to the **closest and most available** pod.

## Why Test Kubernetes?

Now that we have a basic understanding about Kubernetes, let's move onto why we need to test it. 

The fact is, Kubernetes does not secure itself. Kubernetes security is a **shared** responsibility.
- Cloud providers protect the infrastructure and control plane
- Operators configure RBAC and policies
- Developers secure their apps and images

*Did you know, most breaches don't come from Kubernetes bugs - they come from misconfigurations and overly permissive settings.*

That is why testing Kubernetes becomes extremely important so that we can catch those misconfigs before the bad guys do. When you look at a Kubernetes cluster, it isn't 1 system; it's a mesh of API's and agents. This means that the Attack Surface for a cluster is big.
- if our dashboard is open to public, or has weak authentication mechanisms then that is a problem.
- It the images running on our pod are outdated and vulnerable, that would lead to elevated access.
- If there is lack of policies implemented on the network, an attacker can move laterally like it's one big flag LAN.
- Since new pods, namespaces and images are spun up regularly, there is always a chance of a vulnerability to sneak into our cluster putting it in danger.

This is where Kubernetes testing comes in. Tools like **[Kube-bench](https://github.com/aquasecurity/kube-bench)**, **[Trivy](https://github.com/aquasecurity/trivy)** and **[Falco](https://github.com/falcosecurity/falco)** help us automate validation and detect issues before they turn into breaches.

### Practical 2 - Benchmarking Our Cluster With Kube-bench

**Scenario**: *Perform a CIS kubernetes benchmarking assessment against Minikube's Kubernetes cluster.*

- For the sake of simplicity, we can just use the Kubebench tool against the standard cluster created using **`minikube`**

```shell
$ minikube start
```

- Download the latest release from **Kube-bench**'s [Github](https://github.com/aquasecurity/kube-bench/releases) repository.

```shell
# installing the binary
$ sudo apt install .\<NAME>.deb
```

- Finally we can run the tool

```shell
$ kube-bench run
[INFO] 1 Master Node Security Configuration
[INFO] 1.1 Master Node Configuration Files
[FAIL] 1.1.1 Ensure that the API server pod specification file permissions are set to 644 or more restrictive (Automated)
[FAIL] 1.1.2 Ensure that the API server pod specification file ownership is set to root:root (Automated)
[FAIL] 1.1.3 Ensure that the controller manager pod specification file permissions are set to 644 or more restrictive (Automated)
[FAIL] 1.1.4 Ensure that the controller manager pod specification file ownership is set to root:root (Automated)
[FAIL] 1.1.5 Ensure that the scheduler pod specification file permissions are set to 644 or more restrictive (Automated)
[FAIL] 1.1.6 Ensure that the scheduler pod specification file ownership is set to root:root (Automated)
[FAIL] 1.1.7 Ensure that the etcd pod specification file permissions are set to 644 or more restrictive (Automated)
[FAIL] 1.1.8 Ensure that the etcd pod specification file ownership is set to root:root (Automated)
[WARN] 1.1.9 Ensure that the Container Network Interface file permissions are set to 644 or more restrictive (Manual)
[WARN] 1.1.10 Ensure that the Container Network Interface file ownership is set to root:root (Manual)
[FAIL] 1.1.11 Ensure that the etcd data directory permissions are set to 700 or more restrictive (Automated)
[FAIL] 1.1.12 Ensure that the etcd data directory ownership is set to etcd:etcd (Automated)
[FAIL] 1.1.13 Ensure that the admin.conf file permissions are set to 644 or more restrictive (Automated)
[FAIL] 1.1.14 Ensure that the admin.conf file ownership is set to root:root (Automated)
[FAIL] 1.1.15 Ensure that the scheduler.conf file permissions are set to 644 or more restrictive (Automated)
[FAIL] 1.1.16 Ensure that the scheduler.conf file ownership is set to root:root (Automated)
[FAIL] 1.1.17 Ensure that the controller-manager.conf file permissions are set to 644 or more restrictive (Automated)
[FAIL] 1.1.18 Ensure that the controller-manager.conf file ownership is set to root:root (Automated)
[FAIL] 1.1.19 Ensure that the Kubernetes PKI directory and file ownership is set to root:root (Automated)
[WARN] 1.1.20 Ensure that the Kubernetes PKI certificate file permissions are set to 644 or more restrictive (Manual)
[WARN] 1.1.21 Ensure that the Kubernetes PKI key file permissions are set to 600 (Manual)
[INFO] 1.2 API Server
[WARN] 1.2.1 Ensure that the --anonymous-auth argument is set to false (Manual)
[PASS] 1.2.2 Ensure that the --basic-auth-file argument is not set (Automated)
[PASS] 1.2.3 Ensure that the --token-auth-file parameter is not set (Automated)
[PASS] 1.2.4 Ensure that the --kubelet-https argument is set to true (Automated)
[PASS] 1.2.5 Ensure that the --kubelet-client-certificate and --kubelet-client-key arguments are set as appropriate (Automated)
....
....
....

== Summary policies ==
0 checks PASS                                                                                                                                                
0 checks FAIL
24 checks WARN
0 checks INFO

== Summary total ==
40 checks PASS                                                                                                                                               
35 checks FAIL
47 checks WARN
0 checks INFO
```

## Closure

With this, we have a solid understanding of what Kubernetes is, why is it required, and why testing it is important. In the next blog, I'll talk about how we can perform reconnaissance and gain initial foothold in the cluster.

Until next time :)

---
