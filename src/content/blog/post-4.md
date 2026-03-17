---
title: "A Developer's Guide to Edge AI Workloads with Metro AI Suite"
excerpt: "In our previous blog, \"Intersection to Infrastructure: Enabling Edge AI with Metro AI Suite,\" we explored why a pragmatic, end‑to‑end pipeline is essential for building scene‑aware traffic analytics at the edge. The Smart Intersection sample application demonstrates a practical, modular pattern for multi‑camera traffic understanding; combining edge‑side object detection, scene‑level synchronized time stamping, and dashboards."
publishDate: 'March 3 2026'
isFeatured: true
tags:
  - Computer Vision
  - OpenEdge Platform
  - Intel
---

In our previous blog, “Intersection to Infrastructure: Enabling Edge AI with Metro AI Suite,” we explored why a pragmatic, end‑to‑end pipeline is essential for building scene‑aware traffic analytics at the edge. The Smart Intersection sample application demonstrates a practical, modular pattern for multi‑camera traffic understanding; combining edge‑side object detection, scene‑level synchronized time stamping, and dashboards. All components are packaged so developers can deploy and test the entire workflow locally, using Kubernetes or Helm, making it easy to replicate, customize, and scale.

## Architectural Layers

The Smart Intersection application is a set of containerized microservices, deployable with Docker or orchestrated through Kubernetes for larger edge installations.

At a high level, the system includes an Inference Service that uses DL Streamer to process video streams and publish detection metadata to MQTT, a Scene Controller that applies intersection‑specific configuration to generate fused multi‑camera tracks, and an Analytics Service that aggregates these tracks into metrics stored in InfluxDB and visualized through Grafana. Node‑RED enables optional workflow customization and external integrations.

This modular separation, covering inference, fusion, analytics, storage, and visualization, keeps the pipeline flexible, scalable, and easy to extend across different intersection layouts and deployments.

![Smart Intersection Application](/smartintersection.webp)

### Data Ingestion and Stream management

Video feeds form the foundation of the workload. The application can process multiple camera streams delivered through RTSP or via prerecorded video files for testing and simulation.

A MediaMTX server handles stream ingestion and redistribution. By acting as a lightweight RTSP/RTMP gateway, it decouples camera sources from the inference pipeline and enables flexible stream fan‑out, routing, and replacement of cameras without modifying the AI services.

### AI Inference Pipeline

After ingestion, frames enter the DL Streamer‑based inference pipeline. DL Streamer handles decoding, batching, preprocessing, model inference, and object tracking entirely as a streaming graph. This is the heart and brain of the entire application, which also determines the pipeline scalability. AI model choices, stream resolution, stream density, etc., help dictate compute requirements to run the workloads performantly and cost effectively. Optimizing such a pipeline for the desired throughput drives one of the most important business outcomes — i.e. scaling applications cost-effectively for enhanced total cost of ownership.
Object detection models optimized with OpenVINO provide hardware‑efficient execution on Intel processors and integrated or discrete accelerators available on edge devices.

The pipeline supports multi‑camera concurrent processing, allowing inference throughput to scale with the number of cameras or increased stream resolutions. Instead of passing raw frames downstream, the inference service outputs compact metadata, object bounding boxes, classifications (if enabled), track IDs, and confidence scores, reducing network and compute overhead for analytics services.

![Smart Intersection Dashboard](/si-dashboard.webp)

### Lightweight Messaging and Event Flow

Inference results and system events flow through a lightweight, event‑driven messaging layer. MQTT serves as the backbone for distributing inference metadata, analytics updates, and control signals. This decoupled communication pattern allows storage, analytics, dashboards, and automation tools to subscribe to relevant data streams independently. MQTT, compared to alternatives like Kafka, RabbitMQ, or NATS, is lightweight, easy to deploy on low powered constrained devices, and optimized for fast, small metadata updates, making it a strong fit for real‑time, multi‑service edge AI workloads like Smart Intersection.

Node‑RED complements MQTT by providing a visual, low‑overhead orchestration layer. Developers can implement event logic, external system hooks, or custom routing without modifying core microservices, making the system extensible and easy to integrate with third‑party applications.

