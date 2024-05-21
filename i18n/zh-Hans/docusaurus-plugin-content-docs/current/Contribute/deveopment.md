---
sidebar_position: 2
title: 创建一个 Operator 项目
sidebar_label: 创建一个 Operator 项目
---
# 创建一个 Operator 项目

## 初始化一个 Operator 项目

```bash
bash hack/install-operator-sdk

operator-sdk init --domain kubeagi.k8s.com.cn --component-config true --owner kubeagi --project-name arcadia --repo github.com/kubeagi/arcadia
```

## 创建一个 CRD

```bash
operator-sdk create api --resource --controller --namespaced=true --group arcadia --version v1alpha1 --kind Laboratory
```

### 在 CRD 发生变化后重新生成

```bash
make generate && make manifests
```

### 基础控制器协调

`// 注意:` 为便于理解，无需将其写入源文件。

```go
func (r *KnowledgeBaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (result ctrl.Result, err error) {
	logger := ctrl.LoggerFrom(ctx)
	logger.V(1).Info("Start KnowledgeBase Reconcile") // Note: V(1) means debug log level
	kb := &arcadiav1alpha1.KnowledgeBase{}
	if err := r.Get(ctx, req.NamespacedName, kb); err != nil {
		// There's no need to requeue if the resource no longer exists.
		// Otherwise, we'll be requeued implicitly because we return an error.
		logger.V(1).Info("Failed to get KnowledgeBase")
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}
	logger = logger.WithValues("Generation", kb.GetGeneration(), "ObservedGeneration", kb.Status.ObservedGeneration, "creator", kb.Spec.Creator) // Note: add log value is optional
	logger.V(1).Info("Get KnowledgeBase instance")

	// Add a finalizer.Then, we can define some operations which should
	// occur before the KnowledgeBase to be deleted.
	// More info: https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers
	if newAdded := controllerutil.AddFinalizer(kb, arcadiav1alpha1.Finalizer); newAdded {
		logger.Info("Try to add Finalizer for KnowledgeBase")
		if err = r.Update(ctx, kb); err != nil {
			logger.Error(err, "Failed to update KnowledgeBase to add finalizer, will try again later")
			return ctrl.Result{}, err
		}
		logger.Info("Adding Finalizer for KnowledgeBase done")
		return ctrl.Result{Requeue: true}, nil
	}

	// Check if the KnowledgeBase instance is marked to be deleted, which is
	// indicated by the deletion timestamp being set.
	if kb.GetDeletionTimestamp() != nil && controllerutil.ContainsFinalizer(kb, arcadiav1alpha1.Finalizer) {
		logger.Info("Performing Finalizer Operations for KnowledgeBase before delete CR")
		// TODO perform the finalizer operations here, for example: remove vectorstore data?
		logger.Info("Removing Finalizer for KnowledgeBase after successfully performing the operations")
		controllerutil.RemoveFinalizer(kb, arcadiav1alpha1.Finalizer)
		if err = r.Update(ctx, kb); err != nil {
			logger.Error(err, "Failed to remove finalizer for KnowledgeBase")
			return ctrl.Result{}, err
		}
		logger.Info("Remove KnowledgeBase done")
		return ctrl.Result{}, nil
	}
	
    // Note: do you logic here

	return result, err
}
```

### 更新 helm 依赖项

如果更新了 deploy/charts/arcadia 软件包的 helm 依赖关系，则应运行 `helm dependency update`，它将更新 `Chart.lock`，否则 helm 软件包将在构建阶段失败。

## 更新 GraphQL

arcadia 在 apiserver 中同时使用 RESTAPI 和 GraphQL，和大模型对话的 chat 接口和上传文件接口使用 RESTAPI，对资源进行增删改查的接口使用 GraphQL。

GraphQL 的创建方式如下：

### 创建 `.graphqls` 文件

首先在 [schema](https://github.com/kubeagi/arcadia/blob/main/apiserver/graph/schema) 目录创建后缀为 `graphqls` 的 [GraphQL schema](https://graphql.org/learn/schema/) 文件。

### 更新 `.go` 模版文件

我们使用 [gqlgen](https://github.com/99designs/gqlgen) 项目从 graphqls 文件自动生成模版 go 代码，在生成之前，我们需要更新 [gqlgen 配置文件](https://github.com/kubeagi/arcadia/blob/main/gqlgen.yaml)，然后执行命令 `make gql-gen` ，就会自动在 [generated](https://github.com/kubeagi/arcadia/tree/main/apiserver/graph/generated) 和 [impl](https://github.com/kubeagi/arcadia/tree/main/apiserver/graph/impl) 文件夹中生成需要的 go 模版文件，一般我们需要在 [impl](https://github.com/kubeagi/arcadia/tree/main/apiserver/graph/impl) 文件夹中的 `xxx.resolvers.go` 文件中增加我们的业务逻辑。在当前项目中，我们的做法是在 [pkg](https://github.com/kubeagi/arcadia/tree/main/apiserver/pkg) 中增加 xxx 目录来编写具体的业务逻辑，`xxx.resolvers.go` 文件只引用上述逻辑的函数。

至此，GraphQL 服务端部分更新完成。下面的步骤需要和 [yuntijs](https://github.com/yuntijs/bff-sdk-generator) 项目一起使用，方便低代码前端 yuntijs 自动生成 SDK 来方便前端开发。

### 创建 `.gql` 文件

首先在 [schema](https://github.com/kubeagi/arcadia/blob/main/apiserver/graph/schema) 目录创建后缀为 `gql` 的文件, 包含了对资源常见的 query 和 mutation 请求内容。

### 生成 YuntiJS 低代码 SDK 文件

在开发中，上述改动提交 pull request 后，经过代码 review，合并代码过程中，[Github action](https://github.com/kubeagi/arcadia/actions/workflows/build_bff_sdk.yaml) 将自动执行生成过程，并推送结果到 [npm](https://www.npmjs.com/package/@yuntijs/arcadia-bff-sdk) 中，前端通过引用 `@yuntijs/arcadia-bff-sdk` 即可使用该 SDK。如果是手动执行，只需要在环境中执行 `make sdk` 即可。
