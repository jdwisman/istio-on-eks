# Security - Peer Authentication

## Prerequisites and Initial Setup

Please begin with Prerequisites listed in Module 4 Readme: https://github.com/istio-on-eks/tree/security/modules/04-security

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
   Workload mTLS mode: PERMISSIVE
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
   Workload mTLS mode: STRICT
Applied PeerAuthentication:
   default.istio-system
Skipping Gateway information (no ingress gateway pods)
```

You can also check this by hovering your mouse over the Lock icon in the Kiali banner, which should now look like this:

![Screenshot 2024-02-28 at 5 23 40 PM](https://github.com/jdwisman/istio-on-eks/assets/71530829/bb458794-ab5a-4db4-9ae9-fee023930bcf)



## Validation

Next we will run curl commands from another pod to test and verify that mTLS is enabled. While we already confirmed that configuration with the istioctl command, we need to look at debug logs to confirm that everything is working as expected. First we need to determine which pods are running so we know what to test. We'll try the frontend pod, where we will need both the pod name as well as the corresponding Istio sidecar. Let's get the full name of the pod, so we can enable debug logs on it. Note that your pod name will be different from mine.

```
$ kubectl get pods -n workshop
NAME                              READY   STATUS    RESTARTS   AGE
catalogdetail-5896fff6b8-pxpdn    2/2     Running   0          13d
catalogdetail2-7d7d5cd48b-8sl8k   2/2     Running   0          13d
frontend-78f696695b-t49rv         2/2     Running   0          13d
productcatalog-64848f7996-w7wm8   2/2     Running   0          13d
```

Now, let's use the istioctl command to enable debug logs for the frontend pod.

```
$ istioctl pc log frontend-78f696695b-t49rv -n workshop --level debug
```

Next, lets find the specific service we want to test, in this case, frontend.

```
$ kubectl get svc -n workshop
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
catalogdetail    ClusterIP   172.20.183.210   <none>        3000/TCP   13d
frontend         ClusterIP   172.20.3.138     <none>        9000/TCP   13d
productcatalog   ClusterIP   172.20.138.232   <none>        5000/TCP   13d
```

As you can see, the frontend service is running with cluster IP 172.20.3.138 in my environment (yours may be different).  We also need the Fully Qualified Domain Name (FQDN) of the service.  In Kubernetes, those are normally [service_name].[namespace].svc.cluster.local. So in this case, it would be frontend.workshop.svc.local.

Now we can run a container image that includes network testing utilities such as curl:
```
kubectl run -i --tty curler --image=public.ecr.aws/k8m1l3p1/alpine/curler:latest --rm
```

Now we can send a request to port 9000, which should get rejected as we don't have the proper certificate to authenticate to mTLS. Note that the IP address that it resolves to should be the same as what we saw before.

```
# curl -v frontend.workshop.svc.cluster.local:9000
*   Trying 172.20.3.138:9000...
* Connected to frontend.workshop.svc.cluster.local (172.20.3.138) port 9000 (#0)
> GET / HTTP/1.1
> Host: frontend.workshop.svc.cluster.local:9000
> User-Agent: curl/7.77.0
> Accept: */*
> 
* Recv failure: Connection reset by peer
* Closing connection 0
curl: (56) Recv failure: Connection reset by peer
```
Next, we need to view our debug logs.  While Envoy has an ssl.handshake statistic, its not directly queriable via envoy-proxy. However, by enabling debug logs, we can view the results of queries there.

```
$ kubectl logs -f frontend-78f696695b-t49rv -n workshop -c istio-proxy | grep tlsMode
2024-03-14T22:03:26.779477Z     debug   envoy upstream external/envoy/source/common/upstream/upstream_impl.cc:426       transport socket match, socket tlsMode-disabled selected for host with address 10.0.6.92:14268       thread=14
2024-03-14T22:03:26.779503Z     debug   envoy upstream external/envoy/source/common/upstream/upstream_impl.cc:426       transport socket match, socket tlsMode-istio selected for host with address 10.0.8.201:5000  thread=14
```

## Leveraging Amazon Certificate Manager (ACM)

Certificate management can present additional administrative overhead for Istio deployments. One way to simplify this is to use Amazon’s AWS Certificate Manager (ACM) to manage the TLS certs and just use annotations to allow ingressgateway to use those certs. In this section, we will try it out.


First, we need a Certificate Authority. If you don't already have one, you can easily create one with AWS Private Certificate Authority (APCA). In the AWS Console, click on "Create a Private CA". Next, fill out the form and choose "Create CA". You then need to install the CA, by choosing "Actions" then "Install CA". Now you will be able to create certificates with ACM. You will then be able to see the CA in the console, like this:

<img width="1093" alt="Screenshot 2024-03-29 at 2 25 25 PM" src="https://github.com/jdwisman/istio-on-eks/assets/71530829/b488137b-5df7-41e1-b685-9793af8ef745">

After you have a working CA, go to Amazon Certificate Manager (ACM) in the console and choose "Request a Certificate". Choose "Request a Private Certificate" if you are just testing this. Select your Certificate Authority, give the certificate a name, and then choose "Request". The certificate should be available and you will see it in the console like this:

<img width="1048" alt="Screenshot 2024-03-29 at 2 31 13 PM" src="https://github.com/jdwisman/istio-on-eks/assets/71530829/f88f9cd7-5e54-4de4-9ce9-6184be859e44">

Now we need to annotate our ingressgateway so that it will use the certificate from ACM. For this, let's first get the information about the ingress service like this:
```
$ kubectl get svc -n istio-ingress                                                                                                       
NAME            TYPE           CLUSTER-IP    EXTERNAL-IP                                                                     PORT(S)                                      AGE
istio-ingress   LoadBalancer   172.20.58.1   k8s-istioing-istioing-dc506af4a2-d85559e68d4a31fd.elb.us-east-1.amazonaws.com   15021:32486/TCP,80:30979/TCP,443:31160/TCP   31d
```

Next, we need to add the following annotations to the service. Create a file with the following information, copying in the ARN of your certificate:

```
kubectl -n istio-system patch service istio-ingressgateway --patch "$(cat<<EOFmetadata:annotations:service.beta.kubernetes.io/aws-load-balancer-ssl-cert: <<ARN_OF_CERTIFICATE>>service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcpservice.beta.kubernetes.io/aws-load-balancer-ssl-ports: "https"service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "3600"EOF)"
```

