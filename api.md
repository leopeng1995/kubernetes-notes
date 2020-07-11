# API

当我看到 Kubernetes API 的设计只采用了 HTTP REST API 时，是很惊讶的，Kubernetes 没有使用 RPC 而是使用最简单的 HTTP REST，不过仔细想了一下，其实也在情理之中，Kubernetes 的 API 基本都是对各种资源（Pod、Deployment、Ingress 等等）的动作（创建、更新、删除），使用 REST 来描述是非常合适，用 RPC 倒显得画蛇添足了。

加上 `-v=9` 选项即可打印 kubectl 命令使用的接口：

```bash
kubectl get pod -v=9
```

直接访问是不行的，anonymous 用户无权访问：

```bash
curl -k -v -XGET  \
	-H "Accept: application/json;as=Table;v=v1beta1;g=meta.k8s.io, application/json" \
	-H "User-Agent: kubectl/v1.16.6 (darwin/amd64) kubernetes/e7f962b" \
	'https://kubernetes.docker.internal:6443/api/v1/namespaces/default/pods?limit=500'
```

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "pods is forbidden: User \"system:anonymous\" cannot list resource \"pods\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}
```

因此我们需要获取相应账号的 token，列出 secrets：

```
kubectl get secrets
```

这里我选择的是 default：

```
token=$(kubectl describe secret $(kubectl get secrets | grep default-token | gawk '{ print $1 }') | grep token: | gawk '{ print $2 }')
echo $token
```

```bash
curl -k -v -XGET  \
	-H "Accept: application/json;as=Table;v=v1beta1;g=meta.k8s.io, application/json" \
	-H "User-Agent: kubectl/v1.16.6 (darwin/amd64) kubernetes/e7f962b" \
	-H "Authorization: Bearer ${token}" \
	'https://kubernetes.docker.internal:6443/api/v1/namespaces/default/pods?limit=500'
```

这次就可以访问成功了。