Using an event‑driven messaging layer allows the application to remain loosely coupled, letting each service subscribe only to the data it needs and scale independently without overloading the system with constant pub‑sub broadcasts. This reduces bandwidth and compute overhead, improves resilience when downstream services fail, and makes it easy to extend the system by adding new consumers without modifying existing microservices.

Node‑RED further simplifies routing and workflow customization without code changes. The trade‑offs are mainly around managing event ordering, handling bursts of messages, and debugging distributed flows, but the flexibility and efficiency gained make this architecture well‑suited for real‑time, multi‑service edge AI workloads.

### Data Persistence and Analytics Storage

The application uses multiple specialized datastores to handle different data types efficiently:

- PostgreSQL manages structured configuration data such as camera mapping, intersection layout, and system metadata.
- InfluxDB stores time‑series analytics; vehicle counts, occupancy, throughput, and system metrics, optimized for high‑write performance and retention policies.
- Milvus is available for workloads requiring vector storage or similarity search (for example, advanced behavior analytics or embedding‑based retrieval).

Using specialized datastores ensures each type of data is handled by the engine best optimized for it, improving performance and reducing contention. This separation keeps the pipeline responsive and scalable as data volume or analytics complexity grows.

## Observability and Monitoring

Observability is built into the application to help developers understand performance, latency, and system behavior across the pipeline. OpenTelemetry collects metrics and traces from microservices, providing visibility into end‑to‑end processing time, inference throughput, and system health. This layer also improves reliability by exposing bottlenecks, latency spikes, and resource contention early, helping developers diagnose issues across microservices and keep the pipeline performing consistently as conditions change.

Grafana dashboards combine telemetry data with time‑series metrics from InfluxDB to present real‑time and historical views of traffic trends, system load, and service performance. This gives developers an immediate, reliable view of system behavior, making it easier to spot performance issues early, validate that the pipeline is running correctly, and maintain consistent operation as traffic patterns or device conditions change.

![Grafana Monitoring Dashboard](/grafana.webp)

## Security Enablement at the Edge

Security in the Metro AI Suite is built into both the platform and application layers to meet the needs of edge deployments, which often operate unattended and outside traditional data‑center environments. Security in the Metro AI Suite is built into both the platform and application layers to meet the needs of edge deployments, which often operate unattended and outside traditional data‑center environments. For edge AI systems like Smart Intersection, processing continuous real‑world sensor data at the network edge, strong protection is essential to prevent tampering, preserve data integrity, and keep analytics trustworthy. A zero‑trust posture is required; every component must authenticate, all communication must be encrypted, and no service is implicitly trusted, ensuring the system remains hardened even in exposed or physically accessible environments.

### Platform-Level Security

Intel’s hardware‑rooted security establishes trust before workloads run:

- UEFI secure Boot ensures only trusted firmware and OS components load.
- Discrete TPM (dTPM) stores keys, certificates, and authentication secrets securely.
- Memory protection keeps sensitive inference and analytics data isolated during runtime.

### Application-Level Security

At the application layer, Smart Intersection enforces secure communication and isolation:

- TLS protects data exchanged between services like MQTT, InfluxDB, and the portal.
- Separated credentials keep service secrets out of application code.
- Container isolation limits the impact of failures or vulnerabilities.
- Event‑driven decoupling reduces direct service dependencies and shrinks the attack surface.

### Integrated Security in the Edge AI Workflow

These protections are built into normal operation: trust is validated at boot, data paths remain encrypted, and services stay isolated and observable. Combined hardware and software safeguards deliver the reliability and resilience required for real‑world edge AI deployments.

### Deployment Flow

Deploying the Smart Intersection application follows a straightforward sequence:

1. Clone the repository and navigate to the Smart Intersection recipe

```bash 
git clone https://github.com/open-edge-platform/edge-ai-suites.git
cd edge-ai-suites/metro-ai-suite/metro-vision-ai-app-recipe/
```

2. Run the installer

The installer downloads required models, configures application assets, and prepares the environment.

```bash
./install.sh smart-intersection
```

3. Export the admin password

The installer generates a password stored under src/secrets. Export it so dependent services can access it.

```bash
export SUPASS=$(cat ./smart-intersection/src/secrets/supass)
```

