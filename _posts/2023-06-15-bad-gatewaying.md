---
layout: post
title: "ingress-nginx 502 Bad Gatewayin'"
date: 2023-06-15
---

Check out this error, fresh, straight from [_ingress-nginx_](https://kubernetes.github.io/ingress-nginx/) logs:

```
023/06/15 09:44:36 [error] 1317#1317: *12043607 upstream sent too big header while reading response header from upstream
```

Let's analyze the problem to understand and solve it.

_ingress-nginx_ can be thought as a huge [_nginx_](https://nginx.org/en/) reverse-proxy that routes HTTP traffic originating from outside the cluster to our in-cluster services (downstreams, from now on). When _nginx_ routes requests to a downstream and the downstream responds, _nginx_ [first reads the "first part" of the response](https://github.com/nginx/nginx/blob/6af8fe9cc44dbef43af4db5ea09d54ebd4aadbc5/src/http/ngx_http_upstream.c#LL2474C26-L2474C26) (basically, the headers) then continues processing the rest. If the buffer is too small to accomodate the headers returned by the downstream, we get a 502 Bad Gateway.

Unfortunately, we cannot change the problematic downstream to return smaller headers because it is a third-party application distributable. Our only options is to to increase this buffer's size.

Let's see how and if we can affect this buffer size.

_nginx_ controls the upstream buffer size through the configuration key [`proxy_buffer_size`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffer_size) which, according to the documentation, defaults to 4 kB or 8 kB ([a page size](https://github.com/nginx/nginx/blob/6af8fe9cc44dbef43af4db5ea09d54ebd4aadbc5/src/http/modules/ngx_http_proxy_module.c#L3516)). This setting [overrides the `upstream.buffer_size` configuration value](https://github.com/nginx/nginx/blob/6af8fe9cc44dbef43af4db5ea09d54ebd4aadbc5/src/http/modules/ngx_http_proxy_module.c#L464) used to [allocate the headers buffer](https://github.com/nginx/nginx/blob/6af8fe9cc44dbef43af4db5ea09d54ebd4aadbc5/src/http/ngx_http_upstream.c#L2395).

What about the default value of `proxy_buffer_size` for _ingress-nginx_ ? Looks like it is [4 kB](https://github.com/kubernetes/ingress-nginx/blob/686aeac5961f37eaf1ddfa2fa320df4ccf0cf005/internal/ingress/controller/config/config.go#L947).

We know what to do now. We have to configure `proxy_buffer_size` of my _ingress-nginx_, but how ?

_ingress-nginx_ [is configurable](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/) globally using a [_ConfigMap_](https://kubernetes.io/docs/concepts/configuration/configmap/) resource or "locally" using special annotations on the particular [`Ingress`](https://kubernetes.io/docs/concepts/services-networking/ingress/) resource we want to configure. In this...guide (?) I will only show how to change the configuration globally. See [the official documentation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) to learn how to [use annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) instead.

It seems that we need to change the _ConfigMap_ of _ingress-nginx_. Let's head over the documentation to find the name... Wait. Where is the name of the _ConfigMap_ that we have to change ?

If we look [at the documentation](https://kubernetes.github.io/ingress-nginx/user-guide/cli-arguments/) we see that there are command line arguments that can be given to _ingress-nginx_ executable. One of these arguments is [`--configmap`](https://github.com/kubernetes/ingress-nginx/blob/686aeac5961f37eaf1ddfa2fa320df4ccf0cf005/pkg/flags/flags.go#L82), used to specify: "_Name of the ConfigMap containing custom global configurations for the controller._". We can customize `--configmap` on _ingress-nginx_ [_Deployment_](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) resource.

Looking to my cluster we see that the the _ConfigMap_ name specified on _ingress-nginx_ _Deployment_ is `clusterwide-ingress-nginx-controller` in the same namespace where _ingress-nginx_ is deployed.

Here is an excerpt from my cluster of the relevant part in the _Deployment_ of _ingress-nginx_.

```yaml
# ...
spec:
  containers:
    - name: controller
      image:
      args: ...
        - --configmap=$(POD_NAMESPACE)/clusterwide-ingress-nginx-controller
# ...
```

Now, let's change the _ConfigMap_ to specify `proxy_buffer_size` to `16k`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  # ...
  name: clusterwide-ingress-nginx-controller
  # ...
data:
  # ...
  proxy-buffer-size: 16k
```

After making the changes we can check if they were applied by connecting to one _ingress-nginx_ [Pod](https://kubernetes.io/docs/concepts/workloads/pods/). As we can see from the image below, _ingress-nginx_ [Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) did in fact update the `nginx.conf` file to include our changes.

[![ingress-nginx ConfigMap applied to nginx.conf](/assets/ingress-nginx-conf-applied.jpg)](/assets/ingress-nginx-conf-applied.jpg){:target="\_blank"}

If we go back trying the failed request we had at the beginning, it succeeds !

One last question, though: where does _ingress-nginx_ Controller uses the _ConfigMap_ name we just configured ? [After the argumnets we pass are parsed](https://github.com/kubernetes/ingress-nginx/blob/686aeac5961f37eaf1ddfa2fa320df4ccf0cf005/cmd/nginx/main.go#L61) and [the Controller is instantiated](https://github.com/kubernetes/ingress-nginx/blob/686aeac5961f37eaf1ddfa2fa320df4ccf0cf005/cmd/nginx/main.go#L149), the _ConfigMap_ name is used [by a configuration store](https://github.com/kubernetes/ingress-nginx/blob/686aeac5961f37eaf1ddfa2fa320df4ccf0cf005/internal/ingress/controller/nginx.go#L130) that [asks Kubernetes](https://github.com/kubernetes/ingress-nginx/blob/686aeac5961f37eaf1ddfa2fa320df4ccf0cf005/internal/ingress/controller/store/store.go#L807) for a _ConfigMap_ with the name we provide and then stores all the configuration we made.

Nice, looks like we have the full picture now.
