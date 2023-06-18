---
layout: post
title: "ingress-nginx 502 Bad Gatewaying"
date: 2023-06-15
---

Checkout this error, fresh, straight from [`ingress-nginx`](https://kubernetes.github.io/ingress-nginx/) logs:

```
023/06/15 09:44:36 [error] 1317#1317: *12043607 upstream sent too big header while reading response header from upstream
```

Let's analyze the problem.

`ingress-nginx` is basically a huge [`nginx`](https://nginx.org/en/) reverse-proxy that routes HTTP traffic originating from outside the cluster to our in-cluster services (downstreams, from now on). When `nginx` routes the request to a downstream and the downstream responds, `nginx` [first reads the "first part" of the response](https://github.com/nginx/nginx/blob/6af8fe9cc44dbef43af4db5ea09d54ebd4aadbc5/src/http/ngx_http_upstream.c#LL2474C26-L2474C26) (basically, the headers) then continues processing, if the buffer is too small to accomodate the headers returned by the downstream we get into a 502 Bad Gateway.

Ok, we know what causes the problem. Admittedly, though, the error message was quite self-explanatory...

Unfortunately, the downstream code is not under my control, I will need to increase this buffer size to make `nginx` process that downstream responses. Let's see how and if we can affect this buffer size in order to avoid the problem.

`nginx` controls the upstream buffer size through the configuration key [`proxy_buffer_size`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffer_size) which, according to the documentation, by default is 4 Kilobytes or 8 Kilobytes ([a page size](https://github.com/nginx/nginx/blob/6af8fe9cc44dbef43af4db5ea09d54ebd4aadbc5/src/http/modules/ngx_http_proxy_module.c#L3516)). This setting [overrides the `upstream.buffer_size` configuration value](https://github.com/nginx/nginx/blob/6af8fe9cc44dbef43af4db5ea09d54ebd4aadbc5/src/http/modules/ngx_http_proxy_module.c#L464) that is used to [allocate the headers buffer](https://github.com/nginx/nginx/blob/6af8fe9cc44dbef43af4db5ea09d54ebd4aadbc5/src/http/ngx_http_upstream.c#L2395).

What is the default value of `proxy_buffer_size` for `ingress-nginx` ? Looks like it is [4 Kilobytes](https://github.com/kubernetes/ingress-nginx/blob/686aeac5961f37eaf1ddfa2fa320df4ccf0cf005/internal/ingress/controller/config/config.go#L947).

We know what to do now. We have to configure `proxy_buffer_size` of my `ingress-nginx`, but how ?
`ingress-nginx` [is configurable](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/) globally using a [`ConfigMap`](https://kubernetes.io/docs/concepts/configuration/configmap/) resource or "locally" using special annotations on the particular [`Ingress`](https://kubernetes.io/docs/concepts/services-networking/ingress/) resource we want to configure. In this...guide (?) I will show how to change the configuration globally. See [the official documentation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) to learn how to configure `ingress-nginx` using [annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/).

We need to change the `ConfigMap` of `ingress-nginx`. Let's head over the documentation to find the name... Wait. Where is the name of the `ConfigMap` that I should change ?

If we look [at the documentation](https://kubernetes.github.io/ingress-nginx/user-guide/cli-arguments/) we see that there are command line arguments that can be given to `ingress-nginx` executable. One of these flags is [`--configmap`](https://github.com/kubernetes/ingress-nginx/blob/686aeac5961f37eaf1ddfa2fa320df4ccf0cf005/pkg/flags/flags.go#L82), which is used to specify: "_Name of the ConfigMap containing custom global configurations for the controller._". We can set these flags on `ingress-nginx` [`Deployment`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) resource.

Looking in my cluster we see that the the `ConfigMap` name specified on `ingress-nginx` `Deployment` is `clusterwide-ingress-nginx-controller` in the same namespace where `ingress-nginx` is deployed.

[![ingress-nginx ConfigMap name](/assets/ingress-nginx-config-map.jpg)](/assets/ingress-nginx-config-map.jpg){:target="\_blank"}

Ok, let's change the `ConfigMap` to specify `proxy_buffer_size` to `16k`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  ...
  name: clusterwide-ingress-nginx-controller
  ...
data:
  ...
  proxy-buffer-size: 16k
```

Once we apply the changes we can check if they went through by connecting to one of the `ingress-nginx` [Pods](https://kubernetes.io/docs/concepts/workloads/pods/). As you will see from the image below, `ingress-nginx` updated the `nginx.conf` file of the `nginx` instance running in the [Controller's](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) `Pod`.

[![ingress-nginx ConfigMap applied to nginx.conf](/assets/ingress-nginx-conf-applied.jpg)](/assets/ingress-nginx-conf-applied.jpg){:target="\_blank"}

But where does `ingress-nginx` Controller uses the `ConfigMap` name we just configured ? [After the flags are parsed](https://github.com/kubernetes/ingress-nginx/blob/686aeac5961f37eaf1ddfa2fa320df4ccf0cf005/cmd/nginx/main.go#L61) and [the Controller is instantiated](https://github.com/kubernetes/ingress-nginx/blob/686aeac5961f37eaf1ddfa2fa320df4ccf0cf005/cmd/nginx/main.go#L149), the `ConfigMap` name is used [by a configuration store](https://github.com/kubernetes/ingress-nginx/blob/686aeac5961f37eaf1ddfa2fa320df4ccf0cf005/internal/ingress/controller/nginx.go#L130) that [asks Kubernetes](https://github.com/kubernetes/ingress-nginx/blob/686aeac5961f37eaf1ddfa2fa320df4ccf0cf005/internal/ingress/controller/store/store.go#L807) for a `ConfigMap` with the name we provide and then stores all the configuration we made.

Nice, looks like we have the full picture now.

In conclusion, we found out that a downstream was returning responses containing very large headers that would be too big for the default proxy buffer size of nignx. To avoid the problem we increased the size of this buffer to accomodate the large headers returned by the downstream, fixing the 502 Bad Gateway status code.
