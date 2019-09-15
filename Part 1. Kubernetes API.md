# Kubernetes from Scratch, Part 1: Kubernetes API

September 15, 2019

In this installment of the series, I will be running the [`kube-apiserver`][kube-apiserver], then any services it depends on to become operational (even if not useful without the rest of the cluster components).

## The series

In this series, I hope to work through running each of the necessary components of Kubernetes, and devloping a deployment model.

This series includes:

- [Part 0: Kubernetes Architecture][part-0]
- Part 1: Kubernetes API Server up and running

## Approach

I will run each component in Docker so as to not pollute my current machine. To keep things simple, repeatable, and most importantly adjustable with minimal effort I will use docker-compose to capture specifics. 

I will then run the component and watch the logs. As I see errors, I will adjust the deployment until satisfied. 

Once I have a good understanding of how it needs to run, I will then develop the Helm templates so that the comonent(s) can be deployed to Kubernetes with a single place to specify override values.

I should also mention that I'm very time poor at the moment, and so I'm setting _future me_ up for success. This should make up for all the terrible things _past me_ has left _current me_ (me).

## Step 0: Find the kube-apiserver image

I'm going to cheat a bit here. I have Docker for Desktop on my Mac, with Kubernetes enabled. So Instead of lookup on Docker Hub, or GCR.io, I will just see what's being used. If it looks appropriate, I'll probably just use it. If it looks too specific to Kubernetes on Docker for Desktop, then I'll start the search from scratch, or see how kubeadm does it.

```sh
kubectl -n kube-system get pod -l component=kube-apiserver -o yaml | yq  -y '.items[] | .spec.containers[] | .image'
k8s.gcr.io/kube-apiserver-amd64:v1.10.11
```

This looks fairly generic and not tied to Docker for Desktop, although I would like to use a more recent release. I found v1.15.3 here: <https://console.cloud.google.com/gcr/images/google-containers/GLOBAL/etcd-amd64>.

I'm going to run it on it's own with no special flags or dependencies, just to see what logs come out:

```bash
docker run --rm k8s.gcr.io/kube-apiserver-amd64:v1.15.3
```

Nothing happened. I guess the Dockerfile has no entrypoint nor command specified and we have to run the `kube-apiserver` as I have seen in other installations.

```sh
docker run --rm k8s.gcr.io/kube-apiserver-amd64:v1.15.3 kube-apiserver
```

```log
I0915 01:52:00.059047       1 serving.go:312] Generated self-signed cert (/var/run/kubernetes/apiserver.crt, /var/run/kubernetes/apiserver.key)
I0915 01:52:00.059604       1 server.go:560] external host was not specified, using 172.17.0.4
W0915 01:52:00.059912       1 authentication.go:415] AnonymousAuth is not allowed with the AlwaysAllow authorizer. Resetting AnonymousAuth to false. You should use a different authorizer
Error: --etcd-servers must be specified
Usage:
  kube-apiserver [flags]

...

error: --etcd-servers must be specified
```

Alright, the first dependency. We need to get `etcd` up and running. Before I do that, I want to see what happens if I specify an invalid `etcd` server.

```sh
docker run --rm k8s.gcr.io/kube-apiserver-amd64:v1.15.3 kube-apiserver --etcd-servers localhost:1234
```

