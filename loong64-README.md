- `1.克隆源码 `
```
git clone https://github.com/licloud/ingress-nginx.git

git checkout loong64-controller-v1.1.0
```

- `2.编译 `

使用mod管理包的方式进行编译：

```c

git status

go mod tidy

go get -u golang.org/x/sys golang.org/x/net

make build

make image

具体过程如下：

清空依赖：
rm -rf /root/go/pkg
获取依赖：
[root@node01 ingress-nginx]# go mod tidy

编译：
[root@node01 ingress-nginx]# make build
编译报错：
[root@node01 ingress-nginx]# make build
Unable to find image 'k8s.gcr.io/ingress-nginx/e2e-test-runner:v20210916-gd9f96bbbb@sha256:5b434c08e582b58b96867152682c1e754ee609c82390abf3074992d4ec53ed25' locally
make: *** [Makefile:80: build] Interrupt

解决方式：不使用e2e相关进行测试，注释掉其
[root@node01 ingress-nginx]# vim Makefile
#@build/run-in-docker.sh

编译报错：
[root@node01 ingress-nginx]# make build
Building targets for loong64, generated targets in rootfs/bin/loong64 directory.
# golang.org/x/sys/unix
../go/pkg/mod/golang.org/x/sys@v0.0.0-20210817190340-bfb29a6856f2/unix/affinity_linux.go:14:35: undefined: _NCPUBITS
...
make: *** [Makefile:81：build] 错误 2

解决方式：使用最新的net和sys替换原有的
[root@node01 ingress-nginx]# go get -u golang.org/x/sys golang.org/x/net

编译完成
[root@node01 ingress-nginx]# make build
Building targets for loong64, generated targets in rootfs/bin/loong64 directory.
[root@node01 ingress-nginx]# ls rootfs/bin/loong64/
dbg  nginx-ingress-controller  wait-shutdown

```

- `8.打包镜像`

```c
vim Makefile
REGISTRY ?= cr.loongnix.cn
BASE_IMAGE ?= cr.loongnix.cn/google_containers/nginx-slim:0.14

[root@node01 ingress-nginx]# git diff rootfs/Dockerfile
diff --git a/rootfs/Dockerfile b/rootfs/Dockerfile
index 5a8af2a6d..8d976d5fc 100644
--- a/rootfs/Dockerfile
+++ b/rootfs/Dockerfile
@@ -33,11 +33,7 @@ LABEL build_id="${BUILD_ID}"
 
 WORKDIR  /etc/nginx
 
-RUN apk update \
-  && apk upgrade \
-  && apk add --no-cache \
-    diffutils \
-  && rm -rf /var/cache/apk/*
+RUN apt update || apt update
 
 COPY --chown=www-data:www-data etc /etc
 
@@ -60,14 +56,14 @@ RUN bash -xeu -c ' \
     chown -R www-data.www-data ${dir}; \
   done'
 
-RUN apk add --no-cache libcap \
+RUN apt install libcap2 libcap2-bin dumb-init -y \
   && setcap    cap_net_bind_service=+ep /nginx-ingress-controller \
   && setcap -v cap_net_bind_service=+ep /nginx-ingress-controller \
-  && setcap    cap_net_bind_service=+ep /usr/local/nginx/sbin/nginx \
-  && setcap -v cap_net_bind_service=+ep /usr/local/nginx/sbin/nginx \
+  && setcap    cap_net_bind_service=+ep /usr/sbin/nginx \
+  && setcap -v cap_net_bind_service=+ep /usr/sbin/nginx \
   && setcap    cap_net_bind_service=+ep /usr/bin/dumb-init \
   && setcap -v cap_net_bind_service=+ep /usr/bin/dumb-init \
-  && apk del libcap
+  && apt remove libcap2 libcap2-dev -y
 
 USER www-data

[root@node01 ingress-nginx]# make image
...
Successfully built ab1e53f85b79
Successfully tagged cr.loongnix.cn/controller:v1.1.0

验证：
[root@node01 ingress-nginx]# docker run cr.loongnix.cn/controller:v1.1.0
-------------------------------------------------------------------------------
NGINX Ingress controller
  Release:       v1.1.0
  Build:         git-843a16a8f
  Repository:    https://github.com/licloud/ingress-nginx.git
  nginx version: nginx/1.14.2

-------------------------------------------------------------------------------
...
下边错误没有连接k8s-api无需关心
```
