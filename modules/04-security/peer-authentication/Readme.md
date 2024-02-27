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

This section will demonstrate the steps needed to enable Mutual Transport Layer Security (mTLS) for peer authentication between workloads.

By default, Istio will use mTLS for all workloads with proxies configured, however it will also allow plain text.  When the X-Forwarded-Client-Cert header is there, Istio will use mTLS, and when it is missing, it implies that the requests are in plain text.

![image](https://github.com/jdwisman/istio-on-eks/assets/71530829/c9889622-26a5-4ed4-ac89-6721b4b1c356)


<Add Steps to test this here, and see both mTLS and plain text by default>



If we want to force mTLS for all traffic, then we must enable STRICT mode.

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