4. Start application microservices with Docker Compose

This pulls the container images and launches all microservices defined in the recipe.

```bash
docker compose up –d
```

5. Verify running containers

```bash
docker ps
```

6. Access application endpoints

All services are routed through an nginx reverse proxy running at https://localhost.
If accessing remotely, replace localhost with the host machine’s IP address.

The deployment uses self‑signed certificates, so browsers will show warnings.

Default endpoints

- Application portal / reverse proxy: https://localhost
- Grafana: https://localhost/grafana/ (default login: admin / admin)
- InfluxDB: https://localhost:8086
- Node‑RED: https://localhost/nodered/
- DL Streamer pipeline status API: https://localhost/api/pipelines/status

## Edge Constraints and Runtime Profile

Edge AI workloads operate under constraints that shape how the Smart Intersection pipeline is built and tuned. Unlike cloud systems, edge deployments must balance latency, throughput, and limited compute resources close to the data source.

### Latency vs. Throughput

Real‑time traffic analytics require low latency to track objects frame‑to‑frame, while multiple high‑resolution camera feeds increase throughput demands. Adjusting batch sizes, concurrency, or model complexity affects both, so the pipeline must balance these competing needs.

### Memory and I/O Limits

Video decoding, buffering, and metadata handling generate heavy memory and I/O pressure. On devices with limited RAM or shared memory, contention across CPU, GPU, and media pipelines can impact performance. Efficient data flow, minimal copies, optimized buffers, compact metadata is crucial.

### Power and Thermal Constraints

Many edge devices run in fanless or thermally restricted environments. Continuous multi‑camera inference can reach thermal or power limits, requiring careful tuning of concurrency, model load, or inference frequency to avoid throttling.

### Real‑Time vs. Best‑Effort Tasks

The pipeline mixes real‑time workloads (inference, scene fusion) with best‑effort tasks (analytics, dashboards, logging). Prioritizing latency‑critical components helps maintain stability under load.

## Optimizing on Intel Platforms

Optimizing edge AI on Intel platforms requires aligning pipeline design with available hardware.

### CPU Efficiency and Parallelism

Performance depends on effective use of Intel CPUs — vectorization, multithreading, and efficient memory access. Inference, decoding, and preprocessing should use available cores without causing contention, ensuring the pipeline keeps up with incoming video streams.

### OpenVINO Optimization

OpenVINO enables hardware‑aware inference through model compression, operator fusion, and graph optimizations. This lets the same application run efficiently across Intel CPUs, integrated GPUs, and accelerators without major code changes.

### Profiling and Bottleneck Removal

Regular profiling is essential on constrained edge devices. Bottlenecks in decoding, inference, or metadata processing can be addressed by rebalancing compute, adjusting threading or batching, reducing data movement, and tuning buffer sizes. Iterative optimization is key to sustaining performance.

## Deployment, Extension, and Operational Patterns

Metro AI Suite supports multiple deployment scenarios common in edge environments.

### Deployment Models

- Bare‑metal for maximum control and predictable performance
- Containerized deployments for portability across sites
- Kubernetes for multi‑node scaling, automated updates, and distributed operations

### Designed for Extension

Smart Intersection is built to be extended, not used as a fixed template. Developers can add analytics modules, swap in custom models, integrate new sensors, or modify scene logic using Node‑RED — all without reworking the pipeline.

### Operational Best Practices

Edge operations benefit from modular pipelines, regular profiling as workloads evolve, and safe rollout/rollback strategies to minimize risk, especially for remote or unattended deployments. These patterns support stable, scalable operation from pilot to production.

## Getting Started

Building effective edge AI systems requires understanding workload behavior, resource constraints, and the hardware capabilities of the target platform. The Metro AI Suite simplifies this process by providing open, reference workloads, such as the Smart Intersection application that serve as practical blueprints for designing, evaluating, and tuning visual AI pipelines on Intel edge hardware.

With support for multiple deployment models, built‑in observability, and repeatable benchmarking, the suite offers a clear path for developers moving from early experimentation to production‑ready edge deployments. It allows teams to explore real‑world scenarios, optimize pipelines with OpenVINO and DL Streamer, and adapt reference solutions to match their own edge AI use cases.