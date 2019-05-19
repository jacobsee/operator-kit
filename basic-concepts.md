# Basic concepts

## What is a Kubernetes Operator?

This question requires a basic understanding of how Kubernetes itself manages resources. At its core, Kubernetes takes resources (typically written in human-readable YAML files, but could also be JSON), and attempts to ensure that the _actual state_ of the cluster matches the _desired state_ defined in those YAML files. For instance, one might define a `deployment` resource:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

When this resource is applied to the cluster, Kubernetes should fetch the `nginx:1.7.9` container image, and spin up three pods each running NGINX and exposing port 80 as configured above. But _how exactly_ does Kubernetes do this? Let's break down the process just a bit.

The Kubernetes control plane is, at its heart, an API server. We can see this in the definition above, because we have to define _where_ the resource that we're about to define lives. In this case, the `Deployment` object that we're defining lives in `apiVersion: apps/v1`, which is called the __API Group & Version__. The group+version `apps/v1` is built in to newer versions of Kubernetes, so we don't have to do any extra work to help the API server know what we're talking about - it's already there. The API server, recognizing that it knows how to handle `Deployment` resources in `apps/v1`, stores this object in [its internal database, etcd](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/).

Storing this `Deployment` object in etcd is fantastic, but the API server itself actually _does not know_ how to take the definition here, and perform the actions necessary to make sure that the cluster state matches what we asked for. This is the job of another component, called the [_deployment controller_](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go). The deployment controller is an application internal to Kubernetes which is constantly running, connected to the API server via a long-lived change notification feed called a __watch__ - specifically, you would say that it "has a watch on `Deployment` objects", since watches are specific to different kinds of resources. Since the deployment controller has a watch on `Deployment` objects, it is immediately notified by the API server whenever a `Deployment` object is created, modified, or deleted. The deployment controller has all the logic required to make sure that the cluster state is as it should be whenever a change to a `Deployment` object happens.

To recap, when we apply a `Deployment` object to the cluster:

1. The Kubernetes API server, knowing how to handle `Deployment` objects in the `apps/v1` version group, stores the object in its database - etcd.
2. The Kubernetes API server, recognizing that an app (the deployment controller) has a watch on `Deployment` objects, sends a message to the connected deployment controller informing it that a change has occurred.
3. The deployment controller reads the new desired state of the cluster from the newly-applied `Deployment` object. It computes the difference between this desired state, and the observed current state of the cluster.
4. Knowing the difference between the current cluster state and the desired cluster state, the deployment controller performs any actions required to bring the cluster state in sync with the desired state.

An operator is any custom implementation of __exactly this same behavior__.

The Kubernetes API is _extensible_, which is to say that it can be made aware of new resource types, through something called a __Custom Resource Definition__, or CRD. A CRD looks something like this:

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: foos.samplecontroller.k8s.io
spec:
  group: samplecontroller.k8s.io
  version: v1alpha1
  names:
    kind: Foo
    plural: foos
  scope: Namespaced
```

This CRD makes a new resource group and version, called `samplecontroller.k8s.io/v1alpha1`; and in that group and version, a resource type with a `Kind` of `Foo`. Importantly, this CRD __does not__ actually instantiate a resource of kind `Foo`. It simply makes the kind `Foo` available for use later on. If you have a background in object-oriented programming, you can think of a CRD as a class definition. Just because you have a class definition does not mean that you have an instance of the class - it just means that you won't have any type errors as you continue to develop your app. After this CRD has been applied to the cluster, your favorite cluster management command-line tool (whether it be `kubectl` or `oc`) will be able to work with it as a first-class citizen. `oc get foos`, `oc describe foo <foo-name>`, `oc delete foo <foo-name>` are all valid commands.

As previously stated, applying a CRD to the cluster does not actually instantiate a `Foo` in the cluster. We need some more YAML for that - an actual definition of a `Foo` object:

```yaml
apiVersion: samplecontroller.k8s.io/v1alpha1
kind: Foo
metadata:
  name: example-foo
spec:
  deploymentName: example-foo
  replicas: 1
```

Notice how we needed to tell the Kubernetes API server that we wanted to use our new API group and version, `samplecontroller.k8s.io/v1alpha1`, and that we wanted a resource of kind `Foo`. At this point, we have both a custom resource definition, and a custom resource itself. Going back to our object-oriented programming analogy, the custom resource is an object - an instantiation of the class that we've previously defined. It also has some new values defined. It has a name, which lives in `metadata.name`. Kubernetes uses this to identify the object to human users later. It also has two values defined in its `spec`, `deploymentName` and `replicas`, which are arbitrary values. The contents of the `spec` field of a custom resource are totally up to the developer to define - Kubernetes does not hold you to any contracts. You can think of the `spec` field as the instance variables of your object. Whatever you need to know in order to make your object function should go in there.

All of this is only _one half_ of what you need to make your operator useful. CRDs and CRs don't help anybody if there's no logic in there to act on them! To make that happen, we will use Operator-SDK and Golang, which will be introduced in depth in later sections. Suffice it to say - we will need to build an app. It will live in a `Deployment` on your cluster just like any other app, but will have some properties and special permissions that allow it to communicate directly with the Kubernetes API server to get its job done.