---
title: 'Real-Time Crowd Density Estimation Using CSRNet and OpenVINO'
excerpt: Crowd counting is a deceptively hard problem. The traditional approach of running a person detector, count bounding boxes, collapses almost immediately in real-world dense scenes. When people are packed tightly, occlusion causes detectors to miss 40–60% of individuals. The count becomes meaningless precisely when it matters most. This post walks through a pipeline I built for real-time crowd density estimation using CSRNet and OpenVINO. The system processes an overhead video feed, generates per-frame density maps, applies spatial zone analysis, performs temporal aggregation, and flags anomalies; all running locally with no cloud dependency.
publishDate: 'March 17 2026'
isFeatured: true
tags:
  - Computer Vision
  - OpenEdge Platform
  - Intel
---

Crowd counting is a deceptively hard problem. The traditional approach of running a person detector, count bounding boxes, collapses almost immediately in real-world dense scenes. When people are packed tightly, occlusion causes detectors to miss 40–60% of individuals. The count becomes meaningless precisely when it matters most.
This post walks through a pipeline I built for real-time crowd density estimation using CSRNet and OpenVINO. The system processes an overhead video feed, generates per-frame density maps, applies spatial zone analysis, performs temporal aggregation, and flags anomalies; all running locally with no cloud dependency.

The full code is at [github.com/arunimakanavu/crowd-density](https://github.com/arunimakanavu/crowd-density).

---

## The Problem with Just Counting People

The first instinct when building a crowd counter is to run a person detector and count bounding boxes. It works reasonably well when people are spread out. But in a genuinely dense crowd, bodies overlap, heads disappear behind shoulders, and the detector starts missing people at exactly the moment you need the most accurate count.

Instead of detecting individuals, CSRNet takes a different approach entirely. It looks at the image and produces a density map, a spatial heatmap where the value at each region represents how much crowd is concentrated there. To get the total count for an area, you simply sum the values in that region. The model never tries to find where each person is. It learns the overall distribution of people in the scene, which means occlusion hurts it much less.

This is the right tool for dense crowds, and it is the foundation the entire pipeline is built on.

---

## Why OpenVINO

Once you have a model you want to deploy, you need to actually run it efficiently. OpenVINO is Intel's inference framework and it handles this well, but there is a more specific reason it was chosen here.

Deploying a PyTorch model traditionally involves exporting it to an intermediate format called ONNX, then converting that to OpenVINO's own IR format. For CSRNet specifically, this path is fragile. The dilated convolutions in CSRNet's backend have a history of compatibility issues with ONNX export, and getting through that step without errors requires careful version management.

Newer versions of OpenVINO skip the ONNX step entirely. The conversion reads the PyTorch model directly and outputs the IR files in one Python script. No CLI switching, no ONNX compatibility debugging, no intermediate files to manage. You run the conversion once, and from that point on, the pipeline never touches PyTorch again.

At inference time, OpenVINO's device selection is worth highlighting. Switching between CPU, integrated GPU, and NPU requires no code changes at all. The same pipeline runs on a development laptop or an edge deployment box without modification.

---

## How the Pipeline Works

The system processes video frame by frame through seven sequential stages.

Each frame is first preprocessed: color conversion, resize, normalization using ImageNet statistics, and layout conversion to match what the model expects. This happens in a dedicated module and keeps the inference code clean.

The preprocessed frame goes into CSRNet running on OpenVINO. The model outputs a low-resolution density map, roughly one eighth the size of the input frame in each dimension. The global crowd count for the frame is computed by summing this map before it gets upsampled. This is an important detail: summing after upsampling inflates the count by a factor of 64, so the order of operations matters.

The upsampled density map is then passed through zone masking. Each zone is a polygon defined in pixel coordinates. A binary mask is created for each zone and applied to the density map, giving an independent density estimate per zone.

This per-zone output feeds into the temporal aggregation layer, which is where the system starts to reason rather than just measure. Each zone maintains a rolling history of density estimates. From this history, the system computes a smoothed density value, a rate of change between frames, and a long-term trend using linear regression. A single noisy frame cannot trigger an alert. The system looks at patterns over time.

Finally, the anomaly detection layer evaluates these signals against configurable thresholds and classifies each zone's state as NORMAL, WARNING, ALERT, or CRITICAL. Every alert event gets logged with a timestamp, zone ID, alert type, and density value.

---

## Three Kinds of Alerts

The anomaly layer produces three independent alert types, each targeting a different threat pattern.

A SUSTAINED alert fires when a zone's smoothed density stays above a threshold for a configured number of consecutive frames. This is the clearest signal: a zone has been packed beyond its safe capacity for long enough that it is not a transient spike.

A SPIKE or DROP alert fires when density changes faster than a crowd could realistically fill or empty a space under normal conditions. A sudden spike might indicate crowd surge or panic convergence. A sudden drop might indicate evacuation or scatter. These are the hardest alerts to tune well because you need to separate a real event from a camera artifact like a lighting change.

A TREND alert is the most proactive of the three. It fires when the long-term density trend in a zone is consistently increasing, even if the current value has not crossed the high-density threshold yet. This gives operators lead time to act before a situation becomes critical.

---

## Configuring Zones

Zones are defined as polygon coordinates in a JSON config file. The default setup divides a 1920x1080 frame into six equal zones in a 3x2 grid. But you can draw any polygon shape that matches the physical layout of your scene: entry corridors, open plazas, bottleneck passages, wherever you want independent density tracking.

The threshold values are calibrated against observed density output rather than real-world person counts. The right workflow is to run the pipeline on a representative clip first, observe the smoothed density values per zone in the terminal, and set your thresholds just above the normal operating range. For the video used in this project, that range was roughly 0.010 to 0.018 per pixel.

---

## What the Output Looks Like

The annotated output blends a jet colormap heatmap over the original video at around 40% opacity. Hot colours indicate high density regions. Cool colours indicate sparse areas. Zone boundaries are drawn as outlines and color-coded by their current alert state: green for NORMAL, yellow for WARNING, orange for ALERT, and red for CRITICAL.

Each zone displays its estimated count, its occupancy ratio relative to the configured capacity, and its current alert state. A HUD in the top corner shows the global count and video timestamp. When alerts fire, they appear in a panel at the bottom of the frame with the zone ID, alert type, and density value.

The annotated video is saved to disk alongside a JSON event log containing every alert event with full metadata.

---

## A Note on Model Calibration

The pretrained weights used here were trained on a sparse street-level dataset. Applied to a dense overhead crowd scene, the absolute count estimates come out inflated. If you look at the output and the numbers seem very large, this is why.

What remains accurate is the relative distribution across zones and the temporal behavior of the signals. The system correctly identifies which zones are denser than others, correctly tracks how density changes over time, and raises alerts at the right moments. For a proof-of-concept pipeline, this is sufficient. For production use with accurate absolute counts, the weights would need to be replaced with a model trained on dense overhead data, or fine-tuned on annotated frames from your specific scene.

---

Building this was a useful exercise in separating concerns across a multi-stage vision pipeline. The model regresses density. The temporal layer reasons about change. The anomaly layer interprets signals. None of these layers need to know about each other, which makes each one independently tunable and replaceable.

Code and setup instructions: [github.com/arunimakanavu/crowd-density](https://github.com/arunimakanavu/crowd-density)

