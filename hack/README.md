
TODO add toc for file because we will have to add a lot of text for all the test

TODO add how-to/prereq. for e2e tests
TODO add how-to/prereq. for backend build unit tests
TODO add how-to/prereq. for backend build tests

# Get up and running w kpng

Run the local-up-kpng.sh script (make sure you have a kind or other cluster ready).

This will run kpng using incluster access to the apiserver, as a daeamonset.

You can test your changes by making sure you run both the `build` and `install` functions in this script.

This is just a first iteration on a dev env for kpng, feel free to edit/add stuff.

# How it works

This development recipe works by using `kind` to spin up a cluster.
However it cant use a vanilla kind recipe because:
- we need to add labels for kpng to know where its running its kube-proxy containers
- we need to add a kube-proxy service account 
- we also need to tolerate the controlplane node so that kpng runs there
- theres a bug in older kinds wrt kubeproxy mode = none

# Run from source

To run kpng from source, you can run
```
docker build -t myname/kpng:ipvs ./
IMAGE=myname/kpng:ipvs PULL=Never IMAGE=myname:kpng:ipvs BACKEND=ipvs ./kpng-local-up.sh
kind load docker-image myname/kpng:ipvs --name=kpng-proxy
```

# Details

After a few moments, youll see the kpng containers coming up...

thus the recipe has separate 'functions' for each phase of running KPNG.

- setup: setup kind and install it, gopath stuff
- build: compile kpng and push it to a registry
- install: delete the kpng daemonset and redeploy it

# Joining a kind node
When developing `kpng` in certain scenarios it might be required to join a [kind](https://github.com/kubernetes-sigs/kind)
node for better debugging, below the steps:

*<ins>Example: Cluster running with kind and kpng</ins>*:

1. Creating the cluster:  
`kpng> ./hack/test_e2e.sh -i ipv4 -b iptables`

2. Check all nodes are Ready
```
# kubectl get node
NAME                                   STATUS   ROLES                  AGE   VERSION
kpng-e2e-ipv4-iptables-control-plane   Ready    control-plane,master   94m   v1.22.2
kpng-e2e-ipv4-iptables-worker          Ready    <none>                 93m   v1.22.2
kpng-e2e-ipv4-iptables-worker2         Ready    <none>                 94m   v1.22.2
```

3. List docker process name
```
# docker ps --format '{{.Names}}'
kpng-e2e-ipv4-iptables-worker
kpng-e2e-ipv4-iptables-control-plane
kpng-e2e-ipv4-iptables-worker2
```

4. Join the kind node

`# docker exec -it ${NODE_NAME_HERE} sh`

```
# docker exec -it kpng-e2e-ipv4-ipvs-worker sh
# hostname
kpng-e2e-ipv4-iptables-control-plane
```

# E2E - Testing a single scenario of failure
As soon the `./hack/test_e2e.sh -i IP_FAMILY -b BACKEND` finish, it will create a kubernetes cluster based in `kind + kpng`.  
This cluster will keep running until the developer decide to manually remove it with all files generated in the kpng tree.

*<ins>Files generated by test_e2e.sh in the kpng tree</ins>*:
| Filename                | Description                                                              |
|-------------------------|--------------------------------------------------------------------------|
| artifacts               | kubeconfig for the test cluster, reports (junit generated from the test) |
| e2e.test                | The generated e2e.test                                                   |
| ginkgo                  | Ginkgo binary                                                            |
| kubeconfig_tests.conf   | The kubeconfig from the created kind cluster                             |

However, these files can be useful for developers speed up and execute a `single failure test` from the E2E framework.  

The example below focus **only** in the test `should be able to preserve UDP traffic when server pod cycles for a ClusterIP service` and **skip**: `Feature|Federation|PerformanceDNS|Disruptive|Serial|LoadBalancer|KubeProxy|GCE|Netpol|NetworkPolicy` tests.

*<ins>Executing a single KPNG test</ins>*:
```
kpng> ./hack/test_e2e.sh -i ipv4 -b iptables
kpng> cd temp/e2e
kpng/e2e2>./ginkgo --nodes="1" \
       --focus="should be able to preserve UDP traffic when server pod cycles for a ClusterIP service" \
       --skip="Feature|Federation|PerformanceDNS|Disruptive|Serial|LoadBalancer|KubeProxy|GCE|Netpol|NetworkPolicy" \
        ./e2e.test \
        -- \
        --kubeconfig="artifacts/kubeconfig_tests.conf" \
        --provider="local" \
        --dump-logs-on-failure=false \
        --report-dir="artifacts/reports" \
        --disable-log-dump=true
```

# Conntrack Notes

**Watching conntrack table**:

The following example keep listening for `UDP packages` with orig port as `12345`
```
#node> watch conntrack -L -p udp --orig-port-src=12345` 
udp      17 115 src=10.244.2.2 dst=10.96.237.45 sport=12345 dport=80 src=10.244.2.4 dst=10.244.2.2 sport=80 dport=12345
```

**Deleting all UDP entries**:
```
#node> conntrack -D conntrack --proto udp
udp      17 118 src=10.244.2.2 dst=10.96.237.45 sport=12345 dport=80 src=10.244.2.3 dst=10.244.2.2
                sport=80 dport=12345 [ASSURED] mark=0 use=1
conntrack v1.4.6 (conntrack-tools): 1 flow entries have been deleted.
```

# Contribute

This is just an initial recipe for learning how kpng works.  Please contribute updates
if you have ideas to make it better.  

- One example of a starting contribution might be
pushing directly to a local registry and configuring kind to use this reg, so dockerhub
isnt required.  
- Or a `tilt` recipe which hot reloads all kpng on code changes.







