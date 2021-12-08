# Definitions

## Default ingress controller

The [default ingress controller][] is responsible for mapping
hostnames to services on your OpenShift cluster. In a typical install,
a wildcard DNS record for `*.apps.<clustername>.<domain>` maps maps
all hostnames in that domain to the ingress address, and then
the ingress routers inspect the hostname in the request (e.g., the
HTTP `Host` field or the SSL [Service Name Indicator][sni]) to figure
out where to route the request.

[default ingress controller]: https://docs.openshift.com/container-platform/4.9/networking/ingress-operator.html
[sni]: https://en.wikipedia.org/wiki/Server_Name_Indication
