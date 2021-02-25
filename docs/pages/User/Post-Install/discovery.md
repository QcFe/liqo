---
title: Add a new cluster to Liqo
weight: 3
---


Once Liqo is installed in your cluster, you can start establishing new *peerings*.
Specifically, you can rely on three different methods to peer with other clusters:

1. **LAN Discovery**: automatic discovery of neighboring clusters available in the same LAN. This looks similar to the automatic discovery of Wi-Fi hotspots, and it is particularly suitable when your cluster is composed of a single node (e.g., in a combination with [K3s](https://k3s.io)).
2. **DNS Discovery**: automatic discovery of the clusters associated with a specific DNS domain (e.g.; *liqo.io*). This is achieved by quering specific DNS entries. This looks similar to the discovery of voice-over-IP SIP servers and it is mostly oriented to big organizations that wish to adopt Liqo in production.
3. **Manual Configuration**: manual addition of specific clusters to the list of known ones. This method is particularly appropriate outside LAN, without requiring any DNS configuration.

## LAN Discovery

Liqo is able to automatically discover any available clusters running on the same L2 Broadcast Domain, as well as to make your cluster discoverable by others.

Using kubectl, you can also manually obtain the list of discovered foreign clusters:

```bash
kubectl get foreignclusters
NAME                                   AGE
ff5aa14a-dd6e-4fd2-80fe-eaecb827cf55   101m
```
To check whether Liqo is configured to automatically attempt to peer with the foreign cluster, you can check the join property of the specific ForeignCluster resource:

```bash
kubectl get foreignclusters ${FOREIGN_CLUSTER} --template={{.spec.join}}
true
```

At any moment, you can enable or disable the peering with a specific cluster setting its join flag accordingly.

You can enable the peering patching this value with:
```bash
kubectl patch foreignclusters "$foreignClusterName" \
  --patch '{"spec":{"join":true}}' \
  --type 'merge'
```

You can disable the peering patching this value with:
```bash
kubectl patch foreignclusters "$foreignClusterName" \
  --patch '{"spec":{"join":false}}' \
  --type 'merge'
```

### Enable and Disable the discovery on LAN

The discovery on LAN can be enabled and disabled updating the flags in the ClusterConfig CR. Lan Discovery can be disabled to avoid unwanted peering with neighbors.

You can enable discovery, by patching:
```bash
kubectl patch clusterconfigs liqo-configuration \
  --patch '{"spec":{"discoveryConfig":{"enableDiscovery": true, "enableAdvertisement": true}}}' \
  --type 'merge'
```

Or disabling it with:
```bash
kubectl patch clusterconfigs liqo-configuration \
  --patch '{"spec":{"discoveryConfig":{"enableDiscovery": false, "enableAdvertisement": false}}}' \
  --type 'merge'
```

## DNS Discovery

The DNS discovery procedure requires two orthogonal actions to be enabled:

1. Register your cluster into your DNS server to make it discoverable by others (the required parameters are available in the section below).
2. Connect to a foreign cluster, specifying the remote domain.

### Register the home cluster

To allow the other clusters to peer with your cluster(s), you need to register a set of DNS records that specify the cluster(s) available in your domain, with the different parameters required to establish the connection.

In a scenario where we have to manage the discovery of multiple clusters, it can be useful to manage the entire set updating it in a unique place.
We only have to know how the Authentication Service is reachable from the external world.

#### Get the Required Values

The first required value is or the __hostname__ or the __IP address__ where it is reachable.
If you specified a name during the installation, it will be reachable with an Ingress (you can get it with `kubectl get ingress -n liqo`),
if not it is exposed with a NodePort Service, so you can get one if the IPs of the nodes of your cluster (`kubectl get nodes -o wide`).

The second required value is the __port__ where it is reachable.
If you are using an Ingress it should be reachable at port `443`. Otherwise if you are using a NodePort Service you can get the port executing
`kubectl get service -n liqo auth-service`, an output example could be:

```txt
NAME           TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
auth-service   NodePort   10.81.20.99   <none>        443:30740/TCP   2m7s
```
where "30740" is the port where the service is listening and can be contacted outside the cluster.

#### DNS Configuration

Now, it is possible to configure the records necessary to enable the DNS discovery process.
In the following example, we present a `bind9`-like configuration for a hypothetical domain `example.com`. It exposes two Liqo-enabled cluster named `liqo-cluster` and `liqo-cluster-2`. The first one exposes the Auth Service at `1.2.3.4:443`, while the second at `2.3.4.1:8443`.

```txt
example.com.                  PTR     liqo-cluster.example.com.
                                      liqo-cluster-2.example.com.

liqo-cluster.example.com.     SRV     0 0 443 auth.server.example.com.
liqo-cluster-2.example.com.   SRV     0 0 8443 auth.server-2.example.com.

auth.server.example.com.      A       1.2.3.4
auth.server-2.example.com.    A       2.3.4.1
```

Remember to adapt the configuration according to your setup, modyfing the urls, ips and ports accordingly.

{{%expand "Expand here to know more about the meaning of each record." %}}

* the `PTR` record lists the Liqo clusters exposed for the specific domain (e.g. `liqo-cluster`).
* the `SRV` record specifies the network parameters needed to connect to the Auth Service of the cluster. You should have a record for each cluster present in the `PTR` record.
  Specifically, it has the following format:
  ```txt
   <cluster-name>._liqo._tcp.<domain>. SRV <priority> <weight> <auth-server-port> <auth-server-name>.
  ```
  where the priority and weight fields are unused and should be set to zero. In this case, the API server is reachable at the address `liqo-cluster-api.server.example.com` through port `6443`.
* The `A` record assigns an IP address to the DNS name of the Auth Service server ( `1.2.3.4` in the above example).

{{% /expand %}}

### Connect to a remote cluster

To leverage the DNS discovery to peer to a remote cluster, it is necessary to specify the remote domain called `SearchDomain`.
For any `SearchDomain`, you need to configure the following parameters:

1. **Domain**: the domain where the cluster you want to peer with is located.
2. **Name**: a mnemonic name to identify the domain.
3. **Join**: to specify whether to automatically trigger the peering procedure.

Using kubectl, it is also possible to perform the following configuration. A `SearchDomain` for the `example.com` domain, may be look like:

```
cat << "EOF" | kubectl apply -f
apiVersion: discovery.liqo.io/v1alpha1
kind: SearchDomain
metadata:
  name: example.com
spec:
  domain: example.com
  autojoin: true
EOF
```

## Manual Configuration

### Forging the ForeignCluster

In Liqo, remote clusters are defined as `ForeignClusters`
To add a new Foreign Cluster, the Auth Service URL is the only required value to make this cluster peerable from the external world.

To get the address and the port where the Authentication Service is running, see the [Get the Required Values](#get-the-required-values) section.

An example of this resource can be:

```yaml
apiVersion: discovery.liqo.io/v1alpha1
kind: ForeignCluster
metadata:
  name: my-cluster
spec:
  join: true # optional (defaults to false)
  authUrl: "https://<ADDRESS>:<PORT>"
```

When you create the ForeignCluster, the Liqo control plane will contact the `authURL` (i.e. the public URL of a cluster authentication server) to retrieve all the required cluster information.

#### Access the cluster configurations

You can get the cluster configurations exposed by the Auth Service endpoint of the other cluster. This allows retrieving the information necessary to peer with the remote cluster.

```bash
curl --insecure https://<ADDRESS>:<PORT>/ids
```

```json
{"clusterId":"0558de48-097b-4b7d-ba04-6bd2a0f9d24f","clusterName":"LiqoCluster0692","guestNamespace":"liqo"}
```