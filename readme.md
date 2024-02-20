# [TURORIAL] - Integrating kubernetes dashboard with R/O access only with  Helm.
This setup may be handy if you need to deploy with helm and provide proper R/O access RBAC. For instance, if you want your developers to check logs only, or look visually at workloads.

## Requirements:
Helm 
Kubernetes cluster (no specific vendor/version. In this guide I am using V1.28)

## Customizing helm chart

- Adding the chart to the repo locally available:
`helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/`

- Downloading and unzipping the chart:
`helm fetch --untar kubernetes-dashboard/kubernetes-dashboard`
Running this command will show a folder with the helm chart content.

## Preparing RBAC

Any file we include in the `template` folder inside the helm chart structure will be applied.
For this, we will provision our RBAC-related files, and place them in our helm chart `template` folder.

- clusterRole: This object will "list" the resources, 
  
``` yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name:  k8s-dashboard-read
rules:
- apiGroups: ["", "apps", "extensions", "batch", "autoscaling"]
  resources: ["deployments", "replicasets", "statefulsets", "daemonsets", "jobs", "cronjobs", "replicationcontrollers", "pods", "services", "configmaps", "secrets", "persistentvolumes", "persistentvolumeclaims, namespaces]
  verbs: ["get", "list"]
- apiGroups: ["events.k8s.io"]
  resources: ["events"]
  verbs: ["get", "list"]
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  verbs: ["get", "list"]

```

This clusterRole is enabling the apigroups that correspond to the objects the dashboard is sharing.
  
- RoleClusterBinding: 

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8s-dashboad-read-cr
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: k8s-dashboard-read
subjects:
- kind: ServiceAccount
  name: k8s-dashboard-user
  namespace: kubernetes-dashboard

```
This object `clusterRoleBinding` is binding the service account (to be created next) to our role. Please note this is a cluster scoped object.

- Service Account: 

``` yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8s-dashboard-user
  namespace: kubernetes-dashboard

```

- Finally we want to provision a secret, that will provide a security bearer token, we will use it to log into the dashboard later.

``` yaml

apiVersion: v1
kind: Secret
metadata:
  name: k8s-dashboard-read-user
  namespace: kube-system
  annotations:
    kubernetes.io/service-account.name: "k8s-dashboard-user"   
type: kubernetes.io/service-account-token  #note we are using the type sa-token to generate an actual token

```

We can place our recently created files in the templates folder. Our next step is literally testing out by installing in our cluster

## Installing our Helm chart:

Issue the following command while connected to your cluster (while in your chart's folder). This command will run as a test, and by using the --dry-run flag we ensure it does not apply, so we can review the manifests we are rendering:

`helm upgrade --install kubernetes-dashboard . --namespace you-dashboard-namespace --history-max 3 -f values.yaml --debug --dry-run`


Once we double-check these manifests, simply run the same command without the dry-run:

``helm upgrade --install kubernetes-dashboard . --namespace you-dashboard-namespace --history-max 3 -f values.yaml --debug`

## Accessing the Dashboard:

The output after installing this chart will give you commands to port-forward the pod running the service: (will be similar to the below, it is just a example)

``` bash

export POD_NAME=$(kubectl get pods -n kube-system -l "app.kubernetes.io/name=kubernetes-dashboard,app.kubernetes.io/instance=kubernetes-dashboard" -o jsonpath="{.items[0].metadata.name}")
  echo https://127.0.0.1:8443/
  kubectl -n kube-system port-forward $POD_NAME 8443:8443
https://127.0.0.1:8443/

```
Let the command running, and copy your bearer token from another terminal:

 `k get secret k8s-dashboard-read-user -o jsonpath={".data.token"} | base64 -d | pbcopy` #pbcopy may not work if you are not using a MAC, just remove it if you use any other unix distro.

Now simply access your localhost address https://127.0.0.1:8443/ in a web browser, and select bearer token authentication.

Now, you should be able to review the dashboard content, only with R/O. For admin-acces, simple provision yaml files that match the access you require.



## Glosary:

`helm upgrade --install kubernetes-dashboard`: Commonly, the flags `upgrade --install` literally will upgrade in case the release `kubernetes-dashboard` is not installed in our cluster, in the `--namespace` release.

`--history=max 3` and `-f values.yaml` refer to the amount of versions we are allowing of this installation in our cluster. Meaning, we can fetch data for the last 3.

`--debug` and `--dry-run` Respectively, debug will show the yaml files to be created by the chart and dry-run will not install it, so it is great to watch which resources are being provisioned at runtime.
`