---
url: https://istio.io/latest/docs/ambient/install/multicluster/multi-primary_multi-network
scrapeDate: 2026-06-18T07:10:31.937Z
library: istio


---

1.  [Documentation](_latest_docs_.md)
2.  [Ambient Mode](_latest_docs_ambient_.md)
3.  [Install](_latest_docs_ambient_install_.md)
4.  [Install Multicluster](_latest_docs_ambient_install_multicluster_.md)
5.  Install ambient multi-primary on different networks

* * *

1.  [Set the default network for `cluster1`](_latest_docs_ambient_install_multicluster_multi-primary_multi-network_.md#set-the-default-network-for-cluster1)
2.  [Configure `cluster1` as a primary](_latest_docs_ambient_install_multicluster_multi-primary_multi-network_.md#configure-cluster1-as-a-primary)
3.  [Install an ambient east-west gateway in `cluster1`](_latest_docs_ambient_install_multicluster_multi-primary_multi-network_.md#install-an-ambient-east-west-gateway-in-cluster1)
4.  [Set the default network for `cluster2`](_latest_docs_ambient_install_multicluster_multi-primary_multi-network_.md#set-the-default-network-for-cluster2)
5.  [Configure cluster2 as a primary](_latest_docs_ambient_install_multicluster_multi-primary_multi-network_.md#configure-cluster2-as-a-primary)
6.  [Install an ambient east-west gateway in `cluster2`](_latest_docs_ambient_install_multicluster_multi-primary_multi-network_.md#install-an-ambient-east-west-gateway-in-cluster2)
7.  [Enable Endpoint Discovery](_latest_docs_ambient_install_multicluster_multi-primary_multi-network_.md#enable-endpoint-discovery)
8.  [Next Steps](_latest_docs_ambient_install_multicluster_multi-primary_multi-network_.md#next-steps)
9.  [Cleanup](_latest_docs_ambient_install_multicluster_multi-primary_multi-network_.md#cleanup)
10.  [See also](_latest_docs_ambient_install_multicluster_multi-primary_multi-network_.md#see-also)

* * *

Follow this guide to install the Istio control plane on both `cluster1` and `cluster2`, making each a primary cluster (this is currently the only supported configuration in ambient mode). Cluster `cluster1` is on the `network1` network, while `cluster2` is on the `network2` network. This means there is no direct connectivity between pods across cluster boundaries.

Before proceeding, be sure to complete the steps under [before you begin](_latest_docs_ambient_install_multicluster_before-you-begin_.md).

In this configuration, both `cluster1` and `cluster2` observe the API Servers in each cluster for endpoints.

Service workloads across cluster boundaries communicate indirectly, via dedicated gateways for [east-west](https://en.wikipedia.org/wiki/East-west_traffic) traffic. The gateway in each cluster must be reachable from the other cluster.

[![Multiple primary clusters on separate networks](https://istio.io/latest/docs/ambient/install/multicluster/multi-primary_multi-network/arch.svg)](_latest_docs_ambient_install_multicluster_multi-primary_multi-network_arch.svg.md)

Multiple primary clusters on separate networks

## Set the default network for `cluster1`

If the istio-system namespace is already created, we need to set the cluster’s network there:
```
$ kubectl --context="${CTX_CLUSTER1}" label namespace istio-system topology.istio.io/network=network1
```
## Configure `cluster1` as a primary

Create the `istioctl` configuration for `cluster1`:

Install Istio as primary in `cluster1` using istioctl and the `IstioOperator` API.
```
$ cat <<EOF > cluster1.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: ambient
  components:
    pilot:
      k8s:
        env:
          - name: AMBIENT_ENABLE_MULTI_NETWORK
            value: "true"
          - name: AMBIENT_ENABLE_BAGGAGE
            value: "true"
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: network1
EOF
```
Apply the configuration to `cluster1`:
```
$ istioctl install --context="${CTX_CLUSTER1}" -f cluster1.yaml
```
## Install an ambient east-west gateway in `cluster1`

Install a gateway in `cluster1` that is dedicated to ambient [east-west](https://en.wikipedia.org/wiki/East-west_traffic) traffic. Be aware that, depending on your Kubernetes environment, this gateway may be deployed on the public Internet by default. Production systems may require additional access restrictions (e.g. via firewall rules) to prevent external attacks. Check with your cloud vendor to see what options are available.
```
$ @samples/multicluster/gen-eastwest-gateway.sh@ \
    --network network1 \
    --ambient | \
    kubectl --context="${CTX_CLUSTER1}" apply -f -
```
Wait for the east-west gateway to be assigned an external IP address:
```
$ kubectl --context="${CTX_CLUSTER1}" get svc istio-eastwestgateway -n istio-system
NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)   AGE
istio-eastwestgateway   LoadBalancer   10.80.6.124   34.75.71.237   ...       51s
```
## Set the default network for `cluster2`

If the istio-system namespace is already created, we need to set the cluster’s network there:
```
$ kubectl --context="${CTX_CLUSTER2}" get namespace istio-system && \
  kubectl --context="${CTX_CLUSTER2}" label namespace istio-system topology.istio.io/network=network2
```
## Configure cluster2 as a primary

Create the `istioctl` configuration for `cluster2`:

Install Istio as primary in `cluster2` using istioctl and the `IstioOperator` API.
```
$ cat <<EOF > cluster2.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: ambient
  components:
    pilot:
      k8s:
        env:
          - name: AMBIENT_ENABLE_MULTI_NETWORK
            value: "true"
          - name: AMBIENT_ENABLE_BAGGAGE
            value: "true"
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster2
      network: network2
EOF
```
Apply the configuration to `cluster2`:
```
$ istioctl install --context="${CTX_CLUSTER2}" -f cluster2.yaml
```
## Install an ambient east-west gateway in `cluster2`

As we did with `cluster1` above, install a gateway in `cluster2` that is dedicated to east-west traffic.
```
$ @samples/multicluster/gen-eastwest-gateway.sh@ \
    --network network2 \
    --ambient | \
    kubectl apply --context="${CTX_CLUSTER2}" -f -
```
Wait for the east-west gateway to be assigned an external IP address:
```
$ kubectl --context="${CTX_CLUSTER2}" get svc istio-eastwestgateway -n istio-system
NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)   AGE
istio-eastwestgateway   LoadBalancer   10.0.12.121   34.122.91.98   ...       51s
```
## Enable Endpoint Discovery

Install a remote secret in `cluster2` that provides access to `cluster1`’s API server.
```
$ istioctl create-remote-secret \
  --context="${CTX_CLUSTER1}" \
  --name=cluster1 | \
  kubectl apply -f - --context="${CTX_CLUSTER2}"
```
Install a remote secret in `cluster1` that provides access to `cluster2`’s API server.
```
$ istioctl create-remote-secret \
  --context="${CTX_CLUSTER2}" \
  --name=cluster2 | \
  kubectl apply -f - --context="${CTX_CLUSTER1}"
```
**Congratulations!** You successfully installed an Istio mesh across multiple primary clusters on different networks!

## Next Steps

You can now [verify the installation](_latest_docs_ambient_install_multicluster_verify_.md).

## Cleanup

Uninstall Istio from both `cluster1` and `cluster2` using the same mechanism you installed Istio with (istioctl or Helm).

Uninstall Istio in `cluster1`:
```
$ istioctl uninstall --context="${CTX_CLUSTER1}" -y --purge
$ kubectl delete ns istio-system --context="${CTX_CLUSTER1}"
```
Uninstall Istio in `cluster2`:
```
$ istioctl uninstall --context="${CTX_CLUSTER2}" -y --purge
$ kubectl delete ns istio-system --context="${CTX_CLUSTER2}"
```
Post-delete verification:

```bash
$ kubectl get ns istio-system --context="${CTX_CLUSTER1}"
$ kubectl get ns istio-system --context="${CTX_CLUSTER2}"
$ kubectl get crd | grep -E "istio\\.io|gateway\\.networking\\.k8s\\.io"
```

## Post-delete verification

After uninstalling Istio from both clusters:

```bash
$ kubectl get ns istio-system --context="${CTX_CLUSTER1}"
Error from server (NotFound): namespaces "istio-system" not found

$ kubectl get ns istio-system --context="${CTX_CLUSTER2}"
Error from server (NotFound): namespaces "istio-system" not found

$ kubectl get crd | grep -E "istio\\.io|gateway\\.networking\\.k8s\\.io"
# Should return no results
```

Verify both namespaces are fully removed and no Istio-related CRDs remain.

## Operational bootstrap gates (W16)

Treat the following as required gates for every cluster pair:

1. Cluster templates for both `cluster1.yaml` and `cluster2.yaml` explicitly set `meshID`, `multiCluster.clusterName`, `network`, and ambient multi-network env flags.
2. `cluster2` primary bring-up is not complete until `istioctl install`, east-west gateway deployment, and external address readiness have all passed.
3. Endpoint discovery bootstrap is complete only after **both** remote secrets are created and visible in `istio-system` on opposite clusters.

Gateway readiness gates:

```bash
$ kubectl --context="${CTX_CLUSTER1}" wait --for=condition=available deploy/istio-eastwestgateway -n istio-system --timeout=5m
$ kubectl --context="${CTX_CLUSTER2}" wait --for=condition=available deploy/istio-eastwestgateway -n istio-system --timeout=5m
$ kubectl --context="${CTX_CLUSTER1}" get svc istio-eastwestgateway -n istio-system
$ kubectl --context="${CTX_CLUSTER2}" get svc istio-eastwestgateway -n istio-system
```

Remote secret verification:

```bash
$ kubectl --context="${CTX_CLUSTER1}" get secret -n istio-system | grep istio-remote-secret-cluster2
$ kubectl --context="${CTX_CLUSTER2}" get secret -n istio-system | grep istio-remote-secret-cluster1
```
## See also