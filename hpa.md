# HPA

首先需要部署 [metrics-server](https://github.com/kubernetes-sigs/metrics-server) 组件，但是在 Mac 的 docker-desktop 上，部署会出现问题，启动参数需要加上 `--kubelet-insecure-tls` （这里可以直接使用 manifests/metrics-server/components-v0.3.6.yaml ）。

```bash
kubectl apply -f manifests/metrics-server/components-v0.3.6.yaml
```

```bash
kubectl autoscale deployment hello-function --cpu-percent=50 --min=1 --max=10
```

需要给 Pod 中的 container 定义资源配额，否则 hpa 会出现 `<unknown>` ：

```yaml
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
```

```
kubectl get hpa
```

测试负载命令：

```
kubectl run -it --rm load-generator --image=busybox /bin/sh

Hit enter for command prompt

while true; do wget -q -O- http://php-apache; done
```