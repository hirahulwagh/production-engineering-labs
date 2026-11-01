# OpenTelemetry Astronomy Shop --- Microservices Deployment & Observability on AWS

![AWS](https://img.shields.io/badge/AWS-EC2-orange)
![Ubuntu](https://img.shields.io/badge/OS-Ubuntu%2026.04%20LTS-E95420)
![Docker](https://img.shields.io/badge/Container-Docker%20%2B%20Compose-2496ED)
![OpenTelemetry](https://img.shields.io/badge/Observability-OpenTelemetry-7B3FE4)
![Status](https://img.shields.io/badge/Deployment-Completed-success)

A hands-on DevOps/SRE project that deploys the **OpenTelemetry Astronomy
Shop Demo**, a distributed e-commerce application made up of multiple
microservices, on an AWS EC2 virtual machine using Docker Compose. The
project also enables a telemetry stack for exploring distributed traces,
metrics, logs, dashboards, and service health.

> **Project type:** Learning / portfolio lab. This is not a
> production-hardened deployment. The demo contains intentionally
> simplified settings and should not be exposed publicly without
> additional security controls.

------------------------------------------------------------------------

## Table of contents

-   [1. Project overview](#1-project-overview)
-   [2. Goals and learning outcomes](#2-goals-and-learning-outcomes)
-   [3. Architecture](#3-architecture)
-   [4. AWS infrastructure](#4-aws-infrastructure)
-   [5. Prerequisites](#5-prerequisites)
-   [6. Launch and connect to EC2](#6-launch-and-connect-to-ec2)
-   [7. Prepare Ubuntu](#7-prepare-ubuntu)
-   [8. Install Docker Engine and
    Compose](#8-install-docker-engine-and-compose)
-   [9. Clone the OpenTelemetry Demo](#9-clone-the-opentelemetry-demo)
-   [10. Understand the Compose files](#10-understand-the-compose-files)
-   [11. Deploy the core
    microservices](#11-deploy-the-core-microservices)
-   [12. Enable the full demo
    services](#12-enable-the-full-demo-services)
-   [13. Enable the observability
    stack](#13-enable-the-observability-stack)
-   [14. Validate the application](#14-validate-the-application)
-   [15. Access the observability
    tools](#15-access-the-observability-tools)
-   [16. Resource and health checks](#16-resource-and-health-checks)
-   [17. Issues encountered and
    resolutions](#17-issues-encountered-and-resolutions)
-   [18. Useful operational commands](#18-useful-operational-commands)
-   [19. Security considerations](#19-security-considerations)
-   [20. Cleanup and cost control](#20-cleanup-and-cost-control)
-   [21. Suggested repository
    structure](#21-suggested-repository-structure)
-   [22. Next engineering milestones](#22-next-engineering-milestones)
-   [23. References](#23-references)

------------------------------------------------------------------------

## 1. Project overview

The OpenTelemetry Astronomy Shop is a sample online store designed to
demonstrate a distributed application and observability. A user
interacts with the storefront; backend services communicate with one
another to complete activities such as browsing products, managing a
cart, calculating shipping, processing payment, and completing checkout.

We deployed the upstream application on AWS EC2 with Docker Compose. We
used the upstream Compose configuration in layers:

1.  `compose.yaml` --- core demo services.
2.  `compose.full.yaml` --- additional full-demo components, including
    Kafka, accounting, and fraud detection.
3.  `compose.observability.yaml` --- observability components, including
    Jaeger, Prometheus, OpenSearch, Grafana, and OpAMP.

The application was started in stages so that we could validate each
layer and troubleshoot resource constraints before moving forward.

### What was completed

-   Provisioned an Ubuntu EC2 instance.
-   Installed Docker Engine, Docker CLI, containerd, Buildx, and the
    Docker Compose plugin.
-   Cloned the official OpenTelemetry Demo repository.
-   Started the core demo services.
-   Added the full-demo services.
-   Added the observability services.
-   Opened the storefront and completed a test purchase; the UI
    confirmed **"Your order is complete!"** and displayed an order ID.
-   Confirmed Prometheus was reachable and returned successful
    health/readiness responses.
-   Opened Grafana and viewed the Demo/Linux dashboard with host
    metrics.
-   Investigated a Grafana responsiveness / upstream timeout issue and
    identified a restrictive 500 MiB container memory limit as a likely
    contributor. The Compose file was edited to raise the Grafana limit
    to 1000M; the final recreated-container verification should be
    recorded after it is run.

------------------------------------------------------------------------

## 2. Goals and learning outcomes

This project is intended to provide practical experience with:

-   AWS EC2 provisioning and Linux administration.
-   Docker images, containers, networks, resource limits, and Docker
    Compose.
-   Multi-service application deployment and service dependencies.
-   Environment variables and layered Compose configuration.
-   Troubleshooting disk exhaustion, container health, and application
    timeouts.
-   OpenTelemetry instrumentation and telemetry collection.
-   Distributed tracing with Jaeger.
-   Metrics collection and querying with Prometheus.
-   Dashboards and visualization with Grafana.
-   Log storage/search with OpenSearch.
-   Operational checks using `docker compose`, `docker inspect`,
    `docker logs`, `docker stats`, `curl`, `free`, and `df`.

------------------------------------------------------------------------

## 3. Architecture

### High-level request and telemetry flow

``` mermaid
flowchart TB
    U[User / Browser] --> P[Frontend Proxy - Envoy :8080]
    P --> F[Frontend]
    F --> A[Ad]
    F --> C[Cart]
    F --> CO[Checkout]
    F --> PC[Product Catalog]
    F --> R[Recommendation]
    CO --> PAY[Payment]
    CO --> CUR[Currency]
    CO --> E[Email]
    CO --> S[Shipping]
    PC --> DB[(PostgreSQL)]
    C --> V[(Valkey)]
    S --> Q[Quote]
    AC[Accounting] --> K[Kafka]
    FD[Fraud Detection] --> K

    F -. traces / metrics / logs .-> OT[OpenTelemetry Collector]
    A -. telemetry .-> OT
    C -. telemetry .-> OT
    CO -. telemetry .-> OT
    PC -. telemetry .-> OT
    OT --> J[Jaeger - Traces]
    OT --> PR[Prometheus - Metrics]
    OT --> OS[OpenSearch - Logs]
    PR --> G[Grafana]
    J --> G
    OS --> G
```

The diagram is conceptual. Actual service-to-service calls and telemetry
pipelines are defined by the upstream application configuration and may
vary by release.

### Main application components

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  Frontend                            Store user interface

  Frontend Proxy (Envoy)              Entry point and routing to frontend
                                      and selected tools

  Checkout                            Coordinates the checkout workflow

  Cart                                Shopping cart operations

  Product Catalog                     Product information; uses
                                      PostgreSQL

  Payment                             Payment simulation

  Currency                            Currency conversion

  Shipping                            Shipping calculation

  Quote                               Quote/shipping-related dependency

  Recommendation                      Product recommendations

  Ad                                  Advertisement service

  Email                               Order email simulation

  Image Provider                      Product images

  Flagd / Flagd UI                    Feature-flag service and UI

  Load Generator                      Generates demo traffic

  Kafka                               Event streaming used by full-demo
                                      components

  Accounting                          Accounting demo service

  Fraud Detection                     Fraud-detection demo service

  Astronomy DB                        PostgreSQL database

  Valkey Cart                         In-memory cart dependency
  -----------------------------------------------------------------------

### Observability components

  -----------------------------------------------------------------------
  Component                           Purpose
  ----------------------------------- -----------------------------------
  OpenTelemetry Collector             Receives, processes, and exports
                                      telemetry

  Jaeger                              Distributed trace exploration

  Prometheus                          Metrics scraping/storage and PromQL
                                      queries

  Grafana                             Dashboards and data-source
                                      visualization

  OpenSearch                          Log indexing/search backend

  OpAMP Server                        OpenTelemetry agent management
                                      protocol demo component
  -----------------------------------------------------------------------

**Important distinction:** OpenTelemetry is the instrumentation and
telemetry framework; Jaeger, Prometheus, OpenSearch, and Grafana are
separate tools used to store, query, or visualize telemetry.

------------------------------------------------------------------------

## 4. AWS infrastructure

### Environment used in this deployment

The following values are based on the EC2 terminal session and
filesystem/memory checks captured during the implementation.

  Setting                           Value used / observed
  --------------------------------- -----------------------------------------------
  Cloud provider                    AWS
  Service                           Amazon EC2
  Region                            `ap-south-1` (Mumbai)
  Operating system                  Ubuntu Server 26.04 LTS
  Architecture                      64-bit x86 (`x86_64`)
  CPU                               2 vCPUs
  Memory                            Approximately 8 GiB (Linux reported 7.6 GiB)
  EBS volume                        30 GiB
  Root filesystem after expansion   Approximately 28 GiB usable
  Filesystem                        ext4
  Login user                        `ubuntu`
  Application source                Official OpenTelemetry Demo GitHub repository

**Instance type and AMI identifier:** The exact EC2 instance-type string
and AMI ID were not preserved in the terminal transcript available for
this README. The observed machine had 2 vCPUs and approximately 8 GiB
RAM. Do not infer an instance type or AMI ID from those values; add the
exact values from the EC2 console if you want the infrastructure record
to be fully reproducible.

### Recommended sizing

For this complete Docker Compose stack, use a VM with at least the
resources tested here: 2 vCPUs, approximately 8 GiB RAM, and 30 GiB EBS.
This is a lab baseline, not a guaranteed production sizing
recommendation. Load generation, OpenSearch, and Grafana can increase
resource consumption. For heavier traffic or concurrent Kubernetes
experiments, use a larger instance or separate the workloads.

### EBS sizing lesson

The original root filesystem was only about 6.7 GiB even though the
attached EBS disk was later shown as 30 GiB. Expanding the EBS volume
alone did not automatically expand the partition and ext4 filesystem. We
had to grow both the partition and filesystem; see the troubleshooting
section.

------------------------------------------------------------------------

## 5. Prerequisites

-   AWS account with permission to create EC2 instances, security
    groups, and EBS volumes.
-   SSH client (Windows PowerShell, Windows Terminal, macOS Terminal, or
    Linux shell).
-   EC2 key pair (`.pem`).
-   A browser to access the storefront and observability interfaces.
-   Basic Linux and Docker command-line knowledge.

------------------------------------------------------------------------

## 6. Launch and connect to EC2

### 6.1 Launch the instance

In the AWS Console:

1.  Open **EC2 → Instances → Launch instances**.
2.  Give the instance a name, for example `otel-astronomy-shop`.
3.  Select the Ubuntu Server 26.04 LTS 64-bit x86 AMI used for the lab.
4.  Select a compute size with 2 vCPUs and approximately 8 GiB RAM (the
    actual instance type used should be confirmed from the EC2 console).
5.  Create or select a key pair and download the `.pem` file.
6.  Configure the root EBS volume as 30 GiB, preferably gp3.
7.  Create/select a security group.
8.  Launch the instance and wait until the instance checks pass.

### 6.2 Security group guidance

For a safer setup:

  -----------------------------------------------------------------------
  Port              Protocol          Source            Purpose
  ----------------- ----------------- ----------------- -----------------
  22                TCP               Your public IP    SSH
                                      `/32`             administration

  8080              TCP               Your public IP    Storefront /
                                      `/32`             Envoy proxy

  9090              TCP               Your public IP    Prometheus, if
                                      `/32`             direct access is
                                                        needed
  -----------------------------------------------------------------------

Do not expose Grafana, Jaeger, OpenSearch, PostgreSQL, Kafka, or
internal microservice ports to `0.0.0.0/0`. Prefer accessing internal
tools through the proxy or an SSH tunnel. The demo's published dynamic
ports are not a reason to open every port in the security group.

### 6.3 Connect from Windows PowerShell

Move to the directory containing the key:

``` powershell
cd C:\Users\<YOUR_USER>\Desktop\ServerConnectionFiles
```

Connect using the public DNS name shown in the EC2 console:

``` powershell
ssh -i "your-key.pem" ubuntu@<EC2-PUBLIC-DNS>
```

The public IP/DNS can change after a stop/start unless an Elastic IP is
associated. Always use the current value shown in the EC2 console.

### 6.4 Verify the operating system and resources

``` bash
cat /etc/os-release
uname -a
nproc
free -h
lsblk
df -hT /
```

Expected observations from our instance:

-   Ubuntu 26.04 LTS.
-   2 logical CPUs.
-   About 7.6 GiB memory visible to Linux.
-   30 GiB EBS disk after resizing.
-   About 28 GiB root filesystem after partition/filesystem expansion.

------------------------------------------------------------------------

## 7. Prepare Ubuntu

Update package metadata and install basic utilities:

``` bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release git
```

Check disk and memory before installing Docker:

``` bash
df -hT /
free -h
```

Keep sufficient free disk space for image pulls, writable container
layers, and logs.

------------------------------------------------------------------------

## 8. Install Docker Engine and Compose

We installed Docker from Docker's official Ubuntu APT repository.

### 8.1 Install prerequisites

``` bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
```

### 8.2 Add Docker's official GPG key

``` bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### 8.3 Add the Docker APT repository

``` bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 8.4 Install Docker Engine and plugins

``` bash
sudo apt-get update

sudo apt-get install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

### 8.5 Enable and verify Docker

``` bash
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager

sudo docker version
sudo docker compose version
sudo docker buildx version
sudo systemctl status containerd --no-pager
```

Versions observed in this lab included Docker Engine/CLI `29.9.2` and
Docker Compose plugin `5.6.1`. Package versions can change over time.

We used `sudo docker` throughout the lab. This avoids changing the
user's group membership. Membership in the `docker` group effectively
grants root-level privileges, so only add trusted users to it.

------------------------------------------------------------------------

## 9. Clone the OpenTelemetry Demo

We used the official upstream repository:

``` bash
git clone https://github.com/open-telemetry/opentelemetry-demo.git
cd opentelemetry-demo
```

Inspect the source tree:

``` bash
ls -la
```

Important files in the upstream repository include:

``` text
compose.yaml
compose.full.yaml
compose.observability.yaml
otel-config.yml
src/
README.md
```

The application source and Compose definitions belong to the
OpenTelemetry project. This portfolio repository should document our AWS
deployment, operational decisions, and troubleshooting rather than
copying or claiming ownership of the upstream application.

------------------------------------------------------------------------

## 10. Understand the Compose files

Docker Compose can merge multiple YAML files. Each additional `-f` file
adds services or overrides configuration from earlier files.

  -----------------------------------------------------------------------
  File                                Role
  ----------------------------------- -----------------------------------
  `compose.yaml`                      Core microservices and base
                                      dependencies

  `compose.full.yaml`                 Adds full-demo components such as
                                      Kafka, accounting, and fraud
                                      detection

  `compose.observability.yaml`        Adds Jaeger, Prometheus,
                                      OpenSearch, Grafana, and OpAMP
  -----------------------------------------------------------------------

### Validate the merged configuration

Core services:

``` bash
sudo docker compose -f compose.yaml config --services
```

Full demo:

``` bash
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  config --services
```

Full demo plus observability:

``` bash
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  -f compose.observability.yaml \
  config --services
```

The last command should list the application services and observability
components, including `jaeger`, `prometheus`, `opensearch`, `grafana`,
and `opamp-server`.

Why this matters: checking the merged service list catches missing or
incorrectly ordered Compose files before starting containers.

------------------------------------------------------------------------

## 11. Deploy the core microservices

From the repository directory:

``` bash
cd ~/opentelemetry-demo

sudo docker compose up -d
```

This starts the core demo using the default Compose file. The upstream
images are pulled or built according to the Compose configuration.

Check the deployment:

``` bash
sudo docker compose ps
sudo docker ps
```

Check health status:

``` bash
sudo docker compose ps --all
```

Inspect a service's logs if it is unhealthy or repeatedly restarting:

``` bash
sudo docker logs <container-name> --tail 100
```

Wait for dependencies and health checks to settle before adding the next
layer.

------------------------------------------------------------------------

## 12. Enable the full demo services

Add the full-demo Compose file:

``` bash
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  up -d
```

This layered configuration adds the full-demo components, including
Kafka, accounting, and fraud detection.

Validate:

``` bash
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  ps
```

Confirm the merged service list:

``` bash
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  config --services
```

------------------------------------------------------------------------

## 13. Enable the observability stack

Add the observability Compose file while retaining the core and
full-demo files:

``` bash
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  -f compose.observability.yaml \
  up -d
```

Validate the complete stack:

``` bash
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  -f compose.observability.yaml \
  ps
```

List all running containers and published ports:

``` bash
sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

The exact host ports for some services are dynamically assigned by
Compose. Do not assume that Grafana or Jaeger will always use the same
high-numbered host port after a container is recreated.

### Why use all three files?

The three-file command makes the deployment intent explicit:

-   Base application services.
-   Additional full-demo services.
-   Observability backends and UI components.

Use the same three files, in the same order, for later `ps`, `logs`,
`config`, `up`, and targeted service operations.

------------------------------------------------------------------------

## 14. Validate the application

### 14.1 Find the EC2 public IP

Use the current public IPv4 address shown in the EC2 console. Do not
hardcode an old address in a long-lived README.

### 14.2 Open the storefront

``` text
http://<EC2-PUBLIC-IP>:8080/
```

The browser should display the Astronomy Shop storefront.

### 14.3 Complete a test purchase

1.  Open the storefront.
2.  Browse the product catalogue.
3.  Add one or more products to the cart.
4.  Open the cart and proceed to checkout.
5.  Complete the demo checkout flow.
6.  Confirm the order-completion page and order ID.

During this implementation, the checkout completed successfully and the
browser displayed **"Your order is complete!"** with an order ID.

### 14.4 Verify the frontend proxy from the EC2 host

``` bash
curl -I --max-time 15 http://localhost:8080/
```

A successful HTTP response indicates the proxy is responding. If the
browser cannot connect, check the EC2 security group, current public IP,
and container status.

------------------------------------------------------------------------

## 15. Access the observability tools

### 15.1 Prometheus

The host port was explicitly published as `9090` in the observed
deployment.

Open:

``` text
http://<EC2-PUBLIC-IP>:9090/
```

Try the query interface:

``` text
http://<EC2-PUBLIC-IP>:9090/query
```

Check readiness from EC2:

``` bash
curl -i --max-time 10 http://localhost:9090/-/ready
curl -i --max-time 10 http://localhost:9090/-/healthy
```

Prometheus responded successfully during validation.

### 15.2 Grafana

The demo's frontend proxy is configured to route Grafana under:

``` text
http://<EC2-PUBLIC-IP>:8080/grafana/
```

The Grafana container also received a dynamically assigned host port.
Find its current mapping:

``` bash
sudo docker port grafana 3000
```

Check its health endpoint from inside the container:

``` bash
sudo docker exec grafana \
  sh -c 'wget -S -O- -T 15 http://127.0.0.1:3000/api/health'
```

The dashboard used during the lab was the Demo/Linux dashboard, which
displayed host information and system metrics such as CPU, memory, and
uptime.

For a Grafana data source that runs in the same Compose network, use the
Docker service DNS name and internal port where appropriate (for
Prometheus, `http://prometheus:9090`), not the EC2 public IP.

### 15.3 Jaeger

The frontend proxy route is intended to expose the Jaeger UI at:

``` text
http://<EC2-PUBLIC-IP>:8080/jaeger/ui
```

Find the direct mapped UI port:

``` bash
sudo docker port jaeger 16686
```

Use the current mapping if direct access is enabled and allowed by your
security policy. Prefer the proxy or an SSH tunnel rather than opening
the Jaeger port publicly.

### 15.4 OpenSearch

Find its mapped port:

``` bash
sudo docker port opensearch 9200
```

OpenSearch is used as a log backend in this demo stack. Do not expose it
publicly.

### 15.5 Inspect all published ports

``` bash
sudo docker ps --format "table {{.Names}}\t{{.Ports}}"
```

This is the authoritative way to find the current host-to-container port
mapping for the running containers.

------------------------------------------------------------------------

## 16. Resource and health checks

A multi-service demo can consume significant memory and disk. Check the
EC2 host and containers regularly.

### Host memory and CPU

``` bash
free -h
nproc
uptime
top
```

### Root filesystem and block devices

``` bash
df -hT /
lsblk
```

### Docker storage

``` bash
sudo docker system df
```

### Container resource usage

``` bash
sudo docker stats --no-stream
```

### Container status

``` bash
sudo docker ps
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  -f compose.observability.yaml \
  ps
```

### Inspect a container's configured memory limit

``` bash
sudo docker inspect grafana \
  --format='Memory={{.HostConfig.Memory}}'
```

A value of `524288000` bytes represents a 500 MiB limit. A value around
`1048576000` bytes represents a 1000 MiB limit.

------------------------------------------------------------------------

## 17. Issues encountered and resolutions

This section records the actual problems seen during the lab, why they
occurred, and how they were addressed. The purpose is to make a future
deployment repeatable and to avoid losing time on the same problems.

### Issue 1 --- Docker build failed with `no space left on device`

**Observed error:**

``` text
failed to solve: failed to compute cache key:
write tmp file: write /var/lib/containerd/.../updatedat.tmp:
no space left on device
```

**Investigation:**

``` bash
df -hT /
lsblk
sudo docker system df
```

The root filesystem was only about 6.7 GiB and reached 100% usage. The
EBS block device was expanded to 30 GiB, but the root partition and
filesystem had not yet grown to use the new capacity.

**Resolution:**

Install the growpart utility if it is not already present:

``` bash
sudo apt update
sudo apt install -y cloud-guest-utils
```

Identify the root disk and partition first:

``` bash
lsblk
```

In this lab the root disk was `/dev/nvme0n1` and the root partition was
`/dev/nvme0n1p1`. Confirm these names on your own instance before
running the next commands.

Grow partition 1:

``` bash
sudo growpart /dev/nvme0n1 1
```

Expand the ext4 filesystem:

``` bash
sudo resize2fs /dev/nvme0n1p1
```

Verify:

``` bash
df -hT /
lsblk
```

After expansion, the observed root filesystem was approximately 28 GiB,
with about 22 GiB available.

**Important:** Increasing the EBS volume size in AWS is not enough by
itself. The partition and filesystem may also need to be expanded. Never
copy the device name blindly; inspect `lsblk` first.

### Issue 2 --- Disk space was consumed by container images and build cache

**Investigation:**

``` bash
sudo docker system df
```

This showed Docker image storage and build cache usage. The demo
pulls/builds multiple images, so disk usage increases quickly.

**Operational approach:**

-   Allocate enough EBS capacity before deploying the full stack.
-   Check `df -h` and `docker system df` before rebuilding.
-   Remove only resources that are confirmed to be unused.
-   Avoid running broad prune commands while the application is in use.

For example, `docker image prune` removes dangling images;
`docker system prune` is broader. Read the command's impact before
confirming it. Do not use `docker system prune --volumes` casually
because it can remove data volumes.

### Issue 3 --- Grafana returned `504 Gateway Timeout`

**Observed symptoms:**

-   Grafana through the frontend proxy returned `504 Gateway Timeout`.
-   A direct request to the Grafana host port also timed out.
-   A request to Grafana's own `/api/health` endpoint from inside the
    Grafana container timed out.
-   The proxy container did not contain `wget`, so an attempted
    in-container test failed with `wget: not found`. That result did not
    prove Grafana itself was down.

**Useful investigation commands:**

``` bash
sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

sudo docker stats --no-stream

sudo docker logs grafana --since 15m 2>&1 | tail -100

sudo docker exec grafana \
  sh -c 'wget -S -O- -T 15 http://127.0.0.1:3000/api/health'

sudo docker port grafana 3000
```

**Finding:**

Grafana was configured with a 500 MiB container memory limit in
`compose.observability.yaml`. During investigation, `docker stats`
showed Grafana close to that limit (approximately 494 MiB / 500 MiB).
OpenSearch was also using a substantial amount of memory.

This made the restrictive Grafana memory limit a plausible contributor
to the slow responses. It was not proven to be the only possible cause.

**Configuration change made:**

A backup was created:

``` bash
cp compose.observability.yaml compose.observability.yaml.bak
```

The Grafana memory limit was edited from:

``` yaml
deploy:
  resources:
    limits:
      memory: 500M
```

to:

``` yaml
deploy:
  resources:
    limits:
      memory: 1000M
```

**Next verification required after the edit:**

Recreate only Grafana, leaving the rest of the running stack untouched:

``` bash
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  -f compose.observability.yaml \
  up -d --no-deps --force-recreate grafana
```

Then verify the configured limit and health endpoint:

``` bash
sudo docker inspect grafana \
  --format='Memory={{.HostConfig.Memory}}'

sudo docker exec grafana \
  sh -c 'wget -S -O- -T 15 http://127.0.0.1:3000/api/health'

sudo docker stats --no-stream grafana
sudo docker port grafana 3000
```

If Grafana still times out, inspect its logs, host memory, OpenSearch
memory, and proxy routing. Increasing a memory limit does not guarantee
a fix if the underlying cause is a datasource, database, plugin, or
application issue.

### Issue 4 --- Dynamic host ports changed after container recreation

**Observed behavior:**

Services such as Grafana and Jaeger were published on high-numbered host
ports. Grafana's mapped host port changed between container recreations
(for example, `32805` and `32806` appeared in separate checks).

**Reason:**

Compose configuration can publish a container port without specifying a
fixed host port. Docker then selects an available host port.

**Resolution / best practice:**

``` bash
sudo docker port grafana 3000
sudo docker port jaeger 16686
sudo docker ps --format "table {{.Names}}\t{{.Ports}}"
```

Always check the current mapping rather than saving a dynamic port in
documentation. The frontend proxy's fixed entry point was host port
`8080`; Prometheus was explicitly mapped to host port `9090`.

### Issue 5 --- A command was entered without its Docker Compose context

**Observed:**

The shell returned:

``` text
up: command not found
```

**Reason:**

`up` is not a standalone Linux command. It is a Docker Compose
subcommand.

**Correct usage:**

``` bash
sudo docker compose up -d
```

For the complete stack, include all three Compose files as shown
earlier.

### Issue 6 --- A test utility was missing in the proxy container

**Observed:**

``` text
wget: not found
```

**Reason:**

The frontend proxy image did not include `wget`. This is a limitation of
the debugging tool available in that container, not evidence that the
target service is unavailable.

**Better approach:**

Run the test from the host or from the target container:

``` bash
sudo docker exec grafana \
  sh -c 'wget -S -O- -T 15 http://127.0.0.1:3000/api/health'
```

If the target image also lacks the tool, use a temporary diagnostic
container on the same Docker network or test from the host using the
published port.

### Issue 7 --- Full stack resource pressure

**Observed:**

The host had about 7.6 GiB RAM and 2 vCPUs. At one point, the
application and observability containers together used several GiB of
memory. Grafana and OpenSearch were among the larger consumers.

**Operational lesson:**

-   Monitor `free -h` and `docker stats`.
-   Avoid running a second heavy environment (for example, a full
    Minikube stack) at the same time on this same EC2 instance.
-   Reduce load-generator activity if the host becomes constrained.
-   Consider a larger EC2 instance for concurrent Kubernetes and Compose
    labs.
-   Treat container memory limits as controls to review, not as proof
    that the host has enough RAM.

------------------------------------------------------------------------

## 18. Useful operational commands

Run commands from the upstream repository directory
(`~/opentelemetry-demo`).

### Start / update the complete stack

``` bash
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  -f compose.observability.yaml \
  up -d
```

### Check service status

``` bash
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  -f compose.observability.yaml \
  ps
```

### Follow logs for one service

``` bash
sudo docker logs -f grafana
sudo docker logs -f otel-collector
sudo docker logs -f prometheus
```

### Restart one service

``` bash
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  -f compose.observability.yaml \
  restart <service-name>
```

### Recreate one service after changing its Compose configuration

``` bash
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  -f compose.observability.yaml \
  up -d --no-deps --force-recreate <service-name>
```

### Stop the stack but retain containers and data

``` bash
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  -f compose.observability.yaml \
  stop
```

### Remove the stack

``` bash
sudo docker compose \
  -f compose.yaml \
  -f compose.full.yaml \
  -f compose.observability.yaml \
  down
```

`down` removes the Compose containers and network. It does not normally
remove named volumes unless `-v` is supplied. Do not add `-v` unless you
intentionally want to delete persisted data.

------------------------------------------------------------------------

## 19. Security considerations

This deployment is a learning lab. Before using it beyond a private lab:

-   Restrict SSH to a trusted source IP or use AWS Systems Manager
    Session Manager.
-   Restrict web access to your own IP or place the application behind a
    properly configured load balancer and TLS.
-   Do not expose internal microservice, database, Kafka, OpenSearch, or
    telemetry ports to the public internet.
-   Do not commit PEM keys, passwords, tokens, `.env` files containing
    secrets, or cloud credentials to GitHub.
-   Review the upstream demo's default credentials and configuration
    before exposing any UI.
-   Use TLS and authentication for externally accessible dashboards.
-   Patch the OS and Docker packages regularly.
-   Add monitoring and alerting for disk, memory, CPU, container health,
    and application availability.
-   Use IAM roles with least privilege rather than storing AWS
    credentials on the instance.

------------------------------------------------------------------------

## 20. Cleanup and cost control

EC2 compute, EBS storage, snapshots, and public IPv4 addresses can incur
charges depending on the account and region.

When the lab is not in use:

1.  Stop the Compose stack if you want to preserve the instance and its
    local data:

    ``` bash
    sudo docker compose \
      -f compose.yaml \
      -f compose.full.yaml \
      -f compose.observability.yaml \
      stop
    ```

2.  Stop the EC2 instance from the AWS Console if you do not need it
    running.

3.  Review EBS volumes, snapshots, Elastic IPs, and other billable
    resources.

4.  Terminate the instance only when you no longer need it. Confirm
    whether the root EBS volume is configured for deletion on
    termination before proceeding.

Stopping containers does not stop EC2 billing for the instance while it
is running. Stopping EC2 generally stops compute charges, but attached
EBS storage can continue to incur charges.

------------------------------------------------------------------------

## 21. Suggested repository structure

Create a **new repository** for this portfolio implementation rather
than changing the upstream OpenTelemetry Demo repository. Suggested
name:

`otel-astronomy-shop-devops`

The upstream application is a dependency; this repository should contain
your deployment documentation, architecture, operational evidence, and
future DevOps automation.

Suggested structure:

``` text
otel-astronomy-shop-devops/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── aws-ec2-setup.md
│   ├── docker-compose-deployment.md
│   ├── observability.md
│   ├── troubleshooting.md
│   ├── runbooks/
│   │   ├── disk-full.md
│   │   ├── grafana-timeout.md
│   │   └── container-unhealthy.md
│   └── incidents/
│       └── incident-001-grafana-timeout.md
├── diagrams/
│   ├── architecture.mmd
│   └── deployment-flow.mmd
├── screenshots/
│   ├── storefront/
│   ├── grafana/
│   ├── jaeger/
│   └── prometheus/
├── scripts/
│   ├── preflight-check.sh
│   └── health-check.sh
├── jenkins/
│   └── Jenkinsfile
├── k8s/
│   └── README.md
└── LICENSE
```

Do not copy the upstream repository wholesale into this new repository.
Link to it and document the exact version or commit you tested. Add
screenshots only after checking that they do not reveal public IPs,
account IDs, credentials, tokens, or other sensitive information.

### Create the repository on GitHub

1.  Open GitHub and select **New repository**.
2.  Repository name: `otel-astronomy-shop-devops`.
3.  Description:
    `AWS EC2 deployment of OpenTelemetry Astronomy Shop microservices with Docker Compose and observability`.
4.  Choose Public if you are comfortable publishing the lab details;
    otherwise choose Private.
5.  Add the README by uploading this file or initialize the repository
    with a README and replace it.
6.  Add the supporting `docs/`, `diagrams/`, `screenshots/`, and
    `scripts/` directories as they are created.

### Push from your local machine

After creating the empty repository on GitHub:

``` bash
git init
git add README.md docs diagrams screenshots scripts
git commit -m "docs: add Astronomy Shop deployment documentation"
git branch -M main
git remote add origin https://github.com/<YOUR_GITHUB_USERNAME>/otel-astronomy-shop-devops.git
git push -u origin main
```

If you create the repository with a README on GitHub first, clone it
locally or pull/reconcile the remote history before pushing.

------------------------------------------------------------------------

## 22. Next engineering milestones

The Docker Compose deployment is the first milestone. The next steps
should build on this working baseline rather than trying to implement
everything at once.

### Milestone 1 --- Finish operational validation

-   Recreate Grafana with the updated 1000M limit.
-   Verify `/api/health`, dashboard loading, and datasource queries.
-   Capture final `docker compose ps`, `docker stats`, and host resource
    outputs.
-   Record the exact EC2 instance type, AMI ID, Docker versions, and
    tested Git commit.

### Milestone 2 --- Observability and SRE exercises

-   Trace a checkout request across multiple services in Jaeger.
-   Query application and host metrics in Prometheus.
-   Build or refine Grafana dashboards.
-   Explore logs in OpenSearch.
-   Generate controlled traffic and observe the effect on latency,
    errors, and resource usage.
-   Use feature flags to simulate supported demo failure scenarios.
-   Write incident reports with detection, impact, timeline, root cause,
    mitigation, and follow-up actions.

### Milestone 3 --- CI/CD

Build a Jenkins pipeline for the deployment repository or a controlled
application fork:

1.  Checkout source.
2.  Run formatting and tests.
3.  Build container images where applicable.
4.  Run static analysis.
5.  Scan images for vulnerabilities.
6.  Push versioned images to a container registry.
7.  Deploy to a test environment.
8.  Run smoke tests and publish deployment status.

Do not claim that the upstream repository already includes this Jenkins
pipeline. Treat the pipeline as a portfolio engineering extension and
implement it in this repository.

### Milestone 4 --- Kubernetes

After the Compose deployment and operational exercises are stable:

-   Study the upstream Kubernetes deployment method and chart values for
    the exact release.
-   Deploy to a suitably sized Kubernetes environment.
-   Configure ingress, service discovery, secrets, resource
    requests/limits, and persistent storage.
-   Reuse the observability concepts and compare Compose versus
    Kubernetes operations.

The current EC2 instance has 2 vCPUs and approximately 8 GiB RAM.
Running the complete Compose stack and a full Kubernetes demo
simultaneously may cause resource pressure. Stop the Compose stack or
use a larger/separate instance for Kubernetes experiments.

### Milestone 5 --- Production-minded improvements

-   TLS and authentication.
-   Least-privilege IAM and network access.
-   Centralized secrets management.
-   Backup and restore procedures.
-   Resource capacity planning.
-   SLOs, alerting, and error-budget policy.
-   Repeatable infrastructure provisioning with Terraform.

------------------------------------------------------------------------

## 23. References

-   **Upstream OpenTelemetry Demo:**
    https://github.com/open-telemetry/opentelemetry-demo
-   **OpenTelemetry documentation:** https://opentelemetry.io/docs/
-   **Docker Engine installation on Ubuntu:**
    https://docs.docker.com/engine/install/ubuntu/
-   **Docker Compose documentation:** https://docs.docker.com/compose/
-   **AWS EC2 documentation:** https://docs.aws.amazon.com/ec2/
-   **CloudChamp project walkthrough:**
    https://www.youtube.com/watch?v=nWDCWe5OwjQ
-   **Project reference notes:**
    https://cloudchamp.notion.site/Otel-Astronomy-Shop-Demo-App-ea4a469e0e06425b9ef7dcfa84b37f50

The CloudChamp links are reference material. The actual deployment
documented here is based on the commands and outputs captured during our
EC2 implementation.

------------------------------------------------------------------------

## Project status

  -----------------------------------------------------------------------
  Area                                Status
  ----------------------------------- -----------------------------------
  EC2 provisioning                    Completed

  Ubuntu preparation                  Completed

  Docker Engine and Compose           Completed
  installation                        

  Upstream repository clone           Completed

  Core Compose deployment             Completed

  Full demo layer                     Completed

  Observability layer                 Deployed

  Storefront validation               Completed

  Test checkout                       Completed

  Prometheus validation               Completed

  Grafana dashboard exploration       Completed; intermittent timeout
                                      investigated

  Grafana 1000M limit change          Edited in Compose; container
                                      recreation and final verification
                                      still to be recorded

  CI/CD pipeline                      Planned

  Kubernetes deployment               Planned

  SLOs, alerts, and incident runbooks Planned
  -----------------------------------------------------------------------

------------------------------------------------------------------------

**Author:** Rahul Wagh\
**Focus:** DevOps Engineering · Cloud · SRE
