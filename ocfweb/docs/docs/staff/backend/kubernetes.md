[[!meta title="Kubernetes"]]

At the OCF we are in the process of moving production services from
[[Mesos/Marathon|doc staff/backend/mesos]] to [Kubernetes][kubernetes]. In this
document we will explain the design of our Kubernetes cluster while also
touching briefly on relevant core concepts. This page is _not_ a `HOWTO` for
deploying services or troubleshooting a bad cluster. Rather, it is meant to
explain architectural considerations such that current work can be built upon.
Although, reading this document will help you both deploy services in the OCF
Kubernetes cluster and debug issues when they arise.

## Kubernetes

Kubernetes is a container orchestration system open sourced by Google. Its main
purpose is to schedule services to run on a cluster of computers while
abstracting away the existence of the cluster from the services. The design of
Kubernetes is loosely based on Google's internal orchestration system Borg.
Kubernetes is now maintained by the [Cloud Native Computing Foundation][cncf],
which is a part of the Linux Foundation. Kubernetes can flexibly handle
replication, impose resource limits, and recover quickly from failures.

## Kubernetes Cluster

A Kubernetes cluster consists of "master" nodes and "worker" nodes. In short,
master nodes share state to manage the cluster and schedule jobs to run on
workers. It is considered best practice to run an odd number of masters, and
currently our cluster has three masters.

### Masters

Kubernetes masters share state via a distributed key-value store. The reason
for the key-value store is high availability: if one master goes down, a new
master node will subsume control of the cluster in a leader election to ensure
that no services incur downtime. [etcd][etcd-io] is an
implementation of the [Raft][raft] protocol chosen by the Kubernetes project
to act as the key-value store. It's three main goals are:

1. Leader elections in case of failure.
2. Log replication across all masters.
3. Ensuring log integrity across all masters.

### Why the odd number of masters?

Consider a cluster of *N* members. When masters form quorum to agree on cluster
state, quorum must have _at least_ ⌊*N*/2⌋+1 members. Every new odd number in a
cluster with *M* > 1 masters adds one more node of fault tolerance.  Therefore,
adding an extra node to an odd numbered cluster gives us nothing. If interested
read more [here][failure-tolerance].

### Workers

Workers are the brawn in the Kubernetes cluster. While master nodes are
constantly sharing data, managing the control plane, and scheduling services,
workers primarily run [pods][pod]. `kublet` is the service that executes pods
as dictated by the control plane, performs health checks, and recovers from pod
failures should they occur. Workers also run an instance of `kube-proxy`, which
forwards control plane traffic to the correct `kublet`.

### Pods

In the Kubernetes world, pods are the smallest computing unit. A pod is made up
of one or more containers. The difference between a pod and a standalone
container is best illustrated by an example. Consider [ocfweb][ocfweb]; it is
comprised of several containers—the web container, static container, and worker
container.  In Kubernetes, together these three containers form one pod, and it
is pods that can be scaled up or down. A failure in any of these containers
indicates a failure in the entire pod. It should be noted that when writing
Kubernetes services we don't actually deal in pods but one further abstraction,
deployments, which create pods for us.

## OCF Kubernetes Cluster Bootstrapping

Since almost all OCF architecture is bootstapped using Puppet, it was necessary
for us to do the same with Kubernetes. We rely on the
[puppetlabs-kubernetes][kubernetes-module] module to handle initial
bootstrapping and bolt OCF specific configurations on top of it.
`puppetlabs-kubernetes` performs two crucial tasks:

1. Installs `etcd`, `kubelet`, `kube-proxy`, and `kube-dns`, initializes the
   cluster, and applies a networking backend.
2. Generates the required [PKI for Kubernetes and etcd][kubernetes-pki].

Do note that `puppetlabs-kubernetes` is still very much a work in progress. If
you notice an issue in the module you are encouraged to write a patch and send
it upstream.

### Kubernetes PKI

All the private keys and certs for the PKI are in the puppet private share, in
`/opt/puppet/shares/private/kubernetes`. We won't go into detail of everything
contained there, but much of Kubernetes and `etcd` communication is
authenticated using client certificates. All the necessary items for workers
are included in `os/Debian.yaml`, although adding a new master to the cluster
requires re-running [kubetool][puppetlabs-kubetool] to generate new `etcd
server` and `etcd peer` certs.

### OCF Kubernetes Configuration

Currently, the OCF has three Kubernetes masters: (1) `deadlock`, (2) `coup`,
and (3) `autocrat`. A Container Networking Interface (`cni`) is the last piece
required for a working cluster. The `cni`'s purpose is to faciltate intra-pod
communication. `puppetlabs-kubernetes` supports two choices: `weave` and
`flannel`. Both solutions work out-the-box, and we've had success with
`flannel` thus far so we've stuck with it.

