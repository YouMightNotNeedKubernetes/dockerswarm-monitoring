> [!WARNING]
> This documentation and stacks are currently a work-in-progress.

# About
A overview of Observability, Telemetry and Monitoring platform.

## Architecture Overview
This is the architecture overview of the whole system working together.

We are using the Grafana Labs’ opinionated observability stack which includes: Loki-for logs, Grafana - for dashboards and visualization, Tempo - for traces, and Mimir - for metrics.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://github.com/YouMightNotNeedKubernetes/dockerswarm-monitoring-for-scale-guide/assets/4363857/859a1172-db2a-4865-9f0c-ff596aff05c5">
  <source media="(prefers-color-scheme: light)" srcset="https://github.com/YouMightNotNeedKubernetes/dockerswarm-monitoring-for-scale-guide/assets/4363857/41fb45ba-6a3c-4ab5-b549-37dbad9f8e44">
  <img alt="Architecture Overview" src="https://github.com/YouMightNotNeedKubernetes/dockerswarm-monitoring-for-scale-guide/assets/4363857/41fb45ba-6a3c-4ab5-b549-37dbad9f8e44">
</picture>

> See [dockerswarm-monitoring-for-scale-guide: A documentation on how to get started with Docker Swarm Monitoring](https://github.com/YouMightNotNeedKubernetes/dockerswarm-monitoring-for-scale-guide)

## Components
These are the components that will be instrumented to gather Metrics, Logs and Traces.

![image](https://github.com/YouMightNotNeedKubernetes/dockerswarm-monitoring-guide/assets/4363857/95c63ad5-1cf9-4d12-8d2a-89185e3673c0)

## Docker Swarm Service Discovery
A proxy service for accessing **Docker Engine API** via **Docket** socket.

**Prometheus**

Prometheus can be deploy on the Docker Swarm’s manager nodes directly. But if you doesn’t have access to the manager node or wish to deploy the service on the worker nodes instead to help reduce strain on the manager node, you can configure Prometheus to use the “dockerswarm_sd_server”.

**Promtail**

By design, **Promtail** requires access to the **Docker Engine API** to perform **Service Discovery** and fetch container logs via file in the `/var/lib/docker/containers` directory on each nodes in the cluster. 

As **Promtail** is deployed globally (*daemonset in Kubernetes terms*) across all nodes both manager and workers. Accessing to **Docker Swarm** specific API only possible on the **Swarm Manager** node only.

The `dockerswarm_sd_server` provide a simple proxy to the **Docker Engine API** (*with limited capabilities*) by running an agent on one (or more) on **Docker Swarm’s manager**.

This allows the worker nodes to perform **Service Discovery** and allow **Promtail** to discover and collect logs for each of the nodes.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://github.com/socheatsok78/dockerswarm_sd_server/assets/4363857/babd8ddc-d2d6-45b1-8995-401ec3b7319d">
  <source media="(prefers-color-scheme: light)" srcset="https://github.com/socheatsok78/dockerswarm_sd_server/assets/4363857/a59d6061-48da-40d5-8ed0-669ba9794e9c">
  <img alt="Overview" src="https://github.com/socheatsok78/dockerswarm_sd_server/assets/4363857/a59d6061-48da-40d5-8ed0-669ba9794e9c">
</picture>

> See [dockerswarm_sd_server: A simple server provide proxying to Docker Engine API for using with Prometheus/Promtail \("dockerswarm_sd_configs" scrape_configs\)](https://github.com/socheatsok78/dockerswarm_sd_server)


## Kubernetes compatible labels
This diagram show what the labels used for adding **Kubernetes Compatible Labels**.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://github.com/YouMightNotNeedKubernetes/prometheus/assets/4363857/0939b290-3d74-42a3-8807-3beed504614a">
  <source media="(prefers-color-scheme: light)" srcset="https://github.com/YouMightNotNeedKubernetes/prometheus/assets/4363857/0ec926bc-457e-450b-901a-76d651d4e7bf">
  <img alt="Kubernetes Compatible Labels" src="https://github.com/YouMightNotNeedKubernetes/prometheus/assets/4363857/0ec926bc-457e-450b-901a-76d651d4e7bf">
</picture>

> For advance relabeling check the “**Prometheus/Promtail's Kubernetes compatible labels**” documents.

### Prometheus/Promtail's Kubernetes compatible labels

There are the labels that can be use to create **Kubernetes compatible labels** based on the deployment of the containers, either via **Docker Swarm mode** or via **Docker Compose**.

**Docker Swarm**
| prometheus  | promtail    | dockerswarm_tasks                                            | dockerswarm_cadvisor                                         |
|-------------|-------------|--------------------------------------------------------------|--------------------------------------------------------------|
| cluster     | cluster     |                                                              |                                                              |
| __replica__ | __replica__ |                                                              |                                                              |
|             |             |                                                              |                                                              |
| instance    | __host__    | __meta_dockerswarm_node_hostname                             |                                                              |
| job         | job         | __meta_dockerswarm_service_label_com_docker_stack_namespace + __meta_dockerswarm_service_name(*2) | container_label_com_docker_stack_namespace + container_label_com_docker_swarm_service_name(*2) |
| namespace   | namespace   | __meta_dockerswarm_service_label_com_docker_stack_namespace  | container_label_com_docker_stack_namespace                   |
| deployment  | deployment  | __meta_dockerswarm_service_label_com_docker_stack_namespace  | container_label_com_docker_stack_namespace                   |
| pod         | pod         | __meta_dockerswarm_service_name                              | container_label_com_docker_swarm_service_name                |
| container   | container   | __meta_dockerswarm_service_name + __meta_dockerswarm_task_slot + <br>__meta_dockerswarm_task_id | name                                                         |


**Docker and Docker Compose**
| prometheus  | promtail    | docker                                                       | docker_cadvisor                                              |
|-------------|-------------|--------------------------------------------------------------|--------------------------------------------------------------|
| cluster     | cluster     |                                                              |                                                              |
| __replica__ | __replica__ |                                                              |                                                              |
|             |             |                                                              |                                                              |
| instance    | __host__    |                                                              |                                                              |
| job         | job         | *__meta_docker_container_label_com_docker_compose_project + __meta_docker_container_label_com_docker_compose_service* | *container_label_com_docker_compose_project + container_label_com_docker_compose_service* |
| namespace   | namespace   | *__meta_docker_container_label_com_docker_compose_project*   | *container_label_com_docker_compose_project*                 |
| deployment  | deployment  | *__meta_docker_container_label_com_docker_compose_project*   | *container_label_com_docker_compose_project*                 |
| pod         | pod         | *__meta_docker_container_label_com_docker_compose_project + __meta_docker_container_label_com_docker_compose_service* | *container_label_com_docker_compose_project + container_label_com_docker_compose_service* |
| container   | container   | *__meta_docker_container_name(*1)*                           | *name*                                                       |

> See [scrape_configs: A collections of Prometheus/Promtail's scrape_configs.](https://github.com/YouMightNotNeedKubernetes/scrape_configs)

## OpenTelemetry

![image](https://github.com/YouMightNotNeedKubernetes/dockerswarm-monitoring-guide/assets/4363857/40718ca1-3a09-4944-b91f-89a868dff0b7)


The agent collector deployment pattern consists of applications — instrumented with an OpenTelemetry Instrumentation using OpenTelemetry protocol (OTLP) — or other collectors (using the OTLP exporter) that send telemetry signals to a collector instance running with the application or on the same host as the application (such as a sidecar or a daemonset).

### **OpenTelemetry and Prometheus Labels mapping**
By convention, `job` and `instance` labels distinguish targets and are expected to be present on metrics exposed on a Prometheus pull exporter (a “federated” Prometheus endpoint) or pushed via Prometheus remote-write.

In OTLP, the `service.name`, `service.namespace`, and `service.instance.id` triplet is required to be unique, which makes them good candidates to use to construct `job` and `instance`. In the collector Prometheus exporters, the `service.name` and `service.namespace` attributes MUST be combined as `<service.namespace>/<service.name>`, or `<service.name>` if `namespace` is empty, to form the `job` metric label. The `service.instance.id` attribute, if present, MUST be converted to the `instance` label; otherwise, `instance` should be added with an empty value.

| Docker Stack                                        | OpenTelemetry         |
| --------------------------------------------------- | --------------------- |
| `{{.Service.Name}}`                                 | `service.name`        |
| `{{.Service.Labels["com.docker.stack.namespace"]}}` | `service.namespace`   |
| `{{.Task.ID}}`                                      | `service.instance.id` |
> **Note**
> Application running via **Docker Compose** require manual labeling via **Environment Variables**

## Important Notice

Currently the design is primarily focus on container running in **Swarm mode**. 

But we can configure **Prometheus/Promtail** to scrape metrics and logs from generic containers or containers running via **Docker Compose** as well.
