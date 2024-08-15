# High-Availability & Sharding of Flux Components
The document https://fluxcd.io/flux/installation/configuration/sharding/ provides a good explanation of how sharding of Flux components can be implemented such that each shard processes flux objects that are specific to it.

The `sharding-example` branch of this repository shows how we can use the sharding example presented in this document and use it to deploy Flux components in High Availability.

## What this repository does
1. It deploys Flux components with 3 shards in a Highly Available configuration by applying node labels and using kustomize to patch Deployments of Flux components with `nodeAffinity` configuration.
2. It installs `cert-manager` and `postgres-operator` in their own namespaces using Flux.
3. It makes sure that sources and the CRDs associated with `cert-manager` and `postgres-operator` are installed before anything else (see [clusters/staging/infrastructure.yaml](clusters/staging/infrastructure.yaml)). This is important because some applications may use the CRDs.

## Future additions
1. Add `rethinkdb-operator`, `msr-operator` and MSR application once I figure out why flux appears to hang when installing [rethinkdb-operator-1.0.1.yaml](https://gist.githubusercontent.com/wahabmk/d74f3263b18b967f9a697c6aec088cc7/raw/29f8990d5f81bba4c658209c47a117c18bf3fda3/rethinkdb-operator-1.0.1.yaml).
2. Currently Flux components are deployed in an HA configuration but the sharding is not being taken advantage of by configuring apps to run on specific shards.

## How to Deploy
1. Create a kind cluster with 3 worker nodes:
```sh
kind create cluster --name mycluster --config kind-config.yaml
```

2. Check if all 3 worker nodes are running with the `sharding.fluxcd.io/key:<shard>` label applied to each:
```sh
➜  kubectl get nodes -o custom-columns=NAME:.metadata.name,LABEL:.metadata.labels 
NAME                      LABEL
mycluster-control-plane   map[beta.kubernetes.io/arch:arm64 beta.kubernetes.io/os:linux kubernetes.io/arch:arm64 kubernetes.io/hostname:mycluster-control-plane kubernetes.io/os:linux node-role.kubernetes.io/control-plane: node.kubernetes.io/exclude-from-external-load-balancers:]
mycluster-worker          map[beta.kubernetes.io/arch:arm64 beta.kubernetes.io/os:linux kubernetes.io/arch:arm64 kubernetes.io/hostname:mycluster-worker kubernetes.io/os:linux sharding.fluxcd.io/key:shard1]
mycluster-worker2         map[beta.kubernetes.io/arch:arm64 beta.kubernetes.io/os:linux kubernetes.io/arch:arm64 kubernetes.io/hostname:mycluster-worker2 kubernetes.io/os:linux sharding.fluxcd.io/key:shard2]
mycluster-worker3         map[beta.kubernetes.io/arch:arm64 beta.kubernetes.io/os:linux kubernetes.io/arch:arm64 kubernetes.io/hostname:mycluster-worker3 kubernetes.io/os:linux sharding.fluxcd.io/key:shard3]
```

3. Bootstrap flux into the cluster with this repository:
```sh
flux bootstrap git \
--token-auth=true \
--url=https://github.com/wahabmk/fun-with-fluxcd \
--username=<username> \
--password=$GITHUB_PAT \
--branch=sharding-example \
--path=clusters/staging \
--verbose
```

4. Verify that all shards for each Flux component are running:
```sh
➜  kubectl -n flux-system get deployments      
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
helm-controller               1/1     1            1           23m
helm-controller-shard1        1/1     1            1           23m
helm-controller-shard2        1/1     1            1           23m
helm-controller-shard3        1/1     1            1           23m
kustomize-controller          1/1     1            1           23m
kustomize-controller-shard1   1/1     1            1           23m
kustomize-controller-shard2   1/1     1            1           23m
kustomize-controller-shard3   1/1     1            1           23m
notification-controller       1/1     1            1           23m
source-controller             1/1     1            1           23m
source-controller-shard1      1/1     1            1           23m
source-controller-shard2      1/1     1            1           23m
source-controller-shard3      1/1     1            1           23m
```

5. Verify that the shard for each pod is running on the node labeled with that shard:
```sh
➜  kubectl -n flux-system get pod -o=custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName --all-namespaces --sort-by='{.metadata.name}' | grep 'shard'
helm-controller-shard1-56ddb5d4dd-wn6ng           Running   mycluster-worker
helm-controller-shard2-8bcbf5675-l5fwt            Running   mycluster-worker2
helm-controller-shard3-65d458fffd-z2h76           Running   mycluster-worker3
kustomize-controller-shard1-59b59859db-qwvp9      Running   mycluster-worker
kustomize-controller-shard2-584d676df5-264g2      Running   mycluster-worker2
kustomize-controller-shard3-6664d554d4-khpq5      Running   mycluster-worker3
source-controller-shard1-7476469fb4-bj469         Running   mycluster-worker
source-controller-shard2-7fd5ffdf6d-89ndw         Running   mycluster-worker2
source-controller-shard3-d5b7747d8-5cczk          Running   mycluster-worker3
```
