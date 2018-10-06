# Securing Kubernetes workloads with TLS using the Certificate Manager add-on and Google Cloud DNS.

`cert-manager` is a Kubernetes add-on to automate the management and issuance of TLS certificates from various issuing sources. This recipe demonstrates using `cert-manager` to request TLS certificates from [Let's Encrypt](https://letsencrypt.org/).

Fortunately there is a [helm chart](https://github.com/helm/charts/tree/master/stable/cert-manager) that will do most of the heavy lifting for you.

## Pre-Requisites

* You'll need a Kubernetes cluster installed and the `helm` client.
* Ensure that `kubectl get nodes` responds correctly to ensure your cluster is working and responding.
* Ensure that `helm version` shows that tiller is installed on your cluster.

## Create values.yaml

Create a values.yaml file:

```yaml
## ensure human friendly resource naming
nameOverride: cert-manager
fullnameOverride: cert-manager

## ensure RBAC is enabled
rbac:
  create: true

## restrict pod resources
resources:
  requests:
    cpu: 10m
    memory: 32Mi
```

## Use helm to deploy cert-manager

Use helm to install the `cert-manager` addon in a namespace that you can restrict access to (since it will use a secret that contains auth to your gcloud account). Some people like to use the existing `kube-system` namespace, but it's good to be more explicit and give it its own namespace of `cert-manager`.

```bash
$ helm install --namespace cert-manager -n cert-manager \
  --values values.yaml stable/cert-manager

NAME:   cert-manager
LAST DEPLOYED: Fri Oct  5 13:25:23 2018
NAMESPACE: cert-manager
STATUS: DEPLOYED

...
...
```

After a few minutes check the new `ingress-controller` namespace to see if your pods are running:

```bash
$ kubectl -n cert-manager get all
NAME                                READY     STATUS    RESTARTS   AGE
pod/cert-manager-7f9fb4c8bc-4cpq6   1/1       Running   0          29s

NAME                           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager   1         1         1            1           30s

NAME                                      DESIRED   CURRENT   READY     AGE
replicaset.apps/cert-manager-7f9fb4c8bc   1         1         1         30s

```

## Create a certificate issuer

In order for `cert-manager` to integrate with Let's Encrypt you need to set up a cluster issuer. This tells `cert-manager` to use the [ACME Protocol](https://ietf-wg-acme.github.io/acme/draft-ietf-acme-acme.html) to request certificates from Let's Encrypt.

You also need to choose if you want to validate your ownership of the domain via `http` or `dns`. For `http` you need a working `ingress controller` to manage the routes and traffic for the given hostname, for `dns` you provide credentials to the issuer to create a `txt` field.

The following example will use the `dns` method to authenticate to Google Cloud DNS to bypass the added complexity of using an `ingress` resource.

Create a Google Cloud service account and Kubernetes secret to use:

```bash
$ gcloud iam service-accounts create cert-manager \
    --display-name "Service account for cert-manager"
Created service account [cert-manager].

$ gcloud projects add-iam-policy-binding <GCP_PROJECT_ID> \
    --role='roles/dns.admin' \
    --member='serviceAccount:cert-manager@<GCP_PROJECT_ID>.iam.gserviceaccount.com'
...

$ gcloud iam service-accounts keys create credentials.json \
    --iam-account cert-manager@pgtm-pczarkowski.iam.gserviceaccount.com
```

Create a secret containing the service account key:

```bash
$ kubectl -n cert-manager create secret \
    generic letsencrypt-gcp-credentials \
  --from-file=credentials.json=credentials.json
secret/external-dns created

```

Create a Kubernetes manifest that sets up a certificate issuer for both the Let's Encrypt Staging and Production APIs:

__cluster-issuer.yaml__
```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: '<email@address.com>'
    privateKeySecretRef:
      name: letsencrypt-staging
    dns01:
      providers:
        - name: letsencrypt-staging-gcp
          clouddns:
            # A secretKeyRef to a google cloud json service account
            serviceAccountSecretRef:
              name: letsencrypt-gcp-credentials
              key: credentials.json
            # The project in which to update the DNS zone
            project: <google-cloud-project-id>
---
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: '<email@address.com>'
    privateKeySecretRef:
      name: letsencrypt-prod
    dns01:
      providers:
        - name: gcp-prod
          clouddns:
            # A secretKeyRef to a google cloud json service account
            serviceAccountSecretRef:
              name: letsencrypt-gcp-credentials
              key: credentials.json
            # The project in which to update the DNS zone
            project: <google-cloud-project-id>

```

Deploy the newly created manifest:

```bash
$ kubectl apply -n cert-manager -f cluster-issuer.yaml
clusterissuer.certmanager.k8s.io/letsencrypt-staging created
clusterissuer.certmanager.k8s.io/letsencrypt-prod created
```

Check the `cert-manager` logs to check it can connect to Google Cloud DNS:

```bash
$ k -n cert-manager logs deployment/cert-manager
...
...
I1005 18:59:36.770485       1 logger.go:88] Calling GetAccount
I1005 18:59:36.877050       1 setup.go:93] letsencrypt-staging: verified existing registration with ACME server
I1005 18:59:36.877399       1 controller.go:154] clusterissuers controller: Finished processing work item "letsencrypt-staging"
I1005 18:59:37.176411       1 setup.go:93] letsencrypt-prod: verified existing registration with ACME server
I1005 18:59:37.176593       1 controller.go:154] clusterissuers controller: Finished processing work item "letsencrypt-prod"
```

## Deploy an Application to Kubernetes and secure with TLS Certificates

Create a certificate for your URL:

```
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: example
spec:
  secretName: example-tls
  issuerRef:
    name: letsencrypt-staging
  commonName: app.example.com
  dnsNames:
  - app.example.com
  acme:
    config:
    - dns01:
        provider: letsencrypt-staging-gcp
      domains:
      - app.example.com
```

Check that `cert-manager` created your secret and populated it with the key and cert:

```
$ k get secret example-tls -o yaml
apiVersion: v1
data:
  tls.crt: ...
  tls.key: ...
kind: Secret
metadata:
  annotations:
    certmanager.k8s.io/alt-names: app.example.com
    certmanager.k8s.io/common-name: app.example.com
    certmanager.k8s.io/issuer-kind: ClusterIssuer
    certmanager.k8s.io/issuer-name: letsencrypt-staging
  creationTimestamp: 2018-10-05T19:12:52Z
  labels:
    certmanager.k8s.io/certificate-name: example
  name: example-tls
  namespace: default
  resourceVersion: "4699573"
  selfLink: /api/v1/namespaces/default/secrets/example-tls
  uid: a82dd011-c8d2-11e8-a0e5-42010a000b0a
type: kubernetes.io/tls
```

Deploy a basic NGINX application:

```bash
$ kubectl run example --image=nginx:1.13.5-alpine
deployment.apps/example created

$ kubectl expose deployment example --type=NodePort --port 80
service/example exposed

```

Create an ingress manifest:

> Note: you need to have a working ingress controller deployed for this to work.

__ingress.yaml__

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example
spec:
  tls:
  - secretName: example-tls
  backend:
    serviceName: example
    servicePort: 80
```

Deploy the ingress manifest:

```bash
$ kubectl apply -f ingress.yaml
ingress/example created
```

After a few minutes you should see your deployment running and an address attached to the ingress resource:

```bash
kubectl get deployment,svc,ingress example
NAME                            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/example   1         1         1            1           6m

NAME              TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/example   NodePort   10.100.200.185   <none>        80:30160/TCP   6m

NAME                         HOSTS     ADDRESS        PORTS     AGE
ingress.extensions/example   *         35.201.78.41   80,443        6m
```

However it can take up to 10 minutes for the google load balancer to be provisioned and running correctly. If you see a `503` or `404` wait a while longer and try again.

Eventually it will start working and you'll see the default nginx page:

```bash
curl 35.201.78.41
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

However when you try via SSL (you can use a Host header so you don't need to update DNS) you'll see a cert error:

```bash
$ curl -H "Host: app.example.com" https://35.201.78.41
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html
```

This is because the letsencrypt staging certificate does not have a trusted CA. You can redeploy the certificate above with the production cluster issuer and it will then work correctly:

```bash
$ curl -H "Host: app.example.com" https://35.201.78.41
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

## Conclusion

Integrating with the `cert-manager` add-on to manage TLS certificates is fairly simple and brings a lot of power to Kubernetes as it makes getting TLS certificates very easy.

You can even use annotations on your `ingress` resource to have `cert-manager` automatically create the certificates for you, and if you'd rather not provide your google cloud credentials you can use the `http` method to prove ownership of the domain instead of `dns`.

It gets really exciting when you start tying `cert-manager` and `external-dns` add-ons together and your Kubernetes resources can start to manage both their own DNS and TLS Certificates.