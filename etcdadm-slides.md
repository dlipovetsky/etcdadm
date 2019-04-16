<!-- $theme: default -->

> have speaker notes visible separate from slides
> more war stories
> more comparison with and without etcdadm
> record 45 minute youtube video
> emphasize what etcdadm makes easy, how hard it is without it

---

Kubernetes in Production: Operating etcd with etcdadm
===

[Daniel Lipovetsky](https://github.com/dlipovetsky)
Software Engineer, [Platform9 Systems](https://platform9.com/)
---

April 16, 2019

---

# etcdadm

- CLI to simplify etcd operation, including disaster recovery
- Inspired by lessons learned running Kubernetes in production
- An open-source, community project:
	- https://sigs.k8s.io/etcdadm
- Easy to install
 	- `go get sigs.k8s.io/etcdadm`
	- Binary releases coming soon

---

# Lessons Learned in Production
## Some definitions

- Kubernetes Control plane
	- Group of stateless components
		- apiserver
		- controller-manager
		- scheduler
	- One stateful component
		- etcd

---

# Lessons Learned in Production

1. API uptime is critical.
	- Without the API, the cluster is a zombie.
	- All CRD-based services need the API.
0. Many API outages are etcd outages.
	- Check component statuses, or apiserver log.
0. Permanent etcd cluster outages happen.
	- User deleted wrong set of instances.
0. Have a **manual** recovery process.
0. Periodic backups are important, but try to recover the latest state, if possible.

---

# How to ensure API uptime

- There are two strategies:
  - Tolerate partial failure.
  - Reduce recovery time.
- You need both.

---

# How to tolerate partial failure

- Deploy multiple control plane replicas.
  - Easier
  - No performance penalty
- Deploy multiple etcd members.
  - Harder
  - Performance penalty

---

# How to reduce recovery time

- Write a service to automate recovery.
	- More complex and less flexible
	- Depends on external APIs to 
	- Hard to debug and patch
	- Must itself tolerate failure.  
- Have a manual recovery process.
	- Can be made simple with tooling
	- Has no dependencies
	- Easy to debug and patch

> service implementation varies with environment
> tooling is generic across environments

---

# etcdadm
- Goals:
	- Make it easy to tolerate partial failure
	- Make it easy to have a manual recovery process
	- Work without dependencies on external services like DNS, or networked storage
	- Compose well with other tools
		- Use kubeadm to deploy control plane replicas
- Let's demo!
  - How to deploy a multi-member cluster
  - How to scale the cluster down
  - How to recover from a partial failure
  - How to recover from a disaster

---

# How to deploy a multi-member cluster

- Deploy all members atomically
	- Discovery service
	- DNS
	- Static
- Deploy one member, then scale up

---

# How to deploy a multi-member cluster

- etcdadm is designed to deploy one member, then scale up
	- One mechanism to understand
	- No dependencies on DNS or discovery service
	- Easily understood failure
	- Must deploy members sequentially

---

# How to deploy a multi-member cluster
## Create the first member
```shell
172.0.0.1> etcdadm init
```
Behind the scenes
1. Generate CA, server and client certificates
0. Write configuration
0. Create and start systemd service

> Certs are surprisingly hard to get right.
> Configuration includes I/O priority and memory protection.
---

# How to deploy a multi-member cluster
## Scale up

1. Copy CA cert/key
```shell
172.0.0.1> rsync /etc/etcd/pki/ca.* 172.0.0.2:/etc/etcd/pki
```
2. Join the cluster
```
172.0.0.2> etcdadm join https://172.0.0.2:2379
```

Behind the scenes:
1. Add member using etcd API
0. Discover all members using etcd API
0. Write configuration
0. Create and start systemd service

---

# How to scale down

1. Leave the cluster
```
172.0.0.2> etcdadm reset
```

Behind the scenes:
1. Discover identity of local member
0. Remove member using etcd API
0. Stop and remove systemd service
0. Remove configuration and data

---

# How to handle etcd failure

## Some definitions
Partial failure
- A quorum of members is available
- Examples: Planned maintenance, network partition, hard disk failure

Complete failure
- A quorum of members is not available
- Examples: Data center outage, networked storage failure

---

# How to prepare for a planned partial failure

First, consider how many failures your cluster can tolerate.
Then, choose how to prepare:
- Do nothing.
	- High risk.
- Migrate the member.
	- A special procedure.
- Replace the member.
	- Reuses the scaling procedure: Scale up, then down.
---
# How to prepare for a planned partial failure

Replace the member prior to maintenance; etcdadm makes this easy.

1. Copy CA cert/key
```shell
172.0.0.2> rsync /etc/etcd/pki/ca.* 172.0.0.3:/etc/etcd/pki
```

2. Leave the cluster
```shell
172.0.0.2> etcdadm reset
```

3. Join the cluster
```shell
172.0.0.3> etcdadm join https://172.0.0.1:2379
```

---

# How to recover from an unplanned partial failure

If the data is on disk and the member is reachable:

- Check for insufficient disk space `df -h /var/lib/etcd`

- Check for a changed IP. If the IP changed, update the member's peer and client URLs. Then start the etcd service.

- Something else? See [this great KubeCon talk on debugging etcd](https://kccna18.sched.com/event/GrYJ/debugging-etcd-joe-betz-jingyi-hu-google). 

---

# How to recover from an unplanned partial failure
If the data is not on disk, or the member is unreachable, remove the failed member from the list. Then scale up.
 
1. Identify the permanently failed member.
```shell
> etcdctl.sh member list
7675368186969f2a, started, isv-daniel-537-10-105-16-229platform9.sys, https://10.105.16.229:2380, https://10.105.16.229:2379
7a085789484825b5, started, isv-daniel-537-10-105-16-195platform9.sys, https://10.105.16.195:2380, https://10.105.16.195:2379
ffe8a15189b30b53, started, isv-daniel-537-10-105-17-41platform9.sys, https://10.105.17.41:2380, https://10.105.17.41:2379
```

2. Remove the member.
```shell
> etcdctl.sh member remove $MEMBER_ID
```

---


# How to recover from an unplanned complete failure

Fetch a backed-up snapshot, or take a snapshot of some available member

Create a new cluster from a snapshot.

```shell
etcdadm init --snapshot /tmp/etcd.snapshot
```

---

# Roadmap

- Implement automation that invokes the etcdadm CLI.
- Join multiple members in parallel.
- Improve upgrade support

- What feature would **you** like to see? File an issue in github.com/kubernetes-sigs/etcdadm/issues

- Find us in **#etcdadm** in [kubernetes slack](https://kubernetes.slack.com)

---

# Thank you!

Thanks to everyone at [Platform9 Systems](https://platform9.com) and the [Cluster Lifecycle Special Interest Group](https://github.com/kubernetes/community/tree/master/sig-cluster-lifecycle).

---

# Q&A