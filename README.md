# Ceph Object Storage Traffic Management

## Round-Robin DNS

## Static DNS

## IPVS

## Routing

# HAProxy

The intent of this ansible playbook is to help folks operating Ceph clusters
that are providing object storage services deploy and configure HAProxy on
each of their storage hosts running a ceph-radosgw daemon. While HAProxy is
often used for balancing load across backends, this playbook does not use it
in this capacity. Instead HAProxy is being used because it offers a rich set
of HTTP proxy features that can improve the robustness of Ceph object storage
services. This playbook expects ceph-radosgw daemons to be configured such that
they bind to an address on a loopback interface. 

<p align="center">
  <img src="https://raw.githubusercontent.com/mmgaggle/ceph-haproxy/master/diagram.png" />
</p>

# Background

Starting with the Firefly (v0.80) release of Ceph, the Ceph Object Gateway has
ran on Civetweb. Civetweb is embedded into the ceph-radosgw daemon. This is in
contrast with earlier releases that used the Apache webserver with FastCGI. One
challenge with Civetweb is that it requires a thread per connection. We have
found that running HAProxy in front of ceph-radosgw helps in keeping the
thread pool small while handling high connection counts. To make this work we
keep connections between clients and HAProxy open, and close connections between
HAproxy and ceph-radosgw after each HTTP request. This is accomplished with the
[http-server-close](https://cbonte.github.io/haproxy-dconv/1.9/configuration.html#option%20http-server-close) HAProxy configuration option.

# Timeouts

# SSL/TLS

There are only two SSL/TLS configurations that apply when http-server-close is
employed. The configurations are SSL/TLS bridging/re-encryption or SSL/TLS
offload. 
