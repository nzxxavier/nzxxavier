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