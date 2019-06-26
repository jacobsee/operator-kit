# Using Operator-SDK

This section is going to be relatively thin. This is because the usage of operator-sdk CLI is already well-documented. The following doc should be considered required reading for moving forward.

[https://github.com/operator-framework/getting-started](https://github.com/operator-framework/getting-started)

Nevertheless, the core steps are these:

## Create your project

```sh
> operator-sdk new <name-of-your-operator>
```

for instance,

```sh
> operator-sdk new memcached-operator
```

This should initialize a project for you, which you can then `cd` into. It also runs `dep ensure` (`dep` is a Golang package manager) to make sure that all dependencies are available.

## Add a Custom Resource Definition (CRD) to your project

```sh
> operator-sdk add api --api-version=<your-api-version> --kind=<your-crd-name>
```

such as 

```sh
> operator-sdk add api --api-version=cache.example.com/v1alpha1 --kind=Memcached
```

Normally, your API version should look like a domain with a Kube-style version in the path, and your `kind` should be capitalized.

Among other thing, this creates a file that defines the contents and types of the `spec` and `status` fields that your Custom Resource objects will have. In this case, our file was created at `pkg/apis/cache/v1alpha1/memcached_types.go`.

```Golang
type MemcachedSpec struct {
}
type MemcachedStatus struct {
}
```

If you'd like your CR to have an input `size` and an output `message`, you can define that using Go types like this, as an example.

```Golang
type MemcachedSpec struct {
    Size int32 `json:"size"`
}
type MemcachedStatus struct {
    Message string `json:"message"`
}
```

Operator-SDK actually _requires_ that the first character of the type definition be capitalized, and the first character of the JSON representation be lower case. The case of the rest of the letters _must be the same_ for the automated tooling to work!

Anytime you edit the type definition, you _must_ rerun the code generator:

```sh
> operator-sdk generate k8s
```

## Add a controller to your project

After completing the above steps, we've defined a CRD for our operator, but have not implemented any logic for it. Almost anytime you create a CRD in your project, you will also need a _controller_, which is logic that backs the CRD. We'll continue using the Memcached example.

```sh
> operator-sdk add controller --api-version=cache.example.com/v1alpha1 --kind=Memcached
```

This creates a file at `pkg/controller/memcached/memcached_controller.go`. Importantly, this file contains the _reconcile loop_.

TODO: Flesh out reconcile loop section.

## Trying it all out

There are multiple paths forward that we can take here.

To try your operator out quickly, head over to [trying it out](trying-it-out.md).

To incorporate your operator into a CI/CD pipeline, visit [operator pipelines](operator-pipelines.md).
