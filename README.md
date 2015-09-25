# k8s-service-proxy

[![Docker Repository on Quay.io](https://quay.io/repository/robszumski/k8s-service-proxy/status "Docker Repository on Quay.io")](https://quay.io/repository/robszumski/k8s-service-proxy)

A Kubernetes service reverse proxy that requires no configuration.

This container is designed to be the entrypoint for HTTP traffic into your cluster.

## How It Works

Requests to a domain are proxied based on the host header to a Kubernetes service residing in the `public` namespace. The Kubernetes DNS addon is required for use, since we're simply sending traffic to `http://$domain.public.svc.cluster.local`.

Using a separate namespace for these services protects you from exposing all of your services to the internet. Any domain that doesn't have a corresponding service will return a 502.

In my cluster, the Kubernetes DNS resolver is configured as `10.3.0.10`.

The nginx config is bare bones and simple to understand:

```
server {
	listen 80;
        server_name ~^((?<subdomain>.+)\.)?(?<domain>.+)\.(?<tld>.*)$;

        location / {
          resolver        10.3.0.10;
          proxy_pass      http://$domain.public.svc.cluster.local;
          proxy_redirect off;
          proxy_set_header        Host    $domain.$tld;
          proxy_set_header        X-Real-IP $remote_addr;
          proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
```

TODO: add if statement to handle subdomains

## Try It Out

Before following these steps, have a fully functioning Kubernetes cluster with the DNS addon configured. Configure `kubectl` to connect to this cluster.

First, clone this repo:

```
git@github.com:robszumski/k8s-service-proxy.git
```

Next, create a service and replication controller for nginx. These will be placed in the `default` namespace. You'll see a warning because we're exposing this service publically via a NodePort at `32100`:

```
$ kubectl create -f nginx-rc.yml
replicationcontrollers/nginx-router
$ kubectl create -f nginx-service.yml
You have exposed your service on an external port on all nodes in your
cluster.  If you want to expose this service to the external internet, you may
need to set up firewall rules for the service port(s) (tcp:32100) to serve traffic.

See http://releases.k8s.io/HEAD/docs/user-guide/services-firewalls.md for more details.
services/nginx
```

While our proxy container downloads, let's start a sample webapp to test our routing with. First, create the `public` namespace:

```
$ kubectl create -f public-ns.yml
namespaces/public
```

Create our test app in the new namespace:

```
$ kubectl create -f test-rc.yml
replicationcontrollers/test
$ kubectl create -f test-service.yml
services/test
```

Once everything has started up, you can curl the NodePort while specifying the host header:

```
$ curl -H "Host: test.com" 172.17.4.99:32100
<h1>Hello version 1.0.0</h1>
```

Success! We just dynamically exposed a Kubernetes service to the internet without any configuration.

If something went wrong, you'll see nginx return a 502. This happens any time you provide a domain that doesn't have a service:

```
<html>
<head><title>502 Bad Gateway</title></head>
<body bgcolor="white">
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx</center>
</body>
</html>
```
