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

# Step 3 - Pull package image
imgpkg pull -b "$PKG_IMAGE_URL" -o PACKAGE-DIR
```
