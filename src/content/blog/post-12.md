---
title: From DL Streamer Pipelines to Multimodal Decisions: Understanding Data Flow with Weld Defect Detection
excerpt: Your DL Streamer pipeline detects defects. What happens next?
publishDate: 
tags:
  - Computer Vision
  - OpenEdge Platform
  - Intel
  - DL Streamer
---
Your DL Streamer pipeline detects defects. What happens next?

In the previous article, “Building a DL Streamer Application: From Pipeline Architecture to Model Integration,” we have seen how to construct a DL Streamer pipeline that ingests video, performs inference, and generates metadata. That pipeline is the vision analytics core of the system.
However, the real-world edge AI systems are not built around a single pipeline; they are built around how multiple components communicate and work together.
This article focuses on Weld Defect Detection Sample Application as an example, to explain data flow across pipelines, how results are routed through services, and how multimodal fusion combines outputs from different analytics layers to produce a final decision.

## System Overview: Moving Beyond a Single Pipeline
The weld defect detection application is designed as a distributed multimodal system. It consists of multiple cooperating services instead of a monolithic pipeline.
At a high level, the system is composed of several interconnected components that work together to process, analyze, and interpret data. The Weld Data Simulator acts as the entry point, generating synchronized video and time-series data that emulate a real welding process. This data is then consumed by the DL Streamer Pipeline Server, which performs vision-based defect detection and produces structured metadata from the video stream.
In parallel, the time-series data is processed by the Time-Series Analytics component, which analyzes process signals and detects anomalies over time. The outputs from both these pipelines, vision detections and time-series anomaly signals are then sent to the Fusion Analytics module, where they are combined to produce a final decision about weld quality.
To enable communication between all these components, the system uses an MQTT Broker as a central messaging backbone. Video streaming is handled by MediaMTX along with WebRTC (via Coturn), allowing processed video frames and overlays to be delivered in real time to users. Meanwhile, InfluxDB and Grafana are used for storing and visualizing analytical data, providing dashboards for monitoring system outputs. Finally, SeaweedFS (S3-compatible storage) is used to archive frames and metadata, ensuring that historical data is available for further analysis and traceability. Instead of a single linear pipeline, the system is structured as parallel analytics paths with a fusion layer.

### Vision Pipeline: DL Streamer in Context
The DL Streamer Pipeline Server is responsible for processing the video stream within the system. It reads frames from an RTSP source provided by MediaMTX, which simulates or delivers live camera input. These frames are then passed through a trained defect detection or classification model, enabling the system to analyze weld quality in real time. For each frame, the pipeline generates structured metadata that captures the inference results, such as whether a defect is detected and the associated confidence score. This metadata is published to downstream components via MQTT, allowing other services in the architecture to consume and act on it. In parallel, the pipeline also produces annotated video streams with overlays (such as bounding boxes or labels), which are sent through the streaming layer for visualization in user interfaces like Grafana.
A typical output from this pipeline looks like:
```json
{
  "timestamp": 1716200000,
  "defect_detected": true,
  "confidence": 0.92
}
```
This output does not trigger decisions by itself. It is an intermediate signal used by downstream components.

### Time-Series Analytics Pipeline
Parallel to the vision pipeline, the system processes time-series data derived from the welding process.
This flow includes:
- Data ingestion via MQTT and Telegraf
- Storage in InfluxDB
- Execution of anomaly detection functions (UDFs)
The time-series pipeline produces processed outputs, not raw signals. For example:
```json
{
  "timestamp": 1716200000,
  "anomaly_detected": true,
  "score": 0.91
}
```
This is important:
- The system does not fuse raw sensor values
- It fuses analytics outputs from independent pipelines

## MQTT: The Central Data Backbone
The system relies heavily on MQTT for communication, acting as the central messaging backbone that connects all components. Instead of tightly coupling services, each component publishes its outputs to specific MQTT topics and subscribes to the data it needs. For example, the vision pipeline (DL Streamer) publishes defect detection metadata, while the time-series analytics pipeline publishes anomaly detection results based on process data. The fusion layer then subscribes to both of these streams, aligns them using timestamps, and publishes a final decision back to MQTT. This event-driven design ensures that components remain independent, making the system more flexible, scalable, and easier to extend or modify without affecting the entire architecture.

### Fusion Analytics: The Decision Layer
The fusion analytics service, implemented in fusion.py, is the component that turns separate analytics outputs into a final decision.
The role of the fusion layer is simple in structure but critical in function. It receives outputs from both analytics pipelines; the vision pipeline, which provides a defect_detected signal, and the time-series pipeline, which produces an anomaly_detected signal. These events are first aligned using timestamps to ensure they correspond to the same point in time or process instance. Once aligned, the fusion layer applies configurable logical rules, such as AND or OR conditions, to determine whether a defect should be flagged. Based on this evaluation, it produces and publishes a single fused decision, which represents the final system output used for alerting, visualization, and downstream processing.

