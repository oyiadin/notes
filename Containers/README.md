## 随手记

### 常见组件之间的关系

- Kubernetes -> docker-shim (Deprecated) -> Docker (moby) -> containerd -> containerd-shim -(OCI)-> runC -> OS
- Kubernetes -(CRI)-> CRI-compatible runtime -(OCI)-> OCI-compatible tool -> OS
    - CRI-compatible runtimes: containerd, cri-o, ...
    - OCI-compatible tools: runC, crun, ...

## OCI Specs

OCI (Open Container Initiative) consists of three specs:

1. the runtime specs: how to run an unpacked file system bundle
2. the image specs: how to bundle and unpack a container file system
3. the distribution specs: how to distribute the images

读懂这些规范并不难，篇幅也不长，这里记录一些笔记。

但是其中 image specs 貌似没怎么有用，因为镜像一般借助 image registry 来分发，而 registry 遵守的是 distribution specs，其中并不涉及 image spces 里的 index、manifest 等概念/结构。

distribution specs 基本与 docker registry API 一致，主要有这些 API：

- `GET /v2/<name>/tags/list`
- `GET /v2/<name>/manifests/<ref>`
- `GET /v2/<name>/blobs/<digest>`

    完整列表参考 https://github.com/opencontainers/distribution-spec/blob/v1.0.1/spec.md#endpoints

- tags 示例： https://docker.m.daocloud.io/v2/nginx/tags/list
- manifests 示例： https://docker.m.daocloud.io/v2/nginx/manifests/latest
- blobs 示例： https://docker.m.daocloud.io/v2/nginx/blobs/sha256:5fb9a81470f3644c474192baf0827a34749286cb6d933091d4d4463ea4f9c495

可自行访问查看响应。

tags 不用多说。manifest 的结构很简单：

```json5
{
   "schemaVersion": 1,
   "name": "docker.io/nginx",
   "tag": "latest",
   "architecture": "amd64",
   "fsLayers": [
      {
         "blobSum": "sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4"
      },
      {
         "blobSum": "sha256:9b1e1e7164db75ad0f64e8deeb33e771d455fa590126b2e16d25e5a75fc6f517"
      }
      # ...
   ],
   "history": ["...省略..."],
   "signatures": ["...省略..."]
}
```

我们最感兴趣的就是 fsLayers，取其中的 blobSum 作为 digest，直接下载 blobs 可得一个 gzip 格式的文件，对这个文件解压即可得到某一层 layer 的文件。

这个过程中跟 image specs 根本就没关系，只需要下载 manifest 文件，逐层下载、解压 blobs 即可得到一个“unpacked file system bundle”。但是要符合 runtime specs 还得继续做一些简单的处理，继续往下：

https://github.com/opencontainers/runtime-spec/blob/v1.0.2/bundle.md

runtime specs 对 fs bundle 的结构的描述很简单：

- 一个 config.json 存储了配置信息
- 一个 rootfs 目录，由 config.json 中的 root.path 属性所指向

rootfs 目录一般用 overlay2 来实现，其中已拥有所有 layer 叠加的结果。

config.json 的规范见此： https://github.com/opencontainers/runtime-spec/blob/v1.0.2/config.md

其中可以包含 mounts、process（cmd、cwd、env、args、rlimits、capabilities、uid、gid、umask等各种配置）、hostname。对于 linux，还可以指定 namespace 的 pid。

config.json 中还可以配置各种生命周期 hooks，如 prestart、startContainer、poststop 等。

这里有一个完整的 config.json 示例： https://github.com/opencontainers/runtime-spec/blob/v1.0.2/config.md#configuration-schema-example

可以看到 config.json 基本把 rootfs、mounts、namespace、hostname、entrypoint、capabilities 等各种信息都给包揽了，但是缺失了网络的配置。因为网络比较特殊，存在各种各样的实现，难以统一到一个模型中，所以也预留了 hooks 的口子可以执行自定义的命令来实现网络的打通。

runtime specs 针对各个操作系统还做了一些特殊要求，比如针对 Linux，就要求 rootfs 中必须已存在 /proc、/sys 等目录，并在容器中挂载为对应的类型。还可以配置 sysctl、seccomp、cgroup（如：memory（limits、swappiness等）、cpu（shares、quota等））

除此之外，runtime specs 还定义了几个必须实现的容器状态（creating、created、running、stopped）以及 13 个生命周期事件。除此之外无其他特别特殊的东西了。

所以经过上边的分析，其实可以说 runc 遵守的所谓 OCI，更准确的说应该是 OCI 其中的 runtime specs。
