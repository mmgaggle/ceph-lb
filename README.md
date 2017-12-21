# Introduction

Ceph Object Gateway is an object storage interface built on top of librados
to provide applications with a RESTful gateway to Ceph Storage Clusters. Ceph
Object Storage supports two interfaces:

* S3: compatible with a large subset of the Amazon S3 API
* Swift: compatible with a large subset of the OpenStack Swift API.

The objective of this ansible playbook is to present and help operators
implement a few different traffic management strategies for collections of Ceph
Object Gateways. Each traffic management strategy is composed of two stages,
a load balancing stage, and proxy stage.

# Load Balancing Stage

There are several options for balancing load across storage hosts. Operators
should consider the traffic volume and fault tolerence required for their
environment, while keeping in mind the complexity of their selected strategy.

## Client Side with Dnsmasq

The dnsmasq role configures a dnsmasq service on each client that resolves the
domain name for the object storage API endpoint to one of the storage hosts. To
balance the load across storage hosts, each storage host should have the same
number of clients. The mapping of storage hosts to clients is configured through
an ansible variables file. The resulting configuration is by no means fault
tolerent and is only appropriate for lab environments.

## Keepalived with Direct Routing

The keepalived role configures keepalived on each storage host with direct
routing. This solution is fault tolerent, but active/passive. It has the
ability to scale egress beyond the throughput of a single host, but not
ingress. This may make it unsuitable for environments that expect a large
volume of object writes.

## Quagga with Equal Cost Multipath

Quagga is a routing daemon that has the ability to form an adjacency between
the host it is deployed on and another router. The quagga role configures
quagga on each storage host to advertise a route to a virtual IP address on
a loopback interface. Upstream routers balance load across the routes provided
by each storage host with equal cost multipath. With proper hashing load will
be shared across hosts by flow. If quagga crashes or is stopped on a storage
host, the route will be withdrawn and it will stop recieving traffic. Systemd
can be employed to stop the quagga service if either the haproxy or radosgw
service crashes or stops. Using upstream switching equipment with Broadcom
Smart-Hash technology can further increase the robustness of this approach.

# Proxy Stage

HAProxy is often used for balancing load across multiple backend servers. The
role provided by this playbook does not use HAProxy for load balancing, but it
does use a number of it's other features because they improve the robustness of
Ceph Object Storage services. This is the only option we provide for the proxy
stage.

## Connection Management

Starting with the Firefly (v0.80) release of Ceph, the Ceph Object Gateway has
ran on Civetweb. Civetweb is embedded into the ceph-radosgw daemon. This is in
contrast with earlier releases that used the Apache webserver with FastCGI. One
challenge with Civetweb is that it requires a thread per connection. We have
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