Fusion Logic
The fusion process does not analyze images or sensor signals directly. Instead, it applies logical rules such as AND or OR.
Example:

```python
def fuse_events(vision_event, ts_event, rule="AND"):
    vision_flag = vision_event["defect_detected"]
    ts_flag = ts_event["anomaly_detected"]

    if rule == "AND":
        final_defect = vision_flag and ts_flag
    elif rule == "OR":
        final_defect = vision_flag or ts_flag
    else:
        final_defect = False

    return {
        "timestamp": vision_event["timestamp"],
        "vision_flag": vision_flag,
        "ts_flag": ts_flag,
        "final_defect": final_defect
    }
```
### Why This Matters
Each individual pipeline has limitations:
- Vision may produce false positives due to noise
- Time-series analytics may flag anomalies unrelated to visible defects
Fusion improves reliability by combining them:
- AND rule: Higher precision but only flag defect if both pipelines agree
- OR rule: Higher sensitivity but flag defect if either pipeline detects an issue
This makes fusion the core decision engine of the system.

## Data Flow: Step-by-Step
The full system flow can be understood as a sequence of transformations:
1. Data Generation
The system begins with the Weld Data Simulator, which generates synchronized data streams that mimic a real welding process. It produces a video stream over RTSP, which represents the visual weld process, and time-series data over MQTT, which captures process-related signals. These two streams are time-aligned, ensuring that both visual and process data correspond to the same welding instance.
2. Parallel Processing
Once generated, the data flows through two independent processing pipelines. The DL Streamer pipeline consumes the RTSP video stream, performs inference using a trained model, and produces structured detection metadata for each frame. At the same time, the time-series analytics pipeline processes the incoming data signals and applies anomaly detection logic. This results in a separate stream of anomaly outputs, representing whether the process conditions deviate from expected behavior.
3. Fusion
The fusion service acts as the decision layer that brings together the outputs of both pipelines. It consumes the detection metadata from the vision pipeline and the anomaly results from the time-series pipeline. These events are aligned using timestamps to ensure they correspond to the same point in time. The fusion logic then applies rule-based conditions, such as AND or OR, to determine the final defect status, producing a unified decision.
4. Distribution
The final output from the fusion layer is distributed to different systems depending on its purpose. It is published to MQTT for real-time alerts, enabling immediate response. It is also visualized in Grafana dashboards, where users can monitor system behavior and trends. Additionally, results are stored in storage systems for archival and further analysis. This ensures that the system supports real-time operations as well as historical insights.

## Output and Visualization
The system exposes results through multiple channels, allowing different components and users to access the data in ways that suit their needs. MQTT is used for publishing real-time alerts and events, enabling downstream systems to react immediately to detected defects or anomalies. At the same time, the Grafana UI provides a visual interface where users can monitor system behavior, including anomaly signals from the time-series pipeline and fused decisions from the fusion layer.
For visual inspection, the system delivers an annotated video stream through WebRTC, where users can see the processed video along with overlays such as detection results. In addition to real-time outputs, the system also ensures persistence through S3-compatible storage (SeaweedFS), where frames, metadata, and results are archived for later use.
This multi-channel approach enables several capabilities: users can monitor the system in real time, inspect historical data for analysis or debugging, and trace how decisions were made across the different components of the architecture.

## Real-Time vs Analytical Paths
The architecture supports different usage paths: 
- Hot Path (Real-Time)
Fusion → MQTT → alerts
- Warm Path (Visualization)
Annotated video → WebRTC → Grafana
- Cold Path (Archival)
Data → storage → offline analysis
All three paths originate from the same pipeline outputs.

## Common Integration Challenges
Working with this architecture introduces challenges:
- Timestamp misalignment
Fusion fails if events cannot be matched
- Schema inconsistency
Vision and time-series outputs must share structure
- Incorrect fusion logic
Choosing wrong rule (AND vs OR) affects results
- Tight coupling between components
Avoid embedding fusion logic inside pipelines

The recommended approach is:
- Keep pipelines independent
- Use MQTT for communication
- Let fusion remain a separate service
- Understanding Sensor Fusion in This System
Although this is often called a sensor fusion system, the implementation is more structured than directly combining signals.
Instead of:
raw temperature + video → decision
The system uses:

This layered approach has several advantages:
- Easier to scale
- Easier to debug
- Independent model improvements
- Ability to add new modalities later
For example, you could add:
- acoustic data
- vibration sensors
- additional vision models
without changing the fusion logic significantly.

## Conclusion
The weld defect detection application demonstrates how to move from a single DL Streamer pipeline to a full multimodal system.
In this architecture:
- DL Streamer handles vision inference
- Time-series analytics handles sensor anomaly detection
- Fusion combines both into a final decision
The key idea is that intelligence does not come from a single model, but from how multiple systems interact.
In the next article, the focus will shift to the upstream part of this pipeline; model development and optimization. You will explore how models are trained, prepared, and integrated into DL Streamer pipelines, and how tools like the DL Streamer coding agent can accelerate development.
