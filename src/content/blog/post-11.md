---
title: Building a DL Streamer Application; From Pipeline Architecture to Model Integration
excerpt: Building a real-time video analytics application can seem complex at first; especially when you need to ingest live streams, process frames efficiently, and run AI models on them. 
publishDate: June 2 2026
tags:
  - Computer Vision
  - OpenEdge Platform
  - Intel
  - DL Streamer
---
Building a real-time video analytics application can seem complex at first; especially when you need to ingest live streams, process frames efficiently, and run AI models on them. This is where DL Streamer helps. Built on top of the GStreamer multimedia framework, DL Streamer allows you to construct media analytics applications as pipelines, where video data flows through stages such as decoding, preprocessing, inference, and output. This pipeline-based approach simplifies development by turning what would otherwise be complex, multi-stage logic into a configurable data flow. 
In the blog  Intel® DL Streamer: Simplifying Media Analytics Optimized for the Edge, we have seen how the framework is designed to make it easier to develop, optimize, and deploy video analytics workloads on edge systems by combining media processing with AI inference in a single pipeline. 
This article focuses on how to construct such a pipeline using the DL Streamer Pipeline Server. It explains how the pipeline is structured, how a model is integrated and configured, and how data flows from raw frames to inference results.

## Understanding Application Architecture
A DL Streamer–based application is organized as a combination of model artifacts, pipeline configurations, and execution logic, rather than a single monolithic program. This separation ensures that different parts of the system can be updated independently.
Key components typically include:
Model files: OpenVINO IR (.xml, .bin) used for inference
Pipeline definition: Describes elements and how data flows between them
Configuration files: Parameters for preprocessing, inference, and outputs
Execution scripts: Used to start and manage the pipeline
This structure allows you to modify one part, such as swapping a model or changing preprocessing, without affecting the rest of the application.

At the core of the application is the pipeline, which acts as a processing chain. Data flows through a sequence of transformations, where each stage performs a specific task.

Each stage has a clear role:
- Source: Reads frames from a file or stream
- Preprocessing: Converts frames into model-ready format
- Inference: Runs the model and produces predictions
- Postprocessing: Transforms raw outputs into structured results
- Output: Sends results to downstream systems (e.g., MQTT, storage, UI)
We will walk through each of these stages in further sections. 
The key architectural idea is that pipelines are built from modular, composable elements.
Each element performs a single responsibility, accepts defined input formats and produces predictable outputs.
Because of this design, elements can be:
- Replaced (e.g., file input → RTSP stream)
- Reconfigured (e.g., CPU → GPU inference)
- Reused across different pipelines
This modularity enables flexibility and scalability without requiring major code changes.

In the weld defect detection system, the DL Streamer pipeline server represents this processing chain. It ingests video frames, performs inference, and produces metadata. This output is then consumed by downstream components such as messaging systems and analytics services, making the pipeline a core building block within a larger architecture.

## The Inference Element: Core of Model Integration 
The inference element is the core component where model integration happens within a DL Streamer pipeline. It is responsible for taking prepared input data, running it through a trained model, and producing structured results that downstream elements can use.
At a high level, the inference element performs three key functions:
Loads the model into memory (typically an OpenVINO IR model)
Runs inference on incoming frames or tensors
Outputs results (such as detections, classifications, or embeddings) as metadata
This makes it the central bridge between raw data processing and intelligent decision-making.

### Model Configuration
To use the inference element, you need to specify where the model is located and how it should be executed. This is done through configuration parameters.
Typical configuration includes:
Model path: Location of .xml and .bin files
Device selection: CPU, GPU, or other accelerators
Batch size: Number of inputs processed together
Inference settings: Performance and precision options
For example, the pipeline must explicitly point to the model files so that the inference element can load them at runtime.

### Input Requirements
The inference element expects inputs in a specific tensor format, which must match the model’s expectations. This is why preprocessing is critical.
Common input requirements include:
Shape: e.g., 1 x 3 x 416 x 416 (batch × channels × height × width)
Data type: typically, float32 or uint8
Color format: RGB or BGR
Normalization: scaling pixel values to a required range
If these requirements are not met, inference may fail or produce incorrect results. The preprocessing stage ensures that raw frames are transformed into this expected format.

### Output Structure
The inference element produces outputs as metadata attached to each frame. The exact structure depends on the model type.
Typical outputs include:
Bounding boxes (for detection models)
Confidence scores
Class labels
Raw tensors (optional)
This metadata is then passed to postprocessing or directly to downstream systems such as MQTT or analytics services.

