# Postgres 17 on Kubernetes

Application-developer-oriented annotated Kubernetes manifests to deploy Postgres 17 on Kubernetes.

* `Deployment` with only 1 replica.
* `PersistentVolume` using `hostPath` (maybe exploring Minikube's `csi` at some point).
* Tiny part of the config is slightly tailored for `Minikube`-consideration, not for production.

Postgres is a database, deployment of stateful service in Kubernetes always come with extra sets of care that need to considered compared to stateless services.

Production-grade stateful deployment can be super complex, that's why it's not uncommon to see Operator for data-intensive technology (such as: `Strimzi`, `CockroachDB Operator`, `Prometheus Operator`) being offered to simplify the K8s management for stateful deployment.

This example mainly serves as intro and exploration of the gist behind K8s storage abstractions that power stateful deployment.

## Handling data persistence using `Volume`

Handling stateful-ness means dealing with `Volume`. Just like a typical Docker container. Files/data in containers in each Pod are ephemeral by default. Thus, mounting volumes(s) is typically the obvious solution to handle data persistence. On top of that, a `Volume` can be used as file-sharing mechanism between containers within a Pod.

> To be explicit, the backend of K8s `Volume` can vary greatly, the backend of a `Volume` doesn't have to conform to a typical disk/filesystem-like. In fact, it's not uncommon to use other Kubernetes resources (such as: `ConfigMap`/`Secret`) to be mounted as Volume in a container. There are several other unique [backend such as `emptyDir` ](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) that may be used to solve unique kind of use-cases.

At its core, a `Volume` is a directory, possibly with some data in it, which then mounted to container(s) in a `Pod`.

To deploy `postgres` service, we're going to use `Volume` that uses `PersistentVolumeClaim` as its backend. The lifetime of the underlying  `PersistentVolume` resource is detached from any containers lifetime. Thus, this is suitable to be used to preserve data on events of containers restarts.

Before going further, let's first clarify the differences between `PersistentVolumeClaim` and `PersistentVolume` (they're different resource kind).

## `PersistentVolume (PV)`  & `PersistentVolumeClaim (PVC)`

Despite its name `PersistentVolume` is not a valid backend for a `Volume`. It can't be used *directly*. Instead, you need to use `PersistentVolumeClaim` in order to mount   `PersistentVolume` as a volume.

When you create `PersistentVolumeClaim`, k8s will determine how to fulfill this claim using any available  `PersistentVolume` that can match its requirements. When you create `PersistentVolumeClaim` you specify these requirements. (this also mean, you can accidentally misconfigure your PVC that won't get fulfilled by the PV you expect, or the other way around).

Example of PVC fulfillment: two available PV in a cluster each have `10Gi` and `5Gi` unused capacity. If the PVC asks for `8Gi` capacity. It's not possible to use PV with only `5Gi` available to fulfill this PVC.

### Why the needs for `PersistentVolumeClaim`? Why not directly using `PersistentVolume`?

It's related to how storage is abstracted in Kubernetes, `PersistentVolumeClaim` is made unaware of the underlying physical mechanism on how its data will get accessed or stored. The `PersistentVolume` takes care of such details as its internal details. In turn, various physical storage mechanism are supported by `PersistentVolume` and  `PersistentVolumeClaim` is completely unaware of it.

It's useful to see these separation from perspective of responsibility during application development. The creation of `PersistentVolume` and `PersistentVolumeClaim` creation is detached during application development lifecycle.

It's helpful to imagine 2 parties often involved in application development that use k8s for its deployment. The K8s administrator (typically dedicated devops role)...


