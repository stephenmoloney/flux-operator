Operator for creating and managing instances of [Weaveworks flux](https://github.com/weaveworks/flux), [helm-operator](https://github.com/weaveworks/flux/blob/master/site/helm/helm-integration.md) and [tiller, the cluster component of Helm](https://github.com/kubernetes/helm).

Use-cases:

* Doing GitOps without a monorepo. You can easily split your manifests into repos per team, project or namespace.
* Simplify the deployment of your Flux, Tiller, and helm-operator instances.
* Easily manage Flux RBAC policies to prevent any single Flux instance from having access to the entire cluster.
* Use Helm without violating your RBAC policies.

# Installation

To deploy to your cluster:

```
kubectl apply -f deploy/flux-crd.yaml
kubectl apply -f deploy/fluxhelmrelease-crd.yaml
kubectl apply -f deploy/k8s.yaml
```

# Creating a Flux instance

To create a Flux instance, create a `Flux` CR:

```
apiVersion: flux.codesink.net/v1alpha1
kind: Flux
metadata:
  name: example
spec:
  namespace: default
  gitUrl: ssh://git@github.com/justinbarrick/manifests
  role:
    enabled: true
  args:
    connect: "ws://fluxcloud/"
```

This will create a flux pod called `flux-example` in the `default` namespace.

Settings:

* `namespace`: the namespace to deploy flux to.
* `gitUrl`: the URL to git repository to clone (required).
* `gitBranch`: the git branch to use (default: `master`).
* `gitPath`: the path with in the git repository to look for YAML in (default: `.`).
* `gitPollInterval`: the frequency with which to fetch the git repository (default: `5m0s`).
* `gitSecret`: the Kubernetes secret to use for cloning, if it does not exist it will
               be generated (default: `flux-$name-git-deploy` or `$GIT_SECRET_NAME`).
* `fluxImage`: the image to use for flux (default: `quay.io/weaveworks/flux` or `$FLUX_IMAGE`).
* `fluxVersion`: the version to use for flux (default: `1.4.0` or `$FLUX_VERSION`).
* `clusterRole.enabled`: if enabled, a cluster role will be assigned to the service
                         account (default: `false`).
* `clusterRole.rules`: the list of rbac rules to use (default: full access to all resources).
* `role.enabled`: if enabled, a role will be assigned to the service
                  account (default: `false`).
* `role.rules`: the list of rbac rules to use (default: full access to all resources in the namespace).
* `tiller.enabled`: whether or not to deploy a tiller instance in the same namespace (default: false).
* `tiller.tillerImage`: the image to use with tiller (default: `gcr.io/kubernetes-helm/tiller` or `$TILLER_IMAGE`)
* `tiller.tillerVersion`: the image version to use with tiller (default: `v2.9.1` or `$TILLER_VERSION`)
* `args`: a map of args to pass to flux without `--` prepended.
* `helmOperator.enabled`: whether or not to deploy a helm-operator instance in the same namespace (default: false).
* `helmOperator.helmOperatorImage`: the image to use with helm-operator (default: `quay.io/weaveworks/helm-operator` or `$HELM_OPERATOR_IMAGE`).
* `helmOperator.helmOperatorVersion`: the image version to use with helm-operator (default: `master-1dfdc61` or `$HELM_OPERATOR_VERSION`).
* `helmOperator.chartPath`: the chart path to use with Helm Operator (default: `.`).
* `helmOperator.gitPollInterval`: the frequency with which to sync Git and the charts (default: the flux `gitPollInterval` or, if not set, `3m0s`).
* `helmOperator.gitUrl`: the URL of the git repository to use if it is different than the primary flux `gitUrl`.

You can also override some of the defaults by setting environment variables on the
operator itself:

* `GIT_SECRET_NAME`: the git secret name to use.
* `FLUX_IMAGE`: the default image to use for flux.
* `FLUX_VERSION`: the default version to use for flux.
* `HELM_OPERATOR_IMAGE`: the default image to use for helm-operator.
* `HELM_OPERATOR_VERSION`: the default version to use for helm-operator.
* `MEMCACHED_IMAGE`: the default memcached image.
* `MEMCACHED_VERSION`: the default memcached version.
* `TILLER_IMAGE`: the default tiller image.
* `TILLER_VERSION`: the default tiller version.

# Git SSH key

If you already have an SSH key to use with flux, then add it as a secret to Kubernetes:

```
kubectl create secret generic mysecret --from-file=identity=/home/user/.ssh/id_rsa
```

You can then reference the secret as `gitSecret: mysecret` in your Flux YAML.

To get the SSH public key from the SSH key you are using run:

```
ssh-keygen -y -f <(kubectl get secret -o 'go-template={{ .data.identity }}' flux-example-git-deploy |base64 -d)
```

You can then paste this into your Github deploy keys.

# Creating a Tiller instance

The flux-operator can also deploy Tiller with Flux into your namespace. The Tiller
instance uses the same service account as Flux.

To enable, set `tiller.enabled` to true:

```
apiVersion: flux.codesink.net/v1alpha1
kind: Flux
metadata:
  name: example
spec:
  namespace: default
  gitUrl: ssh://git@github.com/justinbarrick/manifests
  role:
    enabled: true
  tiller:
    enabled: true
```

You can now securely use Helm in this namespace:

```
helm --tiller-namespace default ls
helm --tiller-namespace default install stable/memcached
```

## Migrating Tiller

If you already have a default Tiller installation, you can easily start managing
it with the flux-operator:

```
apiVersion: flux.codesink.net/v1alpha1
kind: Flux
metadata:
  name: flux
spec:
  namespace: kube-system
  gitUrl: ssh://git@github.com/justinbarrick/manifests
  clusterRole:
    enabled: true
  tiller:
    enabled: true
```

Save this as `flux.yaml`, delete the old `tiller-deploy` and apply the new one:

```
kubectl delete deployment -n kube-system tiller-deploy
kubectl apply -f ./flux.yaml
```

You should now be able to run `helm ls` and see all of your old deployments.

# Helm Operator

Along with Tiller, it is possible to deploy the Helm Operator. The Helm operator currently
requires access to all namespaces, so a `clusterRole` must be set. This means you should currently
only enable the Helm Operator on one flux.

If Tiller does not already exist in the namespace, also set `tiller.enabled: true`.

```
apiVersion: flux.codesink.net/v1alpha1
kind: Flux
metadata:
  name: example
  namespace: default
spec:
  gitUrl: ssh://git@github.com/justinbarrick/manifests
  clusterRole:
    enabled: true
  tiller:
    enabled: true
  helmOperator:
    enabled: true
```

You should then be able to create a FluxHelmRelease. See the [helm-operator example](https://github.com/weaveworks/flux-helm-test) for more information.

# RBAC

By default, a service account is created and given "get", "watch", "list", permissions on
all namespaces and assigned to the flux pod. The service account is called
`flux-$fluxname`.

You can enable both a role and a cluster role on the service account.

To enable a cluster role, set `clusterRole.enabled`. The default cluster role created
will grant access to all resources in the cluster.

To enable a role, set `role.enabled`. The default role created will grant access to all
resources in the namespace.

You can also set custom RBAC rules by setting `role.rules` or `clusterRole.rules`:

```
apiVersion: flux.codesink.net/v1alpha1
kind: Flux
metadata:
  name: example
  namespace: default
spec:
  gitUrl: ssh://git@github.com/justinbarrick/manifests
  clusterRole:
    enabled: true
    rules:
      - apiGroups: ['*']
        resources: ['*']
        verbs: ['*']
      - nonResourceURLs: ['*']
        verbs: ['*']
```

See [Kubernetes role documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole) for more information.