### Example Configuration
Below is a simplified example from weld defect detection sample application, how an inference element might be defined in a pipeline configuration:

```json
{
  "name": "inference",
  "type": "gvainference",
  "properties": {
    "model": "deployment/models/model.xml",
    "device": "CPU",
    "batch-size": 1,
    "inference-region": "full-frame"
  }
}
```
The inference element encapsulates the complexity of model execution behind a simple interface. By configuring it correctly and ensuring proper input formatting, you can integrate trained models seamlessly into a real-time pipeline.

## Preprocessing: From Raw Frames to Model Input
Before a model can perform inference, raw video frames must be transformed into the exact format the model expects. This step is known as preprocessing, and it is essential for ensuring both correctness and performance. Models are trained on data with specific characteristics, such as resolution, color format, and value range, and any mismatch at runtime can lead to incorrect or inconsistent results.
Preprocessing prepares each frame so that it matches these expectations. Typical transformations include:
Resize: Adjusts frames to the input dimensions required by the model
Color space conversion: Converts formats (e.g., BGR to RGB) depending on model training
Normalization: Scales pixel values to a specific range (e.g., 0–1 or -1 to 1)
Layout conversion: Rearranges data into tensor format (e.g., NHWC → NCHW)
Conceptually, this stage can be viewed as a transformation pipeline:
Raw Frame → Resize → Color Convert → Normalize → Tensor
 
Each step incrementally converts raw image data into a structured tensor that the inference element can consume.
In DL Streamer, preprocessing is not a single function, but a chain of elements configured within the pipeline. These elements operate sequentially, ensuring that by the time data reaches the inference stage, it is fully compliant with the model’s input requirements.
A simplified example configuration might look like:

```json
{
  "name": "preprocessing",
  "elements": [
    {
      "type": "videoscale",
      "properties": {
        "width": 416,
        "height": 416
      }
    },
    {
      "type": "videoconvert",
      "properties": {
        "format": "RGB"
      }
    },
    {
      "type": "tensor_convert"
    }
  ]
}
```
In the weld defect detection use case, preprocessing typically involves resizing each video frame to a fixed resolution (for example, 416×416) and converting it into the tensor format expected by the detection model. This ensures that every frame is processed consistently, allowing the model to produce reliable detection results.

## Postprocessing: From Inference Output to Metadata 
Once inference is complete, the model produces raw outputs that are not immediately usable. These outputs typically include tensors containing prediction data, such as bounding boxes, class probabilities, or raw logits. Postprocessing is the stage where these raw results are transformed into structured, meaningful metadata that can drive decisions and integrations.
Understanding the nature of inference output is the first step. For detection models, the output often consists of:
- Bounding box coordinates (object locations)
- Confidence scores (probability of correctness)
- Class labels or IDs
- Raw tensors representing predictions
However, these outputs may contain redundant or low-confidence detections. They are also often represented in formats that are not directly usable by downstream systems.
Postprocessing converts these raw outputs into actionable information. Common steps include:
- Non-Maximum Suppression (NMS): Removes overlapping bounding boxes for the same object
- Filtering: Discards detections below a confidence threshold
- Label mapping: Converts class IDs into human-readable labels
- Formatting: Structures results as metadata (e.g., JSON or frame-attached metadata)
Conceptually, this stage can be represented as:
Raw Inference Output → Filtering/NMS → Structured Metadata
 
In DL Streamer, postprocessing can be handled through built-in elements or custom logic, depending on the complexity of the model and the desired output format.
A simplified configuration example might look like:
```json
{
  "name": "postprocessing",
  "type": "gvapython",
  "properties": {
    "module": "postprocess.py",
    "class": "PostProcessor"
  }
}
```
In this setup, a custom Python module applies filtering logic, formats results, and prepares metadata for downstream use.
In the weld defect detection example, postprocessing transforms raw detections into interpretable results such as:
- “Defect detected: crack”
- “Location: (x, y, width, height)”
- “Confidence: 0.92”
This structured metadata is then passed along the pipeline, where it can be published to messaging systems, visualized in dashboards, or used by analytics components for further processing.
Postprocessing therefore plays a critical role in bridging model output and real-world application logic, ensuring that inference results are both accurate and usable.

