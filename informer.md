# Informer

注：阅读本笔记前请先阅读 [API](https://github.com/leopeng1995/kubernetes-notes/blob/master/api.md) 。

简单来说，Informer 就是用来提供一种 Kubernetes 资源的本地缓存机制，我理解有点像增量更新，通过 ADD、MODIFIED、DELETE 这几种资源变更事件实现对本地缓存的更新，Client 直接从本地缓存获取 Kubernetes 的资源。

![](static/images/informer1.png)

首先从最简单的 Watch API 入手：

```bash
curl -k https://localhost:6443/api/v1/watch/pods?watch=yes -H "Authorization: Bearer ${token}"
```

事件的基本结构为：

```json
{
  "type": "ADDED", // ADDED, MODIFIED, DELETED 等等
  "object": {
    // 对应的 Kubernetes 资源
  }
}
```

创建一个 curlpod.yaml ：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl-deployment
  #namespace: development
spec:
  selector:
    matchLabels:
      app: curlpod
  replicas: 1
  template:
    metadata:
      labels:
        app: curlpod
    spec:
      containers:
      - name: curlpod
        command:
        - sh
        - -c
        - while true; do sleep 1; done
        image: radial/busyboxplus:curl
```

`kubectl apply -f curlpod.yaml` 创建对应 Kubernetes 资源时，会收到 4 个事件：ADDED、MODIFIED、MODIFIED、MODIFIED（四个事件对应 static/json/informer-apply-event*.json），主要是 `object.status` 的差异：

informer-apply-event1.json ：

```json
{
  "type": "ADDED",
  "object": {
    // ...
    "status": {
      "phase": "Pending",
      "qosClass": "BestEffort"
    }
  }
}
```

informer-apply-event2.json ：

```json
{
  "type": "MODIFIED",
  "object": {
    // ...
    "status": {
      "phase": "Pending",
      "conditions": [
        {
          "type": "PodScheduled",
          "status": "True",
          "lastProbeTime": null,
          "lastTransitionTime": "2020-07-12T16:10:08Z"
        }
      ],
      "qosClass": "BestEffort"
    }
  }
}
```

informer-apply-event3.json ：

```json
{
  "type": "MODIFIED",
  "object": {
    // ...
    "status": {
      "phase": "Pending",
      "conditions": [
        {
          "type": "Initialized",
          "status": "True",
          "lastProbeTime": null,
          "lastTransitionTime": "2020-07-12T16:10:08Z"
        },
        {
          "type": "Ready",
          "status": "False",
          "lastProbeTime": null,
          "lastTransitionTime": "2020-07-12T16:10:08Z",
          "reason": "ContainersNotReady",
          "message": "containers with unready status: [curlpod]"
        },
        {
          "type": "ContainersReady",
          "status": "False",
          "lastProbeTime": null,
          "lastTransitionTime": "2020-07-12T16:10:08Z",
          "reason": "ContainersNotReady",
          "message": "containers with unready status: [curlpod]"
        },
        {
          "type": "PodScheduled",
          "status": "True",
          "lastProbeTime": null,
          "lastTransitionTime": "2020-07-12T16:10:08Z"
        }
      ],
      "hostIP": "192.168.65.3",
      "startTime": "2020-07-12T16:10:08Z",
      "containerStatuses": [
        {
          "name": "curlpod",
          "state": {
            "waiting": {
              "reason": "ContainerCreating"
            }
          },
          "lastState": {},
          "ready": false,
          "restartCount": 0,
          "image": "radial/busyboxplus:curl",
          "imageID": "",
          "started": false
        }
      ],
      "qosClass": "BestEffort"
    }
  }
}
```

informer-apply-event4.json ：

```json
{
  "type": "MODIFIED",
  "object": {
    // ...
    "status": {
      "phase": "Running",
      "conditions": [
        {
          "type": "Initialized",
          "status": "True",
          "lastProbeTime": null,
          "lastTransitionTime": "2020-07-12T16:10:08Z"
        },
        {
          "type": "Ready",
          "status": "True",
          "lastProbeTime": null,
          "lastTransitionTime": "2020-07-12T16:10:10Z"
        },
        {
          "type": "ContainersReady",
          "status": "True",
          "lastProbeTime": null,
          "lastTransitionTime": "2020-07-12T16:10:10Z"
        },
        {
          "type": "PodScheduled",
          "status": "True",
          "lastProbeTime": null,
          "lastTransitionTime": "2020-07-12T16:10:08Z"
        }
      ],
      "hostIP": "192.168.65.3",
      "podIP": "10.1.0.102",
      "podIPs": [
        {
          "ip": "10.1.0.102"
        }
      ],
      "startTime": "2020-07-12T16:10:08Z",
      "containerStatuses": [
        {
          "name": "curlpod",
          "state": {
            "running": {
              "startedAt": "2020-07-12T16:10:09Z"
            }
          },
          "lastState": {},
          "ready": true,
          "restartCount": 0,
          "image": "radial/busyboxplus:curl",
          "imageID": "docker-pullable://radial/busyboxplus@sha256:a68c05ab1112fd90ad7b14985a48520e9d26dbbe00cb9c09aa79fdc0ef46b372",
          "containerID": "docker://f57d33a6178a6d43840d97d6dedd066d297f24cdb597cb6abf5ee30c112451aa",
          "started": true
        }
      ],
      "qosClass": "BestEffort"
    }
  }
}
```

我们可以简单得出以下结论：

1. 在 Event 1 和 2 的时候 Pod 还没有被 Kubernetes 分配具体的机器上，所以 `status.hostIP` 、 `status.podIP` 和 `podIPs` 均为空。
2. `conditions` 里有几个状态，先后顺序为：PodScheduled => Initialized => ContainersReady => Ready ，Event 3 中已经分配了物理 IP （hostIP），但是还没有分配 podIP ，所以此时 Ready 提示 ContainersNotReady ，整体 Pod 的状态也还是 Pending。

`kubectl delete -f curlpod.yaml` 也会产生四个事件，分别是：MODIFIED、MODIFIED、MODIFIED、DELETED。注意，创建的时候 ADDED 是第一个事件，而删除的时候 DELETED 是最后一个事件。我理解这是一个小型的状态机，通过状态的变化而实现优雅上线和优雅下线。