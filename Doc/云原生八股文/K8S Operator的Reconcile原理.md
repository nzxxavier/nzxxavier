# 核心机制

K8S接口的的watch机制，即Operator框架可以通过watch某些资源来实现对这个资源的**无状态的控制循环**
# Reconcile 循环步骤
``` mermaid
flowchart TD
    A[开始<br>Reconcile请求] --> B{“获取自定义资源<br>Custom Resource”}
    B -- 资源不存在 --> C[“清理与资源相关的<br>K8s对象”]
    B -- 资源存在 --> D[“检查实际状态<br>（例如：Pod, Service等）”]
    
    D --> E[“对比期望状态 vs 实际状态<br>并确定需要执行的操作”]
    E --> F{“是否需要更新?”}
    F -- 是 --> G[“执行操作以弥合差异<br>创建/更新/删除对象”]
    F -- 否 --> H[“状态同步<br>无需操作”]
    
    G --> I[“更新资源状态<br>.status字段”]
    C --> I
    H --> I
    
    I --> J[“结束本次Reconcile<br>（返回成功或稍后重试）”]
```
#### 步骤 1：触发事件
Reconcile 循环可以由以下事件触发：
- **自定义资源（CR）的变化**：用户创建、更新或删除了一个 CR（比如 `kubectl apply -f my-app.yaml`）。这是最常见的触发方式。
- **被管理资源的变化**：Operator 所管理的**下游资源**（如 Pod、Deployment）发生了变化（例如，一个 Pod 意外崩溃了）。Operator 需要感知到这种变化并做出反应。
- **定时触发**：有些 Operator 会定期触发 Reconcile，以确保系统状态的健康，即使没有外部事件发生。
- **外部事件**：例如，从外部系统发出的 webhook 通知。
在 Controller-Runtime 框架中，你通过 `Watch` 来注册对这些事件的监听。
#### 步骤 2：获取期望状态
Operator 从 API 服务器中检索到触发此次 Reconcile 的**自定义资源**，并解析其中定义的期望状态。
#### 步骤 3：检查实际状态
Operator 检查 Kubernetes 集群中，与这个 CR 相关的所有资源的当前状态。
- 我为此应用创建的 Deployment 存在吗？
- 它的副本数是否正确？
- 需要的 ConfigMap 和 Service 都存在并配置正确吗？
- Pod 是否在运行？
#### 步骤 4：Diff & 调和

Operator 比较**步骤 2** 和 **步骤 3** 的结果。如果存在差异，它就执行必要的操作来弥合这个差异。
- **如果 Deployment 不存在** -> 创建它。
- **如果 Pod 副本数少了** -> 调整 Deployment 的副本数。
- **如果 ConfigMap 配置变了** -> 更新 ConfigMap 并滚动更新 Pod。
- **如果 CR 被删除** -> 清理所有它创建的资源。
#### 步骤 5：更新状态

Operator 通常会更新 CR 的 `.status` 字段，以反映当前调和的结果和系统的健康状况（例如，`status.phase: Running`）。这为用户提供了可见性。
#### 步骤 6：重新排队
在一次 Reconcile 结束后，Operator 可以请求在特定时间后**重新将同一个 CR 加入队列**，进行下一次 Reconcile。这确保了即使调和操作只完成了一部分，或者中间出了差错，Operator 还能继续尝试。 