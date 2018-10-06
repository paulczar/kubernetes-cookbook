# External-DNS using GCP Cloud DNS

Currently When you create LoadBalancers and Ingress resources on Kubernetes you likely want to go and update DNS records to point at the external IP provided, or alternatively you may pre-register an external IP and code that in your resource.  Either way its a multistep process.

Fortunately there is a Kubernetes add-on [external-dns](https://github.com/kubernetes-incubator/external-dns) which can automatically create DNS records from a number of different DNS providers based on services and ingress resources.  There's even a [helm chart](https://github.com/helm/charts/tree/master/stable/external-dns) that will do most of the heavy lifting for you.

This recipe will take you through using Helm to set up the external-dns controller to manage GCP DNS records.

Before running helm though you'll need to collect some information about both your GCP account for helm to use.  See [values.yaml](https://github.com/helm/charts/blob/master/stable/external-dns/values.yaml) for an example.

## Pre-Requisites

* You'll need a Kubernetes cluster running and the `helm` client installed locally.
* Ensure that `kubectl get nodes` responds correctly to ensure your cluster is working and responding.
* Ensure that `helm version` shows that tiller is installed on your cluster.
* You'll also need a google account and a working understanding of how to navigate the UI or CLI to obtain details and create service accounts.
* You'll also need a working ingress controller (see [gcp/gce-ingress](https://github.com/paulczar/kubernetes-cookbook/tree/master/gcp/gce-ingress) for an example for setting up native ingress on GCP).

## Create values.yaml

Create a values.yaml file containing the following:

```yaml
## Hardcode the resource names to something human friendly
nameOverride: external-dns-gcp

## Watch these resources for new DNS records
sources:
  - service
  - ingress

## use google as the dns provider
provider: google

# Specify the Google project (required when provider=google)
# You'll need to create this secret containing your credentials.json
google:
  project: "<insert project id here>"
  serviceAccountSecret: "google-service-account"

## List of domains that can be managed. Should be managed by Google Cloud DNS
domainFilters: ["<fully.qualified.domain>"]

# These help tell which records are owned by external-dns.
registry: "txt"
txtOwnerId: "k8s"

## Limit external-dns resources
resources:
  limits:
    memory: 50Mi
  requests:
    memory: 50Mi
    cpu: 10m

## ensure RBAC is enabled
rbac:
  create: true
  apiVersion: v1
```

Set the following values which will tell it to read credentials from a secret, will give the resources human friendly names and restrict the resources available to the pods which should be fairly light:

## Create a Namespace and Secret to for the External DNS controller

Create a namespace to run external-dns in:

```bash
$ kubectl create namespace external-dns-gcp
namespace/external-dns-gcp created
```

Create a service account for the external-dns controller to use using the CLI (or skip this and use the CONSOLE or a pre-existing credential.json):

```bash
$ gcloud iam service-accounts create external-dns \
    --display-name "Service account for ExternalDNS on GCP"
Created service account [external-dns].

$ gcloud projects add-iam-policy-binding <GCP_PROJECT_ID> \
    --role='roles/dns.admin' \
    --member='serviceAccount:external-dns@<GCP_PROJECT_ID>.iam.gserviceaccount.com'
...

$ gcloud iam service-accounts keys create credentials.json \
    --iam-account external-dns@<GCP_PROJECT_ID>.iam.gserviceaccount.com
```

Create a secret containing the service account key:

```bash
$ kubectl -n external-dns-gcp create secret \
    generic external-dns \
  --from-file=credentials.json=credentials.json
secret/external-dns created
```

## Use helm to deploy:

Use helm to install the external-dns controller in your new namespace:

```
$ helm install --namespace external-dns-gcp -n external-dns-gcp \
  --values values.yaml stable/external-dns
NAME:   external-dns-gcp
LAST DEPLOYED: Thu Oct  4 12:09:57 2018
NAMESPACE: external-dns-gcp
STATUS: DEPLOYED
...
...
```

After a few minutes check the new `external-dns-gcp` namespace to see if your pods are running:

```bash
$ kubectl -n external-dns-gcp get all
NAME                                   READY     STATUS    RESTARTS   AGE
pod/external-dns-gcp-c7b67b78d-fhhw5   1/1       Running   0          45s

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/external-dns-gcp   ClusterIP   10.100.200.37   <none>        7979/TCP   45s

NAME                               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/external-dns-gcp   1         1         1            1           45s

NAME                                         DESIRED   CURRENT   READY     AGE
replicaset.apps/external-dns-gcp-c7b67b78d   1         1         1         45s

```

Test it out by running an nginx pod and exposing it via a loadbalancer service with an appropriate annotation:

```bash
$ kubectl run example --image=nginx:1.13.5-alpine
deployment.apps/example created

$ kubectl expose deployment example --type=LoadBalancer --port 80
service/example exposed

$ kubectl annotate service example 'external-dns.alpha.kubernetes.io/hostname=app.pivlab.gcp.paulczar.wtf'
service/example annotated
```

You should be able watch the external-dns controller register your new hostname:

```
$ kubectl logs -n external-dns-gcp deployment/external-dns-gcp
...
...
time="2018-10-04T19:35:48Z" level=debug msg="Matched example.demo. (zone: example-zone)"
time="2018-10-04T19:35:48Z" level=debug msg="Considering zone: example-zone (domain: example.demo.)"
time="2018-10-04T19:35:48Z" level=info msg="Change zone: example-zone"
time="2018-10-04T19:35:48Z" level=info msg="Add records: app.example.demo. A [104.154.195.130] 300"
time="2018-10-04T19:35:48Z" level=info msg="Add records: app.example.demo. TXT [\"heritage=external-dns,external-dns/owner=k8s,external-dns/resource=service/default/example\"] 300"
```

After a few minutes for DNS to propogate you should be able to access your example application using the new hostname:

```
$ curl app.example.demo
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```



## Conclusion

Congratulations you've successfully installed and used the external-dns controller on your Kubernetes cluster integrated with Google Cloud DNS.
