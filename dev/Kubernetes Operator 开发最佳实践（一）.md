## 概述

当我们在刚开始学习 Kubernetes Operator 开发时，一般会先学习相关脚手架的配置和使用方法，但大家往往会发现即使掌握了这些方法，仍不清楚该如何编写调谐的代码，例如：如何处理变更、如何处理删除、如何控制调度、状体字段该存储什么内容等等。本文列出了使用 Kubebuilder 开发 Operator 时的一些实践要点。



## 原则

**调谐状态，不要调谐状态的变化。**

Kubernetes 中的资源是对目标环境预期状态的声明，而 Controller 则是保证声明状态与真实状态一致的自动化程序，Controller 工作的过程被称为“调谐”，调谐的结果会回写至资源的 Status 字段中。

在理想的情况下，Controller 不需要知道资源发生了什么变化，只需要在被回调时保证目标环境的当前状态与最新的声明状态一致即可。不过，当目标环境只能被增量更新、或者在被更新过程中需要清理旧数据时，Controller 还是需要知道资源被变更前后的差异。



## 校验资源

对于资源的校验方式，可分为语法和语义两类。

### 语法校验

语法校验可以通过 OpenAPI Schema 实现，属于静态校验。定义资源时，在资源的属性上增加注解，可以自动生成 OpenAPI Schema，例如：

