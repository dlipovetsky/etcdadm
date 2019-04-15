<!-- $theme: default -->

> more war stories
> more comparison with and without etcdadm
> record 45 minute youtube video
> emphasize what etcdadm makes easy, how hard it is without it


Kubernetes in Production: Operating etcd with etcdadm
===

Daniel Lipovetsky
Software Engineer, Platform9
---

April 16, 2019

---

# etcdadm

- Inspired by lessons learned running Kubernetes in production.
- CLI to simplify etcd operation, including disaster recovery.
- An open-source, community project: - 
	- https://sigs.k8s.io/etcdadm
- Easy to install
 	- `go get sigs.k8s.io/etcdadm`
	- Binary releases coming soon!

---

# Some definitions

- Kubernetes Control plane
	- Group of stateless components
		- apiserver
		- controller-manager
		- scheduler
	- One stateful component
		- etcd

---

# Lessons Learned in Production

1. Control plane uptime is critical.
	- Without it, the cluster is a zombie.
	- Pods continue to run.
	- Service
	- Can't update Service endpoints
	- 

> go into more detail
> 
2. Permanent etcd cluster failures happen.
3. Have a manual recovery process and tooling to simplify it.
  - If you have to choose between "easy to understand" and "easy to use," choose "easy to understand."
  - Periodic backups are important, but try to recover the latest state, if possible.

> War story of etcd disaster recovery

---

# Control plane uptime

- Two strategies:
  - Tolerate failure
  - Reduce Mean Time To Recovery (MTTR)

---

# Tolerate failure

- Have multiple control plane replicas
  - Easy
  - No performance penalty
- Have multiple etcd members
  - Hard
  - Performance penalty

---

# Reduce MTTR

- Have a recovery process
- Make it easy to follow

---

# Operating etcd with etcdadm
- Bootstrap
- Scale and Heal
- Recover from disaster

---

# Bootstrap

- Multiple methods
  - Discovery service
  - DNS
  - Static
- We recommend bootstrap by scaling
  -  One mechanism to understand
  -  No external dependencies
  -  etcdadm makes this simple

---

# Bootstrap

- First member:
```shell
172.0.0.1> etcdadm init
172.0.0.1> rsync /etc/etcd/pki/ca.* 172.0.0.2:/etc/etcd/pki
172.0.0.1> rsync /etc/etcd/pki/ca.* 172.0.0.3:/etc/etcd/pki
```
- Subsequent members:
```shell
172.0.0.2> etcdadm join https://172.0.0.1:2379
```
```shell
172.0.0.3> etcdadm join https://172.0.0.1:2379
```

---

# Scale and Heal
> seq diagras from isv to show complexity

> brag about etcdadm reset
> 
---

# Recover from Disaster

---

# Roadmap

- Automated operation



---

