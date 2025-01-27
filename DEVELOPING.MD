# Developer Guide for vSphere Charts for Rancher

This guide is going to highlight the steps that are required to develop and maintian the vSphere charts for [CPI](https://github.com/kubernetes/cloud-provider-vsphere) and [CSI](https://github.com/kubernetes-sigs/vsphere-csi-driver) for Rancher. Upstream is only releasing manifests at this point and they do that in version specific branches. We use these upstream manifests to create the charts that we use. This makes the process a touch more complex as each manifest needs compared to the helm chart template to make the appropriate changes. Since we mirror images into our own container registry, we also have to check the manifests every time a release happens to gather the correct image versions to keep our registry synced. Once the charts and images have been updated, we then need to ensure that the charts are updated in both [RKE2-Charts](https://github.com/rancher/rke2-charts), [RKE2](https://github.com/rancher/rke2), and [Rancher Charts](https://github.com/rancher/charts).  

#### You may be asking why both RKE2 and Rancher charts? 

RKE2 embeds the charts that it ships with each version of RKE2 that is released. This means that RKE2 always uses it's own version of the vSphere charts regardless if deployed standalone or through Rancher. The charts being deployed are determined by the selected cloud provider when provisioning. The Rancher charts shouldn't be consumed for RKE2 and currently exist for use with non-RKE2 clusters that are provisioned in Rancher, primarily RKE1. 

#### Upstream releases 

Upstream releases are not immediate when a new version of Kubernetes is released. This means that the charts typically lag behind Kubernets verison by at least a month if not longer. This means that often that users can't upgrade clusters immediately or deploy the newest version that is released in RKE2 or Rancher with full support.

#### Cloud Provider Skew Policy

Cloud providers, cloud controller manager, in Kubernetes support a [skew policy](https://kubernetes.io/releases/version-skew-policy/#kube-controller-manager-kube-scheduler-and-cloud-controller-manager). This allows us to use one minor version older of a CPI on Kubernetes cluster. The primary purpose is to allow upgrades, yet it also gives us the ability to deploy an older CPI before the official one has been released for the new version of Kubernetes. CSI doesn't have these gaurantees explicitly defined and so don't assume with CSI that it will work.

## Chart Design

Both the CSI and CPI charts in this repository are designed to be what have been called **super charts**. A super chart is a chart that uses a helm template function to determine which image versions should be deployed based on the Kubernetes version of the cluster. The image versions and configuration is part of an array of values that use semver to determine if it can be deployed on a Kubernetes version.

By creating a super chart, it means there is only ever one chart and we can ensure that users are always getting the most up to date images based on their Kubernetes version. This technique is used in other charts inside of Rancher.

## Development  

Features and upstream releases are the primary reason to be working in this repository. Upstream releases through either bug fixes or new Kubernetes version support is the majority of the work that is required. Below is a high level of the steps that need to occur.

* Check the upstream manifests for changes and new image versions
* Update the [image-mirror](https://github.com/rancher/image-mirror) repository for new images
* Update the charts with the new images and manifest changes
* Locally compile and deploy to a Kubernetes cluster to ensure that the chart functions
* Check with RKE2 on the next release cycle and plan out PRs for getting these changes included by the target release date.
* Update Rancher Charts with the new versions

Before getting started, it is suggested to create an issue in the Rancher/Rancher repository to track the work that has to occur. It spans several different repositories which is often hard to coordinate. Here is the recommend issue format. 

```
**Is your feature request related to a problem? Please describe.**



**Describe the solution you'd like**



# Checklist for items that need completed

- [ ] Update Image-mirror repo
- [ ] Update manifests in rancher/vsphere-charts
- [ ] Notify about chart "release"
- [ ] Update chart in rancher/charts
- [ ] Update chart in rancher/rke2-charts
- [ ] Updated RKE2 to consume new chart
```

Then for every PR, link back to this issue so everything can be tracked. When you get to RKE2, you will need to create a separate issue in the RKE2 repository in addition to the Rancher one. 

Here are some examples:

* Rancher Issue: https://github.com/rancher/rancher/issues/38188
* RKE2 issue: https://github.com/rancher/rke2/issues/2867

### Step 1: Checking upstream manifests

There are two goals when checking upstream manifests. The first goal is to check to see if any image versions have been updated and the second is to check if any changes in the manifests have occured. 

If you find that image versions have been updated, then you will need to make a pull request to the [image-mirror](https://github.com/rancher/image-mirror) repository first before proceeding. [Here](https://github.com/rancher/image-mirror/pull/261) is an example PR and follow the requirements outlined in the PR template for those changes. After the images have been checked and image-mirror updated with any new images or image versions, then proceed to check check the manifests.

Checking the manifests is best done by cloning down the branch for the version that you want to compare with the helm templates. Then using a diff tool like, Kdiff/Meld/WinMerge, compare each manifest directory with the helm chart directory. Look at the diffs and decide if that diff is brining in a modification or addition that needs to be added to the charts in this repository. If you determine the diff needs brought over, then do so. 

Finally, make sure that the unit tests for each chart has test cases added to existing tests, test are updated, or new tests are added based on the set of changes.

### Step 2: Compile and check locally

At this point you can verify everything passes linting and tests by running:

```
$ make ci

### linting charts ###
helm lint ./charts/rancher-vsphere-cpi/
==> Linting ./charts/rancher-vsphere-cpi/

1 chart(s) linted, 0 chart(s) failed
helm lint ./charts/rancher-vsphere-csi/
engine.go:167: [INFO] Missing required value: .Values.vCenter.clusterId must be provided
==> Linting ./charts/rancher-vsphere-csi/

1 chart(s) linted, 0 chart(s) failed
### packaging charts for testing purposes ###
helm package ./charts/rancher-vsphere-cpi/
Successfully packaged chart and saved it to: /home/jphillips/work/vsphere-charts/rancher-vsphere-cpi-1.4.0.tgz
helm package ./charts/rancher-vsphere-csi/
Successfully packaged chart and saved it to: /home/jphillips/work/vsphere-charts/rancher-vsphere-csi-2.5.1-rancher1.tgz
### removing chart packages ###
rm *.tgz
### setup ###
go mod tidy
### running unit tests ###
go test -v -tags helm ./tests/unit
--- PASS: TestCSITemplateRenderedControllerDeployment (0.53s)
    --- PASS: TestCSITemplateRenderedControllerDeployment/Kubernetes_1.23 (0.09s)
    --- PASS: TestCSITemplateRenderedControllerDeployment/Kubernetes_1.22 (0.09s)
    --- PASS: TestCSITemplateRenderedControllerDeployment/Kubernetes_1.21 (0.09s)
    --- PASS: TestCSITemplateRenderedControllerDeployment/Kubernetes_1.20 (0.09s)
    --- PASS: TestCSITemplateRenderedControllerDeployment/Kubernetes_1.23_with_CSI_Resizer_Enabled (0.09s)
    --- PASS: TestCSITemplateRenderedControllerDeployment/Kubernetes_1.20_with_CSI_Resizer_Enabled (0.08s)
PASS
ok  	github.com/rancher/vsphere-charts/tests/unit	(cached)
```

If everything passes here then it's time to test deployment to a cluster, if something errors, then please correct it and ensure all checks pass.

# Step 3: Deploying the chart

At this stage, it's now safe to actually test the charts on a real cluster. Typically, a RKE1 cluster is deployed on vSphere using Rancher. Then the kubeconfig is downloaded from the cluster and used to deploy the local chart to the cluster using helm. Here is an example. Make sure to supply the vSphere configuration required.

```
$ helm install -f vsphere-test-config.yaml --kubeconfig <path to kube config> cpi ./charts/rancher-vsphere-cpi 
```

Ensure that all pods for both the CPI and CSI charts come up without errors. Make sure that the provider gets set on each node to be vSphere. 

# Step 4: PR to Rancher vSphere Charts Repository

You are now ready to create your PR to the vSphere charts repository. Fill out the PR template and request reviews. Once the PR has been approved and passed the Drone run, you can merge. 

# Step 5: Create RKE2-Charts PR

Now you can pull the chart into the RKE2-Charts repository. [Here](https://github.com/rancher/rke2-charts/pull/250) is an exapmle PR. 

# Step 6: Create RKE2 PR

Once the RKE2-Charts PR has been merged, you can update RKE2. You will need to coordinate with the RKE2 before submitting the PR. This step will also require that you backport the PR to each branch of the still supported versions of RKE2, again coordinate with the RKE2 team before proceeding.

[Here](https://github.com/rancher/rke2/pull/2868) is an example PR and an [example](https://github.com/rancher/rke2/pull/2870) backport.

# Step 7: Create Rancher Charts PR

This is the last step of the process, which is to now pull the charts into the Rancher charts repository. The process for that repository is well documented, just follow the instructions listed in that repository for the workflow. [Here](https://github.com/rancher/charts/pull/1898) is an example PR.