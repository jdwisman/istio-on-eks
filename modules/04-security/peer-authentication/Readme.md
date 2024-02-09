# Security - Peer Authentication

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

