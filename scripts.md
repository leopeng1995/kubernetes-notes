# Scripts

### 清理本地镜像

```bash
#!/usr/bin/env bash
# grep 的是关键字，可以自定义。
image_name=($(docker images | grep localhost | awk '{ print $1 }'))
image_version=($(docker images | grep localhost | awk '{ print $2 }'))

for ((i=0; i<${#image_name[@]}; ++i)); do
   if [ ${image_version[i]} != "latest" ]; then
      # 这里的 if 判断是为了过滤某些不希望清理掉的镜像。 
      if [ ${image_name[i]} != "localhost:5000/leopeng1995/hello-world" ]; then
         eval docker image rm "${image_name[i]}":"${image_version[i]}"
      fi
   fi
done
```

执行完上述脚本后可以使用 `docker image prune` 清理掉 dangling（这里不知道怎么准确翻译，意思是未被使用同时也未被引用） 的镜像。

### 本地开发快速发布

在本地 K8s 环境（Minikube 或者 Docker for Desktop）开发时，通常需要快速编译、发布到本地 Kubernetes 中，因此我整理了一些常用的脚本和模板文件。

注：Dockerfile 文件自行创建，这里不再叙述。

首先是 `template.yml` ，将部署的资源文件中的 `image` 字段的镜像版本号改成可替换的标识符（`<IMAGE_TAG>`）：

```
      containers:
        - image: localhost:5000/leopeng1995/hello-world:<IMAGE_TAG>
```

创建 Makefile 文件：

```
IMAEG_TAG?=latest

docker-build:
	docker build -t leopeng1995/hello-world .

docker-tag:
	export IMAGE_TAG=${IMAGE_TAG}
	docker tag leopeng1995/hello-world localhost:5000/leopeng1995/hello-world:${IMAGE_TAG}

docker-push:
	export IMAGE_TAG=${IMAGE_TAG}
	docker push localhost:5000/leopeng1995/hello-world:${IMAGE_TAG}
```

主要是三个命令，build、tag 和 push 分别完成镜像的编译、标签和发布（这里是发布到本地的 Registry，可自行定义）。

创建 local_publish.sh 文件：

```bash
#!/usr/bin/env bash
pod_name=$(kubectl get pods | grep hello-world | gawk '{ print $1 }' | head -n 1)

echo "Old IMAGE_TAG:"
kubectl describe pods/$pod_name | grep Image: | grep hello-world | gawk '{ print $2 }'

echo "New IMAGE_TAG (填入版本数字):"
read image_tag

export IMAGE_TAG=$image_tag

image_build=true
if [ $image_build = "true" ]
then
  make docker-build && make docker-tag && make docker-push
fi

cat template.yml | sed "s/<IMAGE_TAG>/$IMAGE_TAG/" | kubectl apply -f -
```

该发布脚本实际执行时会输出：

```
No resources found in default namespace.
Old IMAGE_TAG:
error: arguments in resource/name form must have a single resource and name
New IMAGE_TAG (填入版本数字):
```

因为首次发布没有相关的资源，因此这里会查询失败，所以这个时候只需要填入 0.1.0 即可，每次发布都需要执行脚本手动填入新的版本号（希望有哪位大佬能处理一下这个不足）。`image_build` 环境变量可以用来控制是否需要重新编译镜像，有时候只是想用之前的镜像重新发布。