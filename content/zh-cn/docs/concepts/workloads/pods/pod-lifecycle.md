---
title: Pod 的生命周期
content_type: concept
weight: 30
---
<!--
title: Pod Lifecycle
content_type: concept
weight: 30
-->

<!-- overview -->

<!--
This page describes the lifecycle of a Pod. Pods follow a defined lifecycle, starting
in the `Pending` [phase](#pod-phase), moving through `Running` if at least one
of its primary containers starts OK, and then through either the `Succeeded` or
`Failed` phases depending on whether any container in the Pod terminated in failure.

Whilst a Pod is running, the kubelet is able to restart containers to handle some
kind of faults. Within a Pod, Kubernetes tracks different container
[states](#container-states) and determines what action to take to make the Pod
healthy again.
-->
本页面讲述 Pod 的生命周期。
Pod 遵循预定义的生命周期，起始于 `Pending` [阶段](#pod-phase)，
如果至少其中有一个主要容器正常启动，则进入 `Running`，之后取决于 Pod
中是否有容器以失败状态结束而进入 `Succeeded` 或者 `Failed` 阶段。

在 Pod 运行期间，`kubelet` 能够重启容器以处理一些失效场景。
在 Pod 内部，Kubernetes 跟踪不同容器的[状态](#container-states)并确定使
Pod 重新变得健康所需要采取的动作。

<!--
In the Kubernetes API, Pods have both a specification and an actual status. The
status for a Pod object consists of a set of [Pod conditions](#pod-conditions).
You can also inject [custom readiness information](#pod-readiness-gate) into the
condition data for a Pod, if that is useful to your application.

Pods are only [scheduled](/docs/concepts/scheduling-eviction/) once in their lifetime.
Once a Pod is scheduled (assigned) to a Node, the Pod runs on that Node until it stops
or is [terminated](#pod-termination).
-->
在 Kubernetes API 中，Pod 包含规约部分和实际状态部分。
Pod 对象的状态包含了一组 [Pod 状况（Conditions）](#pod-conditions)。
如果应用需要的话，你也可以向其中注入[自定义的就绪态信息](#pod-readiness-gate)。

Pod 在其生命周期中只会被[调度](/zh-cn/docs/concepts/scheduling-eviction/)一次。
一旦 Pod 被调度（分派）到某个节点，Pod 会一直在该节点运行，直到 Pod
停止或者被[终止](#pod-termination)。

<!-- body -->

<!--
## Pod lifetime

Like individual application containers, Pods are considered to be relatively
ephemeral (rather than durable) entities. Pods are created, assigned a unique
ID ([UID](/docs/concepts/overview/working-with-objects/names/#uids)), and scheduled
to nodes where they remain until termination (according to restart policy) or
deletion.
If a {{< glossary_tooltip term_id="node" >}} dies, the Pods scheduled to that node
are [scheduled for deletion](#pod-garbage-collection) after a timeout period.
-->
## Pod 生命期   {#pod-lifetime}

和一个个独立的应用容器一样，Pod 也被认为是相对临时性（而不是长期存在）的实体。
Pod 会被创建、赋予一个唯一的
ID（[UID](/zh-cn/docs/concepts/overview/working-with-objects/names/#uids)），
并被调度到节点，并在终止（根据重启策略）或删除之前一直运行在该节点。

如果一个{{< glossary_tooltip text="节点" term_id="node" >}}死掉了，调度到该节点的
Pod 也被计划在给定超时期限结束后[删除](#pod-garbage-collection)。

<!--
Pods do not, by themselves, self-heal. If a Pod is scheduled to a
{{< glossary_tooltip text="node" term_id="node" >}} that then fails, the Pod is deleted; likewise, a Pod won't
survive an eviction due to a lack of resources or Node maintenance. Kubernetes uses a
higher-level abstraction, called a
{{< glossary_tooltip term_id="controller" text="controller" >}}, that handles the work of
managing the relatively disposable Pod instances.
-->
Pod 自身不具有自愈能力。如果 Pod
被调度到某{{< glossary_tooltip text="节点" term_id="node" >}}而该节点之后失效，
Pod 会被删除；类似地，Pod 无法在因节点资源耗尽或者节点维护而被驱逐期间继续存活。
Kubernetes 使用一种高级抽象来管理这些相对而言可随时丢弃的 Pod 实例，
称作{{< glossary_tooltip term_id="controller" text="控制器" >}}。

<!--
A given Pod (as defined by a UID) is never "rescheduled" to a different node; instead,
that Pod can be replaced by a new, near-identical Pod, with even the same name if
desired, but with a different UID.

When something is said to have the same lifetime as a Pod, such as a
{{< glossary_tooltip term_id="volume" text="volume" >}},
that means that the thing exists as long as that specific Pod (with that exact UID)
exists. If that Pod is deleted for any reason, and even if an identical replacement
is created, the related thing (a volume, in this example) is also destroyed and
created anew.
-->
任何给定的 Pod （由 UID 定义）从不会被“重新调度（rescheduled）”到不同的节点；
相反，这一 Pod 可以被一个新的、几乎完全相同的 Pod 替换掉。
如果需要，新 Pod 的名字可以不变，但是其 UID 会不同。

如果某物声称其生命期与某 Pod 相同，例如存储{{< glossary_tooltip term_id="volume" text="卷" >}}，
这就意味着该对象在此 Pod （UID 亦相同）存在期间也一直存在。
如果 Pod 因为任何原因被删除，甚至某完全相同的替代 Pod 被创建时，
这个相关的对象（例如这里的卷）也会被删除并重建。

{{< figure src="/images/docs/pod.svg" title="Pod 结构图例" class="diagram-medium" >}}

<!--
*A multi-container Pod that contains a file puller and a
web server that uses a persistent volume for shared storage between the containers.*
-->
*一个包含多个容器的 Pod 中包含一个用来拉取文件的程序和一个 Web 服务器，
均使用持久卷作为容器间共享的存储。*

<!--
## Pod phase

A Pod's `status` field is a
[PodStatus](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#podstatus-v1-core)
object, which has a `phase` field.

The phase of a Pod is a simple, high-level summary of where the Pod is in its
lifecycle. The phase is not intended to be a comprehensive rollup of observations
of container or Pod state, nor is it intended to be a comprehensive state machine.

The number and meanings of Pod phase values are tightly guarded.
Other than what is documented here, nothing should be assumed about Pods that
have a given `phase` value.

Here are the possible values for `phase`:
-->
## Pod 阶段     {#pod-phase}

Pod 的 `status` 字段是一个
[PodStatus](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#podstatus-v1-core)
对象，其中包含一个 `phase` 字段。

Pod 的阶段（Phase）是 Pod 在其生命周期中所处位置的简单宏观概述。
该阶段并不是对容器或 Pod 状态的综合汇总，也不是为了成为完整的状态机。

Pod 阶段的数量和含义是严格定义的。
除了本文档中列举的内容外，不应该再假定 Pod 有其他的 `phase` 值。

下面是 `phase` 可能的值：

<!--
Value       | Description
:-----------|:-----------
`Pending`   | The Pod has been accepted by the Kubernetes cluster, but one or more of the containers has not been set up and made ready to run. This includes time a Pod spends waiting to be scheduled as well as the time spent downloading container images over the network.
`Running`   | The Pod has been bound to a node, and all of the containers have been created. At least one container is still running, or is in the process of starting or restarting.
`Succeeded` | All containers in the Pod have terminated in success, and will not be restarted.
`Failed`    | All containers in the Pod have terminated, and at least one container has terminated in failure. That is, the container either exited with non-zero status or was terminated by the system.
`Unknown`   | For some reason the state of the Pod could not be obtained. This phase typically occurs due to an error in communicating with the node where the Pod should be running.
-->
取值 | 描述
:-----|:-----------
`Pending`（悬决）| Pod 已被 Kubernetes 系统接受，但有一个或者多个容器尚未创建亦未运行。此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间。
`Running`（运行中） | Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建。至少有一个容器仍在运行，或者正处于启动或重启状态。
`Succeeded`（成功） | Pod 中的所有容器都已成功终止，并且不会再重启。
`Failed`（失败） | Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止。也就是说，容器以非 0 状态退出或者被系统终止。
`Unknown`（未知） | 因为某些原因无法取得 Pod 的状态。这种情况通常是因为与 Pod 所在主机通信失败。

{{< note >}}
<!--
When a Pod is being deleted, it is shown as `Terminating` by some kubectl commands.
This `Terminating` status is not one of the Pod phases.
A Pod is granted a term to terminate gracefully, which defaults to 30 seconds.
You can use the flag `--force` to [terminate a Pod by force](/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced).
-->
当一个 Pod 被删除时，执行一些 kubectl 命令会展示这个 Pod 的状态为 `Terminating`（终止）。
这个 `Terminating` 状态并不是 Pod 阶段之一。
Pod 被赋予一个可以体面终止的期限，默认为 30 秒。
你可以使用 `--force` 参数来[强制终止 Pod](/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced)。
{{< /note >}}

<!--
Since Kubernetes 1.27, the kubelet transitions deleted pods, except for
[static pods](/docs/tasks/configure-pod-container/static-pod/) and
[force-deleted pods](/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced)
without a finalizer, to a terminal phase (`Failed` or `Succeeded` depending on
the exit statuses of the pod containers) before their deletion from the API server.
-->
从 Kubernetes 1.27 开始，除了[静态 Pod](/zh-cn/docs/tasks/configure-pod-container/static-pod/)
和没有 Finalizer 的[强制终止 Pod](/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced)
之外，`kubelet` 会将已删除的 Pod 转换到终止阶段
（`Failed` 或 `Succeeded` 具体取决于 Pod 容器的退出状态），然后再从 API 服务器中删除。

<!--
If a node dies or is disconnected from the rest of the cluster, Kubernetes
applies a policy for setting the `phase` of all Pods on the lost node to Failed.
-->
如果某节点死掉或者与集群中其他节点失联，Kubernetes
会实施一种策略，将失去的节点上运行的所有 Pod 的 `phase` 设置为 `Failed`。

<!--
## Container states

As well as the [phase](#pod-phase) of the Pod overall, Kubernetes tracks the state of
each container inside a Pod. You can use
[container lifecycle hooks](/docs/concepts/containers/container-lifecycle-hooks/) to
trigger events to run at certain points in a container's lifecycle.

Once the {{< glossary_tooltip text="scheduler" term_id="kube-scheduler" >}}
assigns a Pod to a Node, the kubelet starts creating containers for that Pod
using a {{< glossary_tooltip text="container runtime" term_id="container-runtime" >}}.
There are three possible container states: `Waiting`, `Running`, and `Terminated`.
-->
## 容器状态  {#container-states}

Kubernetes 会跟踪 Pod 中每个容器的状态，就像它跟踪 Pod 总体上的[阶段](#pod-phase)一样。
你可以使用[容器生命周期回调](/zh-cn/docs/concepts/containers/container-lifecycle-hooks/)
来在容器生命周期中的特定时间点触发事件。

一旦{{< glossary_tooltip text="调度器" term_id="kube-scheduler" >}}将 Pod
分派给某个节点，`kubelet`
就通过{{< glossary_tooltip text="容器运行时" term_id="container-runtime" >}}开始为
Pod 创建容器。容器的状态有三种：`Waiting`（等待）、`Running`（运行中）和
`Terminated`（已终止）。

<!--
To check the state of a Pod's containers, you can use
`kubectl describe pod <name-of-pod>`. The output shows the state for each container
within that Pod.

Each state has a specific meaning:
-->
要检查 Pod 中容器的状态，你可以使用 `kubectl describe pod <pod 名称>`。
其输出中包含 Pod 中每个容器的状态。

每种状态都有特定的含义：

<!--
### `Waiting` {#container-state-waiting}

If a container is not in either the `Running` or `Terminated` state, it is `Waiting`.
A container in the `Waiting` state is still running the operations it requires in
order to complete start up: for example, pulling the container image from a container
image registry, or applying {{< glossary_tooltip text="Secret" term_id="secret" >}}
data.
When you use `kubectl` to query a Pod with a container that is `Waiting`, you also see
a Reason field to summarize why the container is in that state.
-->
### `Waiting` （等待）  {#container-state-waiting}

如果容器并不处在 `Running` 或 `Terminated` 状态之一，它就处在 `Waiting` 状态。
处于 `Waiting` 状态的容器仍在运行它完成启动所需要的操作：例如，
从某个容器镜像仓库拉取容器镜像，或者向容器应用 {{< glossary_tooltip text="Secret" term_id="secret" >}}
数据等等。
当你使用 `kubectl` 来查询包含 `Waiting` 状态的容器的 Pod 时，你也会看到一个
Reason 字段，其中给出了容器处于等待状态的原因。

<!--
### `Running` {#container-state-running}

The `Running` status indicates that a container is executing without issues. If there
was a `postStart` hook configured, it has already executed and finished. When you use
`kubectl` to query a Pod with a container that is `Running`, you also see information
about when the container entered the `Running` state.
-->
### `Running`（运行中）     {#container-state-running}

`Running` 状态表明容器正在执行状态并且没有问题发生。
如果配置了 `postStart` 回调，那么该回调已经执行且已完成。
如果你使用 `kubectl` 来查询包含 `Running` 状态的容器的 Pod 时，
你也会看到关于容器进入 `Running` 状态的信息。

<!--
### `Terminated` {#container-state-terminated}

A container in the `Terminated` state began execution and then either ran to
completion or failed for some reason. When you use `kubectl` to query a Pod with
a container that is `Terminated`, you see a reason, an exit code, and the start and
finish time for that container's period of execution.

If a container has a `preStop` hook configured, this hook runs before the container enters
the `Terminated` state.
-->
### `Terminated`（已终止）   {#container-state-terminated}

处于 `Terminated` 状态的容器已经开始执行并且或者正常结束或者因为某些原因失败。
如果你使用 `kubectl` 来查询包含 `Terminated` 状态的容器的 Pod 时，
你会看到容器进入此状态的原因、退出代码以及容器执行期间的起止时间。

如果容器配置了 `preStop` 回调，则该回调会在容器进入 `Terminated`
状态之前执行。

<!--
## Container restart policy {#restart-policy}

The `spec` of a Pod has a `restartPolicy` field with possible values Always, OnFailure,
and Never. The default value is Always.

The `restartPolicy` applies to all containers in the Pod. `restartPolicy` only
refers to restarts of the containers by the kubelet on the same node. After containers
in a Pod exit, the kubelet restarts them with an exponential back-off delay (10s, 20s,
40s, …), that is capped at five minutes. Once a container has executed for 10 minutes
without any problems, the kubelet resets the restart backoff timer for that container.
-->
## 容器重启策略 {#restart-policy}

Pod 的 `spec` 中包含一个 `restartPolicy` 字段，其可能取值包括
Always、OnFailure 和 Never。默认值是 Always。

`restartPolicy` 适用于 Pod 中的所有容器。`restartPolicy` 仅针对同一节点上
`kubelet` 的容器重启动作。当 Pod 中的容器退出时，`kubelet`
会按指数回退方式计算重启的延迟（10s、20s、40s、...），其最长延迟为 5 分钟。
一旦某容器执行了 10 分钟并且没有出现问题，`kubelet` 对该容器的重启回退计时器执行重置操作。

<!--
## Pod conditions

A Pod has a PodStatus, which has an array of
[PodConditions](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#podcondition-v1-core)
through which the Pod has or has not passed. Kubelet manages the following
PodConditions:
-->
## Pod 状况  {#pod-conditions}

Pod 有一个 PodStatus 对象，其中包含一个
[PodConditions](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#podcondition-v1-core)
数组。Pod 可能通过也可能未通过其中的一些状况测试。
Kubelet 管理以下 PodCondition：

<!--
* `PodScheduled`: the Pod has been scheduled to a node.
* `PodHasNetwork`: (alpha feature; must be [enabled explicitly](#pod-has-network)) the
  Pod sandbox has been successfully created and networking configured.
* `ContainersReady`: all containers in the Pod are ready.
* `Initialized`: all [init containers](/docs/concepts/workloads/pods/init-containers/)
  have completed successfully.
* `Ready`: the Pod is able to serve requests and should be added to the load
  balancing pools of all matching Services.
-->
* `PodScheduled`：Pod 已经被调度到某节点；
* `PodHasNetwork`：Pod 沙箱被成功创建并且配置了网络（Alpha 特性，必须被[显式启用](#pod-has-network)）；
* `ContainersReady`：Pod 中所有容器都已就绪；
* `Initialized`：所有的 [Init 容器](/zh-cn/docs/concepts/workloads/pods/init-containers/)都已成功完成；
* `Ready`：Pod 可以为请求提供服务，并且应该被添加到对应服务的负载均衡池中。

<!--
Field name           | Description
:--------------------|:-----------
`type`               | Name of this Pod condition.
`status`             | Indicates whether that condition is applicable, with possible values "`True`", "`False`", or "`Unknown`".
`lastProbeTime`      | Timestamp of when the Pod condition was last probed.
`lastTransitionTime` | Timestamp for when the Pod last transitioned from one status to another.
`reason`             | Machine-readable, UpperCamelCase text indicating the reason for the condition's last transition.
`message`            | Human-readable message indicating details about the last status transition.
-->
字段名称             | 描述
:--------------------|:-----------
`type`               | Pod 状况的名称
`status`             | 表明该状况是否适用，可能的取值有 "`True`"、"`False`" 或 "`Unknown`"
`lastProbeTime`      | 上次探测 Pod 状况时的时间戳
`lastTransitionTime` | Pod 上次从一种状态转换到另一种状态时的时间戳
`reason`             | 机器可读的、驼峰编码（UpperCamelCase）的文字，表述上次状况变化的原因
`message`            | 人类可读的消息，给出上次状态转换的详细信息

<!--
### Pod readiness {#pod-readiness-gate}

Your application can inject extra feedback or signals into PodStatus:
_Pod readiness_. To use this, set `readinessGates` in the Pod's `spec` to
specify a list of additional conditions that the kubelet evaluates for Pod readiness.
-->
### Pod 就绪态        {#pod-readiness-gate}

{{< feature-state for_k8s_version="v1.14" state="stable" >}}

你的应用可以向 PodStatus 中注入额外的反馈或者信号：**Pod Readiness（Pod 就绪态）**。
要使用这一特性，可以设置 Pod 规约中的 `readinessGates` 列表，为 kubelet
提供一组额外的状况供其评估 Pod 就绪态时使用。

<!--
Readiness gates are determined by the current state of `status.condition`
fields for the Pod. If Kubernetes cannot find such a condition in the
`status.conditions` field of a Pod, the status of the condition
is defaulted to "`False`".

Here is an example:
-->
就绪态门控基于 Pod 的 `status.conditions` 字段的当前值来做决定。
如果 Kubernetes 无法在 `status.conditions` 字段中找到某状况，
则该状况的状态值默认为 "`False`"。

这里是一个例子：

```yaml
kind: Pod
...
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
status:
  conditions:
    - type: Ready                              # 内置的 Pod 状况
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
    - type: "www.example.com/feature-1"        # 额外的 Pod 状况
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  containerStatuses:
    - containerID: docker://abcd...
      ready: true
...
```

<!--
The Pod conditions you add must have names that meet the Kubernetes [label key format](/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set).
-->
你所添加的 Pod 状况名称必须满足 Kubernetes
[标签键名格式](/zh-cn/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set)。

<!--
### Status for Pod readiness {#pod-readiness-status}

The `kubectl patch` command does not support patching object status.
To set these `status.conditions` for the pod, applications and
{{< glossary_tooltip term_id="operator-pattern" text="operators">}} should use
the `PATCH` action.
You can use a [Kubernetes client library](/docs/reference/using-api/client-libraries/) to
write code that sets custom Pod conditions for Pod readiness.
-->
### Pod 就绪态的状态 {#pod-readiness-status}

命令 `kubectl patch` 不支持修改对象的状态。
如果需要设置 Pod 的 `status.conditions`，应用或者
{{< glossary_tooltip term_id="operator-pattern" text="Operators">}}
需要使用 `PATCH` 操作。你可以使用
[Kubernetes 客户端库](/zh-cn/docs/reference/using-api/client-libraries/)之一来编写代码，
针对 Pod 就绪态设置定制的 Pod 状况。

<!--
For a Pod that uses custom conditions, that Pod is evaluated to be ready **only**
when both the following statements apply:

* All containers in the Pod are ready.
* All conditions specified in `readinessGates` are `True`.

When a Pod's containers are Ready but at least one custom condition is missing or
`False`, the kubelet sets the Pod's [condition](#pod-conditions) to `ContainersReady`.
-->
对于使用定制状况的 Pod 而言，只有当下面的陈述都适用时，该 Pod 才会被评估为就绪：

* Pod 中所有容器都已就绪；
* `readinessGates` 中的所有状况都为 `True` 值。

当 Pod 的容器都已就绪，但至少一个定制状况没有取值或者取值为 `False`，
`kubelet` 将 Pod 的[状况](#pod-conditions)设置为 `ContainersReady`。

<!--
### Pod network readiness {#pod-has-network}
-->
### Pod 网络就绪 {#pod-has-network}

{{< feature-state for_k8s_version="v1.25" state="alpha" >}}

<!--
After a Pod gets scheduled on a node, it needs to be admitted by the Kubelet and
have any volumes mounted. Once these phases are complete, the Kubelet works with
a container runtime (using {{< glossary_tooltip term_id="cri" >}}) to set up a
runtime sandbox and configure networking for the Pod. If the
`PodHasNetworkCondition` [feature gate](/docs/reference/command-line-tools-reference/feature-gates/) is enabled,
Kubelet reports whether a pod has reached this initialization milestone through
the `PodHasNetwork` condition in the `status.conditions` field of a Pod.
-->
在 Pod 被调度到某节点后，它需要被 Kubelet 接受并且挂载所需的卷。
一旦这些阶段完成，Kubelet 将与容器运行时（使用{{< glossary_tooltip term_id="cri" >}}）
一起为 Pod 生成运行时沙箱并配置网络。
如果启用了 `PodHasNetworkCondition` [特性门控](/zh-cn/docs/reference/command-line-tools-reference/feature-gates/)，
kubelet 会通过 Pod 的 `status.conditions` 字段中的 `PodHasNetwork` 状况来报告
Pod 是否达到了初始化里程碑。

<!--
The `PodHasNetwork` condition is set to `False` by the Kubelet when it detects a
Pod does not have a runtime sandbox with networking configured. This occurs in
the following scenarios:
-->
当 kubelet 检测到 Pod 不具备配置了网络的运行时沙箱时，`PodHasNetwork` 状况将被设置为 `False`。
以下场景中将会发生这种状况：
<!--
* Early in the lifecycle of the Pod, when the kubelet has not yet begun to set up a sandbox for the Pod using the container runtime.
* Later in the lifecycle of the Pod, when the Pod sandbox has been destroyed due
  to either:
  * the node rebooting, without the Pod getting evicted
  * for container runtimes that use virtual machines for isolation, the Pod
    sandbox virtual machine rebooting, which then requires creating a new sandbox and fresh container network configuration.
-->
* 在 Pod 生命周期的早期阶段，kubelet 还没有开始使用容器运行时为 Pod 设置沙箱时。
* 在 Pod 生命周期的末期阶段，Pod 的沙箱由于以下原因被销毁时：
  * 节点重启时 Pod 没有被驱逐
  * 对于使用虚拟机进行隔离的容器运行时，Pod 沙箱虚拟机重启时，需要创建一个新的沙箱和全新的容器网络配置。

<!--
The `PodHasNetwork` condition is set to `True` by the kubelet after the
successful completion of sandbox creation and network configuration for the Pod
by the runtime plugin. The kubelet can start pulling container images and create
containers after `PodHasNetwork` condition has been set to `True`.
-->
在运行时插件成功完成 Pod 的沙箱创建和网络配置后，
kubelet 会将 `PodHasNetwork` 状况设置为 `True`。
当 `PodHasNetwork` 状况设置为 `True` 后，
Kubelet 可以开始拉取容器镜像和创建容器。

<!--
For a Pod with init containers, the kubelet sets the `Initialized` condition to
`True` after the init containers have successfully completed (which happens
after successful sandbox creation and network configuration by the runtime
plugin). For a Pod without init containers, the kubelet sets the `Initialized`
condition to `True` before sandbox creation and network configuration starts.
-->
对于带有 Init 容器的 Pod，kubelet 会在 Init 容器成功完成后将 `Initialized` 状况设置为 `True`
（这发生在运行时成功创建沙箱和配置网络之后），
对于没有 Init 容器的 Pod，kubelet 会在创建沙箱和网络配置开始之前将
`Initialized` 状况设置为 `True`。

<!--
### Pod scheduling readiness {#pod-scheduling-readiness-gate}
-->
### Pod 调度就绪态   {#pod-scheduling-readiness-gate}


{{< feature-state for_k8s_version="v1.26" state="alpha" >}}

<!--
See [Pod Scheduling Readiness](/docs/concepts/scheduling-eviction/pod-scheduling-readiness/) for more information.
-->
有关详细信息，请参阅 [Pod 调度就绪态](/zh-cn/docs/concepts/scheduling-eviction/pod-scheduling-readiness/)。

<!--
## Container probes

A _probe_ is a diagnostic
performed periodically by the
[kubelet](/docs/reference/command-line-tools-reference/kubelet/)
on a container. To perform a diagnostic,
the kubelet either executes code within the container, or makes
a network request.
-->
## 容器探针    {#container-probes}

probe 是由 [kubelet](/zh-cn/docs/reference/command-line-tools-reference/kubelet/) 对容器执行的定期诊断。
要执行诊断，kubelet 既可以在容器内执行代码，也可以发出一个网络请求。

<!--
### Check mechanisms {#probe-check-methods}

There are four different ways to check a container using a probe.
Each probe must define exactly one of these four mechanisms:

`exec`
: Executes a specified command inside the container. The diagnostic
  is considered successful if the command exits with a status code of 0.

`grpc`
: Performs a remote procedure call using [gRPC](https://grpc.io/).
  The target should implement
  [gRPC health checks](https://grpc.io/grpc/core/md_doc_health-checking.html).
  The diagnostic is considered successful if the `status`
  of the response is `SERVING`.

`httpGet`
: Performs an HTTP `GET` request against the Pod's IP
  address on a specified port and path. The diagnostic is
  considered successful if the response has a status code
  greater than or equal to 200 and less than 400.

`tcpSocket`
: Performs a TCP check against the Pod's IP address on
  a specified port. The diagnostic is considered successful if
  the port is open. If the remote system (the container) closes
  the connection immediately after it opens, this counts as healthy.

-->
### 检查机制    {#probe-check-methods}

使用探针来检查容器有四种不同的方法。
每个探针都必须准确定义为这四种机制中的一种：

`exec`
: 在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。

`grpc`
: 使用 [gRPC](https://grpc.io/) 执行一个远程过程调用。
  目标应该实现
  [gRPC 健康检查](https://grpc.io/grpc/core/md_doc_health-checking.html)。
  如果响应的状态是 "SERVING"，则认为诊断成功。

`httpGet`
: 对容器的 IP 地址上指定端口和路径执行 HTTP `GET` 请求。如果响应的状态码大于等于 200
  且小于 400，则诊断被认为是成功的。

`tcpSocket`
: 对容器的 IP 地址上的指定端口执行 TCP 检查。如果端口打开，则诊断被认为是成功的。
  如果远程系统（容器）在打开连接后立即将其关闭，这算作是健康的。

<!--
### Probe outcome
Each probe has one of three results:

`Success`
: The container passed the diagnostic.

`Failure`
: The container failed the diagnostic.

`Unknown`
: The diagnostic failed (no action should be taken, and the kubelet
  will make further checks).

-->
### 探测结果    {#probe-outcome}

每次探测都将获得以下三种结果之一：

`Success`（成功）
: 容器通过了诊断。

`Failure`（失败）
: 容器未通过诊断。

`Unknown`（未知）
: 诊断失败，因此不会采取任何行动。

<!--
### Types of probe

The kubelet can optionally perform and react to three kinds of probes on running
containers:
-->
### 探测类型    {#types-of-probe}

针对运行中的容器，`kubelet` 可以选择是否执行以下三种探针，以及如何针对探测结果作出反应：

<!--
`livenessProbe`
: Indicates whether the container is running. If
  the liveness probe fails, the kubelet kills the container, and the container
  is subjected to its [restart policy](#restart-policy). If a container does not
  provide a liveness probe, the default state is `Success`.

`readinessProbe`
: Indicates whether the container is ready to respond to requests.
  If the readiness probe fails, the endpoints controller removes the Pod's IP
  address from the endpoints of all Services that match the Pod. The default
  state of readiness before the initial delay is `Failure`. If a container does
  not provide a readiness probe, the default state is `Success`.

`startupProbe`
: Indicates whether the application within the container is started.
  All other probes are disabled if a startup probe is provided, until it succeeds.
  If the startup probe fails, the kubelet kills the container, and the container
  is subjected to its [restart policy](#restart-policy). If a container does not
  provide a startup probe, the default state is `Success`.
-->

`livenessProbe`
: 指示容器是否正在运行。如果存活态探测失败，则 kubelet 会杀死容器，
  并且容器将根据其[重启策略](#restart-policy)决定未来。如果容器不提供存活探针，
  则默认状态为 `Success`。

`readinessProbe`
: 指示容器是否准备好为请求提供服务。如果就绪态探测失败，
  端点控制器将从与 Pod 匹配的所有服务的端点列表中删除该 Pod 的 IP 地址。
  初始延迟之前的就绪态的状态值默认为 `Failure`。
  如果容器不提供就绪态探针，则默认状态为 `Success`。

`startupProbe`
: 指示容器中的应用是否已经启动。如果提供了启动探针，则所有其他探针都会被
  禁用，直到此探针成功为止。如果启动探测失败，`kubelet` 将杀死容器，
  而容器依其[重启策略](#restart-policy)进行重启。
  如果容器没有提供启动探测，则默认状态为 `Success`。

<!--
For more information about how to set up a liveness, readiness, or startup probe,
see [Configure Liveness, Readiness and Startup Probes](/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).
-->
如欲了解如何设置存活态、就绪态和启动探针的进一步细节，
可以参阅[配置存活态、就绪态和启动探针](/zh-cn/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)。

<!--
#### When should you use a liveness probe?
-->
#### 何时该使用存活态探针?    {#when-should-you-use-a-liveness-probe}

{{< feature-state for_k8s_version="v1.0" state="stable" >}}

<!--
If the process in your container is able to crash on its own whenever it
encounters an issue or becomes unhealthy, you do not necessarily need a liveness
probe; the kubelet will automatically perform the correct action in accordance
with the Pod's `restartPolicy`.

If you'd like your container to be killed and restarted if a probe fails, then
specify a liveness probe, and specify a `restartPolicy` of Always or OnFailure.
-->
如果容器中的进程能够在遇到问题或不健康的情况下自行崩溃，则不一定需要存活态探针；
`kubelet` 将根据 Pod 的 `restartPolicy` 自动执行修复操作。

如果你希望容器在探测失败时被杀死并重新启动，那么请指定一个存活态探针，
并指定 `restartPolicy` 为 "`Always`" 或 "`OnFailure`"。

<!--
#### When should you use a readiness probe?
-->
#### 何时该使用就绪态探针?      {#when-should-you-use-a-readiness-probe}

{{< feature-state for_k8s_version="v1.0" state="stable" >}}

<!--
If you'd like to start sending traffic to a Pod only when a probe succeeds,
specify a readiness probe. In this case, the readiness probe might be the same
as the liveness probe, but the existence of the readiness probe in the spec means
that the Pod will start without receiving any traffic and only start receiving
traffic after the probe starts succeeding.
-->
如果要仅在探测成功时才开始向 Pod 发送请求流量，请指定就绪态探针。
在这种情况下，就绪态探针可能与存活态探针相同，但是规约中的就绪态探针的存在意味着
Pod 将在启动阶段不接收任何数据，并且只有在探针探测成功后才开始接收数据。

<!--
If you want your container to be able to take itself down for maintenance, you
can specify a readiness probe that checks an endpoint specific to readiness that
is different from the liveness probe.
-->
如果你希望容器能够自行进入维护状态，也可以指定一个就绪态探针，
检查某个特定于就绪态的因此不同于存活态探测的端点。

<!--
If your app has a strict dependency on back-end services, you can implement both
a liveness and a readiness probe. The liveness probe passes when the app itself
is healthy, but the readiness probe additionally checks that each required
back-end service is available. This helps you avoid directing traffic to Pods
that can only respond with error messages.

If your container needs to work on loading large data, configuration files, or
migrations during startup, you can use a
[startup probe](#when-should-you-use-a-startup-probe). However, if you want to
detect the difference between an app that has failed and an app that is still
processing its startup data, you might prefer a readiness probe.
-->
如果你的应用程序对后端服务有严格的依赖性，你可以同时实现存活态和就绪态探针。
当应用程序本身是健康的，存活态探针检测通过后，就绪态探针会额外检查每个所需的后端服务是否可用。
这可以帮助你避免将流量导向只能返回错误信息的 Pod。

如果你的容器需要在启动期间加载大型数据、配置文件或执行迁移，
你可以使用[启动探针](#when-should-you-use-a-startup-probe)。
然而，如果你想区分已经失败的应用和仍在处理其启动数据的应用，你可能更倾向于使用就绪探针。

{{< note >}}
<!--
If you want to be able to drain requests when the Pod is deleted, you do not
necessarily need a readiness probe; on deletion, the Pod automatically puts itself
into an unready state regardless of whether the readiness probe exists.
The Pod remains in the unready state while it waits for the containers in the Pod
to stop.
-->
请注意，如果你只是想在 Pod 被删除时能够排空请求，则不一定需要使用就绪态探针；
在删除 Pod 时，Pod 会自动将自身置于未就绪状态，无论就绪态探针是否存在。
等待 Pod 中的容器停止期间，Pod 会一直处于未就绪状态。
{{< /note >}}

<!--
#### When should you use a startup probe?
-->
#### 何时该使用启动探针？   {#when-should-you-use-a-startup-probe}

{{< feature-state for_k8s_version="v1.20" state="stable" >}}

<!--
Startup probes are useful for Pods that have containers that take a long time to
come into service. Rather than set a long liveness interval, you can configure
a separate configuration for probing the container as it starts up, allowing
a time longer than the liveness interval would allow.
-->
对于所包含的容器需要较长时间才能启动就绪的 Pod 而言，启动探针是有用的。
你不再需要配置一个较长的存活态探测时间间隔，只需要设置另一个独立的配置选定，
对启动期间的容器执行探测，从而允许使用远远超出存活态时间间隔所允许的时长。

<!--
If your container usually starts in more than
`initialDelaySeconds + failureThreshold × periodSeconds`, you should specify a
startup probe that checks the same endpoint as the liveness probe. The default for
`periodSeconds` is 10s. You should then set its `failureThreshold` high enough to
allow the container to start, without changing the default values of the liveness
probe. This helps to protect against deadlocks.
-->
如果你的容器启动时间通常超出 `initialDelaySeconds + failureThreshold × periodSeconds`
总值，你应该设置一个启动探测，对存活态探针所使用的同一端点执行检查。
`periodSeconds` 的默认值是 10 秒。你应该将其 `failureThreshold` 设置得足够高，
以便容器有充足的时间完成启动，并且避免更改存活态探针所使用的默认值。
这一设置有助于减少死锁状况的发生。

<!--
## Termination of Pods {#pod-termination}

Because Pods represent processes running on nodes in the cluster, it is important to
allow those processes to gracefully terminate when they are no longer needed (rather
than being abruptly stopped with a `KILL` signal and having no chance to clean up).
-->
## Pod 的终止    {#pod-termination}

由于 Pod 所代表的是在集群中节点上运行的进程，当不再需要这些进程时允许其体面地终止是很重要的。
一般不应武断地使用 `KILL` 信号终止它们，导致这些进程没有机会完成清理操作。

<!--
The design aim is for you to be able to request deletion and know when processes
terminate, but also be able to ensure that deletes eventually complete.
When you request deletion of a Pod, the cluster records and tracks the intended grace period
before the Pod is allowed to be forcefully killed. With that forceful shutdown tracking in
place, the {{< glossary_tooltip text="kubelet" term_id="kubelet" >}} attempts graceful
shutdown.
-->
设计的目标是令你能够请求删除进程，并且知道进程何时被终止，同时也能够确保删除操作终将完成。
当你请求删除某个 Pod 时，集群会记录并跟踪 Pod 的体面终止周期，
而不是直接强制地杀死 Pod。在存在强制关闭设施的前提下，
{{< glossary_tooltip text="kubelet" term_id="kubelet" >}} 会尝试体面地终止
Pod。

<!--
Typically, the container runtime sends a TERM signal to the main process in each
container. Many container runtimes respect the `STOPSIGNAL` value defined in the container
image and send this instead of TERM.
Once the grace period has expired, the KILL signal is sent to any remaining
processes, and the Pod is then deleted from the
{{< glossary_tooltip text="API Server" term_id="kube-apiserver" >}}. If the kubelet or the
container runtime's management service is restarted while waiting for processes to terminate, the
cluster retries from the start including the full original grace period.
-->
通常情况下，容器运行时会发送一个 TERM 信号到每个容器中的主进程。
很多容器运行时都能够注意到容器镜像中 `STOPSIGNAL` 的值，并发送该信号而不是 TERM。
一旦超出了体面终止限期，容器运行时会向所有剩余进程发送 KILL 信号，之后
Pod 就会被从 {{< glossary_tooltip text="API 服务器" term_id="kube-apiserver" >}}上移除。
如果 `kubelet` 或者容器运行时的管理服务在等待进程终止期间被重启，
集群会从头开始重试，赋予 Pod 完整的体面终止限期。

<!--
An example flow:

1. You use the `kubectl` tool to manually delete a specific Pod, with the default grace period
   (30 seconds).
1. The Pod in the API server is updated with the time beyond which the Pod is considered "dead"
   along with the grace period.
   If you use `kubectl describe` to check on the Pod you're deleting, that Pod shows up as
   "Terminating".
   On the node where the Pod is running: as soon as the kubelet sees that a Pod has been marked
   as terminating (a graceful shutdown duration has been set), the kubelet begins the local Pod
   shutdown process.
-->
下面是一个例子：

1. 你使用 `kubectl` 工具手动删除某个特定的 Pod，而该 Pod 的体面终止限期是默认值（30 秒）。

2. API 服务器中的 Pod 对象被更新，记录涵盖体面终止限期在内 Pod
   的最终死期，超出所计算时间点则认为 Pod 已死（dead）。
   如果你使用 `kubectl describe` 来查验你正在删除的 Pod，该 Pod 会显示为
   "Terminating" （正在终止）。
   在 Pod 运行所在的节点上：`kubelet` 一旦看到 Pod
   被标记为正在终止（已经设置了体面终止限期），`kubelet` 即开始本地的 Pod 关闭过程。

   <!--
   1. If one of the Pod's containers has defined a `preStop`
      [hook](/docs/concepts/containers/container-lifecycle-hooks), the kubelet
      runs that hook inside of the container. If the `preStop` hook is still running after the
      grace period expires, the kubelet requests a small, one-off grace period extension of 2
      seconds.
   -->

   1. 如果 Pod 中的容器之一定义了 `preStop`
      [回调](/zh-cn/docs/concepts/containers/container-lifecycle-hooks)，
      `kubelet` 开始在容器内运行该回调逻辑。如果超出体面终止限期时，
      `preStop` 回调逻辑仍在运行，`kubelet` 会请求给予该 Pod 的宽限期一次性增加 2 秒钟。

      {{< note >}}
      <!--
      If the `preStop` hook needs longer to complete than the default grace period allows,
      you must modify `terminationGracePeriodSeconds` to suit this.
      -->
      如果 `preStop` 回调所需要的时间长于默认的体面终止限期，你必须修改
      `terminationGracePeriodSeconds` 属性值来使其正常工作。
      {{< /note >}}

   <!--
   1. The kubelet triggers the container runtime to send a TERM signal to process 1 inside each
      container.
   -->

   2. `kubelet` 接下来触发容器运行时发送 TERM 信号给每个容器中的进程 1。

      {{< note >}}
      <!--
      The containers in the Pod receive the TERM signal at different times and in an arbitrary
      order. If the order of shutdowns matters, consider using a `preStop` hook to synchronize.
      -->
      Pod 中的容器会在不同时刻收到 TERM 信号，接收顺序也是不确定的。
      如果关闭的顺序很重要，可以考虑使用 `preStop` 回调逻辑来协调。
      {{< /note >}}

<!--
1. At the same time as the kubelet is starting graceful shutdown of the Pod, the control plane evaluates whether
   to remove that shutting-down Pod from EndpointSlice (and Endpoints) objects, where those objects represent
   a {{< glossary_tooltip term_id="service" text="Service" >}} with a configured
   {{< glossary_tooltip text="selector" term_id="selector" >}}.
   {{< glossary_tooltip text="ReplicaSets" term_id="replica-set" >}} and other workload resources
   no longer treat the shutting-down Pod as a valid, in-service replica. Pods that shut down slowly
   should not continue to serve regular traffic and should start terminating and finish processing open connections.
   Some applications need to go beyond finishing open connections and need more graceful termination -
   for example: session draining and completion. Any endpoints that represent the terminating pods
   are not immediately removed from EndpointSlices,
   and a status indicating [terminating state](/docs/concepts/services-networking/endpoint-slices/#conditions)
   is exposed from the EndpointSlice API (and the legacy Endpoints API). Terminating
   endpoints always have their `ready` status
   as `false` (for backward compatibility with versions before 1.26),
   so load balancers will not use it for regular traffic.
   If traffic draining on terminating pod is needed, the actual readiness can be checked as a condition `serving`.
   You can find more details on how to implement connections draining
   in the tutorial [Pods And Endpoints Termination Flow](/docs/tutorials/services/pods-and-endpoint-termination-flow/)
-->
3. 在 `kubelet` 启动 Pod 的体面关闭逻辑的同时，控制平面会评估是否将关闭的
   Pod 从对应的 EndpointSlice（和端点）对象中移除，过滤条件是 Pod
   被对应的{{< glossary_tooltip term_id="service" text="服务" >}}以某
   {{< glossary_tooltip text="选择算符" term_id="selector" >}}选定。
   {{< glossary_tooltip text="ReplicaSet" term_id="replica-set" >}}
   和其他工作负载资源不再将关闭进程中的 Pod 视为合法的、能够提供服务的副本。
   关闭动作很慢的 Pod 不应继续处理常规服务请求，而应开始终止并完成对打开的连接的处理。
   一些应用程序不仅需要完成对打开的连接的处理，还需要更进一步的体面终止逻辑 -
   比如：排空和完成会话。任何正在终止的 Pod 所对应的端点都不会立即从 EndpointSlice
   中被删除，EndpointSlice API（以及传统的 Endpoints API）会公开一个状态来指示其处于
   [终止状态](/zh-cn/docs/concepts/services-networking/endpoint-slices/#conditions)。
   正在终止的端点始终将其 `ready` 状态设置为 `false`（为了向后兼容 1.26 之前的版本），
   因此负载均衡器不会将其用于常规流量。
   如果需要排空正被终止的 Pod 上的流量，可以将 `serving` 状况作为实际的就绪状态。
   你可以在教程
   [探索 Pod 及其端点的终止行为](/zh-cn/docs/tutorials/services/pods-and-endpoint-termination-flow/)
   中找到有关如何实现连接排空的更多详细信息。

   {{<note>}}
   <!--
   If you don't have the `EndpointSliceTerminatingCondition` feature gate enabled
   in your cluster (the gate is on by default from Kubernetes 1.22, and locked to default in 1.26), then the Kubernetes control
   plane removes a Pod from any relevant EndpointSlices as soon as the Pod's
   termination grace period _begins_. The behavior above is described when the
   feature gate `EndpointSliceTerminatingCondition` is enabled.
   -->
   如果你的集群中没有启用 EndpointSliceTerminatingCondition 特性门控
   （该门控从 Kubernetes 1.22 开始默认开启，在 1.26 中锁定为默认），
   那么一旦 Pod 的终止宽限期开始，Kubernetes 控制平面就会从所有的相关 EndpointSlices 中移除 Pod。
   上述行为是在 EndpointSliceTerminatingCondition 特性门控被启用时描述的。
   {{</note>}}


<!--
1. When the grace period expires, the kubelet triggers forcible shutdown. The container runtime sends
   `SIGKILL` to any processes still running in any container in the Pod.
   The kubelet also cleans up a hidden `pause` container if that container runtime uses one.
1. The kubelet transitions the pod into a terminal phase (`Failed` or `Succeeded` depending on
   the end state of its containers). This step is guaranteed since version 1.27.
1. The kubelet triggers forcible removal of Pod object from the API server, by setting grace period
   to 0 (immediate deletion).
1. The API server deletes the Pod's API object, which is then no longer visible from any client.
-->
1. 超出终止宽限期限时，`kubelet` 会触发强制关闭过程。容器运行时会向 Pod
   中所有容器内仍在运行的进程发送 `SIGKILL` 信号。
   `kubelet` 也会清理隐藏的 `pause` 容器，如果容器运行时使用了这种容器的话。

1. `kubelet` 将 Pod 转换到终止阶段（`Failed` 或 `Succeeded` 具体取决于其容器的结束状态）。
    这一步从 1.27 版本开始得到保证。

1. `kubelet` 触发强制从 API 服务器上删除 Pod 对象的逻辑，并将体面终止限期设置为 0
   （这意味着马上删除）。

1. API 服务器删除 Pod 的 API 对象，从任何客户端都无法再看到该对象。

<!--
### Forced Pod termination {#pod-termination-forced}
-->
### 强制终止 Pod     {#pod-termination-forced}

{{< caution >}}
<!--
Forced deletions can be potentially disruptive for some workloads and their Pods.
-->
对于某些工作负载及其 Pod 而言，强制删除很可能会带来某种破坏。
{{< /caution >}}

<!--
By default, all deletes are graceful within 30 seconds. The `kubectl delete` command supports
the `--grace-period=<seconds>` option which allows you to override the default and specify your
own value.
-->
默认情况下，所有的删除操作都会附有 30 秒钟的宽限期限。
`kubectl delete` 命令支持 `--grace-period=<seconds>` 选项，允许你重载默认值，
设定自己希望的期限值。

<!--
Setting the grace period to `0` forcibly and immediately deletes the Pod from the API
server. If the pod was still running on a node, that forcible deletion triggers the kubelet to
begin immediate cleanup.
-->
将宽限期限强制设置为 `0` 意味着立即从 API 服务器删除 Pod。
如果 Pod 仍然运行于某节点上，强制删除操作会触发 `kubelet` 立即执行清理操作。

{{< note >}}
<!--
You must specify an additional flag `--force` along with `--grace-period=0` in order to perform force deletions.
-->
你必须在设置 `--grace-period=0` 的同时额外设置 `--force` 参数才能发起强制删除请求。
{{< /note >}}

<!--
When a force deletion is performed, the API server does not wait for confirmation
from the kubelet that the Pod has been terminated on the node it was running on. It
removes the Pod in the API immediately so a new Pod can be created with the same
name. On the node, Pods that are set to terminate immediately will still be given
a small grace period before being force killed.
-->
执行强制删除操作时，API 服务器不再等待来自 `kubelet` 的、关于 Pod
已经在原来运行的节点上终止执行的确认消息。
API 服务器直接删除 Pod 对象，这样新的与之同名的 Pod 即可以被创建。
在节点侧，被设置为立即终止的 Pod 仍然会在被强行杀死之前获得一点点的宽限时间。

{{< caution >}}
<!--
Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
-->
马上删除时不等待确认正在运行的资源已被终止。这些资源可能会无限期地继续在集群上运行。
{{< /caution >}}

<!--
If you need to force-delete Pods that are part of a StatefulSet, refer to the task
documentation for
[deleting Pods from a StatefulSet](/docs/tasks/run-application/force-delete-stateful-set-pod/).
-->
如果你需要强制删除 StatefulSet 的 Pod，
请参阅[从 StatefulSet 中删除 Pod](/zh-cn/docs/tasks/run-application/force-delete-stateful-set-pod/) 的任务文档。

<!--
### Garbage collection of Pods {#pod-garbage-collection}

For failed Pods, the API objects remain in the cluster's API until a human or
{{< glossary_tooltip term_id="controller" text="controller" >}} process
explicitly removes them.

The Pod garbage collector (PodGC), which is a controller in the control plane, cleans up terminated Pods (with a phase of `Succeeded` or
`Failed`), when the number of Pods exceeds the configured threshold
(determined by `terminated-pod-gc-threshold` in the kube-controller-manager).
This avoids a resource leak as Pods are created and terminated over time.
-->
### Pod 的垃圾收集    {#pod-garbage-collection}

对于已失败的 Pod 而言，对应的 API 对象仍然会保留在集群的 API 服务器上，
直到用户或者{{< glossary_tooltip term_id="controller" text="控制器" >}}进程显式地将其删除。

Pod 的垃圾收集器（PodGC）是控制平面的控制器，它会在 Pod 个数超出所配置的阈值
（根据 `kube-controller-manager` 的 `terminated-pod-gc-threshold` 设置）时删除已终止的
Pod（阶段值为 `Succeeded` 或 `Failed`）。
这一行为会避免随着时间演进不断创建和终止 Pod 而引起的资源泄露问题。

<!--
Additionally, PodGC cleans up any Pods which satisfy any of the following conditions:
1. are orphan pods - bound to a node which no longer exists,
2. are unscheduled terminating pods,
3. are terminating pods, bound to a non-ready node tainted with [`node.kubernetes.io/out-of-service`](/docs/reference/labels-annotations-taints/#node-kubernetes-io-out-of-service), when the `NodeOutOfServiceVolumeDetach` feature gate is enabled.

When the `PodDisruptionConditions` feature gate is enabled, along with
cleaning up the pods, PodGC will also mark them as failed if they are in a non-terminal
phase. Also, PodGC adds a pod disruption condition when cleaning up an orphan
pod (see also:
[Pod disruption conditions](/docs/concepts/workloads/pods/disruptions#pod-disruption-conditions)).
-->
此外，PodGC 会清理满足以下任一条件的所有 Pod：
1. 孤儿 Pod - 绑定到不再存在的节点，
2. 计划外终止的 Pod
3. 终止过程中的 Pod，当启用 `NodeOutOfServiceVolumeDetach` 特性门控时，
   绑定到有 [`node.kubernetes.io/out-of-service`](/zh-cn/docs/reference/labels-annotations-taints/#node-kubernetes-io-out-of-service)
   污点的未就绪节点。

若启用 `PodDisruptionConditions` 特性门控，在清理 Pod 的同时，
如果它们处于非终止状态阶段，PodGC 也会将它们标记为失败。
此外，PodGC 在清理孤儿 Pod 时会添加 Pod 干扰状况（另请参阅：
[Pod 干扰状况](/zh-cn/docs/concepts/workloads/pods/disruptions#pod-disruption-conditions)）。

## {{% heading "whatsnext" %}}

<!--
* Get hands-on experience
  [attaching handlers to container lifecycle events](/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/).

* Get hands-on experience
  [configuring Liveness, Readiness and Startup Probes](/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

* Learn more about [container lifecycle hooks](/docs/concepts/containers/container-lifecycle-hooks/).

* For detailed information about Pod and container status in the API, see
  the API reference documentation covering
  [`.status`](/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodStatus) for Pod.
-->
* 动手实践[为容器生命周期时间关联处理程序](/zh-cn/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/)。
* 动手实践[配置存活态、就绪态和启动探针](/zh-cn/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)。
* 进一步了解[容器生命周期回调](/zh-cn/docs/concepts/containers/container-lifecycle-hooks/)。
* 关于 API 中定义的有关 Pod 和容器状态的详细规范信息，
  可参阅 API 参考文档中 Pod 的 [`.status`](/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodStatus) 字段。
