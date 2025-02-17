# OpenShift GitOps Repo

This is an example on how I would structure a 1:1 (repo-to-single cluster)
git repo for OpenShift. This can be modified for other, [non OLM enabled](https://github.com/christianh814/example-kubernetes-go-repo),
Kubernetes clusters by just changing a few files.

This example assumes (as I mentioned in the 1:1 part above) that it's a
single repo for a single cluster. However, this can be modified (quite
easily) for poly/mono repos or for multiple clusters. This is meant as
a good starting point and not what your final repo will look like.

This is based on Argo CD but the same principals can be applied to Flux.

# Structure

Below is an explanation on how this repo is laid out. You'll notice
that I use [Kustomize](https://kustomize.io/) heavily. I do this since I
follow the [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)
principal when it comes to YAML files.

|#|Directory Name|Description|
|---|----------------|-----------------|
| 1. |`cluster-XXXX`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;| This is the cluster name. This name should be unique to the specific cluster you're targeting. For OpenShift, I like to use the output of `kubectl get infrastructure cluster -o jsonpath='{.status.infrastructureName}'`|
| 2. | `bootstrap` | This is where bootstrapping specifc configurations are stored. These are items that get the cluster/automation started. They are usually install manifests.<br /><br /> `base` is where are the "common" YAML would live and `overlays` are configurations specific to the cluster.<br /><br />The `kustomization.yaml` file in `default` overlay has `cluster-XXXX/components/applicationsets/` and `cluster-XXXX/components/argocdproj/` as a part of it's `bases` config.|
| 3. | `components` | This is where specific components for the GitOps Controller lives (in this case Argo CD).<br /><br />`applicationsets` is where all the ApplicationSets YAMLs live and `argocdproj` is where the ArgoAppProject YAMLs live.<br /><br />Other things that can live here are RBAC, Git repo, and other Argo CD specific configurations (each in their repsective directories).|
| 4. | `core` | This is where YAML for the core functionality of the cluster live. Here is where the Kubernetes administrator will put things that is necissary for the functionality of the cluster.<br /><br />Under `gitops-controller` is where you are using Argo CD to manage itself. The `kustomization.yaml` file uses `cluster-XXXX/bootstrap/overlays/default` in it's `bases` configuration. This `core` directory gets deployed as an applicationset which can be found under `cluster-XXXX/components/applicationsets/core-components-appset.yaml`.<br /><br />To add a new "core functionality" workload, one needs to add a directory with some yaml in the `core` directory. See the `container-security-operator` directory as an example.|
| 5. | `tenants` | This is where the workloads for this cluster live.<br /><br />Similar to `core`, the `tenants` directory gets loaded as part of an ApplicationSet that is under `cluster-XXXX/components/applicationsets/tenants-appset.yaml`.<br /><br />This is where Devlopers/Release Engineers do the work. They just need to commit a directory with some YAML and the applicationset takes care of creating the workload.<br /><br />**Note** that `bgd-blue/kustomization.yaml` file points to another Git repo. This is to show that you can host your YAML in one repo, or many repos.|

# Testing

To see this in action, just apply this repo.

```shell
until kubectl apply -k https://github.com/napsetsre/user-defined-metrics/cluster-XXXX/bootstrap/overlays/default; do sleep 3; done
```

This should give you 4 applications

```shell
$ kubectl get applications -n openshift-gitops
NAME                          SYNC STATUS   HEALTH STATUS
bgd-blue                      Synced        Healthy
container-security-operator   Synced        Healthy
myapp                         Synced        Healthy
gitops-controller             Synced        Healthy
```

Backed by 2 applicationsets

```shell
$ kubectl get appsets -n openshift-gitops
NAME      AGE
cluster   110s
tenants   110s
```

Visit the Argo CD UI to see it visualized, get the route by running (use "LOG IN VIA OPENSHIFT"):

```shell
kubectl get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}'
```

# Enjoy

Fork and Enjoy

> :warning: Feel free to fork this and play around with it, but remember if you'll have to change the applicationsets configuration and where the bases are pointing to if you're chanigng names of things
