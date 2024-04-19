---
title: 'k8s常见问题总结[持续更新...]'
date: 2024-04-29T21:31:44+08:00
description: ""
tags: ["k8s", "kubernetes", "docker", "containerd", "pod"]
categories: ["k8s"]
draft: false
article.showTableOfContents: true
---

# 1. 怎么手动停止pod
在k8s集群中， 你可能会遇到pod无法删除的问题。 比如pod的terminationGracePeriodSeconds设置的很长， 可以通过下面方式来停止：

1. 进入pod 容器中，查看阻塞的进程pid，kill掉改进程。 
2. 如果阻塞进程是1号进程， 是无法通过kill的手段操作的，可以登录所在的机器查看容器的进程

{{<alert>}}
直接终止容器内的进程可能会导致应用状态不一致或数据丢失。在执行这些操作之前，最好确保了解Pod中运行的应用以及相应的风险
{{</alert>}}

如果使用containerd作为容器runtime， 使用`ctr`命令查找到对应的容器
```bash
ctr -n k8s.io container ls | grep $imageName
#获取到taskID
ctr -n k8s.io task kill taskID
```

或者使用`crictl`命令
```bash
crictl ps -a  | grep $podName 
#获取到containerID
crictl stop $containerID
```


{{<alert>}}
1. 为什么`ctr c ls`无法查看pod的容器。 因为在k8s中，所有k8s的containerd的容器、镜像等魔人均位于的`k8s.io`的namespace下，这个和k8s pod的namespace是不一样的。 而containerd默认在default namespace下<br>
2. containerd 的container概念和task 与docker的container概念有差异。 即contaienrd创建container后，并不会启动容器。 `ctr start` 容器后，才会拉起进程, 称为task。
{{</alert>}}


# 2. 如何更新configmap
有时候我们会将一组文件创建为一个configmap资源，并通过以下命令

```bash
#path为一组配置文件路径
kubectl create cm configmap-demo --from-file=$path/
```

但是更新的时候，没办法使用kubectl apply的命令，会提示以下错误。kubectl apply不支持`--from-file` 的参数。

```bash
error: unknown flag: --from-file
See 'kubectl apply --help' for usage.
```

但是我们可以通过管道的方式来完成， 通过`--dry-run=client` 参数将会打印出要创建到apiserver上的内容， 然后该内容将作为单个文件通过管道被apply成功
```bash
kubectl create cm configmap-demo --from-file=$path/ --dry-run=client -o yaml | k apply -f -
```

可以通过`kubectl help apply` 查看支持的参数，如果configmap是是单个文件的话，则可以通过-f参数来更新
```
kubectl apply cm configmap-demo -f $filename
```

# 3. 如何筛选符合条件的Pod/Node等
## LabelSelector
labelSeletor包含两种语义的表达， `Equality-based`和 `Set-Based`， 实际上`Equality-based`是`Set-Based`的一种特化场景。
### Equality-based
基于等式的条件表达式， 即key, value的条件为 `=`  `==`  `!=` ，其中前两个是等价的。 这种情况下，就要求value必须是唯一的，且key, value是在筛选时必须给定。

通过kubectl工具可以实现这些筛选，用法如下
`kubectl get pod -l key1=value1,key2=value2`  

### Set-Based
基于集合的条件表达式，这种方式将筛选条件表达为 `key Operator value`  三元组。 其中Operator包含了`in` `notin` 和 `exists`。比如，

```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - { key: tier, operator: In, values: [cache] }
    - { key: environment, operator: NotIn, values: [dev] }
```

而对于以下需求，情况就稍显复杂,  就需要依赖Set-Based语义来表达。
- 仅筛选包含该key的资源  `kubect get pod -l $key`
- 筛选不包含该key的  `kebectl get pod -l $key exists`
- 符合某个key，但是value可以有多个选择的  `kubectl get pods -l $key in ($value1, $value2)`
- 符合某个key，但是value不符合某些选项的 `kubectl get pods -l $key notin ($value1, $value2)`


