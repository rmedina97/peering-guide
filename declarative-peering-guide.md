
# Declarative peering

Declarative peering is supported starting from Liqo 1.0. This allows the creation of a set of Custom Resources (CRs) that describe the peering with a remote cluster, which is automatically established once the CRs are applied on both clusters.
This approach simplifies automation, GitOps, and continuous delivery. For example, a Git repository may contain the manifests describing the peerings, while an instance of [ArgoCD](https://argo-cd.readthedocs.io) synchronizes these changes on the clusters, creating and destroying the peerings accordingly.

This document analyzes how to declaratively configure each of the Liqo modules for two clusters:

- Networking
- Authentication
- Offloading

>**Note** The labels on the templates provided in this guide are all mandatory.

## Tenant namespace

In both clusters, it is necessary to create a namespace, called the tenant namespace, which will contain all the custom resources (CRs).  
Each tenant namespace must refer to the peering with a specific cluster; therefore, a distinct tenant namespace must be created for each peering.  
Tenant namespaces **can have arbitrary names** and do not need to follow the Liqo pattern `liqo-tenant-xxx`.

The following is a basic template of a tenant namespace with the **mandatory labels**:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    liqo.io/remote-cluster-id: <REMOTE_CLUSTER_ID>
    liqo.io/tenant-namespace: "true"
  name: <LOCAL_TENANT_NAMESPACE>
spec: {}
```

Where:  
`LOCAL_TENANT_NAMESPACE` is the tenant namespace of the local cluster;  
`REMOTE_CLUSTER_ID` is the remote cluster’s ID, set during Liqo installation. It can be obtained by launching the following command on the **remote cluster**:

**liqoctl**

```bash
liqoctl info --get clusterid
```
**kubectl**

```bash
kubectl get configmaps -n liqo liqo-clusterid-configmap \
  --template {{.data.CLUSTER_ID}}
```

### Example of a Tenant Namespace (Consumer Cluster)

The following is an example of tenant namespace named `liqo-tenant-provider`on the consumer cluster, which refers to the peering with a cluster with id `provider`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    liqo.io/remote-cluster-id: provider
    liqo.io/tenant-namespace: "true"
  name: liqo-tenant-provider
spec: {}
```

### Example of a Tenant Namespace (Provider Cluster)

The following is an example of tenant namespace named `liqo-tenant-consumer` on the provider cluster, which refers to the peering with the cluster with id `consumer`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    liqo.io/remote-cluster-id: consumer
    liqo.io/tenant-namespace: "true"
  name: liqo-tenant-consumer
spec: {}
```

## Declarative network configuration

By default, the network connection between clusters is set up using a secure channel created with [Wireguard](https://www.wireguard.com/).  
Usually, one cluster (usually the provider) hosts the server gateway that exposes a UDP port, which the client gateway (usually on the consumer cluster) needs to reach.  
In this guide, **the client gateway will be configured on the consumer cluster, and the server gateway on the provider cluster**, since this is the most common setup.  
That said, since the network peering setup is independent from the offloading role of the clusters (consumer vs. provider), it’s possible to swap the client/server roles if that works better in a different environment.

### Creating and exchanging the network configurations (both clusters)

The two clusters that need to be connected require the network configuration of the other, which is provided via the `Configuration` CR, which basic template with the **mandatory labels** is the following:

```yaml
apiVersion: networking.liqo.io/v1beta1
kind: Configuration
metadata:
  labels:
    liqo.io/remote-cluster-id: <REMOTE_CLUSTER_ID>
  name: <RESOURCE_NAME>
spec:
  remote:
    cidr:
      external: <EXTERNAL_CIDR_REMOTE_CLUSTER>       
      pod: <POD_CIDR_REMOTE_CLUSTER>     
```
Where:  
`REMOTE_CLUSTER_ID` is the remote cluster’s ID, set during Liqo installation;  
`RESOURCE_NAME` is an arbitrary name assigned to the resource;  
`EXTERNAL_CIDR_REMOTE_CLUSTER` is the external CIDR of the remote cluster (set at Liqo install time);  
`POD_CIDR_REMOTE_CLUSTER` is the internal pod CIDR of the remote cluster (also set at Liqo install time);  

**This Configuration resource needs to be applied on both clusters** and **must include the pod and external CIDRs of the other cluster** (e.g., cluster A’s config includes cluster B’s CIDRs, and vice versa).

You can check the pod and external CIDRs configured on a cluster by running:

```bash
kubectl get networks.ipam.liqo.io -n liqo external-cidr -o=jsonpath={'.status.cidr'}
kubectl get networks.ipam.liqo.io -n liqo pod-cidr -o=jsonpath={'.status.cidr'}

# Sample output
10.70.0.0/16
10.42.0.0/16
```

### Creating and exchanging the Wireguard keys (both clusters)

In order to enable authentication and encryption, WireGuard requires a key pair (private and public) **for each gateway**.  
Those keys can be generated either [via the Wireguard utility tool](https://www.wireguard.com/quickstart/#key-generation) or with OpenSSL, as shown below:

```bash
openssl genpkey -algorithm X25519 -outform der -out private.der
openssl pkey -inform der -in private.der -pubout -outform der -out public.der

# Get the Wireguard private key
echo "Private key:"; cat private.der | tail -c 32 | base64
# Get the Wireguard public key
echo "Public key:"; cat public.der | tail -c 32 | base64
```

After generating the key pairs for each cluster (provider and consumer), a Secret resource must be created in each cluster to store its respective keys. The following template **includes the required labels**.

```yaml
apiVersion: v1
kind: Secret
metadata:
  labels:
    liqo.io/remote-cluster-id: <REMOTE_CLUSTER_ID>
    networking.liqo.io/gateway-resource: "true"
  name: <RESOURCE_NAME>
  namespace: <LOCAL_TENANT_NAMESPACE>
type: Opaque
data:
  privateKey: <WIREGUARD_PRIVATE_KEY>
  publicKey: <WIREGUARD_PUBLIC_KEY>
```

Where:  
`WIREGUARD_PRIVATE_KEY` and `WIREGUARD_PUBLIC_KEY` refer to the base64-encoded key pair generated before for the local cluster;  
`REMOTE_CLUSTER_ID` is the remote cluster’s ID, set during Liqo installation;  
`LOCAL_TENANT_NAMESPACE` is the tenant namespace of the local cluster;  
`RESOURCE_NAME` is an arbitrary name assigned to the resource;  

Additionally, each cluster must define a PublicKey resource **containing the public key of the remote cluster**, as shown in the template below, which **includes the required labels**:

```yaml
apiVersion: networking.liqo.io/v1beta1
kind: PublicKey
metadata:
  labels:
    liqo.io/remote-cluster-id: <REMOTE_CLUSTER_ID>
    networking.liqo.io/gateway-resource: "true"
  name: <RESOURCE_NAME>
  namespace: <LOCAL_TENANT_NAMESPACE>
spec:
  publicKey: <REMOTE_WIREGUARD_PUBLIC_KEY>
```

Where:  
`REMOTE_WIREGUARD_PUBLIC_KEY` is the public key of the remote cluster;  
`REMOTE_CLUSTER_ID` is the remote cluster’s ID, set during Liqo installation;  
`LOCAL_TENANT_NAMESPACE` is the tenant namespace of the local cluster;  
`RESOURCE_NAME` is an arbitrary name assigned to the resource;  

#### Example Resources to Be Applied (Provider Cluster)

The following example resources illustrate the configuration to be applied on the provider cluster

The first resource is a Secret containing the WireGuard key pair generated for the provider cluster itself:

```yaml
apiVersion: v1
kind: Secret
metadata:
  labels:
    liqo.io/remote-cluster-id: consumer
    networking.liqo.io/gateway-resource: "true"
  name: gw-keys
  namespace: liqo-tenant-consumer
type: Opaque
data:
  privateKey: IIDYUpVU5r/R3pxp7xnpEU31Ky2+Th7pSWQWyEKKVG8=
  publicKey: SU+Ba3jBNgofBrZtVHwad1VJotcVs4TmgpOdFQhx/UE=
```

The remote-cluster-id label must match the ID of the consumer cluster, while the namespace must refer to the local tenant namespace of the provider cluster.

The second resource is a PublicKey, which contains the public key of the consumer cluster:

```yaml
apiVersion: networking.liqo.io/v1beta1
kind: PublicKey
metadata:
  labels:
    liqo.io/remote-cluster-id: consumer
    networking.liqo.io/gateway-resource: "true"
  name: gw-publickey
  namespace: liqo-tenant-consumer
spec:
  publicKey: ahPstVhybRiIyxSPhZgfWkc5vMUaJTFHaCcTr2KEW2A=
```

### Configuring the Server Gateway (Provider Cluster)

By default, Liqo establishes inter-cluster communication through a WireGuard tunnel. This section describes the configuration of the gateway server, which acts as the endpoint to which the client gateway connects.

The `GatewayServer` custom resource defines the server-side configuration and must be applied to the cluster designated as the server in the tunnel setup.  
When defining the `GatewayServer` resource, it is essential to **specify the `secretRef` field**, referencing the previously generated WireGuard key pair.  
Under the `.spec.endpoint` field, it is possible to configure a fixed `nodePort` or `loadBalancerIP` (if supported by the infrastructure provider) to expose the gateway on a specific UDP port or IP address. This allows the client-side configuration to be determined in advance.  
The following is an example of a `GatewayServer` resource applied to the provider cluster. It exposes the WireGuard gateway using a `NodePort` service on port `30742`, with **the mandatory labels**:

```yaml
apiVersion: networking.liqo.io/v1beta1
kind: GatewayServer
metadata:
  labels:
    liqo.io/remote-cluster-id: <CONSUMER_CLUSTER_ID>
  name: server
  namespace: <PROVIDER_TENANT_NAMESPACE>
spec:
  endpoint:
    port: 51840
    serviceType: NodePort
    nodePort: 30742
  mtu: 1340
  secretRef:
    name: <WIREGUARD_KEYS_SECRET_NAME>
  serverTemplateRef:
    apiVersion: networking.liqo.io/v1beta1
    kind: WgGatewayServerTemplate
    name: wireguard-server
    namespace: liqo
```

Where:  
`CONSUMER_CLUSTER_ID` is the ID of the consumer cluster where the gateway client will run;  
`PROVIDER_TENANT_NAMESPACE` is the tenant namespace in the provider cluster where the gateway server will be deployed;  
`WIREGUARD_KEYS_SECRET_NAME` is the name of the secret containing the server’s WireGuard key pair;

### Configuring the Client Gateway (Consumer Cluster)

The client gateway will be configured on the consumer cluster to establish a connection with the provider's exposed WireGuard service.

The `GatewayClient` custom resource specifies the configuration parameters of the client gateway, including the endpoint information required to initiate the connection with the gateway server.

The following is an example of a `GatewayClient` resource, to be applied on the consumer cluster, with **the mandatory labels**:

```yaml
apiVersion: networking.liqo.io/v1beta1
kind: GatewayClient
metadata:
  creationTimestamp: null
  labels:
    liqo.io/remote-cluster-id: <PROVIDER_CLUSTER_ID>
  name: client
  namespace: <CONSUMER_TENANT_NAMESPACE>
spec:
  clientTemplateRef:
    apiVersion: networking.liqo.io/v1beta1
    kind: WgGatewayClientTemplate
    name: wireguard-client
    namespace: liqo
  secretRef:
    name: <WIREGUARD_KEYS_SECRET_NAME>
  endpoint:
    addresses:
    - <REMOTE_IP>
    port: 30742
    protocol: UDP
  mtu: 1340
```

Where:  
`PROVIDER_CLUSTER_ID` the ID of the provider cluster hosting the gateway server;  
`CONSUMER_TENANT_NAMESPACE` is the tenant namespace in the consumer cluster where the gateway client will be deployed;   
`WIREGUARD_KEYS_SECRET_NAME` is the name of the secret containing the client’s WireGuard key pair;  
`REMOTE_IP`: is the IP address of a node in the provider cluster (in the case of a `NodePort` service), or the load balancer IP/FQDN if a `LoadBalancer` service is used;

### Summary of Network Configuration

In order to establish network connectivity between clusters, the following resources must be defined and applied **in both clusters**:

- `Configuration` resource containing the network parameters of the peer cluster
- `Secret` resource holding the local WireGuard public and private keys
- `PublicKey` resource including the WireGuard public key of the peer cluster

Additionally:

- the provider cluster must define a `GatewayServer` resource
- the consumer cluster must define a `GatewayClient` resource to connect to the provider’s gateway server

Once all the required resources are in place, the gateway client should be able to connect to the server and establish the WireGuard tunnel.

The status of the connection can be verified by inspecting the corresponding `Connection` resource, as described [here](./inter-cluster-network.md#connection-crds).

## Declarative Configuration of Cluster Authentication

This section describes how to configure authentication between clusters, enabling the consumer cluster to request resources from the provider cluster.

When authentication is manually configured, **it is the responsibility of the user to provide credentials that grant the necessary permissions** for the consumer cluster to operate correctly.

For additional information regarding authentication mechanisms in Kubernetes, refer to [the official documentation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/).  
Guidance on how to issue a client certificate can be found [here](https://kubernetes.io/docs/tasks/tls/certificate-issue-client-csr/).

Refer to the [EKS documentation](https://docs.aws.amazon.com/eks/latest/userguide/cluster-auth.html) for details on how access control is handled in this environment.

>**Note** Client certificate-based authentication is [not directly supported on Amazon EKS](https://aws.amazon.com/blogs/containers/managing-access-to-amazon-elastic-kubernetes-service-clusters-with-x-509-certificates/).
Refer to the [EKS documentation](https://docs.aws.amazon.com/eks/latest/userguide/cluster-auth.html) for details on how access control is handled in this environment.

### Role Binding for the Consumer Cluster User (Provider Cluster)

After creating the credentials that the consumer cluster will use to authenticate with the provider, it is necessary to assign the minimal set of permissions required for the consumer to operate.  
It is important to note that **the consumer cluster does not create workloads directly on the provider cluster**. At this stage, the consumer only requires permission to create Liqo resources, such as `ResourceSlice` objects, which are then subject to approval by the provider.

To assign the appropriate permissions, the user associated with the consumer cluster must be bound to the `liqo-remote-controlpane` ClusterRole. This is accomplished by creating a `RoleBinding` custom resource within the tenant namespace of the provider cluster, as illustrated in the following example, which includes **the mandatory labels and role definition**:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    liqo.io/remote-cluster-id: <CONSUMER_CLUSTER_ID>
  name: liqo-binding-liqo-remote-controlplane
  namespace: <PROVIDER_TENANT_NAMESPACE>
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: liqo-remote-controlplane
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: <USER_COMMON_NAME>
```

Where:  
`USER_COMMON_NAME` corresponds to the CN (Common Name) field of the X.509 certificate used by the consumer cluster for authentication;
`CONSUMER_CLUSTER_ID` is the ID of the consumer cluster;  
`PROVIDER_TENANT_NAMESPACE` is the tenant namespace in the provider cluster;

### Tenant Resource Creation for the Consumer Cluster (Provider Cluster)

To enable authentication and resource negotiation from the consumer cluster, the provider cluster must define a `Tenant` resource. This resource is used to manage authorization policies of the remote consumer.  
For instance, if it is necessary to prevent the consumer cluster from negotiating additional resources, the tenantCondition field can be set to `Cordoned`, which halts any further resource negotiation.

In the context of declarative configuration, where no automatic handshake occurs between clusters, the Tenant must be explicitly configured to accept `ResourceSlice` objects from the consumer. This is achieved by setting the `authzPolicy` field to `TolerateNoHandshake`, as illustrated in the following example:

```yaml
apiVersion: authentication.liqo.io/v1beta1
kind: Tenant
metadata:
  labels:
    liqo.io/remote-cluster-id: <CONSUMER_CLUSTER_ID>
  name: my-tenant
  namespace: <PROVIDER_TENANT_NAMESPACE>
spec:
  clusterID: <CONSUMER_CLUSTER_ID>
  authzPolicy: TolerateNoHandshake
  tenantCondition: Active
```

Where:  
`CONSUMER_CLUSTER_ID` is the ID of the consumer cluster;  
`PROVIDER_TENANT_NAMESPACE` is the tenant namespace in the provider cluster;

### Adding the Credentials to the Consumer Cluster (Consumer Cluster)

The credentials previously created in the provider cluster must be made available to the consumer cluster. This is achieved by creating an identity `Secret` containing a kubeconfig file with the necessary credentials to authenticate and interact with the provider cluster.

An example of the identity Secret is provided below, with **the mandatory labels and annotations**:

```yaml
apiVersion: v1
data:
  kubeconfig: <BASE64_KUBECONFIG_USER>
kind: Secret
metadata:
  labels:
    liqo.io/identity-type: ControlPlane
    liqo.io/remote-cluster-id: <PROVIDER_CLUSTER_ID>
  annotations:
    liqo.io/remote-tenant-namespace: <PROVIDER_TENANT_NAMESPACE>
  name: cplane-secret
  namespace: <CONSUMER_TENANT_NAMESPACE>
```

Where:  
`PROVIDER_TENANT_NAMESPACE` is the tenant namespace in the provider cluster;  
`CONSUMER_TENANT_NAMESPACE` is the tenant namespace in the consumer cluster;  
`PROVIDER_CLUSTER_ID` is the ID of the provider cluster;  
`BASE64_KUBECONFIG_USER` is the kubeconfig file, encoded in base64, containing the credentials of the user previously created in the provider cluster;

Once this Secret is created, the `liqo-crd-replicator` component in the consumer cluster initiates the replication of resources and enables the creation of `ResourceSlice` resources targeting the provider cluster. This allows the consumer to begin negotiating resources with the provider.

The following log output from the `liqo-crd-replicator` pod confirms that the replication process has started correctly:

```text
 k logs -n liqo liqo-crd-replicator-68f9f55dfc-bc4l8 --tail 5

I1107 10:05:29.249872       1 crdReplicator-operator.go:94] Processing Secret "liqo-tenant-cl02/kubeconfig-controlplane-cl02"
I1107 10:05:29.254977       1 reflector.go:90] [cl02] Starting reflection towards remote cluster
I1107 10:05:29.255001       1 reflector.go:131] [cl02] Starting reflection of offloading.liqo.io/v1beta1, Resource=namespacemaps
I1107 10:05:29.255035       1 reflector.go:131] [cl02] Starting reflection of authentication.liqo.io/v1beta1, Resource=resourceslices
I1107 10:05:29.355741       1 reflector.go:163] [cl02] Reflection of authentication.liqo.io/v1beta1, Resource=resourceslices correctly started
```

### Summary of Authentication Configuration

To configure authentication, the following resources must be created:

**On the provider cluster:**

- set of credentials to be used by the consumer cluster
- RoleBinding resource binding the credentials to the `liqo-remote-controlplane` ClusterRole
- `Tenant` resource associated with the consumer cluster

**On the consumer cluster:**

- Secret resource containing the kubeconfig with the credentials required to access the provider cluster

This setup enables the consumer cluster to authenticate with the provider and initiate the resource negotiation process.

>**Note**: When inspecting the status of the Liqo consumer cluster, authentication will appear as disabled. This indicates that no user credentials have been configured for the provider cluster to authenticate against the consumer cluster, meaning that resource negotiation is unidirectional.  
To enable bidirectional resource negotiation, it is necessary to repeat the same authentication setup steps in the opposite direction, as creating credentials on the consumer cluster and pass them to the provider cluster.

## Declarative Configuration of Namespace Offloading

While offloading is independent from the network, which means that it is possible to negotiate resources and configure a namespace offloading without the inter-cluster network enabled, **a [working authentication configuration](#declarative-configuration-of-clusters-authentication) is a pre-requisite to enable offloading**.

### Requesting Resources: Configure a `ResourceSlice` (Consumer Cluster)

The `ResourceSlice` custom resource defines the computational resources requested by the consumer cluster from a remote provider cluster. It must be created in the tenant namespace of the consumer cluster and is automatically propagated to the provider cluster, which may accept or reject the request.

The following is an example of a `ResourceSlice` resource:

```yaml
apiVersion: authentication.liqo.io/v1beta1
kind: ResourceSlice
metadata:
  annotations:
    liqo.io/create-virtual-node: "true"
  creationTimestamp: null
  labels:
    liqo.io/remote-cluster-id: <PROVIDER_CLUSTER_ID>
    liqo.io/remoteID: <PROVIDER_CLUSTER_ID>
    liqo.io/replication: "true"
  name: <VIRTUAL_NODE_NAME>
  namespace: <CONSUMER_TENANT_NAMESPACE>
spec:
  class: default
  providerClusterID: <PROVIDER_CLUSTER_ID>
  resources:
    cpu: 20
    memory: 128Gi
```

Where:  
`PROVIDER_CLUSTER_ID` the ID of the provider cluster hosting the gateway server;    
`CONSUMER_TENANT_NAMESPACE` is the tenant namespace in the consumer cluster;  
`VIRTUAL_NODE_NAME` is the name that will be assigned to the virtual node representing the provider cluster within the consumer cluster;

If the provider cluster accepts the request, a virtual node will be created on the consumer cluster, exposing the requested resources and acting as a proxy for the provider cluster.

For additional details on the `ResourceSlice` and `VirtualNode` resources, refer to [the dedicated documentation section](./offloading-in-depth.md#create-resourceslice).

### Enabling Offloading and Remote Availability of Kubernetes Resources

By default, virtual nodes **are not eligible targets for workload scheduling unless offloading is explicitly enabled in the namespace from which the workload is deployed**. This is achieved by creating a `NamespaceOffloading` resource, as shown below:

```yaml
apiVersion: offloading.liqo.io/v1beta1
kind: NamespaceOffloading
metadata:
  name: offloading
  namespace: <NAMESPACE_NAME_TO_OFFLOAD>
spec:
  clusterSelector:
    nodeSelectorTerms: []
  namespaceMappingStrategy: DefaultName
  podOffloadingStrategy: LocalAndRemote
```

Where:  
`NAMESPACE_NAME_TO_OFFLOAD` is the name of the namespace to offload;

This resource must be created in the namespace intended to be offloaded to remote clusters.

For instance, the configuration above enables the offloading of the chosen namespace to all available provider clusters. The `podOffloadingStrategy: LocalAndRemote` policy allows pods in the namespace to be scheduled either locally or remotely, depending on the Kubernetes scheduler’s decisions (e.g., a remote virtual node with more available resources may be favored over a saturated local node).

>**Note**:When inspecting the status of the Liqo provider cluster, offloading is reported as disabled. This behavior is expected and indicates that the provider cluster is not configured to offload its workloads to the consumer cluster.  
To enable bidirectional offloading, using different namespaces on both clusters, the same offloading configuration steps must be performed in the opposite direction, from the provider to the consumer cluster.

Refer to [the namespace offloading documentation](../../usage/namespace-offloading.md#namespace-mapping-strategy) for more in-depth explanations.

>**Note** Currently, the `NamespaceOffloading` resource **must be created before scheduling any pod** intended to run on a remote cluster.  
If a pod is created before the namespace has been offloaded, it will remain indefinitely in the `Pending` state—even after the offloading configuration is applied.  
**It is therefore crucial to offload the namespace first**, before initiating pod scheduling.