# kubernetes ingress-nginx 설치 가이드
> Nginx.Inc의 nginx-ingress-controller와 다르게 kubernetes에서 제공하는 ingress구현체로 같은 nginx기반으로 만들어졌지만 차이점이 존재한다.

- install
    ```sh
    kubectl apply -f mandatory.yaml
    kubectl apply -f service-nodeport.yaml
    ```

- 설치 후 해당 네임스페이스에서 ingress resource 적용시 작동

- [ingress-nginx 와 nginx-ingress-controller 차이점](https://github.com/nginxinc/kubernetes-ingress/blob/master/docs/nginx-ingress-controllers.md)

| Aspect or Feature | kubernetes/ingress-nginx | nginxinc/kubernetes-ingress with NGINX | nginxinc/kubernetes-ingress with NGINX Plus |
| --- | --- | --- | --- |
| **Fundamental** |
| Authors | Kubernetes community | NGINX Inc and community |  NGINX Inc and community |
| NGINX version | [Custom](https://github.com/kubernetes/ingress-nginx/tree/master/images/nginx) NGINX build that includes several third-party modules | NGINX official mainline [build](https://github.com/nginxinc/docker-nginx) | NGINX Plus |
| Commercial support | N/A | N/A | Included |
| **Load balancing configuration** |
| Merging Ingress rules with the same host | Supported | Supported | Supported |
| HTTP load balancing extensions - Annotations | See the [supported annotations](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/annotations.md) | See the [supported annotations](configmap-and-annotations.md) | See the [supported annotations](configmap-and-annotations.md)|
| HTTP load balancing extensions -- ConfigMap | See the [supported ConfigMap keys](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/configmap.md) | See the [supported ConfigMap keys](configmap-and-annotations.md) | See the [supported ConfigMap keys](configmap-and-annotations.md) |
| TCP/UDP | Supported via a ConfigMap | Supported via a ConfigMap with native NGINX configuration | Supported via a ConfigMap with native NGINX configuration |
| Websocket  | Supported | Supported via an [annotation](https://github.com/nginxinc/kubernetes-ingress/tree/master/examples/websocket) | Supported via an [annotation](https://github.com/nginxinc/kubernetes-ingress/tree/master/examples/websocket) |
| TCP SSL Passthrough | Supported via a ConfigMap | Not supported | Not supported |
| JWT validation | Not supported | Not supported | Supported |
| Session persistence | Supported via a third-party module | Not supported | Supported |
| Configuration templates *1 | See the [template](https://github.com/kubernetes/ingress-nginx/blob/master/rootfs/etc/nginx/template/nginx.tmpl) | See the [templates](https://github.com/nginxinc/kubernetes-ingress/tree/master/internal/nginx/templates) | See the [templates](https://github.com/nginxinc/kubernetes-ingress/tree/master/internal/nginx/templates) |
| **Deployment** |
| Command-line arguments *2 | See the [arguments](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/cli-arguments.md) | See the [arguments](cli-arguments.md) | See the [arguments](cli-arguments.md) |
| TLS certificate and key for the default server | Required as a command-line argument/ auto-generated | Required as a command-line argument | Required as a command-line argument |
| Helm chart | Supported | Supported | Supported |
| **Operational** |
| Reporting the IP address(es) of the Ingress controller into Ingress resources | Supported | Supported | Supported |
| Extended Status | Supported via a third-party module | Not supported | Supported |
| Prometheus Integration | Supported | Supported | Supported |
| Dynamic reconfiguration of endpoints (no configuration reloading) | Supported with a third-party Lua module | Not supported | Supported |
