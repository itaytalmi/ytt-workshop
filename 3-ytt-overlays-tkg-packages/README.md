# ytt Overlays for TKG Packages

This example demonstrates ytt overlays for TKG packages in situations where customizations are needed. Typically, a customization for a TKG package would be needed to implement a functionality that is not provided out of the box as part of the TKG package.
In this example, we will use the same example we used in the previous workshop, to implement node selectors and tolerations, but this time, apply it to a TKG package. The package we will use for this example is Cert-Manager. The Cert-Manager package is a good example as it contains several Kubernetes deployment resources, for which we can apply the node selectors and tolerations as part of this demonstration.

## Retrieving the TKG Package Files

Before you start working on the implementation of your overlay, you need to inspect the TKG package files and get familiar with the involved resources (the resources deployed as part of the package). Knowing the structure of the package helps build the logic for your overlay.

The first step would be to retrieve the package files locally:

```bash
# Step 1 - Get package latest version
PKG_VERSIONS=($(tanzu package available list cert-manager.tanzu.vmware.com -n tkg-system -o json | jq -r ".[].version" | sort -t "." -k1,1n -k2,2n -k3,3n))
PKG_VERSION=${PKG_VERSIONS[-1]}
echo "$PKG_VERSION"

# Step 2 - Get package image URL
PKG_IMAGE_URL=$(kubectl -n tkg-system get packages "cert-manager.tanzu.vmware.com.${PKG_VERSION}" -o jsonpath='{.spec.template.spec.fetch[0].imgpkgBundle.image}')
echo $PKG_IMAGE_URL

# Step 3 - Pull package image locally
imgpkg pull -b "$PKG_IMAGE_URL" -o ./pkg-files-tmp
```

Under the `./pkg-files-tmp/config` directory, you can now inspect the complete configuration files contained in the package.
In this case, we are interested in `./pkg-files-tmp/config/_ytt_lib/bundle/config/upstream/cert-manager.yaml`, which contains the Cert-Manager resources.
If you search this manifest for `kind: Deployment`, you should find three deployment resources. If you look at `./pkg-files-tmp/config/schema.yaml`, you can also see all the values accepted by the package. These are configurable parameters that can be set by the user. In this case, node selectors and tolerations
cannot be provided natively as part of this schema. Meaning, we have to come up with a solution for that using a ytt overlay.

We will now use the same overlay we used in workshop #2.

Our overlay is located in the `overlay-deployment-tolerations-node-selectors.yaml` file. Let's go over the contents:

```yaml
#@ load("@ytt:overlay", "overlay")
#@overlay/match by=overlay.subset({"kind": "Deployment"}),expects="1+"
---
spec:
  template:
    spec:
      #@overlay/match missing_ok=True
      nodeSelector:
        #@overlay/match missing_ok=True
        custom.tkg/node-type: infra
      #@overlay/match missing_ok=True
      tolerations:
        #@overlay/append
        - key: custom.tkg/node-type
          value: infra
          operator: Equal
          effect: NoSchedule
```

First, we load the ytt's `overlay` module, since we will be using `matchers`.

We then implement a `matcher` to look for resources of kind `Deployment` in the manifest, and we specify that at least 1 deployment is expected to be found.

We then add the logic that needs to be added/appended to our deployment once located by the matcher.
In this case, we are adding a node selector matching a node selector of `custom.tkg/node-type: infra` and toleration matching a node taint of `custom.tkg/node-type=infra:NoSchedule`. Note that to insert a section such as `nodeSelector` or `tolerations` to the manifest, you specify `#@overlay/match missing_ok=True` before the section, which tells ytt to add this section if it doesn't already exist. To add an item to a map such as `nodeSelector`, you then also use `#@overlay/match missing_ok=True`, and to add an item to an array such as `tolerations`, you use `#@overlay/append`. This is all shown in our example overlay.

To apply the package with the overlay, you can use the following Tanzu CLI command:

```bash
PKG_VERSIONS=($(tanzu package available list cert-manager.tanzu.vmware.com -n tkg-system -o json | jq -r ".[].version" | sort -t "." -k1,1n -k2,2n -k3,3n))
PKG_VERSION=${PKG_VERSIONS[-1]}
echo "$PKG_VERSION"

tanzu package install cert-manager \
--package "cert-manager.tanzu.vmware.com" \
--version "$PKG_VERSION" \
--ytt-overlay-file overlay-deployment-tolerations-node-selectors.yaml \
--namespace tkg-packages
```

If you look at the Cert-Manager deployments (`cert-manager`, `cert-manager-cainjector`, or `cert-manager-webhook`), you should see the node selectors and tolerations are present.

For example:

```bash
kubectl get deployment cert-manager -n cert-manager -o yaml | grep -i tolerations: -A4
```

Output:

```bash
      tolerations:
      - effect: NoSchedule
        key: custom.tkg/node-type
        operator: Equal
        value: infra
```
