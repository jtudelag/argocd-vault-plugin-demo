Repo to test ArgoCD Vault Plugin with OpenShift GitOps and the following backends:

1. Delinea Secret Server

# ArgoCD Vault Plugin (AVP)

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

NOTE: For disconected environments, it is advised to [build the AVP container image]
(https://argocd-vault-plugin.readthedocs.io/en/stable/installation/#custom-image-and-configuration-via-sidecar) in advance and push it to your private registry. The image should not only contain the AVP binary,
but any other tool required to render Kubernetes manifests such as kustomize, helm, etc.

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

# AVP Demo with Delinea Secret Server



Here we are going to show how to make AVP work with Delinea Secret Server (DSS).

## Deploy OpenShift GitOps Operator

1. First deploy OpenShift GitOps operator. You can use the following [repo](https://github.com/jtudelag/gitops-multicluster-onfiguration/tree/main/base/openshift-gitops-operator).

## Create User and Secret in DSS

1. Now, you need to create a user in DSS that has permissions to access it via API,
using username and password authentication. Unfortunately, AVP does not support
token based authentication for [DSS](https://argocd-vault-plugin.readthedocs.io/en/stable/backends/#delinea-secret-server).

2. Now create a secret that the user has permissions to read. For the shake of this
demo, create a simple "Usernam/Password" secret and populate it with some values.

3. Delinea Secret Server only allows to reference secrets by their ID, so note the ID
of your secret for later.

## Build the SidecarContainerImage

1. Now clone this repo, and go to the `buld/` folder.

```bash
cd https://github.com/jtudelag/argocd-vault-plugin-demo.git
cd build/
```

2. Build the container image using podman
```bash
podman build -f Containerfile_avp -t myregistry/myrepo/argocd-avp-plugin:1.17.0

```

3. Push it to your registry
```bash
podman push myregistry/myrepo/argocd-avp-plugin:1.17.0
```

## Deploy OpenShift GitOps with AVP plugins

1. Now clone the following repo [https://github.com/jtudelag/gitops-multicluster-onfiguration.git](https://github.com/jtudelag/gitops-multicluster-onfiguration.git), and go to the `base/openshift-gitops-instance-avp/` folder.

```bash
git clone https://github.com/jtudelag/gitops-multicluster-onfiguration.git
cd base/openshift-gitops-instance-avp/manifests/
```

2. Edit and populate the secret `avp-delinea-backend-secret.yaml` with DSS credentials.
Change the following variables:

* AVP_DELINEA_URL
* AVP_DELINEA_USER
* AVP_DELINEA_PASSWORD

```bash
cd base/openshift-gitops-instance-avp/manifests/

cp avp-delinea-backend-secret.yaml.sample avp-delinea-backend-secret.yaml

vi avp-delinea-backend-secret.yaml

---
apiVersion: v1
kind: Secret
metadata:
  name: avp-delinea-backend-secret
  namespace: argocd-avp-gitops
stringData:
  AVP_TYPE: delineasecretserver
  AVP_DELINEA_URL: https://XXXX
  AVP_AUTH_TYPE: userpass
  AVP_DELINEA_USER: XXXX
  AVP_DELINEA_PASSWORD: YYYY
type: Opaque
```

3. Replace the image `quay.io/jtudelag/argocd-avp-plugin:1.17.0` for all the sidecar containers using the one you just build.
```bash
cd base/openshift-gitops-instance-avp/manifests/
vi argocd-avp.yaml
```

4. Deploy ArgoCD with all the required config.
```bash
cd base/openshift-gitops-instance-avp/
kustomize build . | oc apply -f-
```

## Update Your Secrets

1. Fork and then clone this repo.
```bash
https://github.com/<YOUR_GITHUB_USER>/argocd-vault-plugin-demo.git
```

2. Change secrets ID in the secrets example folder. Yoi can look for all the ocurrences
anc hange those.
```bash
cd secrets_examples/
grep -R 'avp.kubernetes.io/path' *
helm/templates/helm-secret.yaml:    avp.kubernetes.io/path: '2'
kustomize/manifests/kustomize-secret.yaml:    avp.kubernetes.io/path: '2'
kustomize_with_helm/helm_secret/templates/helm-secret.yaml:    avp.kubernetes.io/path: '2'
simple/secret-example.yaml:    avp.kubernetes.io/path: '2'
```

3. Now go to the argocd-apps folder and change the repoURL with your repo URL.
```bash
grep repoURL *
app-helm-secret.yaml:    repoURL: 'https://github.com/jtudelag/argocd-vault-plugin-demo.git'
app-kustomize-secret.yaml:    repoURL: 'https://github.com/jtudelag/argocd-vault-plugin-demo.git'
app-kustomize-with-helm-secret.yaml:    repoURL: 'https://github.com/jtudelag/argocd-vault-plugin-demo.git'
app-simple-secret.yaml:    repoURL: 'https://github.com/jtudelag/argocd-vault-plugin-demo.git'
```

4. Push all the changes to Github.

5. Now crate all the ArgoCD applications.
```bash
oc apply -f argocd-apps/
```

6. You should see all your ArgoCD Apps healthy, and the following secrets just created:
```bash
oc get app
NAME                         SYNC STATUS   HEALTH STATUS
helm-secret                  Synced        Healthy
kustomize-secret             Synced        Healthy
kustomize-with-helm-secret   Synced        Healthy
simple-secret                Synced        Healthy

oc get secret | tail -4
helm-secret                                                Opaque                                2      65m
kustoize-with-helm-secret                                  Opaque                                2      65m
kustomize-secret                                           Opaque                                2      65m
simple-secret                                              Opaque                                2      65m
```

7. Make sure the secret values from DSS are really populated into your Kubernetes secrets!
ArgoCD application might be healthy and the Kubernetes secrets created, and still the data might not
be populated.

```bash
oc extract secret/helm-secret --to=-
oc extract secret/kustoize-with-helm-secret --to=-
oc extract secret/kustomize-secret --to=-
oc extract secret/simple-secret --to=-
```
