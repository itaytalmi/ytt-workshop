# YTT Basic YAML

This example demonstrates YTT templating for a simple YAML file.
The YAML manifest is a real Netplan manifest. Since a Netplan configuration can contain many different options, YTT can handle the logic for the different conditions.
This example aims to demonstrate how to manipulate and template this YAML file.

>Note that the YTT syntax is based on Starlark, which is heavily influenced by Python. If you are familiar with Python, you will notice many similarities.

## Read Base YAML

Our `base.yaml` file contains the minimal configuration required by Netplan, with no customization.

```yaml
---
network:
  version: 2
  ethernets:
    ens192:
      dhcp4: true
```

To read this file using YTT, you can run:

```bash
ytt -f base.yaml
```

The output will be the content of the file. This is the basic Netplan configuration, for DHCP.

## Apply YAML Logic and Customizations

The `base.yaml` manifest only contains DHCP configuration, but Netplan also supports specifying static IP configuration. Using YTT, we can apply logic and conditions to manipulate and template our manifest.
In this exercise, we want to allow the user to specify a static IP configuration, and use YTT to apply the logic to the manifest based on the provided parameters.

We will now be working with the `overlay.yaml` file to implement the templating logic.

First, we need to load the modules we will be using. As you can see in lines 1 and 2, we are loading the `overlay` and the `data` modules.

```yaml
#@ load("@ytt:overlay", "overlay")
#@ load("@ytt:data", "data")
```

The ytt’s Overlay feature provides a way to combine YAML structures with the help of annotations. We know the [overlay](https://carvel.dev/ytt/docs/latest/lang-ref-ytt-overlay) module is needed because we are going to use `matchers` to locate the relevant YAML document and resource in the document that needs to be modified.

We know the [data](https://carvel.dev/ytt/docs/latest/ytt-data-values) module is needed because we are going to be reading data values parameters from a values file.

Next, we need to implement a condition (line 4). In this case, we want to generate the YAML manifest based on user inputs. We know that if an IP address, a CIDR and a default gateway are specified, the generated configuration should be for static IP, and the DHCP parameters should be omitted.

```yaml
#@ if data.values.ip_address and data.values.cidr and data.values.gateway:
```

That opens the first condition. If an IP address, CIDR, and gateway are all defined, the logic will apply:

```yaml
#@ ip_cidr_notation = data.values.ip_address + "/" + data.values.cidr
#@overlay/match by=overlay.subset({"network":{"version": 2}})
---
network:
  ethernets:
    ens192:
      #@overlay/remove
      dhcp4: true
      #@overlay/match missing_ok=True
      addresses: #@ [ip_cidr_notation]
      #@overlay/match missing_ok=True
      routes:
        - to: default
          via: #@ data.values.gateway
```

Let's break it down:

First, the `ip_cidr_notation` variable is declared and concatenates the IP address and the CIDR in the required format (CIDR format).

Then, the `overlay/match` method is used to locate the YAML document in the manifest that needs to be modified. In this case, we are locating the document by searching for `network.version = 2`.

By using the `#@overlay/remove` method, we remove the `dhcp4: true` part from the manifest since DHCP is not necessary, and then append the relevant parts: `addresses` and `routes`, specifying the IP address and the default gateway. The `#@overlay/match missing_ok=True` is used as a matcher, to determine where to insert the configuration. the `missing_ok=True` flag is used to declare that if the inserted section is not already present in the manifest, ytt is "permitted" to add it to append the information.

So, we can now modify our `values.yaml` file and set the `ip_address`, `cidr`, and `gateway` variables. For example:

```yaml
#@data/values
---
ip_address: "192.168.1.50"
cidr: "24"
gateway: "192.168.1.1"
dns_servers: ""
search_paths: ""
```

And use ytt to load the `values.yaml` and the `overlay.yaml` files in addition to `base.yaml` this time:

```bash
ytt -f values.yaml -f base.yaml -f overlay.yaml
```

We will get the following result:

```yaml
network:
  version: 2
  ethernets:
    ens192:
      addresses:
      - 192.168.1.50/24
      routes:
      - to: default
        via: 192.168.1.1
```

The other two conditions perform the same thing for DNS servers and DNS search paths.

If we would like ytt to also render the configuration for DNS servers and search paths, we can can modify our `values.yaml` file and set the `dns_servers` and `search_paths` variables. For example:

```yaml
#@data/values
---
ip_address: "192.168.1.50"
cidr: "24"
gateway: "192.168.1.1"
dns_servers: "192.168.2.1,192.168.2.2"
search_paths: "mydomain.com,mydomain.io"
```

And run ytt again:

```bash
ytt -f values.yaml -f base.yaml -f overlay.yaml
```

We will get the following result:

```yaml
network:
  version: 2
  ethernets:
    ens192:
      addresses:
      - 192.168.1.50/24
      routes:
      - to: default
        via: 192.168.1.1
      nameservers:
        addresses:
        - 192.168.2.1
        - 192.168.2.2
        search:
        - mydomain.com
        - mydomain.io
```

Note that in the above example, the DNS servers and the search paths are specified in a comma-separated format in the `values.yaml`. However, ytt then splits those strings to convert them to a YAML array of strings. For example: `#@ data.values.search_paths.split(",")`.

If desired, you can also specify the data values from the CLI, instead of the `values.yaml` file, using the `-v` flag. This flag functionality is similar to the Helm `--set` parameter, allowing to pass supported parameters. For example:

```bash
ytt -f values.yaml -f base.yaml -f overlay.yaml \
-v ip_address="192.168.1.50" \
-v cidr="24" \
-v gateway="192.168.1.1" \
-v dns_servers="192.168.2.1,192.168.2.2" \
-v search_paths="mydomain.com,mydomain.io"
```
