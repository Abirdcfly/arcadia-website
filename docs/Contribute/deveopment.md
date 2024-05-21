---
sidebar_position: 2
title: Create an Operator Project
sidebar_label: Create an Operator Project
---
# Create an Operator Project

## Init a new operator project

```bash
bash hack/install-operator-sdk

operator-sdk init --domain kubeagi.k8s.com.cn --component-config true --owner kubeagi --project-name arcadia --repo github.com/kubeagi/arcadia
```

## Create a CRD

```bash
operator-sdk create api --resource --controller --namespaced=true --group arcadia --version v1alpha1 --kind Laboratory
```

### Regenerate after changes on CRD

```bash
make generate && make manifests
```

### Base controller reconcile

`// Note:` For ease of understanding only, it does not need to be written to the source file

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

### Update helm dependency

If you update the helm dependency of deploy/charts/arcadia package, you should run `helm dependency update`, it'll update `Chart.lock`, or the helm package will fail at build stage.

## Update GraphQL

arcadia uses both RESTAPI and GraphQL in apiserver, using RESTAPI for the chat API to talk to the LLM and for uploading files, and GraphQL for Create, Read, Update, and Delete resources.

GraphQL is created as follows:

### Create `.graphqls`

First create a [GraphQL schema](https://graphql.org/learn/schema/) file with the suffix `graphqls` in the [schema](https://github.com/kubeagi/arcadia/blob/main/apiserver/graph/schema) directory.

### Update `.go`

We use the [gqlgen](https://github.com/99designs/gqlgen) project to automatically generate the template go code from the graphqls file, before generating it we need to update the [gqlgen configuration file](https://github.com/kubeagi/arcadia/blob/main/gqlgen.yaml), then execute the command `make gql-gen` and it will automatically generate the template go code in [generated](https://github.com/kubeagi/arcadia/tree/main/apiserver/graph/generated) and[impl](https://github.com/kubeagi/arcadia/tree/main/apiserver/graph/impl) folders. Generally, we need to update the go template file `xxx.resolvers.go` in [impl](https://github.com/kubeagi/arcadia/tree/main/apiserver/graph/impl) folder to add our business logic. In the current project, we do this by adding the `xxx` directory to [pkg](https://github.com/kubeagi/arcadia/tree/main/apiserver/pkg) to write the specific business logic, and the `xxx.resolvers.go` file only references these functions.

At this point, the GraphQL server-side part is complete. The following steps need to be used together with the [yuntijs](https://github.com/yuntijs/bff-sdk-generator) project to facilitate low-code front-end yuntijs auto-generation of SDK for front-end development.

### Create `.gql`

First create a file in the [schema](https://github.com/kubeagi/arcadia/blob/main/apiserver/graph/schema) directory with the suffix `gql`, containing the contents of common query and mutation requests for resources.

### Generate YuntiJS SDK

In development, after the above changes are submitted to the pull request, and after code review, during the code merge process, [GitHub action](https://github.com/kubeagi/arcadia/actions/workflows/build_bff_sdk.yaml) will automatically execute the generation process and push the result to [npm](https://www.npmjs.com/package/@yuntijs/arcadia-bff-sdk), and the front-end can use the SDK by referencing `@yuntijs/arcadia-bff-sdk`. For manual execution, just run `make sdk` in your environment.
