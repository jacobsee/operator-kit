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

The Kubernetes control plane is, at its heart, an API server. We can see this in the definition above, because we have to define _where_ the resource that we're about to define lives. In this case, the `Deployment` object that we're defining lives in `apiVersion: apps/v1`. The version `apps/v1` is built in to newer versions of Kubernetes, so we don't have to do any extra work to help the API server know what we're talking about - it's already there. The API server, recognizing that it knows how to handle `Deployment` resources in `apps/v1`, stores this object in [its internal database, etcd](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/).

Storing this `Deployment` object in etcd is fantastic, but the API server itself actually _does not know_ how to take the definition here, and perform the actions necessary to make sure that the cluster state matches what we asked for. This is the job of another component, called the [_deployment controller_](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go). The deployment controller is an application internal to Kubernetes which is constantly running, connected to the API server via a long-lived change notification feed called a __watch__ - specifically, you would say that it "has a watch on `Deployment` objects", since watches are specific to different kinds of resources. Since the deployment controller has a watch on `Deployment` objects, it is immediately notified by the API server whenever a `Deployment` object is created, modified, or deleted. The deployment controller has all the logic required to make sure that the cluster state is as it should be whenever a change to a `Deployment` object happens.

To recap, when we apply a `Deployment` object to the cluster:

1. The Kubernetes API server, knowing how to handle `apps/v1` `Deployment` objects, stores the object in its database - etcd.
2. The Kubernetes API server, recognizing that an app (the deployment controller) has a watch on `Deployment` objects, sends a message to the connected deployment controller informing it that a change has occurred.
3. The deployment controller reads the new desired state of the cluster from the newly-applied `Deployment` object. It computes the difference between this desired state, and the observed current state of the cluster.
4. Knowing the difference between the current cluster state and the desired cluster state, the deployment controller performs any actions required to bring the cluster state in sync with the desired state.