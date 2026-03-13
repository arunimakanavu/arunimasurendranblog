---
title: 'Casting Defect Detection with Autoencoder'
description: The Anomaly Detection Model is an autoencoder-based anomaly detection system fine-tuned for industrial casting defect inspection.
publishDate: 'October 24 2025'

tag:
- Computer Vision
- Edge AI
---

## Overview

The **Anomaly Detection Model** is an **autoencoder-based anomaly detection system** fine-tuned for industrial **casting defect inspection**. It identifies whether a metal casting image is *normal (OK)* or *defective* by reconstructing input images and analyzing reconstruction errors, achieving excellent performance with a final training loss of 0.0005 and a suggested anomaly threshold of 0.0004.

This model is designed for **Edge AI deployment**, optimized via **ONNX** and **OpenVINO IR** formats to run efficiently on low-power Intel edge devices.

## Repository Structure

```
anomalydetectionmodel/
├── casting_autoencoder.pth       # PyTorch model weights
├── casting_autoencoder.onnx      # ONNX export for deployment
├── inference.py                  # Inference script using OpenVINO
├── model.bin                     # Optional model binary
├── model.xml                     # OpenVINO IR model file
├── modelcard.yaml                # Model card metadata
├── requirements.txt              # Python dependencies
└── train_model.py                # Training script

```

## Model Architecture

The autoencoder consists of:

* **Encoder**: 3-layer convolutional encoder (3 → 16 → 32 → 64 channels)
* **Decoder**: 3-layer transposed convolutional decoder (64 → 32 → 16 → 3 channels)
* **Input**: 304×304 RGB images (converted from grayscale)
* **Output**: Reconstructed images with sigmoid activation

## Training Results

```
Epoch [6/10] - Loss: 0.0007
Epoch [7/10] - Loss: 0.0007
Epoch [8/10] - Loss: 0.0006
Epoch [9/10] - Loss: 0.0005
Epoch [10/10] - Loss: 0.0005
Model saved to casting_autoencoder.pth
Suggested anomaly threshold: 0.0004
```

## Dataset

The model was trained using the [Real-life Industrial Dataset of Casting Product](https://www.kaggle.com/datasets/ravirajsinh45/real-life-industrial-dataset-of-casting-product) from Kaggle.

### Dataset Structure

```
casting_data/
├── train/          # Normal casting images only
│   └── ok/
└── test/           # Defective casting images for threshold calibration
    └── def_front/
```

## Installation

### Requirements

```bash
pip install -r requirements.txt
```

### Dependencies

* openvino
* numpy
* opencv-python
* matplotlib

## Usage

### Training

```bash
python train_model.py
```

### Inference

```bash
python inference.py
```

#### Sample Inference Output

```
(venv) root:~aonmalydetectionmodel$ python3 inference.py
/aonmalydetectionmodel/venv/lib/python3.12/site-packages/openvino/runtime/__init__.py:10: DeprecationWarning: The `openvino.runtime` module is deprecated and will be removed in the 2026.0 release. Please replace `openvino.runtime` with `openvino`.
  warnings.warn(
[INFO] Reconstruction error: 0.000448
[INFO] Reconstruction saved to reconstruction.png
Defective Casting Detected 
```

## Model Outputs

* `casting_autoencoder.pth` - PyTorch model weights
* `casting_autoencoder.onnx` - ONNX export for deployment
* `model.xml` and `model.bin` - OpenVINO IR for inference
* `reconstruction.png` - Generated during inference
* Calibrated anomaly threshold based on defective samples

## Anomaly Detection Process

1. **Training Phase**: Model learns to reconstruct normal casting images
2. **Threshold Calibration**: Uses defective samples to determine optimal threshold
3. **Inference**: Images with reconstruction error > threshold are flagged as defective

## Performance

* **Final Training Loss**: 0.0005
* **Suggested Threshold**: 0.0004
* **Model Type**: Unsupervised anomaly detection
* **Architecture**: Convolutional Autoencoder

## Applications

* Quality control in metal casting manufacturing
* Real-time defect detection on production lines
* Automated visual inspection systems
* Edge deployment in industrial environments

## Model Features

* **Unsupervised Learning**: Trained only on normal samples
* **Real-time Capable**: Optimized for edge deployment
* **ONNX Compatible**: Ready for production deployment
* **Automatic Thresholding**: Self-calibrating anomaly detection
* **Industrial Grade**: Tested on real manufacturing data

## Technical Details

* Symmetric encoder-decoder architecture
* Stride-2 convolutions for downsampling
* Transposed convolutions for upsampling
* ReLU activation in hidden layers
* Sigmoid output activation for pixel reconstruction

## Future Enhancements

* Multi-scale feature extraction
* Attention mechanisms for defect localization
* Integration with production line systems
* Real-time inference optimization

## Citation

* Original PCB anomaly detection framework from Intel Edge AI Suite
* Casting product dataset from Kaggle (ravirajsinh45)

## License

This project is released under the MIT License.

