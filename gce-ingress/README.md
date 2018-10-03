# GCP Native Ingress Controller for PKS on Google Cloud

By default PKS is installed to GCP without an ingress controller leaving this up to the deployer of the cluster to do themselves. As the operator you can choose to install any of the ingress controllers that you like, it makes a lot of sense to use the native controller that manages GCP loadbalancers and URL maps.

Fortunately there is a [helm chart](https://github.com/helm/charts/tree/master/stable/gce-ingress) that will do most of the heavy lifting for you.

Before running helm though you'll need to collect some information about both your GCP account and your PKS cluster for helm to use.  See [values.yaml](values.yaml) for an example.

## Pre-Requisites

* You'll need a PKS cluster installed onto GCP and the `helm` client.
* Ensure that `kubectl get nodes` responds correctly to ensure your cluster is working and responding.
* Ensure that `helm version` shows that tiller is installed on your cluster.
* You'll also need a google account and a working understanding of how to navigate the UI or CLI to obtain details and create service accounts.

## Create values.yaml

Create a values.yaml file (use [values.yaml](https://raw.githubusercontent.com/helm/charts/master/stable/gce-ingress/values.yaml) as a template):

```bash
$ wget https://raw.githubusercontent.com/helm/charts/master/stable/gce-ingress/values.yaml
```

Set the following values which will tell it to read credentials from a secret, will give the resources human friendly names and restrict the resources available to the pods which should be fairly light:

```yaml
nameOverride: "ingress-controller"
fullnameOverride: "ingress-controller"
secret: ingress-controller-credentials

controller:
  resources:
     requests:
      cpu: 10m
      memory: 50Mi

defaultBackend:
  resources:
    limits:
      cpu: 10m
      memory: 20Mi
    requests:
      cpu: 10m
      memory: 20Mi
```

Next set your GCP specific values.  You should know the values to set based on how you installed PKS, but if not, they're pretty easy to find in the google cloud console.

```yaml
config:
  projectID: <google cloud project ID>
  network: <network in which kubernetes is installed>
  subnetwork: <subnet in which kubernetes is installed>
  # a common prefix for your nodes
  nodeInstancePrefix: vm-
  # a common tag for your worker nodes.
  nodeTags: service-instance-<pks cluster UUID>-worker
  # tokenUrl should probably be left as nil
  tokenUrl: "nil"
```

## Use helm to deploy:

Use helm to install the ingress controller in a namespace that you can restrict access to (since it will use a secret that contains auth to your gcloud account). Some people like to use the existing `kube-system` namespace, but it's good to be more explicit and give it its own namespace of `ingress-controller`.

```
$ helm install --namespace ingress-controller -n ingress-controller \
  --values values.yaml stable/gce-ingress

NAME:   ingress-controller
LAST DEPLOYED: Wed Oct  3 15:15:08 2018
NAMESPACE: ingress-controller
STATUS: DEPLOYED
...
...
```

After a few minutes check the new `ingress-controller` namespace to see if your pods are running:

```bash
$ kubectl -n ingress-controller get pods
NAME                                          READY     STATUS              RESTARTS   AGE
ingress-controller-97c895599-w8twn            0/1       ContainerCreating   0          2m
ingress-controller-backend-78bd8bd8c9-xl79n   1/1       Running             0          2m
```

Note the ingress-controller pod is set in `ContainerCreating` mode.  This is because you still need to add a secret containing the GCP credentials. You can create a locked down service account that only has access to modify load balancers and firewall via the gcloud console (or for demo purposes use your main credentials). Once you have a google account you wish to use you should have a JSON file with the credentials in it.  You can create a suitable secret from that file:

```bash
$ kubectl -n ingress-controller create secret generic ingress-controller-credentials \
  --from-file=key.json=</path-to/gcp-account-key.json>
secret/ingress-controller-credentials created
```

Now your pods should be running:

```bash
$ kubectl -n ingress-controller get pods
NAME                                          READY     STATUS    RESTARTS   AGE
ingress-controller-97c895599-w8twn            1/1       Running   0          11m
ingress-controller-backend-78bd8bd8c9-xl79n   1/1       Running   0          11m
```

Test it out by running an nginx pod and exposing it via an ingress rule in the default namespace:

```bash
$ kubectl apply \
-f https://raw.githubusercontent.com/paulczar/pks-demos/master/gce-ingress/example/deployment.yaml \
-f https://raw.githubusercontent.com/paulczar/pks-demos/master/gce-ingress/example/ingress.yaml \
-f https://raw.githubusercontent.com/paulczar/pks-demos/master/gce-ingress/example/service.yaml
deployment.extensions/example created
ingress.extensions/example created
service/example created

```

After a few minutes you should see your deployment running and an address attached to the ingress resource:

```bash
kubectl get deployment,svc,ingress example
NAME                            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/example   1         1         1            1           6m

NAME              TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/example   NodePort   10.100.200.185   <none>        80:30160/TCP   6m

NAME                         HOSTS     ADDRESS        PORTS     AGE
ingress.extensions/example   *         35.201.78.41   80        6m
```

However it can take up to 10 minutes for the google load balancer to be provisioned and running correctly. If you see a `404` error that looks like the following, just wait a while longer:

```bash
$ curl 35.201.78.41
<!DOCTYPE html>
<html lang=en>
  <meta charset=utf-8>
  <meta name=viewport content="initial-scale=1, minimum-scale=1, width=device-width">
  <title>Error 404 (Not Found)!!1</title>
...
...
  <a href=//www.google.com/><span id=logo aria-label=Google></span></a>
  <p><b>404.</b> <ins>That’s an error.</ins>
  <p>The requested URL <code>/</code> was not found on this server.  <ins>That’s all we know.</ins>
```

Eventually it will start working and you'll see the default nginx page:

```bash
curl 35.201.78.41
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

## Conclusion

Congratulations you've successfully installed and used the GCP Ingress Controller on your PKS (or other) Kubernetes cluster running on Google Cloud.
