---
title: A Scalable Approach to Zero Shot and Few Shot Vision Learning at the Edge with Geti Instant Learn
excerpt: Many computer vision projects at the edge begin before enough data is available to train traditional models. In these early stages, collecting large datasets and building highly specialized models can be time-consuming, costly, and often impractical. This is especially true in real-world environments where conditions change often or problems need to be solved quickly. 
publishDate: April 24, 2026
tags:
  - Computer Vision
  - OpenEdge Platform
  - Intel
---

Many computer vision projects at the edge begin before enough data is available to train traditional models. In these early stages, collecting large datasets and building highly specialized models can be time-consuming, costly, and often impractical. This is especially true in real-world environments where conditions change often or problems need to be solved quickly.  For example, a manufacturing line may introduce a new product before defects are fully understood, or in healthcare, radiology and ultrasound teams may want to identify anomalies or quality issues in new imaging workflows without large, annotated datasets. 
Geti Instant Learn is built for these realities. It enables teams to develop and deploy zero-shot and few-shot vision models directly at the edge, using simple visual and vision or text-based prompts to configure model behavior. 

## The foundation: Geti Instant Learn Library

The Geti Instant Learn Library forms the base layer of the framework, and provides the core capabilities required to develop and deploy zero‑shot and few‑shot vision solutions.  It is a standalone Python library that runs without the full application stack or a GUI, making it easy to embed in custom services, batch or offline pipelines, or use from the command line via its CLI for scripting and automation.
The library includes ready-to-use zero-shot and few-shot vision based modular inference pipeline, and support for both visual and text‑based prompt configuration. It enables reference‑to‑target image matching through similarity scoring and configurable thresholds, while supporting multiple execution backends such as PyTorch and OpenVINO. 
At the core of these pipelines are foundation model backbones that have been pre-trained on large and diverse image and image-text datasets. They capture high-level semantic information rather than task-specific patterns, they are well suited for zero-shot and few-shot scenarios. 

## Zero‑shot and few‑shot vision at the edge

The core capability of Geti Instant Learn is its ability to run zero-shot and few-shot vision models efficiently on edge hardware. Zero-shot learning allows the system to recognize new objects or concepts it has never explicitly seen before, using semantic understanding encoded in the model, while few-shot learning enables adaptation using only a small number of reference examples.  Prompts specify what the model should look for, while the runtime itself remains an optimized and consistent vision pipeline that executes feature extraction, similarity matching, and segmentation.
Prompt-Based Configuration and Deterministic Execution
Prompts are best understood as configuration inputs to control the task performed by a deployed model. They define the object or concept of interest, specifying visual similarity that should be evaluated by the pipeline, and determine which regions of an image or frame are considered relevant for output. 

### Image Similarity Concept in Visual Prompting

A reference image defines the target visual appearance and includes a mask or bounding box to more precisely localize the object of interest. The few-shot detection flow in Geti Instant Learn relies on visual similarity derived from reference images and looks for those patterns in the target images. Matcher algorithm computes similarity scores across spatial locations by comparing these features and looking for regions exceeding a predefined similarity threshold. The final output is typically produced as segmentation masks or detections highlighting areas matching closely with the referenced visual concept.

### Few-Shot Learning Through Text-based Prompting

Text-based prompting allows vision models to be guided using natural language descriptions instead of example images. By describing what to look for in words, users can define new objects or concepts without providing annotated reference data. It uses SAM-3 (Segment Anything Model 3), designed for prompt-based concept segmentation in images and videos. In SAM-3, these descriptions are transformed into language embeddings that act as semantic references during inference. Rather than comparing image features to a reference image, the model compares them to the meaning encoded in the text prompt. Because the prompt directly conditions the model’s output, SAM-3 can generalize to new categories and tasks while maintaining a single, consistent inference pipeline.

## Developer-First Modular Vision Framework 

Geti Instant Learn provides a flexible and extensible vision framework built around modular pipelines, allowing developers to easily combine, replace, and experiment with different backbones, matchers, and segmentation components. The supported algorithms, including SAM3, Matcher, SoftMatcher, EfficientSAM3, PerDino, GroundedSAM, and DinoTXT, are selected for their strong generalization, promptability, and suitability for zero-shot and few-shot tasks across domains.
The framework offers unified dataset handling and evaluation APIs, making it simple to benchmark and compare approaches using a consistent interface. Multiple execution backends are supported, with PyTorch enabling rapid research and prototyping, and OpenVINO providing optimized, production-ready deployment for edge environments.

## Geti Instant Learn Application

The Geti Instant Learn Application serves as the operational layer of the framework and is built directly on top of the Geti Instant Learn Library to support real-world usage scenarios. Its primary role is to enable rapid prototyping, evaluation, and deployment of zero-shot and few-shot vision workflows without requiring custom integration or model development. 
It runs inference pipelines continuously, manages both live inputs such as cameras and batch inputs such as video files or image collections, and supports multiple projects running in parallel on the same system. The application also provides visualization and control interfaces that simplify prompt configuration, result inspection, and pipeline management. Vision inference itself is always executed by the library, while the application focuses on orchestration, data flow, and user interaction. 
The annotation workflow in the application is designed to be intuitive and low-effort. To annotate an object, users simply hover the mouse over the object in a reference image, and it is automatically highlighted for selection. This removes the need to manually draw polygons over objects, resulting in a faster and more user-friendly annotation experience.

## High‑level Execution pipeline
At a high level, the application is designed to process visual data as it flows through the system, from the moment it is captured to the moment results are produced. This stream-based approach makes it well suited for real-time and continuous use cases, such as cameras running on the factory floor or video feeds at the edge.
Under the hood, the application is built around a simple three-stage execution pipeline: Source → Processor → Sink. This structure keeps input handling, inference, and output delivery clearly separated, while allowing the system to run continuously and efficiently in production environments.
The source stage is responsible for acquiring input data such as images, video files, or live camera streams and feeding frames into the pipeline. Sources focus only on reliable data ingestion and do not apply any vision logic. User configuration enters the system here in the form of prompts, which may include text descriptions, reference images with optional masks, or bounding boxes defining the region of interest.
The processor stage integrates directly with the Geti Instant Learn Library and performs all vision inference. Prompts are applied as configuration to deployed zero-shot or few-shot models with fixed weights. For each frame, the processor executes a consistent inference pipeline that includes feature extraction, similarity matching, and segmentation or detection. No retraining or weight updates occur at runtime, and inference remains deterministic across runs.
The sink stage consumes the inference results and makes them available for downstream use. Outputs can include segmentation masks, bounding boxes, confidence scores, and similarity maps, and can be routed to different sinks for visualization, export, recording, or integration with other systems. For interactive use, sinks may stream low-latency visual output to a user interface, while other sinks support batch or offline workflows. 

## Get Started with Geti Instant Learn Application

### Prerequisites:
uv version ~= 0.10.x (Python package manager)  
node version v24.x   
npm version 11.x  
just (Command runner)

### Running the Application 

Run the application with a single command. 

```bash
just device=xpu application/dev 
```
Access the application through `https://localhost:3000`

## Closing Thoughts and Next Steps 

Geti Instant Learn is designed to help teams vision models at the edge to reduce complexity and save time. Model behavior is controlled through configuration, while inference remains deterministic, efficient, and suitable for production environments.
Looking ahead to the upcoming 2026.1 release, continued optimization for edge deployment remains a key focus. Planned enhancements include model quantization, reduced‑precision inference, and related improvements in memory usage, latency, and power efficiency. These efforts aim to extend support to a broader range of edge hardware while maintaining predictable performance.
