# Portworx Asserts Integration (Portworx Operator)

This is will show the full Asserts<>Portworx intergration by configuring existing
services Portworx ships with as well as adding supplemental exporters to produce
more metrics about the kubernetes cluster and its nodes.


## Installing Portworx

This assumes Portworx is being or has been installed via the Portworx Operator version 2.11

Install the Portworx Operator:

```
kubectl apply -f 'https://install.portworx.com/2.11?comp=pxoperator'
```

Install other Portworx Services:

```
kubectl apply -f 'https://install.portworx.com/2.11?operator=true&mc=false&b=true&kd=type%3Dpd-standard%2Csize%3D150&s=%22type%3Dpd-standard%2Csize%3D150%22&c=px-cluster-4d683bc9-5dc7-4c94-8819-2b71fd6bba70&gke=true&stork=true&csi=true&mon=true&tel=false&st=k8s&promop=true'
```


## Configuring Existing Portworx Resources

### Prometheus-Operator

The Portworx Prometheus-Operator is configured with command line flag `--namespaces=kube-system`, this only allows it to pickup ServiceMonitors from that namespace. Change to allow the asserts namespace:

```
kubectl edit deployment px-prometheus-operator -n kube-system
```

### Prometheus Scrape itself

Label the px-prometheus service so the ServiceMonitor can pick it up:

```
kubectl label service px-prometheus app=px-prometheus
```

Apply the prometheus ServiceMonitor:

```
kubectl apply -f manifests/prometheus-servicemonitor.yaml
```

and change: `-namespaces=kube-system` to `-namespaces=kube-system,asserts`

### Scraping kubelet metrics

Apply a ServiceMonitor to allow Prometheus to scrape the kubelet metrics. This will provide cadvisor and well as some other k8s metrics:

```
kubectl apply -f manifests/kubelet-servicemonitor.yaml
```

### Create the Portworx Storage Class

```
kubectl apply -f manifests/portworx-sc.yaml
```


## Installing New Exporters 

### Instaling kube-state-metrics

Add the prometheus-community helm repo:

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

```

Install:

```
helm upgrade --install kube-state-metrics prometheus-community/kube-state-metrics -f helm/kube-state-metrics-values.yaml -n kube-system
```

### Installing node-exporter

Install:

```
helm upgrade --install node-exporter prometheus-community/prometheus-node-exporter -f helm/node-exporter-values.yaml
```

## Installing Asserts

Install:

```
helm repo add asserts https://asserts.github.io/helm-charts
helm repo update
helm install asserts asserts/asserts -n asserts -f helm/asserts-values.yaml --create-namespace
```

## Verify and Access

Once all containers are initialized and running:

```bash
kubectl get pods -l app.kubernetes.io/instance=asserts
```

You can then login to the asserts-ui by running:

```bash
kubectl port-forward svc/asserts-ui 8080
```

And opening your browser to [http://localhost:8080](http://localhost:8080)
you will be directed to the Asserts Registration page. There you can acquire
a license as seen [here](https://docs.asserts.ai/getting-started/self-hosted/helm-chart#see-the-data)

## Configuring Promethueus DataSources

Configure your Prometheus DataSource which Asserts will connect to
and query by following [these instructions](https://docs.asserts.ai/integrations/data-source/prometheus)

For Portworx-Operator you can set the url to `http://px-prometheus.kube-system.svc.cluster.local:9090`

## Uninstalling the Chart

To uninstall/delete the `asserts` deployment:

```console
helm delete asserts
```

The command removes all the Kubernetes components but PVC's associated with the chart and deletes the release.

To delete the PVC's associated with `asserts`:

```bash
kubectl delete pvc -l app.kubernetes.io/instance=asserts
```

> **Note**: Deleting the PVC's will delete all asserts related data as well. 