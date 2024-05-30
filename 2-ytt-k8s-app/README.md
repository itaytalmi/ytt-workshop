# ytt - Kubernetes Deployment Manifests

This example demonstrates ytt templating for a Kubernetes deployment manifest. This example uses a sample web application.

## Templating and Functions

The `app.yaml` manifest contains the Kubernetes resources for a namespace, deployment and service.

```yaml
#@ load("@ytt:data", "data")

#@ def labels():
app: #@ data.values.app_name
#@ end

---
apiVersion: v1
kind: Namespace
metadata:
  name: #@ data.values.namespace
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: #@ data.values.namespace
  name: #@ data.values.app_name
spec:
  selector:
    matchLabels: #@ labels()
  template:
    metadata:
      labels: #@ labels()
    spec:
      containers:
      - name: #@ data.values.app_name
        image: #@ data.values.app_image
        ports:
        - name: http
          containerPort: #@ data.values.app_port
---
apiVersion: v1
kind: Service
metadata:
  namespace: #@ data.values.namespace
  name: #@ data.values.app_name
  labels: #@ labels()
spec:
  type: #@ data.values.svc_type
  ports:
  - port: #@ data.values.svc_port
    targetPort: http
  selector: #@ labels()
```

As you can see, there are several operations performed here:

- The `labels` function is defined and contains the `app` label and the application name (e.g., `spring-petclinic`) as the value. The reason we define this function is that labels are required by multiple fields. The deployment resource requires them for `spec.matchLabels` and `spec.template.metadata.labels` and the service resource requires them for `metadata.labels` and `spec.selector`. Since these fields all must match, we would have to set `app` labels for all these fields. Since we are using a function here, we only need to define the labels once in a function, and then call the function everywhere. This saves a lot of duplicate code.
- We read all the relevant information required for our deployment from the `values.yaml` file, using the ytt `data` module. We consume the namespace, application name, application image, port numbers, and more, from the `values.yaml` file. Our `values.yaml` file contains:

```yaml
#@data/values-schema
---
namespace: dev
app_name: spring-petclinic
app_image: springio/petclinic:latest
app_port: 8080
svc_port: 80
svc_type: ClusterIP
```

We can use the ytt CLI to get the resulting (templated) manifest:

```bash
ytt -f app.yaml -f values.yaml
```

Example output:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: dev
  name: spring-petclinic
spec:
  selector:
    matchLabels:
      app: spring-petclinic
  template:
    metadata:
      labels:
        app: spring-petclinic
    spec:
      containers:
      - name: spring-petclinic
        image: springio/petclinic:latest
        ports:
        - name: http
          containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  namespace: dev
  name: spring-petclinic
  labels:
    app: spring-petclinic
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: http
  selector:
    app: spring-petclinic
```

We can also pass this output directly to kubectl to deploy the resources:

```bash
ytt -f app.yaml -f values.yaml | kubectl apply -f -
```

We can also use the kapp CLI to deploy the resources.

```bash
ytt -f app.yaml -f values.yaml | kapp deploy -y -a spring-petclinic -f -
```

## Adding an Overlay

There are customizations you may need to apply to your deployments. For ytt to render these customizations, you have to implement overlays.
In this example, we will implement an overlay to apply tolerations and node selectors to our sample app from the previous sections.

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

To include the overlay in our deployment, we can add it to our ytt syntax:

```bash
ytt -f app.yaml -f values.yaml -f overlay-deployment-tolerations-node-selectors.yaml
```

Example output:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: dev
  name: spring-petclinic
spec:
  selector:
    matchLabels:
      app: spring-petclinic
  template:
    metadata:
      labels:
        app: spring-petclinic
    spec:
      containers:
      - name: spring-petclinic
        image: springio/petclinic:latest
        ports:
        - name: http
          containerPort: 8080
      nodeSelector:
        custom.tkg/node-type: infra
      tolerations:
      - key: custom.tkg/node-type
        value: infra
        operator: Equal
        effect: NoSchedule
---
apiVersion: v1
kind: Service
metadata:
  namespace: dev
  name: spring-petclinic
  labels:
    app: spring-petclinic
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: http
  selector:
    app: spring-petclinic
```

As you can see, the node selectors and tolerations were applied and added to our deployment using the overlay.
