= OpenShift Egress: DNS proxy mode
== Overview
OpenShift egress router runs a service that redirects traffic to a specified remote server, using a private source IP address that is not used for anything else.
The service allows pods to talk to servers that are set up to only allow access from whitelisted IP addresses.
DNS proxy mode for openshift egress router is recommended for TCP based services with IP addresses or domain names.

== Deploying an Egress Router DNS Proxy Pod

In _DNS proxy mode_, the egress router runs as a DNS proxy for TCP based services.

. Create the pod using the following as an example:
+
[source, yaml]
----
apiVersion: v1
kind: Pod
metadata:
  name: egress-dns-proxy
  labels:
    name: egress-dns-proxy
  annotations:
    pod.network.openshift.io/assign-macvlan: "true" <1>
spec:
  initContainers:
  - name: egress-router-setup
ifdef::openshift-enterprise[]
    image: registry.access.redhat.com/openshift3/ose-egress-router
endif::openshift-enterprise[]
ifdef::openshift-origin[]
    image: openshift/origin-egress-router
endif::openshift-origin[]
    securityContext:
      privileged: true
    env:
    - name: EGRESS_SOURCE <2>
      value: 192.168.12.99
    - name: EGRESS_GATEWAY <3>
      value: 192.168.12.1
    - name: EGRESS_ROUTER_MODE <4>
      value: dns-proxy
  containers:
  - name: egress-dns-proxy
ifdef::openshift-enterprise[]
    image: registry.access.redhat.com/openshift3/ose-egress-router-dns-proxy
endif::openshift-enterprise[]
ifdef::openshift-origin[]
    image: openshift/origin-egress-router-dns-proxy
endif::openshift-origin[]
    env:
    - name: EGRESS_DNS_PROXY_DEBUG <5>
      value: "1"
    - name: EGRESS_DNS_PROXY_DESTINATION <6>
      value: |
        # Egress routes for Project "Foo", version 5

        80  203.0.113.25

        100 example.com

        8080 203.0.113.26 80

        8443 foobar.com 443
----
<1> The `pod.network.openshift.io/assign-macvlan annotation` creates a Macvlan
network interface on the primary network interface, and then moves it into the
pod's network name space before starting the *egress-router* container. Preserve
the quotation marks around `"true"`. Omitting them results in errors.
<2> An IP address from the physical network that the node itself is on and is
reserved by the cluster administrator for use by this pod.
<3> Same value as the default gateway used by the node itself.
<4> This tells the egress router image that it is being deployed as
part of DNS proxy, and so it should not set up iptables
redirecting rules.
<5> This is optional and setting this env will display DNS proxy log output on stdout.
<6> This uses the YAML syntax for a multi-line string; see below for
details.
+
Each line of `EGRESS_DNS_PROXY_DESTINATION` can be one of two types:
+
- `<port> <remote address>` - This says that incoming
connections to the given `<port>` should be proxied to the same
TCP port on the given `<remote address>`. `<remote address>` can be an IP address or DNS name.
In case of DNS name, DNS resolution is done at runtime.
In the example above, the first line proxies TCP traffic from
local port 80 to port 80 on 203.0.113.25. The second line proxies TCP traffic from
local port 100 to port 100 on example.com.
- `<port> <remote address> <remote port>` - As above, except
that the connection is proxied to a different `<remote port>` on
`<remote address>`. In the example above, the third line
proxies local port 8080 to remote port 80 on 203.0.113.26 and the fourth line
proxies local port 8443 to remote port 443 on foobar.com.

. Ensure other pods can find the pod's IP address by creating a service to point to the egress router:
+
[source, yaml]
----
apiVersion: v1
kind: Service
metadata:
  name: egress-dns-svc
spec:
  ports:
  - name: con1
    protocol: TCP
    port: 80
    targetPort: 80
  - name: con2
    protocol: TCP
    port: 100
    targetPort: 100
  - name: con3
    protocol: TCP
    port: 8080
    targetPort: 8080
  - name: con4
    protocol: TCP
    port: 8443
    targetPort: 8443
  type: ClusterIP
  selector:
    name: egress-dns-proxy
----
+
Your pods can now connect to this service. Their connections are proxied to
the corresponding ports on the external server, using the reserved egress IP
address.

You can also specify the `EGRESS_DNS_PROXY_DESTINATION` using a
ConfigMap.

