Repo to test ArgoCD Vault Plugin with OpenShift GitOps and the following backends:

1. Delinea Secret Server

# ArgoCD Vault Plugin

[ArgoCD Vault plugin](https://github.com/argoproj-labs/argocd-vault-plugin)
is the solution that ArgoCD community has come up to solve the issue of
secret management with GitOps in general.

This plugin can be used not just for secrets but also for deployments,
configMaps or any other Kubernetes resource.

## ArgoCD Config Management Plugins

It is implemented in the form of an a native [ArgoCD Config Management Plugin]
(https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/).

The way a Config Management Plugin (CMP) works is the following:

```The Argo CD "repo server" component is in charge of building Kubernetes manifests based on some source files from a Helm, OCI, or git repository. When a config management plugin is correctly configured, the repo server may delegate the task of building manifests to the plugin.```

## Defining ConfigManagementPlugins (CMP)

`ConfigManagementPlugin` [manifests](https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/#write-the-plugin-configuration-file) allow to define and configure plugins
in a declarative way. Altough they look like Kubernetes objects, they are not.

The most important parts of the manifest are the: `spec.generate.command` and `spec.discovery`.

* The `spec.generate.command` runs in the Application source directory each time manifests are generated.
Standard output must be ONLY valid Kubernetes Objects in either YAML or JSON. A non-zero exit code will fail manifest generation.

* The `spec.discover` is where discovery rules can be coded in order for the plugin
to be run. `spec.discover` definition is optional. If not discovery rules are defined,
the plugin would only be used when set explicitely in the Application manifest. For
an `spec.discover.find.command` in the repository's root directory. To match, it
must exit with status code 0 and produce non-empty output to standard out.

## Registering Plugin Sidecar

For ArgoCD to register a CMP plugin, the way is to run a sidecar container in the
repo server pod, that contains the CMP configuration.

One important aspect is that there has to be [one sidecar per CMP configuration]
(https://github.com/argoproj/argo-cd/discussions/12278#discussioncomment-4863196).

The ArgoCD operator allows to define sidecar containers for the repo server pod.

For each CMP plugin correctly registered in ArgoCD, a socket `*.sock` is created inside
the repo server pod:

```bash
oc get pods | grep repo
argocd-avp-repo-server-cf699746f-pzh82   4/4     Running   0          25m

oc rsh argocd-avp-repo-server-cf699746f-pzh82
Defaulted container "argocd-repo-server" out of: argocd-repo-server, avp, avp-kustomize, avp-helm, copyutil (init)

ls /home/argocd/cmp-server/plugins/

avp-helm.sock  avp-kustomize.sock  avp.sock
```

## Consuming CMP Plugins

Plugins can be explicitely specified in the [Application manifests](https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/#using-a-config-management-plugin-with-an-application), or
executed based on the discovery rules.

For example, in the following Application, the plugin is named explicitely:
```yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    path: guestbook
    plugin:
      name: avp-helm
```

When a plugin is not named explicitely in the Application manifest, it will be
executed only if any of the discovery rules match.

## Troubleshooting CMP

The best way is to look at the logs of the repo server and the sidecar containers.

For example, to see the logs of one of the plugins, we assuming the sidecar is
named `avp`, we can run the following command:

```bash
oc logs -c avp-helm argocd-avp-repo-server-cf699746f-pzh82
time="2023-12-04T15:03:51Z" level=info msg="ArgoCD ConfigManagementPlugin Server is starting" built="2023-10-30T19:24:59Z" commit=ebab8ec6259a6997fa3f310cddc539cb0c76b442 version=v2.8.4+ebab
8ec
time="2023-12-04T15:03:51Z" level=info msg="argocd-cmp-server v2.8.4+ebab8ec serving on /home/argocd/cmp-server/plugins/avp-helm.sock"
time="2023-12-04T15:13:51Z" level=info msg="Alloc=7852 TotalAlloc=16483 Sys=28029 NumGC=7 Goroutines=9"
time="2023-12-04T15:23:51Z" level=info msg="Alloc=7852 TotalAlloc=16491 Sys=28029 NumGC=12 Goroutines=9"
```