# 4. LabelSelector在ListOptions怎么写
除了在kubectl命令或者yaml中使用labelSelector， 我们在代码中使用以下的方式来实现List过程的筛选， 这里仅以Pod为例，实际上LabelSelector同样支持Job，Deployment， ReplicasSet， DeamonSet等多种资源。

```golang
func FilterPods() {
        selector := metav1.LabelSelector{
		MatchLabels:      make(map[string]string),
		MatchExpressions: make([]metav1.LabelSelectorRequirement, 0),
	}
	lsr := metav1.LabelSelectorRequirement{
		Key:      "apps.kruise.io/cloneset-instance-id",
		Operator: metav1.LabelSelectorOpExists,
		Values:   nil,
	}
	selector.MatchExpressions = append(selector.MatchExpressions, lsr)

	listOptions := metav1.ListOptions{LabelSelector: selector.String()} //这里是错误的用法
	podList, err := c.clientSet.CoreV1().Pods(so.Namespace).List(context.TODO(), listOptions)
	if err != nil {
		klog.Errorf("%+v get pod list error %v", so.NamespacedName, err)
		return nil, err
	}
}	
```

会报以下错误
```bash
get pod list error unable to parse requirement: <nil>: Invalid value: "&LabelSelector{MatchLabels:map[string]string{}": name part must consist of alphanumeric characters, '-', '_' or '.', and must start and end with an alphanumeric character (e.g. 'MyName',  or 'my.name',  or '123-abc', regex used for validation is '([A-Za-z0-9][-A-Za-z0-9_.]*)?[A-Za-z0-9]')
```

selector.String()实际上是返回了一个pb格式的string串，并不是LabelSelector string要求的格式`"x equal y,  x in (y1, y2)"`。
正确写法如下

```golang
listOptions := metav1.ListOptions{LabelSelector: metav1.FormatLabelSelector(&selector)}

//或者如果是确定规则，也可以直接hardcode ~ 
listOptions := metav1.ListOptions{LabelSelector: "key1 in (value1, value2)"}
```

`LabelSelector` api给了我们两个字段来表示筛选条件
- MatchLabels，  一个map类型，这个对应的就是equal-based的语义，即`Key Operator Value`是三元组中Opeartor是Equal
- MatchExpression,  则是Set-Based语义标准的`Key Operator Value`模式
这两类在最终语义的表达上，都会被转换为内部的Requirement  结构，即
```golang
type Requirement struct {
	key      string
	operator selection.Operator
	// In huge majority of cases we have at most one value here.
	// It is generally faster to operate on a single-element slice
	// than on a single-element map, so we have a slice here.
	strValues []string
}
```

所以我们看下 LabelSelectorapi是怎么被FormatLabelSelector处理为string的。FormatLabelSelector 实现如下:

```golang
func FormatLabelSelector(labelSelector *LabelSelector) string {
        //这里先将labelSelector api格式转换为内部的Selector格式，具体就是一组[]Requirements表示。
	selector, err := LabelSelectorAsSelector(labelSelector)
	if err != nil {
		return "<error>"
	}

	l := selector.String()
	if len(l) == 0 {
		l = "<none>"
	}
	return l
}

//其中，LabelSelectorAsSelector使用了默认的internalSelector, 这是一个[]Requirement类型。
type internalSelector []Requirement
func (s internalSelector) String() string {
   var reqs []string
   for ix := range s {
      reqs = append(reqs, s[ix].String())
   }
   return strings.Join(reqs, ",")
}
```

如果我们的label条件仅包含`Equal`语义，或者`In Values`中仅有一个值。那么我们还可以通过 `LabelSelectorAsMap` 方法，将`LabelSelector` api转换为一个`map[string]string`格式，然后对map进行序列化。

```golang
labelMap, err := metav1.FormatLabelSelector(&selector)
listOptions := metav1.ListOptions{LabelSelector: metav1.FormatLabelSelector(&selector)}
```

{{<alert>}}
`LabelSelectorAsMap` 这种方式不适合筛选条件包含 `In`，`NotIn`， `Exists`， `NotExist`。 除非In的条件仅有一个Value
{{</alert>}}
