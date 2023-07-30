# k8s **Affinity与 taint/toleration的区别解释**

## k8s **taint toleration的介绍和使用**

参考文章: https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/taint-and-toleration/

Kubernetes 中 Taint 和 Toleration 是配合使用来进行污点容忍的机制,可以用来避免 Pod 被调度到不合适的 Node 上。

### **Taint 的作用:**

Taint 应用于 Node 上,用于标记该 Node 不宜调度某些 Pod。添加 Taint 后,如果 Pod 没有对应 Toleration,则不会被调度到该 Node。

Taint 效果取决于其效果:

- NoSchedule:表示不能将 Pod 调度到该 Node。
- PreferNoSchedule:表示尽量避免将 Pod 调度到该 Node。
- NoExecute:表示不能将 Pod 调度到该 Node,如果已在 Node 上运行也会驱逐。

### **Toleration 的作用:**

Toleration 设置在 Pod 上,用于容忍(Tolerate)某些 Taint。如果 Pod 可以容忍 Node 的 Taint,则可以调度到该 Node。

Toleration 指定三个参数:

- key:Taint 的 key
- operator:TolerationOperator 操作符,如 "Equal"、"Exists"
- effect:Taint 效果,可选 NoSchedule、PreferNoSchedule 或 NoExecute

**Taint 和 Toleration 的使用:**

1. 在 Node 上添加 Taint:
    
    ```yaml
    kubectl taint nodes node1 key=value:NoSchedule
    
    ```
    
2. 在 Pod spec 中添加 Toleration:
    
    ```yaml
    tolerations:
    - key: "key"
      operator: "Equal"
      value: "value"
      effect: "NoSchedule"
    
    ```
    
3. 这样带有 Toleration 的 Pod 就可以调度到被 Taint 的 Node 上了。

总结:Taint 和 Toleration 实现污点容忍,可以灵活控制 Pod 的调度,避免调度到不合适的 Node。

## k8s Affinity**的介绍和使用**

### 参考文章：

https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/workload-resources/pod-v1/#NodeAffinity

Kubernetes 中的 Affinity 主要分为 nodeAffinity、podAffinity 和 podAntiAffinity 三种,用于 attract 和 repel Pod 在 Node 上的调度。

### **nodeAffinity**

nodeAffinity 根据节点的标签引入吸引 Pod 到特定节点的作用力。例如:

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchExpressions:
      - key: disktype
        operator: In
        values:
        - ssd

```

这个示例会尽量将 Pod 调度到带有 label `disktype=ssd` 的 Node 上。

### **podAffinity**

podAffinity 试图将属于同一个服务的 Pod 调度到相同的拓扑域,例如:

```yaml
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values:
        - store
    topologyKey: "kubernetes.io/hostname"

```

这个示例会尽量将标签为 `app=store` 的 Pod 调度到相同 Node。

### **podAntiAffinity**

podAntiAffinity 与 podAffinity 相反,尽量将 Pod 分散到不同拓扑域,例如:

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - web
      topologyKey: "kubernetes.io/hostname"

```

这个示例会尽量不将 `app=web` 的 Pod 调度到相同 Node。

综上,Affinity 提供了强大的调度吸引策略,能满足复杂的业务需求。

## **Affinity与 taint toleration的**区别

Affinity 和 taint toleration 在 Kubernetes 中都是用于影响 Pod 的调度的机制,主要区别如下:

- Affinity 是一种让 Pod 被吸引到满足指定规则的节点上的机制。它分为 nodeAffinity、podAffinity 和 podAntiAffinity 两种。
- nodeAffinity 允许 Pod 根据节点的标签来调度,podAffinity 和 podAntiAffinity 分别定义了 Pod 之间的亲和性和反亲和性。
- Taint 和 toleration 则是让节点可以排斥不符合要求的 Pod 的机制。Taint 应用于节点上,toleration 应用于 Pod 上。
- 如果 Pod 可以容忍(tolerate)节点的 taint,则该 Pod 可以被调度到该节点上,否则不能。
- Affinity 是“吸引”机制,taint toleration 是“排斥”机制。前者增加调度到目标节点的可能性,后者减少不合适节点的可能性。
- **Affinity 优先于 taint toleration。如果一个节点既有匹配的亲和性要求,又存在不可容忍的 taint,那么 Pod 还是会被调度到该节点**。
- 通常 Affinity 和 taint toleration 结合使用,实现让指定的 Pod 优先调度到特定节点的效果。

总体来说,Affinity 是 positives 的吸引策略,taint toleration 是 negatives 的排斥策略。两者共同形成 Kubernetes 强大的调度机制。