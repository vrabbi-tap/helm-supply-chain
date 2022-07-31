# TAP Supply Chain to deploy a helm chart using FluxCD

## Why do this

Many organizations already have helm charts developed that are fine tuned to their applications needs.
While moving to the TAP native, knative deployment mechanism is very appealing to many, not all solutions are solved in this manner. Having a way to deploy a helm chart via TAP, while still enabling the rest of the benefits of the platform (security, testing, visibility, abstraction, etc.) is a key part in TAP gaining adoption.

## Requirments

* this has only been tested in internet accessible environments. in Air Gapped deployments, your mileage may vary
* this has only been tested with the GitOps supply chain mechanism. With RegistryOps, your mileage may vary

## How to do this

### Overview

TAP already includes a key component for this which is The FluxCD Source Controller.
With the source controller, we have access to 2 CRDs that interest us:

1. HelmRepository
2. HelmChart

While these 2 resources are great, we need one other CRD which is in charge of the actual deployment and reocncilliation of the helm releases, which is called HelmRelease.

This CRD is provided as part of the FluxCD Helm Controller.

### Installing FluxCD Helm Controller

FluxCD Helm Controller can be easily installed from the Tanzu Community Edition package via the following commands:

```bash
kubectl create ns tap-extra-packages
kubectl apply -f https://raw.githubusercontent.com/vmware-tanzu/community-edition/main/addons/packages/fluxcd-helm-controller/metadata.yaml -n tanzu-package-repo-global
kubectl apply -f https://raw.githubusercontent.com/vmware-tanzu/community-edition/main/addons/packages/fluxcd-helm-controller/0.17.2/package.yaml -n tap-extra-packages
tanzu package install -n tap-extra-packages fluxcd-helm-controller -p helm-controller.fluxcd.community.tanzu.vmware.com -v 0.17.2
```

### Configuring a Helm Repository

Once we have FluxCD installed, we can create a HelmRepository resource to allow discovery and usage of the relevant charts. For this example we will use a demo repo i have hosted via Github Pages in this repository under the docs folder:

```bash
kubectl apply -f example/helm-repo.yaml
```

### Create the custom Cartographer config template

The next step is to create the cartographer resources to allow us to utilize the helm controller, in our platform.

```bash
kubectl apply -f config-template-helm.yaml
```

### Creating the custom supply chain
The first step is to fill in the values.yaml file in this repo based on your environment. These can be the same values as you have supplied for the ootb_supply_chain section in your tap installation values.

Once you have set the values in the file, we can now render and apply the supply chain manifest:
```bash
ytt -f values.yaml -f supply-chain.yaml | kubectl apply -f -
```

### Adding the needed RBAC
As we are adding a new resource type, that will be stamped out by the platform, we will need to add the relevant RBAC rules in our runtime and iterate clusters. The easiest way is to add a ClusterRole with the label 'apps.tanzu.vmware.com/aggregate-to-deliverable: "true"'. By using this label, we tell kubernetes to aggregate this role into the OOTB deliverable ClusterRole which we can use to give developers access to in the platform.
```bash
kubectl apply -f cluster-role.yaml
```

### Testing out the new supply chain
To test out the new supply chain, we can use the example workload in this repo
```bash
tanzu apps workload apply -f example/workload.yaml
```
