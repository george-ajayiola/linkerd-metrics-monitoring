# Linkerd and Prometheus Setup Guide

This repository provides a step-by-step guide to installing and configuring **Prometheus** and **Grafana** for monitoring **Linkerd** metrics in a Kubernetes environment.It’s important to note that you should never use the Prometheus that Linkerd Viz installs in production: it uses volatile storage for its metrics, so every time its Pod restarts, you’ll lose all your historical data. Instead, install your own Prometheus and use that.

## Table of Contents

- [Prometheus Installation](#prometheus-installation)
- [Configure Prometheus for Linkerd Metrics](#configure-prometheus-for-linkerd-metrics)
- [Grafana Installation](#grafana-installation)
- [Accessing Grafana](#accessing-grafana)

---

## Prometheus Installation

To monitor Linkerd metrics, you need to set up an external **Prometheus** instance. This will scrape the control plane and proxy metrics in a format consumable by both users and Linkerd components like the web dashboard.

### Step 1: Install Prometheus in Your Kubernetes Cluster

If Prometheus isn't already installed, you can install it using **Helm**.

1. Add the Prometheus Helm repository:
    ```bash
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    ```

2. Install Prometheus with Helm:
    ```bash
    helm install prometheus prometheus-community/prometheus --namespace monitoring --create-namespace
    ```

3. Check if Prometheus is running:
    ```bash
    kubectl get pods -n monitoring
    ```

4. Access the Prometheus dashboard:
    ```bash
    kubectl --namespace monitoring port-forward svc/prometheus-server 9090:80
    ```

    You can now access Prometheus at `http://localhost:9090`.

---

## Configure Prometheus for Linkerd Metrics

To scrape **Linkerd** metrics, you'll need to modify the Prometheus configuration.

### Step 2.1: Modify the Prometheus ConfigMap

1. Export the current configuration:
    ```bash
    kubectl get configmap prometheus-server -o yaml > prometheus-config.yaml
    ```

2. Edit the `prometheus-config.yaml` file to add Linkerd scrape jobs:
    ```yaml
    global:
      scrape_interval: 10s
      scrape_timeout: 10s
      evaluation_interval: 10s

    scrape_configs:
      - job_name: 'linkerd-control-plane'
        scrape_interval: 10s
        metrics_path: /metrics
        static_configs:
          - targets: ['linkerd-controller:9999']
      - job_name: 'linkerd-proxy'
        scrape_interval: 10s
        metrics_path: /metrics
        static_configs:
          - targets: ['linkerd-proxy:4191']
    ...
    ```
The running configuration of the builtin prometheus can be used as a reference.
```
kubectl -n linkerd-viz  get configmap prometheus-config -o yaml
```

3. Save and apply the configuration.

### Step 2.2: Verify the Configuration

You can check if Prometheus is scraping the Linkerd jobs by visiting `http://localhost:9090/targets`.

---

## Grafana Installation

To visualize the metrics, you can install **Grafana** and configure it to use Prometheus as a data source.

### Step 3: Install Grafana

1. Add the Grafana Helm repository:
    ```bash
    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo update
    ```

2. Download the **values.yaml** file from Linkerd's GitHub repository:
    ```bash
    curl -O https://raw.githubusercontent.com/linkerd/linkerd2/main/grafana/values.yaml
    ```

3. Modify the `values.yaml` file to point to your Prometheus instance:
    ```yaml
    datasources:
      datasources.yaml:
        datasources:
          - name: Prometheus
            type: prometheus
            url: http://prometheus-server.monitoring.svc.cluster.local:80
            access: proxy
            isDefault: true
    ```
 If you are accessing Grafana directly at http://localhost:3000, then the root_url in the `values.yaml` should not contain /grafana/. It should simply be /.

Correct root_url Setting
Update your grafana.ini configuration to reflect the correct URL. If you are accessing Grafana at http://localhost:3000, the correct root_url should look like this:
  ```yaml
  grafana.ini:
    server:
      root_url: '%(protocol)s://%(domain)s:%(http_port)s/'
   ```
If Grafana is Accessed via a Subpath
If you are using a reverse proxy (like NGINX) and accessing Grafana on a subpath, such as http://your-domain/grafana/, then the root_url should include the /grafana/ path. In that case, you can use the following:
  ```yaml
  grafana.ini:
    server:
      root_url: '%(protocol)s://%(domain)s:/grafana/'
  ```

4. Install Grafana using Helm:
    ```bash
    helm install grafana -n grafana --create-namespace grafana/grafana -f values.yaml
    ```

---

## Accessing Grafana

After installation, you can access Grafana locally:

1. Use port-forwarding to access Grafana:
    ```bash
    kubectl port-forward -n grafana svc/grafana 3000:80
    ```

2. Navigate to `http://localhost:3000`. The default login credentials are:
    - **Username:** admin
    - **Password:** Retrieve the password using:
        ```bash
        kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode
        ```

---

### Notes

- Make sure to replace namespace placeholders as needed.
- Ensure proper access and security policies are in place when deploying to production.

# References
- [Network Monitoring with the Linkerd Service Mesh](https://buoyant.io/blog/network-monitoring-with-the-linkerd-service-mesh)
- [Bringing your own Prometheus](https://linkerd.io/2.12/tasks/external-prometheus/)