```log
I0915 01:54:39.065367       1 serving.go:312] Generated self-signed cert (/var/run/kubernetes/apiserver.crt, /var/run/kubernetes/apiserver.key)
I0915 01:54:39.065476       1 server.go:560] external host was not specified, using 172.17.0.4
W0915 01:54:39.065532       1 authentication.go:415] AnonymousAuth is not allowed with the AlwaysAllow authorizer. Resetting AnonymousAuth to false. You should use a different authorizer
I0915 01:54:39.066241       1 server.go:147] Version: v1.15.3
I0915 01:54:39.444396       1 plugins.go:158] Loaded 9 mutating admission controller(s) successfully in the following order: NamespaceLifecycle,LimitRanger,ServiceAccount,TaintNodesByCondition,Priority,DefaultTolerationSeconds,DefaultStorageClass,StorageObjectInUseProtection,MutatingAdmissionWebhook.
I0915 01:54:39.444416       1 plugins.go:161] Loaded 6 validating admission controller(s) successfully in the following order: LimitRanger,ServiceAccount,Priority,PersistentVolumeClaimResize,ValidatingAdmissionWebhook,ResourceQuota.
I0915 01:54:40.444480       1 client.go:354] parsed scheme: ""
I0915 01:54:40.444555       1 client.go:354] scheme "" not registered, fallback to default scheme
I0915 01:54:40.444683       1 asm_amd64.s:1337] ccResolverWrapper: sending new addresses to cc: [{localhost:1234 0  <nil>}]
I0915 01:54:40.445108       1 asm_amd64.s:1337] balancerWrapper: got update addr from Notify: [{localhost:1234 <nil>}]
W0915 01:54:40.452577       1 clientconn.go:1251] grpc: addrConn.createTransport failed to connect to {localhost:1234 0  <nil>}. Err :connection error: desc = "transport: Error while dialing dial tcp 127.0.0.1:1234: connect: connection refused". Reconnecting...
W0915 01:54:41.446622       1 clientconn.go:1251] grpc: addrConn.createTransport failed to connect to {localhost:1234 0  <nil>}. Err :connection error: desc = "transport: Error while dialing dial tcp 127.0.0.1:1234: connect: connection refused". Reconnecting...
W0915 01:54:42.975555       1 clientconn.go:1251] grpc: addrConn.createTransport failed to connect to {localhost:1234 0  <nil>}. Err :connection error: desc = "transport: Error while dialing dial tcp 127.0.0.1:1234: connect: connection refused". Reconnecting...
W0915 01:54:45.063604       1 clientconn.go:1251] grpc: addrConn.createTransport failed to connect to {localhost:1234 0  <nil>}. Err :connection error: desc = "transport: Error while dialing dial tcp 127.0.0.1:1234: connect: connection refused". Reconnecting...
...
```

The logs just keep on going while it tries to connect to `etcd`. I also notice some logs about default TLS certs, and some invalid auth option. I get the feeling we're going to want to begin getting the `docker-compose.yml` ready so we can iterate quickly and add/replace options as necessary.

## Step 1: Build an initial docker-compose

Now that we have an idea how to run the container, and knowing that it relies on a lot of command line flags (as shown in the full log ouput when failing to specify `etcd` servers), can setup a base to work from which is easy to adjust and keep track of in source contol.

```yml  
version: "3.7"
services:

  kube-apiserver:
    image: k8s.gcr.io/kube-apiserver-amd64:v${KUBE_VERSION:-1.15.3}
    command: 
      - kube-apiserver
      - --etcd-servers=localhost:1234
    ports:
      - ${KUBE_APISERVER_PORT:-6443}
    networks:
      - kube-master

networks:
  kube-master:
```

Then simply run:

```sh
docker-compose up
# Alternatively, to override a variable:
KUBE_VERSION=1.14.6 docker-compose up
```

The compose file above gets us back to the logs we saw earlier, so that's a good point to launch off from. I foresaw the need for specifying a port to connect to the api server with so I added it now and chose `tcp/6443`.

In the configuration, I'll place variables with sane defaults so it's easy to override things like version or port number.

Before progressing the `kube-apiserver`, I feel we need to move on to getting etcd running.

## Step 2: Spin up `etcd`

I'll keep working off the `docker-compose.yml`, and will make `kube-apiserver` dependent on the `etcd` container.
A recent version of etcd can be found here: <https://console.cloud.google.com/gcr/images/google-containers/GLOBAL/etcd-amd64>.

Much like the `kube-apiserver`, `etcd` also doesn't run without an explicit command. I also found that without the `--listen-client-urls` option, it only listens on localhost and `kube-apiserver` is unable to open a socket.

By specifying only the `--listen-client-urls` option, `kube-apiserver` was still unable to connect. It appears as if `etcd` is responding with a list of urls to connect back on. I found the following option: `--advertise-client-urls`, and since we're using `docker-compose`, we can just specify the service anme which is resolvable via docker DNS in the same docker network (`kube-master`).