### Getting traffic into the cluster

One of the challenges with running Kubernetes on bare-metal is getting traffic
into the cluster. If you are running on `AWS`, `GCP`, or `Azure` you add `type:
LoadBalancer` to your Service definitions and things just work. For bare-metal,
this is an exercise left to the reader.

The figure below demonstrates a request made for `templates.ocf.berkeley.edu`.

```
                                            -------------------------------------------------
                                            |              Kubernetes Cluster               |
                           HAProxy          |                                               |
                          ----------        |                    Ingress         Pod        |
                          |autocrat|        |                   ----------    ----------    |
                          ----------        |                   |Worker1|     | ocfweb |    |
                                            |                   ----------    ----------    |
                                            |                                               |
            *VIP*          HAProxy          |                    Ingress         Pod        |
        ------------      ----------  ✘ SSL / Host: templates   ---------     -----------   |
REQ --> |keepalived| -->  |deadlock|    -->   -------------->   |Worker1| --> |Templates|   |
        ------------      ----------        \                   ---------     -----------   |
                                            |                                               |
                           HAProxy          |                    Ingress         Pod        |
                          ----------        |                   ---------     -----------   |
                          |  coup  |        |                   |Worker3|     | grafana |   |
                          ----------        |                   ---------     -----------   |
                                            -------------------------------------------------
```

All three Kubernetes masters are running an instance of [HAProxy][haproxy].
Furthermore, the masters are all running `keepalived`. The traffic for any
Kubernetes HTTP service will go through the current `keepalived` master, which
holds the virtual IP for all Kubernetes services. The `keepalived` master is
randomly chosen but will move hosts in the case of failure.  `HAProxy` will
terminate ssl and pass the request on to a worker running [Ingress
Nginx][ingress-nginx].  Right now ingress is running as a [NodePort][nodeport]
service on all workers (Note: we can easily change this to be a subset of
workers if our cluster scales such that this is no longer feasible).  The
ingress worker will inspect the `Host` header and forward the request on to the
appropriate pod where the request is finally processed.


### Why didn't we use MetalLB?

`MetalLB` was created so a bare-metal Kubernetes cluster could use `Type:
LoadBalancer` in Service definitions. The problem is, in `L2` mode, it takes a
pool of IPs and puts your service on a random IP in that pool. How one makes
DNS work in this configuration is completely unspecified. We would need to
dynamically update our DNS, which to me sounds like a myraid of outages waiting
to happen. `L3` mode would require the OCF dedicating a router to Kubernetes.


### Why don't we copy Marathan and specify one port per service?

In our current Marathon configuration, we give each service a port on the load
balancer and traffic coming into that port is routed accordingly. First, in
Kubernetes we would emulate this behavior using `NodePort` services, and all
Kubernetes documentation discourages this. Second, it's ugly. Every time we add
a new service we need to modify the load balancer configuration in Puppet. With
our Kubernetes configuration we can add unlimited HTTP services without
touching Puppet.

### Why are we using NodePort services?

The Kubernetes documentation says not to use `NodePort` services in production,
and you just said that above too! True, but we only run _one_ `NodePort`
service, `ingress`, to keep us from needing other `NodePort` services.
SoundCloud also has an interesting blog post about [running NodePort in
production][soundcloud-nodeport].

[kubernetes]: https://kubernetes.io/
[cncf]: https://cncf.io
[etcd-io]: https://github.com/etcd-io/etcd
[raft]: https://raft.github.io/raft.pdf
[failure-tolerance]: https://coreos.com/etcd/docs/latest/faq.html#what-is-failure-tolerance
[pod]: https://kubernetes.io/docs/concepts/workloads/pods/pod/
[ocfweb]: https://github.com/ocf/ocfweb/tree/master/services
[kubernetes-module]: https://github.com/puppetlabs/puppetlabs-kubernetes
[kubernetes-pki]: https://kubernetes.io/docs/setup/certificates
[puppetlabs-kubetool]: https://github.com/puppetlabs/puppetlabs-kubernetes#Setup
[haproxy]: https://en.wikipedia.org/wiki/HAProxy
[ingress-nginx]: https://github.com/kubernetes/ingress-nginx
[nodeport]: https://kubernetes.io/docs/concepts/services-networking/service/#nodeport
[soundcloud-nodeport]: https://developers.soundcloud.com/blog/how-soundcloud-uses-haproxy-with-kubernetes-for-user-facing-traffic
