# repo per app

Following the model described by fluxcd [here](https://fluxcd.io/docs/guides/repository-structure/#repo-per-app)

Applications are developed and mantained in their own repo (which is free to
have many apps). The fluxcd config follows the cluster/tenant model with each
tenant being a seperately permissioned repository for configuring a group of
related application workloads. The relationship is not defined here - it can be
"apps for a product" "apps for a team" "special workloads for a particular
developer" and so on. A cluster tenant is just a cluster owned and controled
collection of rbac rules and namespaces.

This cluster / tenant config seperation is primarily about enabling privilege
separation between cluster wide operations and operations that naturaly scope
to a specific project, team or purpose.


## tenants, environments, developers, applications and namespace allocation

### Summary

* Cluster tenants are simply a unit of configuration management for the
  cluster.
* A cluster tenant is typically a project or team which *has* environments (prod/stage/dev etc)
* The cluster config defines and enables tenant repositories
* The cluster config defines and creates the namespaces for the tenants
* A tenant config repository refers to on or more application repositories
* For each application source the tenant config sets the targetNamespace so the
  imported application manifests are rendered for the correct namespace.
* Developers are considered just a special case of environment. Eg prod stage dev
  dev-$USER though it is entirely feasible to give a user their own cluster
  tenant for special purposes.

### Discussion

The flux cluster repository adds the tenant reference (refering to this repo) and
allocates the namespaces for each supported environment / config. The tenant config
repo (this one) imports the app (or apps) manifests via another flux source.
The app importing source reference (from this repo) must set the targetNamespace on the source.

The source itself is defined in this repo (in sync.yaml) and a
fluxcd/Kustomization is used to patch the sources path *per* environment.


The app repositories do not declare or assign namespaces in their base path.
The fluxcd configuration must arrange to do this to make the manifests
applicable.

The tenant repo is responsible for the reference (or references) to the
application workloads for the tenant. Each application a GitRepository source
and an accompanying fluxcd/Kustomization. That fluxcd/Kustomization (from the
app config base in this repo) is itself patched in the environment/ subpath to
set the appropraite targetNamespace for the environment

### Namespace conventions

Environments are prod/stage/dev/user-$USER

<env>-<tenant-namespaces>-<app>
prod-iona-console-iona-console-iona-app

A specific envirnonment has at least one namespace. And may have many depending
on the needs of the tenant and its applications. For a single application
workspace for a tenant called 'iona-console' the prod namespace would be

prod-iona-console

If tenant iona-console had multiple apps *and* if those apps needed seperate
namespaces, they would look like

prod-iona-console-app1
prod-iona-console-app2

For a namespace per app setup like that, again if necessary, a base shared
namespace would be just prod-iona-console

This allows for further apps, that don't have namespace segregation
requirements, to be placed in just 'prod-iona-console'

## Patching app resources

To patch the manifests of the target app it is necessary to update the
fluxcd/Kustomization that *builds* the apps kustomizations. The
kubernetes/Kustomization  in the config repository cannot directly apply
patches to the manifests refered to by the source

A common confusion here stems from fluxcd's (odd) decision to define there own
kind: Kustomization under their apiVersion: kustomize.toolkit.fluxcd.io/v1beta2

Manifests of the form:

apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization

Are *flux* CRD configurations which modify how the manifests in the configured sources are
built.

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

Are kustomize manifests which *define* the configuration. The configuration
determins the sources and the patches to apply to them during kustomize build.
So it is typical for the config repo to use k8s/Kustomization's to PATCH
fluxcd/Kustomization's such that the referenced manifest repositor is patched.

Ie the standard model is to PATCH the desired patches into the
fluxcd/Kustomization's
