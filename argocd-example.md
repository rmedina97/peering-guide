# ArgoCD Setup

[ArgoCD](https://argo-cd.readthedocs.io) can be used to deploy and synchronize the various Custom Resources (CRs) required for establishing a Liqo peering. This guide outlines the procedure to deploy a unidirectional peering between two clusters — a consumer and a provider — using ArgoCD installed on the consumer cluster. Both clusters must be registered and accessible from the consumer ArgoCD server in order to deploy the necessary CRs.

As with the general guide, this document is structured around the three Liqo modules required to establish a unidirectional peering:

- [Networking](#net)
- [Authentication](#auth)
- [Offloading](#offloading)
- [Unpeer](#unpeer)

## <a id="net"></a>Declarative Network Configuration

 Sample ArgoCD applications for this phase are available in the [argocd-app folder](link alle folder). These applications deploy the CRs located in the net-local (consumer cluster) and net-remote (provider cluster) directories. The CRs provided include randomly generated example data (e.g., WireGuard keys); such data must be adapted to fit the specific target environment. The following section summarizes the parameters that may be customized arbitrarily and those that are dependent on the underlying infrastructure.

### Parameters

The CRs required for the networking phase include several parameters, as also described in the networking section of the [main guide](buttaci la guida primaria). These parameters fall into two categories:

Parameters that may be freely chosen but must remain consistent throughout the CRs:

1. Tenant namespace name
2. Wireguard keys
3. Resources names

Parameters that are constrained by external parameter or by the network setup:

1. Cluster CIDRs
2. Type of Gateway Server (e.g., NodePort, LoadBalancer)
3. Address of the Gateway Server to be referenced by the Gateway Client

In the provided example, a NodePort Gateway Server is used, assuming that both clusters reside within the same network. The Gateway Client then references the IP address of the node hosting the Gateway Server.

>**Note** 
ArgoCD enforces stricter validation rules on CRs compared to manual deployment. In particular, the `Configuration` CR, being namespace-scoped, must be created within a namespace. In thsi guide it is placed in the tenant namespace for simplicity, since it is related to the peering process.

### Deployment

Although the networking phase will only complete successfully once all CRs are applied to their respective clusters, there is no required order of deployment. The peering will become fully operational even if local CRs are applied first and remote CRs are added at a later time.

If the setup is completed correctly, running the `liqoctl info` command on one of the clusters will return output similar to the following:

```bash
┌─ Active peerings ────────────────────────────────────────────────────────────────┐
|  consumer                                                                        |
|      Role:                  Unknown                                              |
|      Networking status:     Healthy                                              |
|      Authentication status: Disabled                                             |
|      Offloading status:     Disabled                                             |
└──────────────────────────────────────────────────────────────────────────────────┘
```