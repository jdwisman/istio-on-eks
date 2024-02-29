# Security - Peer Authentication

## Prerequisites and Initial Setup

Before we proceed to the rest of this post, we need to make sure that the prerequisites are correctly installed. When complete, we will have the Amazon EKS cluster with Istio and the sample application configured.

First, clone the blog example repository:

git clone https://github.com/aws-samples/istio-on-eks.git

Then we need make sure to complete all the below steps. Note: These steps are from [Module 1 – Getting Started](https://github.com/aws-samples/istio-on-eks/tree/main/modules/01-getting-started) that was used in the first Istio blog Getting started with Istio on EKS.

1. [Prerequisites](https://github.com/aws-samples/istio-on-eks/blob/main/modules/01-getting-started/README.md#prerequisites) – Install tools, set up Amazon EKS and Istio, configure istio-ingress and install Kiali using the [Amazon EKS Istio Blueprints for Terraform](https://aws-ia.github.io/terraform-aws-eks-blueprints/patterns/istio/) that we used in the first blog. 
2. [Deploy](https://github.com/aws-samples/istio-on-eks/tree/main/modules/01-getting-started#deploy) – Deploy Product Catalog application resources and basic Istio resources using Helm.
3. [Configure Kiali](https://github.com/aws-samples/istio-on-eks/tree/main/modules/01-getting-started#configure-kiali) – Port forward to Kiali dashboard and Customize the view on Kiali Graph.
4. [Install istioctl](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/#install-hahahugoshortcode860s2hbhb) and add it to the $PATH

NOTE: Do not proceed if you don’t get the below result using the following command:

```sh
kubectl get pods -n workshop
```
```
NAME                              READY   STATUS    RESTARTS   AGE
catalogdetail-658d6dbc98-q544p    2/2     Running   0          7m19s
catalogdetail2-549877454d-kqk9b   2/2     Running   0          7m19s
frontend-7cc46889c8-qdhht         2/2     Running   0          7m19s
productcatalog-5b79cb8dbb-t9dfl   2/2     Running   0          7m19s
```

Note: Do not execute the [Destroy](https://github.com/sridevi1209/istio-on-eks/blob/network-resiliency/modules/01-getting-started/README.md#destroy) section in Module 1

## mTLS Enablement

This section will demonstrate the steps needed to validate and to force Mutual Transport Layer Security (mTLS) for peer authentication between workloads.

By default, Istio will use mTLS for all workloads with proxies configured, however it will also allow plain text.  When the X-Forwarded-Client-Cert header is there, Istio will use mTLS, and when it is missing, it implies that the requests are in plain text.

![image](https://github.com/jdwisman/istio-on-eks/assets/71530829/c9889622-26a5-4ed4-ac89-6721b4b1c356)


After the workshop is up and running from the Getting Started section, you should be able to check and see if mTLS is in Permissive mode (uses mTLS when available but allows plain text) or Strict mode (mTLS required).  Run this command to see which pods are running:

```sh
Admin:~ $ kubectl get pods -n workshop
NAME                              READY   STATUS    RESTARTS   AGE
catalogdetail-5896fff6b8-pxpdn    2/2     Running   0          157m
catalogdetail2-7d7d5cd48b-8sl8k   2/2     Running   0          157m
frontend-78f696695b-t49rv         2/2     Running   0          157m
productcatalog-64848f7996-w7wm8   2/2     Running   0          157m
Admin:~ $
```

And then choose a pod and run this command to see the "Workload mTLS mode":
```sh
$ istioctl x describe pod catalogdetail-5896fff6b8-pxpdn -n workshop
Pod: catalogdetail-5896fff6b8-pxpdn
   Pod Revision: default
   Pod Ports: 3000 (catalogdetail), 15090 (istio-proxy)
--------------------
Service: catalogdetail
   Port: http 3000/HTTP targets pod port 3000
--------------------
Effective PeerAuthentication:
   **Workload mTLS mode: PERMISSIVE**
Skipping Gateway information (no ingress gateway pods)
```

As you can see, by default Istio is in Permissive mode, so while connections will default to use mTLS, plain text will be accepted as well. We can check that mTLS is enabled in Kiali, by looking for the lock icon in the top banner.

![Screenshot 2024-02-28 at 5 04 01 PM](https://github.com/jdwisman/istio-on-eks/assets/71530829/29cacd2b-ccbc-4359-a5a9-c4249d27f767)

We can then check the status of each connection in Kiali. Navigate to the Graph tab and enable Security in the Display menu. Then you will see a Lock icon to show where mTLS encryption is happening in the traffic flow graph.

![Screenshot 2024-02-28 at 5 10 47 PM](https://github.com/jdwisman/istio-on-eks/assets/71530829/ba49ab89-7db0-486d-8bf0-86f15807c5c5)


If we want to force mTLS for all traffic, then we must enable STRICT mode.  Run the following command to force mTLS everywhere Istio is running.

```sh
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
  namespace: "istio-system"
spec:
  mtls:
    mode: STRICT
EOF
```

Now, if we run the same istioctl command to check the mTLS status, we'll get this output instead.

```
$ istioctl x describe pod catalogdetail-5896fff6b8-pxpdn -n workshop
Pod: catalogdetail-5896fff6b8-pxpdn
   Pod Revision: default
   Pod Ports: 3000 (catalogdetail), 15090 (istio-proxy)
--------------------
Service: catalogdetail
   Port: http 3000/HTTP targets pod port 3000
--------------------
Effective PeerAuthentication:
   **Workload mTLS mode: STRICT**
Applied PeerAuthentication:
   default.istio-system
Skipping Gateway information (no ingress gateway pods)
```
