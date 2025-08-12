# Postgres 17 in Kubernetes

Application-developer-oriented reference to deploy Postgres 17 in Kubernetes.

* Setup `Deployment` with 1 replica to focus on the topic of `Volume`.
* Setup `PersistentVolume` using `hostPath` backend, thoroughly explaining typical considerations related to PV provisioning and the subsequent PVC binding.

> Some parts of this config is tailored for local `Minikube`-consideration, but tge generic ideas applies, production considerations will be mentioned when appropriate.

This write-up serves as intro and exploration behind K8s `Volume` and storage abstractions that handles data persistence when deploying stateful service. *Without* glossing over the organizational aspects that should be considered around `PersistentVolume` lifecycle.

---

Postgres is a database, deployment of stateful service in Kubernetes always come with extra sets of care that need to considered compared to stateless services.

Managing production cluster with stateful deployments can be complex, that's why it's not uncommon to see Kubernetes Operator (such as: `Strimzi`, `CockroachDB Operator`, `Prometheus Operator`) being offered to simplify management of stateful deployment in production. But now let's open one part of the trunk and see what's under the hood a little bit.

## Handling data persistence using `Volume`

Having stateful service means dealing with `Volume`. Just like a typical Docker container. Files/data in containers in each Pod are ephemeral by default. Thus, mounting volumes(s) is typically the default approach to handle stateful app's data persistence. On top of that, a `Volume` can also be used as file-sharing mechanism between containers within a Pod.

The backend of a `Volume`, [often referred as "Volume type"](https://kubernetes.io/docs/concepts/storage/volumes/#volume-types), can vary greatly. A volume backend doesn't have to be a typical disk/filesystem-like. In fact, it's not uncommon to use other Kubernetes resources (such as: `ConfigMap`/`Secret`) to be mounted as Volume in a container. There are also several other unique backend, [`emptyDir` being one of them](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir), that may be used to solve unique kind of use-cases.

At its core, a `Volume` is a directory, possibly with some data in it, which is mounted to container(s) in a `Pod`.

To deploy a `postgres` service, we're going to use `Volume` that uses `PersistentVolumeClaim` as its backend. The lifetime of its underlying `PersistentVolume` is detached from any containers lifetime. Thus, this is suitable to preserve states on events of containers restarts.

Before going further, let's first clarify the differences between `PersistentVolumeClaim` and `PersistentVolume` (they're different resource kind).

## `PersistentVolume (PV)` & `PersistentVolumeClaim (PVC)`

Despite of its name, `PersistentVolume` is not a valid backend for a `Volume`. It can't be used *directly*. Instead, you need to use `PersistentVolumeClaim` in order to mount `PersistentVolume` as a volume.

When you create `PersistentVolumeClaim`, k8s will perform various checks to determine how to fulfill this claim, such as:
* Check parameter such as `spec.storageClassName` to determine whether you request statically/dynamically-provisioned PV.
* Check whether any available `PersistentVolume`(s) can match PVC requirements.
* and more..

In production, the responsibility of authoring `PersistentVolumeClaim` manifest not only influenced by app requirements but also how organization cluster being setup. This also means it's possible to misconfigure your PVC if you're unaware about certain configurations/defaults that might be in place within organization cluster, this aspects are often omitted from tutorial.

Simple PVC fulfillment example (Kubernetes official term for this is "binding"): two available PV in a cluster each have `10Gi` and `5Gi` unused capacity. If the PVC asks for `8Gi` capacity. It's not possible to use PV with only `5Gi` available to fulfill this PVC. `PersistentVolumeClaims`s will remain unbound indefinitely if a matching `PersistentVolume` does not exist.

### Why the needs for `PersistentVolumeClaim`? Why not directly using `PersistentVolume`?

This is due to how storage gets abstracted in Kubernetes, `PersistentVolumeClaim` is designed to be more "user-focused", it's unaware of the underlying physical storage that it uses.

Such details are purposely encapsulated inside `PersistentVolume`. Since `PersistentVolume` also need to support multiple storage backend, this abstraction is deliberately put in place by Kubernetes

This way, `PersistentVolumeClaim` doesn't need to be aware how any underlying PV works as long as it provides the storage capabilities it needs.

***But why shouldn't it care? Wouldn't it be easier to just use one resource kind?***

To answer that, it's useful to also consider these separation from organizational responsibility perspective. Since the configuration around `PersistentVolume` and `PersistentVolumeClaim` often handled independently in different stages of application development lifecycle.

### Provisioning & PVC Binding

Consider 2 parties that are typically involved in app development:

In production, the K8s cluster admin (someone responsible to manage organization cluster, typically DevOps folks) may provision the necessary infrastructures and preemptively setup `PersistentVolume`(s) in organization cluster way before application developer need to perform deployment.

This is due to the nature of `PersistentVolume` lifecycle management, many decisions can be quite complex, and influenced by operational cost ($):

* Whether your cluster supports automatic/dynamic PV provisioning based on PVC or strictly static (meaning each PV need to be setup manually before being claimed)
* Various degree of storage backend and quality of service that your organization need to support

Taking the scenario further, to support mission-critical aspect, k8s cluster should likely choose remote persistent storage backend that can withstand cluster/node(s) crashes.

Considering and weighing options such as: AWS EBS, rolling out their own Longhorn/Rook, etc. As you can see these decisions can be quite complex and nuanced, but ideally should be made irrelevant to application developer due to the abstraction that `PersistentVolumeClaim` provides.

Once organization decided on these type of decisions for their cluster, the "claiming" workflow will only be partially influenced by these decisions.

---

Assuming `PersistentVolume`s are ready, responsible author of `PersistentVolumeClaim` can theoretically be isolated and focus on requesting resource that their app needs, detached from the decisions whether such storage is powered by AWS EBS, Rook, or else.

If everything is configured accordingly, the correct `PersistentVolume` will be bind to fulfill the appropriate `PersistentVolumeClaim`.

In case of organization that uses dynamic provisioning, PV will be provisioned automatically based on appropriate `StorageClass` that's declared in each `PersistentVolumeClaim`, thus application developer in an organization must be onboarded of this configuration. If a PV was dynamically provisioned for a new PVC, the provisioned PV will always be bind to that PVC

**A PVC to PV binding is a one-to-one mapping**

## `postgres` Example

With enough basics to cover PV-PVC concepts, you can explore the manifests starting with [ns.yaml](./kubernetes/ns.yaml) and [pv.yaml](./kubernetes/pv.yaml). The manifest is heavily annotated to connect the dots between concept that can seems implicit.

```sh
kubectl apply -f kubernetes
# Port forward nodePort on Windows appropriately
minikube service postgres-svc -n x-minikube-postgres --url
psql postgresql://postgres:pg-password@localhost:37253/app # don't forget to modify assigned port
CREATE TABLE IF NOT EXISTS test_table ( id text primary key );
\dt
\q
```
