# Day 1 GitOps on OKD

This repo is an example of how a cluster can be created and customized using ArgoCD from Day 1.
Instead of manually subscribing to ArgoCD and configuring Application this happens on cluster bootstrap:

### Manifests

`manifests` repo needs to be copied to `openshift` folder during cluster install, so that [Openshift would automatically apply those](https://docs.okd.io/latest/installing/install_config/installing-customizing.html#installation-special-config-kargs_installing-customizing):

```
$ openshift-install create manifests --dir=<installation_directory>
$ cp -r /path/to/checkout/manifests/*.yaml openshift
$ openshift-install create cluster
```

Modify `argo-bootstrap-04-configmap.yaml` to your liking - `Application` should be pointing to a repo with your ArgoCD configuration.

### What's under the hood

openshift-installer would automatically apply manifests from `openshift` folder. In order to setup our GitOps-managed cluster several manifests are sideloaded:
* `olm-subscription` ensures ArgoCD operator would be preinstalled
* `01-namespace` creates a new namespace to run ArgoCD
* `02-serviceaccount` creates a dedicated serviceaccount for ArgoCD bootstrapper
* `03-clusterrolebinding` ensures ArgoCD bootstrapper has permissions to create resources
* `04-configmap` contains necessary manifests which need to be applied to bootstrap ArgoCD:
  * ArgoCD CR declaration with no customizations
  * `argocd` application, which points to the repo with ArgoCD settings
  * clusterrolebinding for bootstrap ArgoCD application controller have permissions to apply changes
* `05-job` is a simple kubernetes job to apply these manifests, when cluster is ready. The manifests to apply are fetched from the configmap above.

On Openshift cluster initialization bootstrap would apply these manifests, the job would wait for cluster to be healthy enough to run user workloads. Once it applies the manifests, OLM would preinstall ArgoCD operator, the operator would install ArgoCD in the namespace and pick up the application settings.

Now ArgoCD would apply the manifests from `argocd-settings`, reconfiguring itself - adding openshift authentication, adding CRB for the server itself and applying additional applications - for instance `cluster-configs`, which adds htpasswd IDP and a new `admins` group.

By the time cluster completes installation, users in `admin` group can log in the cluster and work with ArgoCD without direct access to the operator or `argocd` namespace
