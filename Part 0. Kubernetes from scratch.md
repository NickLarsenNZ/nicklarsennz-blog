# Kubernetes from scratch: Kubernetes Architecture

September 14, 2019

<details>
<summary>Todo</summary>
<p>

- Purpose
- Version (1.14?)
- Kubernetes architecture
- Garden
- BYO Workers (custom container runtime)
- Worker labeling

</p>
</details>  

To help improve my teams efficiency, I'm exploring ways to launch on-prem Kubernetes clusters with minimal effort. While we do run some workloads in the cloud, the nature of the business and some history means we also run a lot on-prem. While there are on-prem options for running Kubernetes (both paid and free), I felt that with a little more effort, we might be able simply instantiate small ephemeral production-grade clusters with low operational overhead.

## The series

In this series, I will build a deployable configuration of all necessary components to stand up a Kubernetes cluster.

While there are many articles on how to deploy a Kubernetes cluster, I have yet to come across one that builds the minimum necessary, leverages containerisation for the control plane, and makes use of common deployment tooling such as [docker-compose] and [Helm].

To see the configurations in full context, jump over to the source at [NickLarsenNZ/kubernetes-byo-worker][source].

This series includes:

- Part 0: Kubernetes Architecture
- [Part 1: Kubernetes API Server up and running][part-1]

For the past year or so, I have mostly worked with managed Kubernetes offerings in the cloud (EKS, AKS, GKE). It has been some time since I provisioned a home-grown Kubernetes cluster, so I spent some time gaining a deeper understanding of the components that make up a Kubernetes cluster. Through this series you will gain a deeper understanding of how the Kubernetes components hang together, and be able to simply launch Kubernetes clusters using common deployment tools.

I will take the following approach:
- Rapidly stand up each component using [docker-compose] and determining the set of variables to cover a wide variety of operating environment.
- Develop a Helm Chart based on the knowledge gained from the rapid iterations using [docker-compose].
- Use an existing Kubernetes cluster as a launchpad for deploying more clusters.
- Attach worker nodes separately. I call this the _BYO Worker_ model.
- Successfully run a test deployment on the cluster

## Kubernetes Architecture

The _thousand-foot_ architectural view of Kubernetes is fairly simple. There are a number of components which tend to be divided into two groups:

- Master Services
- Worker Services

![Kubernetes Architecture][kubernetes-arch]
_The diagram above shows how the components communicate with each other._

The [`kube-apiserver`][kube-apiserver] is the main connection point, and for every worker there are two connections from [`kubelet`][kubelet] and [`kube-proxy`][kube-proxy]. In addition, the [`kube-apiserver`][kube-apiserver] must be able to communicate with each [`kubelet`][kubelet] (for fetching logs, attaching and executing commands, and port-forwarding into Pods). [See here for more detail on the necessary connectivity][kubernetes-comms].

### Master Services

- **`etcd`**: The key-value store for all the state behind Kubernetes. Accessed by the [`kube-apiserver`][kube-apiserver].
- [**`kube-apiserver`**][kube-apiserver]: The face of Kubernetes, all interactions from workers ([`kubelet`][kubelet]), admin users, external tools, etc... come through here.
- [**`kube-scheduler`**][kube-scheduler]: The scheduler process for assigning pods to workers.
- [**`kube-controller-manager`**][kube-controller-manager]: The endless loop watching the current state and updating the [`kube-apiserver`][kube-apiserver] with commands to reach the desired state.

### Worker Services

- [**`kubelet`**][kubelet]: The caretaker of the worker node, it talks to [`kube-apiserver`][kube-apiserver] and takes PodSpecs to apply. Kubelet talks to the local c  CRI (Container Runtime Interface)
- [**`kube-proxy`**][kube-proxy]: The plumber for the worker node, it talks to [`kube-apiserver`][kube-apiserver] and manages iptables rules for access to Pods and Services (redirecting to other worker nodes if necessary).
- **CNI**: Container Networking Interface. For example WeaveNet, Callico, Flannel, etc...
- **CRI**: Container Runtime Interface. For example docker, CRI-O, gvisor, etc...

One thing I have noticed is that many of the diagrams show bi-directional traffic flows between [`kubelet`][kubelet] and [`kube-apiserver`][kube-apiserver] and I'm not 100% certain that is correct.

_**Note:** I have left a lot to be discussed, but I felt like it would detract from the goal of this series. For example, I have not covered additional services such as the Cloud Controller Manager, nor internal DNS for Services._

## BYO Worker

So, the whole idea behind this was to have an easy way to launch master services, then bring your own workers. 

Here's the wish list:

- Single deployment for master services on an existing Kubernetes cluster
- Ability to upgrade the master services, and worker nodes
- Immutable workers with no external access (no SSH, no extra user accounts)
- Bring my own workers, with a container runtime of my choice

## Summary

Based on the component separation of Kubernetes, it should be relatively straightforward (given enough reading) to instantiate the Master Services, and join nodes to them.

Through the rest of the series, I will work toward building that out.

If you're interested in going deeper, read [Part 1: Kubernetes API Server up and running][part-1]

[part-1]: https://github.com/NickLarsenNZ/nicklarsennz-blog/blob/master/Part%201.%20Kubernetes%20API.md
[docker-compose]: https://docs.docker.com/compose/
[Helm]: https://helm.sh/
[source]: https://github.com/NickLarsenNZ/kubernetes-byo-worker
[kelsey-hightower]: https://twitter.com/kelseyhightower
[k8s-hard-mode]: https://github.com/kelseyhightower/kubernetes-the-hard-way
[kubernetes-arch]: ./images/kubernetes_architecture.png
[kubernetes-comms]: https://kubernetes.io/docs/concepts/architecture/master-node-communication/
[kube-apiserver]: https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/
[kube-scheduler]: https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/
[kube-controller-manager]: https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/
[kubelet]: https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/
[kube-proxy]: https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/
