---
myst:
  html_meta:
    description: "Deploy Charmed OpenSearch on LXD virtual machines or on Kubernetes with Juju, including prerequisites, kernel tuning, and bootstrap steps."
---

<!-- vale off -->
(how-to-deploy-anywhere)=
(how-to-deploy-lxd)=
<!-- vale on -->
# How to deploy Charmed OpenSearch

This guide walks you through setting up your environment and deploying Charmed
OpenSearch, covering both the **IAAS/VM** charm (`opensearch`) and the
**Kubernetes** charm (`opensearch-k8s`).

Start here unless you are planning a large, multi-application deployment — in that
case, see [Launch a large deployment](how-to-deploy-large).

Use the tabs below to switch between the two substrates — your selection is
remembered as you scroll through the rest of the page.

If you are a beginner to OpenSearch or Juju and are looking for a more comprehensive
walkthrough of these steps, refer instead to the [Tutorial](tutorial-index).

## Prerequisites

**Juju `v.3.5.3+`**: Install Juju, but do not bootstrap anything yet.

```{note}
See also: [How to install Juju](https://documentation.ubuntu.com/juju/3.6/howto/manage-juju/#install-juju).
```

`````{tab-set}
:sync-group: substrate

````{tab-item} VM
:sync: vm

**LXD `v6.1+`**: Install and initialize
[LXD](https://ubuntu.com/server/docs/lxd-containers), Canonical's lightweight container
hypervisor.

```{note}
See also: [First steps with LXD](https://documentation.ubuntu.com/lxd/latest/tutorial/first_steps/#install-lxd-using-snap).
```
````

````{tab-item} K8s
:sync: k8s

**A Kubernetes cluster**: Any CNCF-conformant Kubernetes `v1.29+` cluster works.
For local development, [Canonical Kubernetes](https://documentation.ubuntu.com/canonical-kubernetes/latest/)
or [MicroK8s](https://microk8s.io/docs/getting-started) are the most convenient options.

The cluster must provide:

* A **default storage class** with dynamic provisioning, for the charm's
  `opensearch-data` and `opensearch-logs` volumes.
* A **load balancer** or another way of exposing services, if you intend to reach the
  cluster from outside Kubernetes.

```{note}
On Canonical Kubernetes, enable the `local-storage` and `load-balancer` features.
On MicroK8s, enable the `hostpath-storage`, `dns`, and `metallb` add-ons.
```
````

`````

**System requirements**: Check that you fulfill the rest of the software and hardware
requirements in the [system requirements page](reference-system-requirements).

## Prepare the substrate

`````{tab-set}
:sync-group: substrate

````{tab-item} VM
:sync: vm

### Disable IPv6 on LXD

Juju does not support IPv6 addresses with LXD. To set the network bridge to have no IPv6
addresses, run the following command after initializing LXD:

```shell
lxc network set lxdbr0 ipv6.address none
```

See [The LXD cloud and Juju](https://canonical.com/juju/docs/juju-cli/3.6/reference/cloud/list-of-supported-clouds/lxd/#constraints)
for more information.
````

````{tab-item} K8s
:sync: k8s

### Check the storage class

Charmed OpenSearch K8s requests two persistent volumes per unit. Confirm that your
cluster has a default storage class that can satisfy them:

```shell
kubectl get storageclass
```

At least one entry must be marked as `(default)`. If none is, either mark one as default
or pass an explicit storage class at deploy time (see [Deploy OpenSearch](#deploy-opensearch)).
````

`````

## Kernel parameter configuration

OpenSearch relies on a number of kernel parameters that are not set to suitable values by
default. How and where you apply them depends on the substrate.

````{note}
The following instructions modify kernel parameters. You can later reset them either
manually or by rebooting.

To take note of the original values, run

```shell
sudo sysctl -a | grep -E 'swappiness|max_map_count|tcp_retries2|file-max'
```
````

`````{tab-set}
:sync-group: substrate

````{tab-item} VM
:sync: vm

Before bootstrapping Juju controllers, the sysconfigs required by OpenSearch must be
enforced. This entails modifying some kernel parameters on the host machine, and creating
a configuration file to apply the same configuration in any new container that gets
deployed.

### Configure sysctl on the host machine

On the **host** machine, add the settings below to a config file:

```shell
sudo tee /etc/sysctl.d/opensearch.conf <<EOF
vm.swappiness = 0
vm.max_map_count = 262144
net.ipv4.tcp_retries2 = 5
EOF
```

Then, apply the new settings:

```shell
sudo sysctl -p /etc/sysctl.d/opensearch.conf
```

### Configure sysctl for new containers

Configure `cloud-init` to set sysctl on each new container that gets deployed.

First, add the configurations to a `cloud-init` user data file:

```shell
cat <<EOF > cloudinit-userdata.yaml
cloudinit-userdata: |
  postruncmd:
    - [ 'echo', 'vm.max_map_count=262144', '>>', '/etc/sysctl.conf' ]
    - [ 'echo', 'vm.swappiness=0', '>>', '/etc/sysctl.conf' ]
    - [ 'echo', 'net.ipv4.tcp_retries2=5', '>>', '/etc/sysctl.conf' ]
    - [ 'echo', 'fs.file-max=1048576', '>>', '/etc/sysctl.conf' ]
    - [ 'sysctl', '-p' ]
EOF
```

Now, there are two options to apply this `cloud-init` configuration:
set it as the default config to be used by every new model created after that,
or set it as a config for a target model.

To set the `cloud-init` script above as **default for all models**, use the
[`model-defaults`](https://juju.is/docs/juju/juju-model-defaults) command:

```shell
juju model-defaults --file=./cloudinit-userdata.yaml
```

To set the `cloud-init` script **for a particular model**, use the
[`model-config`](https://juju.is/docs/juju/juju-model-config) command:

```shell
juju model-config --file=./cloudinit-userdata.yaml --model <model_name>
```
````

````{tab-item} K8s
:sync: k8s

On Kubernetes, kernel parameters are applied per **worker node**, not per container:
`vm.max_map_count`, `vm.swappiness`, and `fs.file-max` are node-wide settings that the
workload pods inherit from the host they are scheduled on.

### Configure sysctl on each Kubernetes node

On **each node** that may run OpenSearch pods, add the settings below to a config file:

```shell
sudo tee /etc/sysctl.d/opensearch.conf <<EOF
vm.swappiness = 0
vm.max_map_count = 262144
fs.file-max = 1048576
EOF
```

Then, apply the new settings:

```shell
sudo sysctl -p /etc/sysctl.d/opensearch.conf
```

```{note}
If your nodes are managed by a cloud provider, prefer the provider's node configuration
mechanism (for example, a node bootstrap script or a machine image) so the settings
survive node replacement.
```

### Configure `net.ipv4.tcp_retries2`

Unlike the parameters above, `net.ipv4.tcp_retries2` is scoped to the **pod's network
namespace** rather than the host, so setting it on the node has no effect on the
workload.

To configure it, deploy the
[`data-platform-k8s-mutator`](https://github.com/canonical/data-platform-k8s-mutator)
charm alongside OpenSearch in the same Kubernetes cluster. The mutator applies the
required sysctl value to the OpenSearch workload pods.

```{note}
This step is optional but recommended for production deployments. Without it, OpenSearch
nodes take longer to detect and recover from network partitions.
```

<!-- TODO: Add the concrete `juju deploy data-platform-k8s-mutator` invocation and any
required configuration options once the charm's interface is finalised. -->
````

`````

## Bootstrap a Juju controller

`````{tab-set}
:sync-group: substrate

````{tab-item} VM
:sync: vm

```shell
juju bootstrap localhost <controller_name>
```

Make sure that the controller's back-end cloud is **not** Kubernetes-based.
````

````{tab-item} K8s
:sync: k8s

Register your Kubernetes cluster with Juju, then bootstrap a controller on it:

```shell
juju bootstrap <k8s_cloud_name> <controller_name>
```

Where `<k8s_cloud_name>` is the name of the Kubernetes cloud known to Juju, for example
`k8s` for Canonical Kubernetes or `microk8s` for MicroK8s.

Make sure that the controller's back-end cloud **is** Kubernetes-based.

```{note}
See also: [How to manage clouds](https://documentation.ubuntu.com/juju/latest/howto/manage-clouds/index.html)
in the Juju documentation.
```
````

`````

You can verify the cloud type of your controllers with:

```shell
juju list-controllers
```

## Create a model

Create a model if you haven't already:

```shell
juju add-model <model_name>
```

Check that the model is of the expected type:

```shell
juju show-model | yq '.[].type'
```

`````{tab-set}
:sync-group: substrate

````{tab-item} VM
:sync: vm

The type must **not** be `caas`.
````

````{tab-item} K8s
:sync: k8s

The type must be `caas`.
````

`````

(deploy-opensearch)=
## Deploy OpenSearch

`````{tab-set}
:sync-group: substrate

````{tab-item} VM
:sync: vm

In a single host deployment with LXD, we recommend using the `testing`
[profile](how-to-optimize-cluster-performance), which will only consume 1G RAM per
container. This is the default.

To deploy OpenSearch, run

```shell
juju deploy opensearch --channel 2/stable -n 3
```

For production deployments, set the `production` profile:

```shell
juju deploy opensearch --channel 2/stable -n 3 --config profile=production
```
````

````{tab-item} K8s
:sync: k8s

The Kubernetes charm requires the `--trust` flag, which grants it the permissions it needs
to manage Kubernetes resources such as Services and StatefulSets on your behalf.

To deploy OpenSearch, run

```shell
juju deploy opensearch-k8s --channel 2/edge -n 3 --trust
```

If your cluster has no default storage class, or you want to pin the charm to a specific
one, pass the storage constraints explicitly:

```shell
juju deploy opensearch-k8s --channel 2/edge -n 3 --trust \
  --storage opensearch-data=<storage_class>,10G \
  --storage opensearch-logs=<storage_class>,2G
```

As with the VM charm, the default
[profile](how-to-optimize-cluster-performance) is `testing`. For production deployments,
set the `production` profile:

```shell
juju deploy opensearch-k8s --channel 2/edge -n 3 --trust --config profile=production
```

```{note}
The charm pulls the OpenSearch workload from an OCI image resource
(`opensearch-image`) rather than from a snap. The image is published together with the
charm revision, so no additional resource needs to be specified at deploy time.
```
````

`````

## Enable TLS

Charmed OpenSearch requires TLS to start, on both the HTTP and Transport layers.
Deploy a TLS certificates provider and integrate it with OpenSearch:

`````{tab-set}
:sync-group: substrate

````{tab-item} VM
:sync: vm

```shell
juju deploy self-signed-certificates --channel 1/stable
juju integrate self-signed-certificates opensearch
```
````

````{tab-item} K8s
:sync: k8s

```shell
juju deploy self-signed-certificates --channel 1/stable
juju integrate self-signed-certificates opensearch-k8s
```
````

`````

```{caution}
Self-signed certificates are not recommended for production. See
[How to enable TLS encryption](how-to-enable-tls-encryption) for the list of supported
certificate providers.
```

## Check the deployment

```shell
juju status --watch 1s
```

The deployment is complete once all units show `active` and `idle` statuses.

## Next steps

* [Launch a large deployment](how-to-deploy-large)
* [Integrate with an application](how-to-integrate-with-an-application)
* [Scale horizontally](how-to-scale-horizontally)
* [Enable monitoring](how-to-monitoring)