## Putting All Together: Complete Pipeline Configuration
Once preprocessing, inference, and postprocessing are defined, they can be combined into a single pipeline configuration. In this format, the pipeline is expressed as a GStreamer string, with named elements that can be configured dynamically through parameters. This approach keeps the execution flow compact while still allowing fine-grained control over individual components.
Below is an example adapted for weld defect detection, following the same structure:
```json
{
  "config": {
    "pipelines": [
      {
        "name": "weld_defect_detection",
        "source": "gstreamer",
        "queue_maxsize": 50,
        "pipeline": "rtspsrc location="rtsp://mediamtx:8554/live.stream" latency=100 name=source ! rtph264depay ! h264parse ! decodebin ! videoconvert ! videoscale ! video/x-raw,format=BGR,width=416,height=416 ! gvainference name=inference inference-region=full-frame ! gvawatermark ! gvametaconvert add-empty-results=true add-rtp-timestamp=true name=metaconvert ! queue ! gvametapublish method=mqtt address="tcp://mqtt-broker:1883" ! appsink name=destination",
        "parameters": {
          "type": "object",
          "properties": {
            "inference-properties": {
              "element": {
                "name": "inference",
                "format": "element-properties"
              }
            }
          }
        }
      }
    ]
  }
}
```
Explanation of Key Sections
- Source and decoding
rtspsrc → rtph264depay → h264parse → decodebin
Handles ingestion of the RTSP stream and converts it into raw video frames.
- Preprocessing
videoconvert → videoscale → caps filter
Ensures frames are resized to 416x416 and formatted correctly (BGR) for the model.
- Inference
gvainference name=inference
Runs the weld defect detection model on each frame. The element is named so its properties can be overridden dynamically.
- Postprocessing and metadata conversion
gvawatermark → gvametaconvert
 Adds visual overlays (optional) and converts inference results into structured metadata.
- Output
gvametapublish (MQTT) → appsink
 Publishes detection results to an MQTT broker and exposes output for further integration.

Configuration Parameters
- Device selection (via inference properties): CPU, GPU, etc.
- Batch size: Controls throughput vs latency
- Queue size: Helps manage buffering in real-time pipelines

This configuration represents a complete, working pipeline:
- Input video is ingested and preprocessed
- The model performs inference
- Results are converted into metadata
- Outputs are published for downstream systems
Each component remains modular, but together they form a cohesive end-to-end weld defect detection pipeline.

## Model Preparation & Loading 
Before a model can be used in a DL Streamer pipeline, it must be trained and packaged in a compatible format. In this workflow, models are typically trained using tools such as Geti, and exported in the OpenVINO Intermediate Representation (IR) format (.xml and .bin).
In this project, model deployment is not configured directly inside the pipeline, but instead referenced through a project.json deployment file. This file acts as a central configuration for the model, including its location, metadata, and runtime parameters.
Practically, this means:
- Model files are stored under a structured directory (for example, /models/.../deployment/)
- The project.json file describes the model setup
- The inference element references this configuration rather than hardcoding everything inline
This approach improves maintainability by separating:
- Pipeline logic (how data flows)
- Model configuration (what model to use and how)
In production environments, this also supports model versioning, since each model can have its own deployment folder and configuration. Updating a model can be done by switching the referenced deployment without modifying the pipeline itself.
This article focuses on how models are integrated into the pipeline. 
Details on model training, optimization, and deployment configuration (including project.json) will be covered in later posts in this series.

## Common Integration Pitfalls
When building and running pipelines, a few common issues appear frequently. Understanding them early can save significant debugging time.
### Model loading failures
Cause: Incorrect file path or unsupported model format
Fix: Verify the .xml path, ensure the .bin file is present, and confirm the model is compatible with OpenVINO
### Pad negotiation errors
Cause: Output format of one element does not match the input expected by the next
Fix: Add or adjust elements like videoconvert, videoscale, or caps filters to ensure format consistency
### Inference performance or timeouts
Cause: Model too heavy for available hardware or frame rate too high
Fix: Use a lighter model, reduce input resolution, or switch to GPU/accelerator
### Memory issues
Cause: Large batch size or excessive buffering
Fix: Reduce batch size, tune queue sizes, and monitor system resource usage

These issues are usually configuration-related and can be resolved without changing the overall pipeline structure.

## Next Steps & Bridge
At this point, the pipeline is able to process video, run inference, and produce structured metadata describing weld defects. While this is a critical milestone, it represents only one part of a complete system. Detection results must still be transmitted, consumed, and acted upon by other components to deliver real value. This leads to the next stage of the architecture: understanding how data flows beyond the pipeline, how it integrates with messaging systems and APIs, and how it is combined with other data sources such as sensor streams. In the next article, the focus will shift to system integration, including metadata handling, MQTT-based communication, and multimodal fusion. The key takeaway is that you have now built the inference engine; the next step is to connect it to the broader system so that insights can drive decisions and actions.
