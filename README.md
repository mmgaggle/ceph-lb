# About

Starting with the firefly (v0.80) release of Ceph, the Ceph Object Gateway has
ran on Civetweb. Civetweb is embedded into the ceph-radosgw daemon. This is in
contrast with earlier releases that used the Apache webserver with FastCGI. One
challenge with Civetweb is that it requires a thread per connection. We have
found that running HAProxy in front of ceph-radosgw to help in keeping the
thread pool small while handling high connection counts. To make this work we
keep connections between clients and HAProxy open, and close connections between
HAproxy and ceph-radosgw after each HTTP request. This is done with the
[http-server-close](https://cbonte.github.io/haproxy-dconv/1.9/configuration.html#option%20http-server-close) option in HAProxy.