```yml
version: "3.7"
services:

  kube-apiserver:
    image: k8s.gcr.io/kube-apiserver-amd64:v${KUBE_VERSION:-1.15.3}
    command: 
      - kube-apiserver
      - --etcd-servers=etcd:2379
    ports:
      - ${KUBE_APISERVER_PORT:-6443}
    networks:
      - kube-master
    depends_on:
      - etcd

  etcd:
    image: k8s.gcr.io/etcd-amd64:${ETCD_VERSION:-3.3.15}
    command:
      - etcd
      - --listen-client-urls=http://0.0.0.0:2379
      - --advertise-client-urls=http://etcd:2379

    ports:
      - "2379"
    volumes:
      - etcd-data:/var/lib/etcd
    networks:
      - kube-master

networks:
  kube-master:

volumes:
  etcd-data:
```

**`docker-compose up` output**:

```log
Recreating kubernetes-from-scratch_etcd_1 ... done
Recreating kubernetes-from-scratch_kube-apiserver_1 ... done
Attaching to kubernetes-from-scratch_etcd_1, kubernetes-from-scratch_kube-apiserver_1
etcd_1            | 2019-09-15 03:09:36.241656 I | etcdmain: etcd Version: 3.3.15
etcd_1            | 2019-09-15 03:09:36.241711 I | etcdmain: Git SHA: 94745a4ee
etcd_1            | 2019-09-15 03:09:36.241721 I | etcdmain: Go Version: go1.12.9
etcd_1            | 2019-09-15 03:09:36.241733 I | etcdmain: Go OS/Arch: linux/amd64
etcd_1            | 2019-09-15 03:09:36.241740 I | etcdmain: setting maximum number of CPUs to 4, total number of available CPUs is 4
etcd_1            | 2019-09-15 03:09:36.241753 W | etcdmain: no data-dir provided, using default data-dir ./default.etcd
etcd_1            | 2019-09-15 03:09:36.242127 I | embed: listening for peers on http://localhost:2380
etcd_1            | 2019-09-15 03:09:36.242210 I | embed: listening for client requests on 0.0.0.0:2379
etcd_1            | 2019-09-15 03:09:36.250189 I | etcdserver: name = default
etcd_1            | 2019-09-15 03:09:36.250209 I | etcdserver: data dir = default.etcd
etcd_1            | 2019-09-15 03:09:36.250223 I | etcdserver: member dir = default.etcd/member
etcd_1            | 2019-09-15 03:09:36.250251 I | etcdserver: heartbeat = 100ms
etcd_1            | 2019-09-15 03:09:36.250261 I | etcdserver: election = 1000ms
etcd_1            | 2019-09-15 03:09:36.250271 I | etcdserver: snapshot count = 100000
etcd_1            | 2019-09-15 03:09:36.250286 I | etcdserver: advertise client URLs = http://etcd:2379
etcd_1            | 2019-09-15 03:09:36.250298 I | etcdserver: initial advertise peer URLs = http://localhost:2380
etcd_1            | 2019-09-15 03:09:36.250312 I | etcdserver: initial cluster = default=http://localhost:2380
etcd_1            | 2019-09-15 03:09:36.252979 I | etcdserver: starting member 8e9e05c52164694d in cluster cdf818194e3a8c32
etcd_1            | 2019-09-15 03:09:36.253027 I | raft: 8e9e05c52164694d became follower at term 0
etcd_1            | 2019-09-15 03:09:36.253049 I | raft: newRaft 8e9e05c52164694d [peers: [], term: 0, commit: 0, applied: 0, lastindex: 0, lastterm: 0]
etcd_1            | 2019-09-15 03:09:36.253060 I | raft: 8e9e05c52164694d became follower at term 1
etcd_1            | 2019-09-15 03:09:36.257974 W | auth: simple token is not cryptographically signed
etcd_1            | 2019-09-15 03:09:36.260204 I | etcdserver: starting server... [version: 3.3.15, cluster version: to_be_decided]
etcd_1            | 2019-09-15 03:09:36.260404 I | etcdserver: 8e9e05c52164694d as single-node; fast-forwarding 9 ticks (election ticks 10)
etcd_1            | 2019-09-15 03:09:36.260921 I | etcdserver/membership: added member 8e9e05c52164694d [http://localhost:2380] to cluster cdf818194e3a8c32
etcd_1            | 2019-09-15 03:09:36.758575 I | raft: 8e9e05c52164694d is starting a new election at term 1
etcd_1            | 2019-09-15 03:09:36.758610 I | raft: 8e9e05c52164694d became candidate at term 2
etcd_1            | 2019-09-15 03:09:36.758637 I | raft: 8e9e05c52164694d received MsgVoteResp from 8e9e05c52164694d at term 2
etcd_1            | 2019-09-15 03:09:36.758660 I | raft: 8e9e05c52164694d became leader at term 2
etcd_1            | 2019-09-15 03:09:36.758675 I | raft: raft.node: 8e9e05c52164694d elected leader 8e9e05c52164694d at term 2
etcd_1            | 2019-09-15 03:09:36.758934 I | etcdserver: setting up the initial cluster version to 3.3
etcd_1            | 2019-09-15 03:09:36.759434 N | etcdserver/membership: set the initial cluster version to 3.3
etcd_1            | 2019-09-15 03:09:36.759466 I | etcdserver/api: enabled capabilities for version 3.3
etcd_1            | 2019-09-15 03:09:36.759505 I | etcdserver: published {Name:default ClientURLs:[http://etcd:2379]} to cluster cdf818194e3a8c32
etcd_1            | 2019-09-15 03:09:36.759892 I | embed: ready to serve client requests
etcd_1            | 2019-09-15 03:09:36.760342 N | embed: serving insecure client requests on [::]:2379, this is strongly discouraged!
kube-apiserver_1  | I0915 03:09:37.852232       1 serving.go:312] Generated self-signed cert (/var/run/kubernetes/apiserver.crt, /var/run/kubernetes/apiserver.key)
kube-apiserver_1  | I0915 03:09:37.852279       1 server.go:560] external host was not specified, using 172.19.0.3
kube-apiserver_1  | W0915 03:09:37.852298       1 authentication.go:415] AnonymousAuth is not allowed with the AlwaysAllow authorizer. Resetting AnonymousAuth to false. You should use a different authorizer
kube-apiserver_1  | I0915 03:09:37.852868       1 server.go:147] Version: v1.15.3
kube-apiserver_1  | I0915 03:09:38.376629       1 plugins.go:158] Loaded 9 mutating admission controller(s) successfully in the following order: NamespaceLifecycle,LimitRanger,ServiceAccount,TaintNodesByCondition,Priority,DefaultTolerationSeconds,DefaultStorageClass,StorageObjectInUseProtection,MutatingAdmissionWebhook.
kube-apiserver_1  | I0915 03:09:38.376650       1 plugins.go:161] Loaded 6 validating admission controller(s) successfully in the following order: LimitRanger,ServiceAccount,Priority,PersistentVolumeClaimResize,ValidatingAdmissionWebhook,ResourceQuota.
kube-apiserver_1  | I0915 03:09:39.378172       1 client.go:354] parsed scheme: ""
kube-apiserver_1  | I0915 03:09:39.378220       1 client.go:354] scheme "" not registered, fallback to default scheme
kube-apiserver_1  | I0915 03:09:39.378298       1 asm_amd64.s:1337] ccResolverWrapper: sending new addresses to cc: [{etcd:2379 0  <nil>}]
kube-apiserver_1  | I0915 03:09:39.378383       1 asm_amd64.s:1337] balancerWrapper: got update addr from Notify: [{etcd:2379 <nil>}]
kube-apiserver_1  | I0915 03:09:39.381649       1 asm_amd64.s:1337] balancerWrapper: got update addr from Notify: [{etcd:2379 <nil>}]
```

## Step n: Connect to `kube-apiserver` via `kubectl`

I expect this to fail initially, and I'm not sure if we'll easily be able to get a successful connection without the `kube-scheduler` or `kube-controller-manager`, but I suspect we can still query `kube-apiserver` with those services down, so long as `etcd` is available.

Let's see if that's true.

```sh
kubectl config set cluster ...
kubectl config set context ...
kubectl config use-context ...

# Moment of truth
kubectl --context=... cluster-info
```

[part-0]: Kubernetes Architecture
[kube-apiserver]: https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/
[kubeadm]: https://github.com/kubernetes/kubeadm