# Portworx Asserts Integration (Portworx Operator)

This will show the full Asserts<>Portworx intergration by configuring existing
services Portworx ships with as well as adding supplemental exporters to produce
more metrics about the kubernetes cluster and its nodes.


## Installing Portworx

This assumes Portworx is being or has been installed via the Portworx Operator version 2.11

See [Install Portworx](https://docs.portworx.com/install-portworx/)


## Configuring Existing Portworx Resources

### Prometheus-Operator

The Portworx Prometheus-Operator is configured with command line flag `--namespaces=kube-system`, this only allows it to pickup ServiceMonitors from that namespace. Change to allow the asserts namespace:

```
kubectl edit deployment px-prometheus-operator -n kube-system
```

and change: `-namespaces=kube-system` to `-namespaces=kube-system,asserts`

### Prometheus Scrape itself

Create a service for the purposes of scraping prometheus since Portworx Operator will undo live
changes to the existing prometheus service. It has no labels which are required to scrape it.

Create the extra prometheus service:

```
kubectl apply -f manifests/prometheus-service.yaml
```

Apply the prometheus ServiceMonitor:

```
kubectl apply -f manifests/prometheus-servicemonitor.yaml
```

> **Note**: normally we would just run `kubectl label service px-prometheus app=px-prometheus -n kube-system` to label the existing prometheus service.

### Scraping kubelet metrics

Apply a ServiceMonitor to allow Prometheus to scrape the kubelet metrics. This will provide cadvisor and well as some other k8s metrics. Note that this servicemonitor might need to change depending on the flavor of k8s being installed on. Check the kubelet service's labels with `kubectl get svc kubelet -n kube-system --show-labels` and set the kubelet serviceMonitor's `matchLabels` appropriately.

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
helm upgrade --install node-exporter prometheus-community/prometheus-node-exporter -f helm/node-exporter-values.yaml -n kube-system
```

## Installing Asserts

Add Repo and Install:

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


## Configuring Prometheus DataSources

Configure your Prometheus DataSource which Asserts will connect to
and query by following [these instructions](https://docs.asserts.ai/integrations/data-source/prometheus)

For Portworx-Operator you can set the url to `http://px-prometheus.kube-system.svc.cluster.local:9090`


## Import Dashboards

3 Portworx dashboards have been modified to work with Grafana v9+. Simple import the dashboards in the `dashboards` directory. Then from here, you can link the Dashboards to Asserts Entities.


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
