# About
A overview of Observability, Telemetry and Monitoring platform.

## Architecture Overview
This is the architecture overview of the whole system working together.

We are using the Grafana Labs’ opinionated observability stack which includes: Loki-for logs, Grafana - for dashboards and visualization, Tempo - for traces, and Mimir - for metrics.

![](image%205.png)
> See [dockerswarm-monitoring-for-scale-guide: A documentation on how to get started with Docker Swarm Monitoring](https://github.com/YouMightNotNeedKubernetes/dockerswarm-monitoring-for-scale-guide)

## Components
These are the components that will be instrumented to gather Metrics, Logs and Traces.

![](image.png)

## Docker Swarm Service Discovery
This service provide the API Endpoint for accessing Docker Engine API from Docker Swarm’s worker nodes.

**Prometheus**
Prometheus can be deploy on the Docker Swarm’s manager nodes directly. But if you doesn’t have access to the manager node or wish to deploy the service on the worker nodes instead to help reduce strain on the manager node, you can configure Prometheus to use the “dockerswarm_sd_server”.

**Promtail**
Promtail required access to Docker Engine API for querying Docker Swarm’s services but only the manager node can perform such operations. And get container logs via file in the `/var/lib/docker/containers` directory.

The “dockerswarm_sd_server” provide a simple proxy to the Docker Engine API on the Docker Swarm’s manager nodes by running an agent on one (or more) on Docker Swarm’s manager and create a proxy to the Docker socket.
![](image%203.png)
> See [dockerswarm_sd_server: A simple server provide proxying to Docker Engine API for using with Prometheus/Promtail \("dockerswarm_sd_configs" scrape_configs\)](https://github.com/socheatsok78/dockerswarm_sd_server)


## Kubernetes compatible labels
This diagram show what the labels used for adding **Kubernetes Compatible Labels**.

![](image%204.png)
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
![](image%202.png)
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