---
title: "Kubernetes Security #4"
date: 2025-12-30
draft: false
summary: "Securing Kubernetes clusters."
tags: ["kubernetes testing"]
categories: ["blog"]
series: ["kubernetes"]
showToc: true
---

This blog is the fourth and final post in the Kubernetes Security series. I'll talk about cluster hardening in this post to conclude this series.
<!--more-->
**Throughout the blog/series, I use 3 tools:**
- [docker](https://www.docker.com/get-started/) - container runtime
- [minikube](https://minikube.sigs.k8s.io/docs/start/) - local single node cluster
- [kubectl](https://kubernetes.io/docs/reference/kubectl/) - tool to interact with cluster

## Is Your Cluster Hardened?

Cluster hardening is not just patching, it's systematically securing the control plane, worker nodes, workloads, and the runtime itself. While there are many things to consider when designing a cluster, it is common to miss out best practices for the sake of convenience.

Let's have a look at different ways we can secure each component present inside a Kubernetes Cluster.

## Securing the Control Plane

The control plane is the **command center** of the cluster. Locking it down should be our number 1 priority. It is essential to have the following:
- Restricting access to the API server
- Avoiding public exposure of the control plane
- Enabling TLS certificate rotation
- Disabling anonymous authentication on the Kubelet

Even if the control plane becomes exposed, these controls significantly reduce blast radius.

## Securing Pods

Most attacks happen at the **pod or workload level**, making this the **first line of defense**. Pod Security Best Practices
1. Containers should **not run as root** unless absolutely required
2. Filesystems should be **immutable** where possible
3. Images should be **regularly scanned** for vulnerabilities
4. Mutable image tags like `:latest` should be **denied**
5. **Privileged containers** should be prevented
6. **Pod Security Admission** should be enforced

Unless there’s a clear operational need (which is rare), containers should run as **non-root users**. Red flags in pod specs include:
- `runAsUser: <UID>` where UID is root
- `allowPrivilegeEscalation: true`

Few examples of prevention are given below:

1. **Deny Privilege Escalation**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  namespace: example-namespace
spec:
  securityContext:
    allowPrivilegeEscalation: false
```

2. **Enforce Non-Root User**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  namespace: example-namespace
spec:
  securityContext:
    runAsNonRoot: true
```

### Pod Security Standards

Kubernetes defines three Pod Security Standards that can be enforced at the namespace or cluster level:
- **Privileged** → Near-unrestricted access; allows known privilege escalations
- **Baseline** → Minimally restrictive; blocks known privilege escalations
- **Restricted** → Heavily restricted; aligns with current pod hardening best practices

**Pod Security Admission** enforces these standards by intercepting API server requests.

**Learn More**
- https://kubernetes.io/docs/concepts/security/pod-security-standards/
- https://kubernetes.io/docs/concepts/security/pod-security-admission/

## Securing Networks and Data Protection

Networking inside a cluster should follow a **zero-trust model**.
- Pods should not communicate unless explicitly allowed
- Default-deny NetworkPolicies act as a firewall baseline
- Use **`mTLS`** (via a service mesh) for encrypted service-to-service traffic
- Encrypt data at rest and rotate secrets regularly

Secrets store sensitive data such as credentials, tokens or SSH keys. By default, secrets are **base64-encoded** but not encrypted. Hence, encryption at rest is strongly recommended. We should also prevent passing secrets via environment variables or `ConfigMaps`. Instead, we can use secrets with CSI drivers (external managers) and workload identity to fetch at runtime.

### Core Network and Access Controls

- Restrict control plane access via firewall and RBAC
- Use TLS for control plane communication
- Create explicit deny policies
- Never store credentials in plain text
- Disable anonymous access
- Use strong authentication

The most important thing is to apply RBAC per team and service account RBAC (Role-Based Access Control) regulates access to cluster resources by assigning permissions to users, groups, or service accounts.

Permissions are defined using:
- **Resources** (pods, secrets, deployments, etc.)
- **Verbs** (`get`, `create`, `delete`, etc.)
RBAC ensures only those who need access have it.

### Networks Policies

Let us consider an example:

```
One of the heads of Kubernetes Laboratories wants to log the results on our latest test subject, so they launch the admin web portal (accessible on port 80) and access their record, sending an API request (on port 8080) to the database (on port 27017). Pods in a Kubernetes cluster are all deployed within the same virtual private network so that they can communicate with each other. However, in some cases, this is not necessary or wanted, and we should harden our cluster by removing these cases.
```

In the example above, our web app needs to communicate with with the API, however the web app does not need to communicate with the database. This communication channel should then be restricted. This is done using a **NetworkPolicy**.

A **NetworkPolicy** defines which pods can communicate with which services or endpoints.
- They are applied at the namespace level
- They control ingress and egress traffic
- They restrict pod-to-pod and cross-namespace communication

Namespaces can also be isolated by adding a default-deny ingress and egress policy clusterwide. Cross namespace allow rules should be explicit.

### Exposure Best Practices

- Prefer **ClusterIP + port-forward** for admin interfaces
- Use **Ingress or LoadBalancer** for user traffic
- If NodePort is unavoidable, keep the node IPs behind a firewall and restrict the source ranges.  

## Runtime Security

Within a Kubernetes cluster, there are many components. These components need to communicate; Kubernetes is entirely API driven. One way in which we can secure API traffic is through encryption.

![](https://cdn.ziomsec.com/k8-sec-p4/1.webp)

> Kubernetes expects all API communication in the cluster to be encrypted by default, with most installation methods supporting the creation and distribution of certificates to cluster components. Before implementing TLS encryption and distributing the certificates, we should establish if a component acts as a client or a server (or both) when communicating in the cluster.

To secure them, first, you would generate a CA (Certificate Authority) cert, which must be present on each component to validate the certificates received. Then, each of the components identified above, such as a client or a server, would need a certificate generated for them.

The certificate creation step would be done using a tool like OpenSSL to first generate the CA certificate, then generate a certificate for each of the components using the following steps:

1. **Generate a private key**

```shell
openssl genrsa --out ca.key 2048
```

2. **Generate a CSR (Certificate Signing Request)**

```shell
openssl req --new --key ca.key --subj "/CN=192.168.0.100" --out ca.csr
```

3. **Generate certificate (signing with CA cert and private key)**

```shell
openssl x509 --req --in ca.csr --signkey ca.key --out ca.crt --days 365
```

Once the certificates have been generated, the corresponding Kubernetes components need to be configured for use. This is done by adding these configurations to the component YAML file.

Implementing TLS encryption like this will implement CIS security benchmarks 1.2.24 - 27, hardening the cluster even further. Even though the certs are created and distributed, they have a validity date and must be renewed.

Additionally, we can monitor syscalls. Syscalls are how user-space processes request privileged operations from the kernel. Runtime security tools monitor these calls to detect abnormal behavior.

### Kubernetes Auditing

Kubernetes auditing records who did what and when. The stages of auditing are:
- **`RequestReceived`** → The audit handler (in the kube-api server) has received the request, but no response has been generated yet.
- **`ResponseStarted`** → This is when the response headers have been sent out, but the response body hasn't.
- **`ResponseCompleted`** → When the response body has been completed and sent out.
- **`Panic`** → This is an event generated when a panic occurs (critical error causing failure)

We use levels to tell Kubernetes how much event data we want to be captured in our audit log. There are four levels in total. As the level increases, more data is captured:
- **`None`** - This tells Kubernetes not to log this request 
- **`Metadata`** - This tells Kubernetes only to log the request metadata 
- **`Request`** - This tells Kubernetes to log the request metadata and the request body
- **`RequestResponse`** - This tells Kubernetes to log the request metadata, request body and response body.

### Security Runtime Enforcement Tools

Security runtime enforcement tools allow you to define policies that can minimize the impact of a threat when it appears in your runtime environment. It does this by restricting the access rights and permissions of resources within the environment. Unlike RBAC, runtime enforcement tools operate at the **kernel level**.

- **APPARMOUR** : It's a kernel module that protects both the operating system and applications from internal and external threats. Policies can be defined in AppArmour that enforce good behaviour, preventing known and unknown (e.g. zero-day attacks) application flaws from being exploited.
- **SECCOMP** : Seccomp (standing for secure computing mode) is an enforcement tool that operates at the kernel level. It works by filtering system calls, only allowing processed to perform certain calls (exit(), sigreturn(), read() and write()) to already open file descriptors (a process unique identifier for a file or other input/output resource).
- **SELINUX** : SELinux is very similar to AppArmour in that it is a Kernel Module which allows access controls to be implemented to protect a runtime environment.

### Falco

![](https://cdn.ziomsec.com/k8-sec-p4/2.webp)

**[Falco](https://falco.org/)** is a runtime threat detection engine. It analyses the behaviour of a system (in our case, imagine a Kubernetes environment), compares the behaviour observed against a list of predefined threat conditions, and, triggers an alert when a match is found.

**Falco** is built with **`eBPF`**. This is a Linux kernel technology that allows engineers to make programs which run securely in the kernel space. It allows programs to run in a protected environment within the kernel space, which ensures the code is safe to run before being executed. These programs can be attached to various hooks and events in the system and allow tools like Falco to have a deeper level of access to kernel operation, allowing for in-depth analysis of security events without increasing the risk of a system crash.

***`Falcosidekick`** is a companion project to Falco that can be enabled during / after configuration. `Falcosidekick` enables the forwarding of Falco events/alerts to 60+ services. This allows for integration to ChatOps tools like Slack and Microsoft Teams when events happen, as well as custom forwarding rules to allow for granular control over this so messages are only sent out for critical events and not spammed in various channels.*

![](https://cdn.ziomsec.com/k8-sec-p4/3.webp)

## Cluster-Level Best Practices

- Enable audit logging
- Implement log monitoring and alerting
- Apply security patches promptly
- Perform vulnerability scans and penetration tests
- Remove obsolete components

As we know, Kubernetes cluster contains nodes and runs a workload by placing containers into pods that run on these nodes. Hence, Kubernetes lives at the highest level of our Kubernetes architecture and comprises all the lower level components. With so many components, the attack surface also increases making it essential to have strong security practices in place.

The **Center for Internet Security (CIS)** provides industry-standard benchmarks for Kubernetes hardening.
- https://www.cisecurity.org/benchmark/kubernetes

Other baselines include **STIGs**:
- https://www.stigviewer.com/stigs 

**[Kube-bench](https://github.com/aquasecurity/kube-bench)** checks cluster configurations against CIS benchmarks and can be run:
- As a pod (with elevated permissions)
- As a container
- Via managed services (EKS, AKS, etc.)

### Kubelet Security

The Kubelet listens on:
- **10250** – authenticated, full access
- **10255** – unauthenticated, read-only (should be disabled)

*The Kubelet-api is used to manage containers running on a node and has full access.* We can find where the kubelet config file is located:

```shell
ps ef | grep kubelet
```
> then we can find the directory in the `--config` flag.

To implement CIS security benchmark 4.2.1 - locking down unauthorized traffic, we can disable anonymous traffic by setting `authentication:anonymous:enabled` to `false`. For example:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
    cacheTTL: 0s
  x509:
    clientCAFile: /var/lib/minikube/certs/ca.crt
```

Authentication Methods
- **X509 Client Certificates**
- **API Bearer Tokens (Webhook authentication)**

Once configured, the Kubelet will no longer accept unauthenticated requests.

## Closure

This concludes the fourth and final post of the [Kubernetes security series](https://ziomsec.com/series/kubernetes/). Each hardening control discussed here exists because there is a corresponding attack path. Understanding how clusters are broken in practice makes it easier to reason about which controls actually matter in your environment. 

Throughout the series, I covered the the security lifecycle across the following areas:
- what Kubernetes is and why it needs to be tested
- reconnaissance and initial access to a cluster
- privilege escalation and post-exploitation techniques
- cluster hardening

*This series doesn’t cover every possible attack or defensive mechanism, and it’s not meant to replace threat modeling or continuous review. Kubernetes changes quickly, and so do the techniques used to abuse it. What should remain constant is the mindset: assume compromise, understand how access can expand, and design clusters to limit blast radius.*

---