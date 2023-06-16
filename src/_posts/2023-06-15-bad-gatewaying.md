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

`ingress-nginx` is basically a huge nginx reverse-proxy that routes HTTP traffic originating from outside the cluster to our in-cluster services (downstreams, from now on). When `ingress-nginx` routes the request to a downstream and the downstream responds, nginx [first reads the "first part" of the response](https://github.com/nginx/nginx/blob/6af8fe9cc44dbef43af4db5ea09d54ebd4aadbc5/src/http/ngx_http_upstream.c#LL2474C26-L2474C26) (basically, the headers) then continues processing, if the buffer is too small for the headers returned by the downstream we get, you guessed it, 502 Bad Gateway.

Ok, we know what causes the problem. Admittedly, though, the error message was quite self-explanatory... Unfortunately, the downstream code is not under my control, I will need to increase this size to have nginx make peace with that downstream. Let's understand how we can affect this buffer size to avoid the problem.

nginx controls the upstream buffer size through the configuration key [`proxy_buffer_size`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffer_size) which, according to the documentation, by default is 4 Kilobytes or 8 Kilobytes ([a page size](https://github.com/nginx/nginx/blob/6af8fe9cc44dbef43af4db5ea09d54ebd4aadbc5/src/http/modules/ngx_http_proxy_module.c#L3516)). This setting [overrides the upstream.buffer_size configuration value](https://github.com/nginx/nginx/blob/6af8fe9cc44dbef43af4db5ea09d54ebd4aadbc5/src/http/modules/ngx_http_proxy_module.c#L464) that is used to [allocate the headers buffer](https://github.com/nginx/nginx/blob/6af8fe9cc44dbef43af4db5ea09d54ebd4aadbc5/src/http/ngx_http_upstream.c#L2395).

What is the default value of `proxy_buffer_size` for `ingress-nginx` ? Looks like it is [4 Kilobytes](https://github.com/kubernetes/ingress-nginx/blob/686aeac5961f37eaf1ddfa2fa320df4ccf0cf005/internal/ingress/controller/config/config.go#L947).

Well, I guess we know what to do now. We have to configure `proxy_buffer_size` of my `ingress-nginx`, but how ?
`ingress-nginx` [is configurable](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/) mainly through global configuration (using a `ConfigMap` resource) or using special annotations on the Ingress resource to configure. In this...guide (?) I will show both approaches.

We start with global configuration. We need to change `ingress-nginx` ConfigMap. Let's head over the documentation to find the name of the `Config`... Wait. Where is the name of `ConfigMap` that I should change ?

Let's not fret.

If we have a look [at the documentation](https://kubernetes.github.io/ingress-nginx/user-guide/cli-arguments/) we see that there are command line arguments that can be given to `ingress-nginx` executable, one of this flags is [`--configmap`](https://github.com/kubernetes/ingress-nginx/blob/686aeac5961f37eaf1ddfa2fa320df4ccf0cf005/pkg/flags/flags.go#L82). Good, now we know where to fo find the name of the ConfigMap.

Looking at my cluster (using Aptakube) the ConfigMap name is ``

TODO: Demonstrate what nginx.confg ingress-nginx changes in the ingress Pods
TODO: Annotations
TODO: All links to names
