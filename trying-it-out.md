# Trying it out

This section is called "trying it out" because neither option presented here should really be used for long-term software development in an OpenShift environment. We don't go into testing, security scanning, working on an operator as a team, etc.

Any techniques for testing your operator listed on this page will also require your cluster to have the CRD applied to it, which requires `cluster-admin` privileges by default.

## Local development with Operator-SDK

Operator-SDK can actually run an operator locally, while pointed at a remote cluster.

You need to define your operator name as an environment variable first:

```sh
> export OPERATOR_NAME=memcached-operator
```

Then, just use the `up local` subcommand to run the operator on your computer:

```sh
> operator-sdk up local --namespace=<my-namespace>
```

Note that you _will not_ see a `Deployment` or `DeploymentConfig` for your operator in this case. It will not appear to be running in the cluster's web UI. If you create a CR in your cluster, the operator running on your computer will react to it, and you'll see any associated logs in your terminal.

## Building the image locally

Operator-SDK can use your local Docker installation on your computer to build an image for your operator, which you can then push to any registry and deploy any way you wish.

```sh
> operator-sdk build quay.io/example/memcached-operator:v0.0.1
```
