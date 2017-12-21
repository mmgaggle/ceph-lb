# Introduction

Ceph Object Gateway is an object storage interface built on top of librados
to provide applications with a RESTful gateway to Ceph Storage Clusters. Ceph
Object Storage supports two interfaces:

* S3: compatible with a large subset of the Amazon S3 API
* Swift: compatible with a large subset of the OpenStack Swift API.

The objective of this ansible playbook is to present and help operators
implement a few different traffic management scenarios for collections of Ceph
Object Gateways. Each traffic management scenario is composed of two stages, a
load balancing stage, and a proxy stage.

# Stage 1: Load Balancing

There are several options for balancing load across storage hosts. Operators
should consider the traffic volume and fault tolerence required for their
environment, while keeping in mind the complexity of their selected scenario.

## Client Side DNS

The ```client_side_dns``` scenario configures a dnsmasq service on each client
that resolves the domain name for the object storage API endpoint to one of the
storage hosts. To balance the load across storage hosts, each storage host
should have the same number of clients. The mapping of storage hosts to clients
is configured with the ```vars/dnsmaq.yml``` variables file. The resulting
configuration is by no means fault tolerent and is only appropriate for lab
environments.

## VRRP with Direct Routing

The ```vrrp_direct_route``` scenario configures keepalived on each storage host
with direct routing. The keepalived daemon runs on both active and passive
storage hosts. All storage hosts running keepalived will use the Virtual
Redundancy Routing Protocol (VRRP). The active storage host sends VRRP
advertisements at periodic intervals; if the backups routers fail to recieve
these advertisements, a new active storage host is elected. 

Direct routing allows storage hosts to process and route packets directly to
the requesting client rather than passing all outgoing packets through the
active storage host. ARP requests for the virtual ip address are filtered by
configuring arptables.

This solution is fault tolerent, but active/passive. It has the ability to scale
egress beyond the throughput of a single host, but not ingress. This may make it
unsuitable for environments that expect a large volume of object writes.

## Route Health Injection

The ```route_health_inject``` scenario configures quagga on each storage host to
advertise a route to the /32 network of a virtual IP address bound to the
loopback interface. Upstream devices use hardware-based equal cost multipath
(ECMP) for load sharing across storage hosts. If quagga crashes or is stopped on
a storage host, the route will be withdrawn and it will stop recieving traffic.
Systemd can be employed to stop the quagga service if either the haproxy or
radosgw service crashes or stops.

This scenario requires upstream devices to be configured, and the required
commands to configure will vary depending on the devices' network operating
system.

# Stage 2: Proxy

HAProxy is often used for balancing load across multiple backend servers. The
role provided by this playbook does not use HAProxy for load balancing, but it
does use a number of it's other features because they improve the robustness of
Ceph Object Storage services. This is the only option we provide for the proxy
stage.

## Connection Management

Starting with the Firefly (v0.80) release of Ceph, the Ceph Object Gateway has
ran on Civetweb. Civetweb is embedded into the ceph-radosgw daemon. This is in
contrast with earlier releases that used the Apache webserver with FastCGI. One
challenge with Civetweb is that it requires a thread per TCP connection. We have
found that running HAProxy in front of ceph-radosgw helps in keeping the
thread pool small while handling high connection counts. To make this work we
keep connections between clients and HAProxy open, and close connections between
HAproxy and ceph-radosgw after each HTTP request. This is accomplished with the
[http-server-close](https://cbonte.github.io/haproxy-dconv/1.9/configuration.html#option%20http-server-close) HAProxy configuration option.

## Timeouts

## SSL/TLS

There are only two SSL/TLS configurations that apply when http-server-close is
employed. The configurations are SSL/TLS bridging/re-encryption or SSL/TLS
offload. 