```go
type ClusterSpec struct {
	// +kubebuilder:validation:Pattern=`https?:\/\/(www\.)?[-a-zA-Z0-9@:%._\+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_\+.~#?&\/\/=]*)`
	ApiServer string `json:"apiserver" yaml:"apiserver"`
	// +kubebuilder:validation:Enum=physical;virtual
	ClusterType ClusterType `json:"clustertype" yaml:"clustertype"`
	// +kubebuilder:validation:Enum=host;worker
	Usage ClusterUsage `json:"usage" yaml:"usage"`
	// +optional
	HostCluster string `json:"hostcluster" yaml:"hostcluster"`
}
```

通过上述代码生成的 CRD yaml 中 Schema 部分为：

```yaml
    schema:
      openAPIV3Schema:
        description: Cluster is the Schema for the clusters API
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: ClusterSpec defines the desired state of Cluster
            properties:
              apiserver:
                pattern: https?:\/\/(www\.)?[-a-zA-Z0-9@:%._\+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_\+.~#?&\/\/=]*)
                type: string
              clustertype:
                enum:
                - physical
                - virtual
                type: string
              hostcluster:
                type: string
              usage:
                description: the usage of cluster, for user use it directry or deploy
                  vcluster on it
                enum:
                - host
                - worker
                type: string
            required:
            - apiserver
            - clustertype
            - usage
            type: object
```



更多关于 CRD 语法校验的方法见：

> https://book.kubebuilder.io/reference/markers/crd-validation.html



### 语义校验

语义校验需要通过代码实现，校验代码可以在 Validating Webhook 中、也可以在 Controller 的 Reconcile 函数中，建议在两个位置均引入校验代码，避免因为未安装 Webhook 导致错误的资源进入调谐流程。

**Validating Webhook 中的校验：**

```go
// ValidateCreate implements webhook.Validator so a webhook will be registered for the type
func (r *Cluster) ValidateCreate() error {
	return ValidateCluster(r, nil, false)
}

// ValidateUpdate implements webhook.Validator so a webhook will be registered for the type
func (r *Cluster) ValidateUpdate(old runtime.Object) error {
	return ValidateCluster(r, old.(*Cluster), false)
}

// ValidateDelete implements webhook.Validator so a webhook will be registered for the type
func (r *Cluster) ValidateDelete() error {
	return ValidateCluster(r, nil, true)
}

func ValidateCluster(new, old *Cluster, isDelete bool) error {
    // validate logic
}
```

**Reconcile 函数中的校验：**

```go
	// 从缓存中获取最后一次调谐成功的声明信息
    lastCluster, err := getLastApply(cluster)
	if err != nil {
		return ctrl.Result{}, err
	}

    // 调用 webhook 中定义的校验函数
	if err := clusterCRD.ValidateCluster(cluster, lastCluster, false); err != nil {
		condition := metav1.Condition{
			Type:    ClusterConditionSecret,
			Status:  "False",
			Reason:  ClusterConditionReason,
			Message: err.Error(),
		}
		cluster.Status.SetConditions([]metav1.Condition{condition}, map[string]bool{ClusterConditionSecret: true})
		if err := r.Status().Update(ctx, cluster); err != nil {
			return ctrl.Result{}, err
		}
		return ctrl.Result{}, err
	}
```



## 缓存声明

当 Controller 的工作涉及到清理历史数据时，我们就需要在资源中缓存前一次调谐成功的声明信息，通常这些信息会被缓存到 Status 字段中。

**资源声明：**

```yaml
spec:
  destination:
    server: https://kubernetes.default.svc
  project: pipeline1
  source:
    path: tekton/overlays/production
    repoURL: ssh://git@gitlab.bluzin.io:2222/tenant1/management.git
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Status中的缓存**：

```yaml
status:
  sync:
    comparedTo:
      destination:
        server: https://kubernetes.default.svc
      source:
        path: tekton/overlays/production
        repoURL: ssh://git@gitlab.bluzin.io:2222/tenant1/management.git
        targetRevision: HEAD
    revision: 4f482dffe0683a0aa1c0acf85d167a9723bbdab8
    status: Synced
```



## 处理删除事件

通过判断 DeletionTimestamp 是否为零来决定调谐是否要进入删除流程。

由于 Controller 针对删除事件的调谐，只需要处理外部服务清理，不需要做语义校验或执行其他逻辑，所以调谐过程中一般会优先处理删除事件。

下面示例代码中对于查询资源的返回错误的处理方法，是使用 IgnoreNotFound() 函数包装了 err，这样处理的作用是当 err 值为 NotFound 时 IgnoreNotFound() 函数会返回 nil，也就是查不到指定资源时，调谐会正常终止。

```go
func (r *ClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	cluster := &clusterCRD.Cluster{}
	if err := r.Get(ctx, req.NamespacedName, cluster); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	if !cluster.DeletionTimestamp.IsZero() {
        // 删除逻辑
    }
}
```



## 管理终结器

终结器是 Controller 对资源第一次执行调谐时在资源的 metadata.finalizers 字段中添加，用来表示 Controller 在这个资源被删除前需要执行一些清理操作，一般情况下，对于一个资源，不同类型的 Controller 负责管理各自的终结器。

当 Controller 收到删除事件时，先判断在 metadata.finalizers 中是否有当前 Controller 所管理的终结器，如果有则执行外部服务的清理，之后移除相关终结器。如果资源中存在多个终结器，相关的 Controller 需要进行各自的清理外部服务和移除终结器的操作。

```go
	// 删除事件
	if !cluster.DeletionTimestamp.IsZero() {
        // 如果资源中没有当前 Controller 管理的终结器，则直接返回
		if !controllerutil.ContainsFinalizer(cluster, FinalizerName) {
			return ctrl.Result{}, nil
		}

        // 清理外部服务
		if err := r.deleteCluster(ctx, *cluster); err != nil {
			return ctrl.Result{}, err
		}

		newCluster := &clusterCRD.Cluster{}
		if err := r.Get(ctx, req.NamespacedName, newCluster); err != nil {
			return ctrl.Result{}, err
		}
        
        // 移除终结器并返回
		controllerutil.RemoveFinalizer(newCluster, FinalizerName)
		if err := r.Update(ctx, newCluster); err != nil {
			return ctrl.Result{}, err
		}
		return ctrl.Result{}, nil
	}

    // 添加终结器
	controllerutil.AddFinalizer(cluster, FinalizerName)
```



## 管理状态

如果我们把 Kubernetes 中的资源想象成是一个 API，那么资源的 Spec 字段就是 API 的入参，调谐过程就是 API 的业务逻辑，而 Status 字段则是 API 的返回值。Status 字段是用来表示 Controller 对资源的调谐结果，以及目标环境的当前状态，一般会存储历史调谐记录、最新调谐结果和报错信息、目标环境的当前属性、声明信息的缓存等等。Status 字段中可以是提供给“人”阅读的内容、也可以是提供给“程序”运行所需的内容。

**如果要存储调谐状态，可以直接使用 apimachinery 项目中的 Condition 类型。**

```go
// CodeRepoStatus defines the observed state of CodeRepo
type CodeRepoStatus struct {
    // 最新调谐结果
	// +optional
	Conditions []metav1.Condition `json:"conditions" yaml:"conditions"`
	// +optional
	ArgoStatus *SyncCodeRepo2ArgoStatus `json:"argoOperator" yaml:"argoOperator"`
}

type SyncCodeRepo2ArgoStatus struct {
    // 上一次成功调谐的声明信息的缓存
	LastSuccessSpec string      `json:"lastSuccessSpec" yaml:"lastSuccessSpec"`
	LastSuccessTime metav1.Time `json:"lastSuccessTime" yaml:"lastSuccessTime"`
    // 目标环境的当前属性
	Url             string      `json:"url" yaml:"url"`
	SecretID        string      `json:"secretID" yaml:"secretID"`
}
```



## 管理返回值

Controller 中的 Reconcile 函数可以通过返回值来控制调谐的调度过程。

Reconcile 函数的第二个返回值是 err，当 err 不为空时，Controller-Runtime 会按照默认的规则重新触发调谐，如果 Reconcile 函数一直返回 err，相关资源会进入降级状态。

Reconcile 函数的第一个返回值是 Result，如果返回一个空的 Result 对象，Controller-Runtime 则不会再主动触发调谐，如果 Result 中的 RequeueAfter 字段有值，Controller-Runtime 则会按照指定时间窗再次发送事件触发调谐。我们可以通过指定 RequeueAfter 的值来实现定期调谐，或实现特定的降级逻辑。

**自定义降级：**

```go
    if reconcileStatusAware, updateStatus := (obj).(apis.ReconcileStatusAware); updateStatus {
        lastUpdate := reconcileStatusAware.GetReconcileStatus().LastUpdate.Time
        lastStatus := reconcileStatusAware.GetReconcileStatus().Status

        status := apis.ReconcileStatus{
            LastUpdate: metav1.Now(),
            Reason:     issue.Error(),
            Status:     "Failure",
        }

        reconcileStatusAware.SetReconcileStatus(status)
        err := r.GetClient().Status().Update(context.Background(), runtimeObj)

        if err != nil {
            log.Error(err, "unable to update status")
            return reconcile.Result{
                RequeueAfter: time.Second,
                Requeue:      true,
            }, nil
        }

        if lastUpdate.IsZero() || lastStatus == "Success" {
            retryInterval = time.Second
        } else {
            // 当前时间减去前一次调谐时间，作为重试时长的计算单位
            retryInterval = status.LastUpdate.Sub(lastUpdate).Round(time.Second)
        }
    } else {
        log.Info("object is not RecocileStatusAware, not setting status")
        retryInterval = time.Second
    }

    return reconcile.Result{
        // 每次调谐失败重试时长翻倍，并设置最大重试时长
        RequeueAfter: time.Duration(math.Min(float64(retryInterval.Nanoseconds()*2), float64(time.Hour.Nanoseconds()*6)))
    }, nil
```



## 参考资料

- kubernetes operators best practices: <https://cloud.redhat.com/blog/kubernetes-operators-best-practices>
- kubebuilder faq: <https://book.kubebuilder.io/faq.html>