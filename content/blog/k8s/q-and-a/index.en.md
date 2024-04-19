---
title: 'k8s Q&A [updating...]'
date: 2024-04-29T21:31:44+08:00
description: ""
tags: ["k8s", "kubernetes", "docker", "containerd", "pod"]
categories: ["k8s"]
draft: false
---

# 1. how to stop pod that still terminating

Sometimes there was a pod keep in terminating status. For example, the terminationGracePeriodSeconds being set to a very large  value.

Do not force delete pod through kubectl delete pod -f option, this will cause container process residual on node unless
you make sure the container on node already terminated.

You can stopping by following ways.
1. attact to pod container, find which PID that block container exit, then kill it;
2. if the block process is init process(PID is 1), then do the following steps.

if k8s using containerd as runtime, try using `ctr` command.
```bash
ctr -n k8s.io container ls | grep $imageName
#find taskID
ctr -n k8s.io task kill taskID
```

Or using `crictl`
```bash
crictl ps -a  | grep $podName 
crictl stop $containerID
```

{{<alert>}}
1. Why `ctr c ls` shows nothing?  All k8s pod containerd run in `k8s.io` namespace, `containerd namespace` is not `POD namespace` in k8s,  containerd use `default` as default namespace
2. `containerd's container` is different from `docker container`.  when contaienrd create container, the container just be created but not started. ctr start will launch container process, this calls task.
{{</alert>}}