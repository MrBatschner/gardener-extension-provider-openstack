# Using the OpenStack provider extension with Gardener as end-user

The [`core.gardener.cloud/v1beta1.Shoot` resource](https://github.com/gardener/gardener/blob/master/example/90-shoot.yaml) declares a few fields that are meant to contain provider-specific configuration.

In this document we are describing how this configuration looks like for OpenStack and provide an example `Shoot` manifest with minimal configuration that you can use to create an OpenStack cluster (modulo the landscape-specific information like cloud profile names, secret binding names, etc.).

## Provider Secret Data

Every shoot cluster references a `SecretBinding` which itself references a `Secret`, and this `Secret` contains the provider credentials of your OpenStack tenant.
This `Secret` must look as follows:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: core-openstack
  namespace: garden-dev
type: Opaque
data:
  domainName: base64(domain-name)
  tenantName: base64(tenant-name)
  username: base64(user-name)
  password: base64(password)
```

Please look up https://docs.openstack.org/keystone/pike/admin/identity-concepts.html as well.

⚠️ Depending on your API usage it can be problematic to reuse the same provider credentials for different Shoot clusters due to rate limits.
Please consider spreading your Shoots over multiple credentials from different tenants if you are hitting those limits.

## `InfrastructureConfig`

The infrastructure configuration mainly describes how the network layout looks like in order to create the shoot worker nodes in a later step, thus, prepares everything relevant to create VMs, load balancers, volumes, etc.

An example `InfrastructureConfig` for the OpenStack extension looks as follows:

```yaml
apiVersion: openstack.provider.extensions.gardener.cloud/v1alpha1
kind: InfrastructureConfig
floatingPoolName: MY-FLOATING-POOL
# floatingPoolSubnetName: my-floating-pool-subnet-name
networks:
# id: 12345678-abcd-efef-08af-0123456789ab
  workers: 10.250.0.0/19
# router:
#   id: 1234
```

The `floatingPoolName` is the name of the floating pool you want to use for your shoot.
If you don't know which floating pools are available look it up in the respective `CloudProfile`.

With `floatingPoolSubnetName` you can explicitly define to which subnet in the floating pool network (defined via `floatingPoolName`) the router should be attached to.

If `networks.id` is an optional field. If it is given, you can specify the uuid of an existing Neutron network (created manually, by other tooling, ...) that should be reused. A new subnet for the Shoot will get created in it get created.
**Warning:** All subnets in the given network must have a router. A subnet without router may cause all subnets in the network to loose connectivity.

The `networks.router` section describes whether you want to create the shoot cluster in an already existing router or whether to create a new one:

* If `networks.router.id` is given then you have to specify the router id of the existing router that was created by other means (manually, other tooling, ...).
If you want to get a fresh router for the shoot then just omit the `networks.router` field.

The `networks.workers` section describes the CIDR for a subnet that is used for all shoot worker nodes, i.e., VMs which later run your applications.

You can freely choose these CIDRs and it is your responsibility to properly design the network layout to suit your needs.

Apart from the router and the worker subnet the OpenStack extension will also create a network, router interfaces, security groups, and a key pair.

## `ControlPlaneConfig`

The control plane configuration mainly contains values for the OpenStack-specific control plane components.
Today, the only component deployed by the OpenStack extension is the `cloud-controller-manager`.

An example `ControlPlaneConfig` for the OpenStack extension looks as follows:

```yaml
apiVersion: openstack.provider.extensions.gardener.cloud/v1alpha1
kind: ControlPlaneConfig
loadBalancerProvider: haproxy
cloudControllerManager:
  featureGates:
    CustomResourceValidation: true
```

The `loadBalancerProvider` is the provider name you want to use for load balancers in your shoot.
If you don't know which types are available look it up in the respective `CloudProfile`.

The `cloudControllerManager.featureGates` contains a map of explicitly enabled or disabled feature gates.
For production usage it's not recommended to use this field at all as you can enable alpha features or disable beta/stable features, potentially impacting the cluster stability.
If you don't want to configure anything for the `cloudControllerManager` simply omit the key in the YAML specification.

## `WorkerConfig`

Each worker group in a shoot may contain provider specific configurations and options. These are contained in the `providerConfig` section of a worker group.
An example of a `WorkerConfig` looks as follows:

```yaml
apiVersion: openstack.provider.extensions.gardener.cloud/v1alpha1
kind: WorkerConfig
serverGroup:
  policy: soft-anti-affinity
```

When you specify the `serverGroup` section in your worker group configuration, a new server group will be created with the configured policy and all machines managed by this worker group will be assigned as members of the created server group.
For users to have access to this feature, it must be enabled on the `CloudProfile` by your operator.

Existing shoots that want to use this feature, have to create **new** worker groups with a server group configuration and migrate their workloads on the new nodes. Updating existing worker groups with server group configuration is currently prohibited.

Please note the following restrictions when deploying workers with server groups:
+ The `serverGroup` and `serverGroup.policy` fields are immutable upon creation. Users are not allowed to change the policy value and neither add or remove a `serverGroup` section from a worker group after it has been created.
+ The `serverGroup` section is optional, but if it is included in the worker configuration, it must contain a valid policy value.
+ The available `policy` values that can be used, are defined in the provider specific section of `CloudProfile`, by your operator.
+ Certain policy values may induce further constraints. Using the `affinity` policy is only allowed when the worker group utilizes a single zone.

## Example `Shoot` manifest (one availability zone)

Please find below an example `Shoot` manifest for one availability zone:

```yaml
apiVersion: core.gardener.cloud/v1alpha1
kind: Shoot
metadata:
  name: johndoe-openstack
  namespace: garden-dev
spec:
  cloudProfileName: openstack
  region: europe-1
  secretBindingName: core-openstack
  provider:
    type: openstack
    infrastructureConfig:
      apiVersion: openstack.provider.extensions.gardener.cloud/v1alpha1
      kind: InfrastructureConfig
      floatingPoolName: MY-FLOATING-POOL
      networks:
        workers: 10.250.0.0/19
    controlPlaneConfig:
      apiVersion: openstack.provider.extensions.gardener.cloud/v1alpha1
      kind: ControlPlaneConfig
      loadBalancerProvider: haproxy
    workers:
    - name: worker-xoluy
      machine:
        type: medium_4_8
      minimum: 2
      maximum: 2
      zones:
      - europe-1a
  networking:
    nodes: 10.250.0.0/16
    type: calico
  kubernetes:
    version: 1.16.1
  maintenance:
    autoUpdate:
      kubernetesVersion: true
      machineImageVersion: true
  addons:
    kubernetes-dashboard:
      enabled: true
    nginx-ingress:
      enabled: true
```

## CSI volume provisioners

Every OpenStack shoot cluster that has at least Kubernetes v1.19 will be deployed with the OpenStack Cinder CSI driver.
It is compatible with the legacy in-tree volume provisioner that was deprecated by the Kubernetes community and will be removed in future versions of Kubernetes.
End-users might want to update their custom `StorageClass`es to the new `cinder.csi.openstack.org` provisioner.
Shoot clusters with Kubernetes v1.18 or less will use the in-tree `kubernetes.io/cinder` volume provisioner in the kube-controller-manager and the kubelet.
