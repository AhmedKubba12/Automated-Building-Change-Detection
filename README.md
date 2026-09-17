# Dubai Building Footprint Change Detection

Deep learning model for detecting changes in building footprints across Dubai using multi-temporal satellite imagery. The model compares imagery from 2019 and 2023 and produces per-pixel change masks that are then georeferenced for use in GIS software.

This repository holds the code, notebooks, and prediction outputs for the project. The trained model weights and the full training dataset are large binary files and are not tracked here (see [Large files](#large-files) below).

## Overview

The goal is to flag where new buildings appeared or existing footprints changed between two points in time. The workflow has four stages:

1. Fine-tune a change detection network on paired Dubai satellite tiles from 2019 and 2023.
2. Search for the decision threshold that gives the best trade-off between precision and recall on a held-out set.
3. Run inference across all tiles with test-time augmentation.
4. Georeference the resulting masks with rasterio and export them as a shapefile that opens directly in ArcGIS.

## Model

The network is an encoder-decoder built on a VGG16 backbone, with channel and spatial attention blocks and a coarse-to-fine decoding path. It was first pretrained on the LEVIR-CD benchmark in a fully supervised setting, then transferred to the Dubai data.

Fine-tuning uses a semi-supervised, mean-teacher setup. A student model is trained on the labelled tiles while an exponential moving average teacher provides consistency targets on the unlabelled tiles. A few details worth calling out:

- The consistency weight ramps up over the first 20 epochs so that noisy early predictions do not destabilise training.
- A burn-in phase lets the student settle before the teacher starts contributing.
- Only part of the VGG16 backbone is unfrozen during transfer, at a lower learning rate than the decoder.

Training settings used for the released checkpoint: learning rate 5e-5, batch size 8, up to 100 epochs, and a labelled train ratio of 0.2.

## Repository structure

```
.
├── Model_Accuracy_Assessment.ipynb        Evaluation on the test set: metrics, confusion matrix, sample visuals
├── Model Predictions Shapefile/           Georeferenced change polygons (ModelPred.shp and companions)
├── Source Code/
│   ├── Dubai Change Detection Model Training.ipynb   Full pipeline: data loader, model, training, inference, export
│   └── Model_Predictions_Tiles/           Predicted change masks as PNG tiles
└── Automated detection of changes in building footprints FINAL 1 (1).pptx   Project presentation
```

## Notebooks

Both notebooks were written to run in Google Colab with the data mounted from Google Drive. If you run them elsewhere, replace the Drive mount and path cells with paths that match your setup.

`Source Code/Dubai Change Detection Model Training.ipynb` covers the whole pipeline end to end: evaluation metrics, the data loader, the network definition, the training loop, threshold selection, inference with test-time augmentation, and the step that georeferences predictions for ArcGIS.

`Model_Accuracy_Assessment.ipynb` loads a trained checkpoint and reports results on the test set, including a confusion matrix and side-by-side sample predictions.

## Requirements

The notebooks install their own dependencies in the first few cells. The main ones are:

- Python 3.9 or later
- PyTorch 2.0 or later and torchvision 0.15 or later
- opencv-python-headless, tensorboardX, numpy (pinned below 2.0 for inference), and protobuf 3.20.x
- rasterio for georeferencing, and a shapefile-capable library for the vector export

A CUDA-capable GPU is recommended for training. Inference runs on CPU but is slower.

## Data

Training data is organised as paired 2019 and 2023 tiles, split into labelled train, validation, and test sets, with a separate pool of unlabelled tiles used by the semi-supervised branch. The labels are binary change masks.

The dataset tiles are not included in this repository because of their size. Point the data path cells in the training notebook at your local copy of the tiles.

## Outputs

Inference produces one change mask per tile. These are saved as PNG tiles in `Source Code/Model_Predictions_Tiles/`, and the merged, georeferenced result is written to `Model Predictions Shapefile/` as a shapefile. Open `ModelPred.shp` in ArcGIS or QGIS to view the detected changes over a basemap.

## Large files

To keep the repository a reasonable size and stay within GitHub's per-file limit, two categories of files are excluded from version control:

- Model weights (`.pth`). The fine-tuned Dubai checkpoint and the pretrained LEVIR-CD checkpoint are each around 200 MB, which is above GitHub's 100 MB limit for a normal repository. Keep them alongside the notebooks locally, or host them separately and link to them.
- The satellite dataset tiles used for training and validation.

These paths are listed in `.gitignore`. If you need to share the weights or data through GitHub, Git LFS is the usual way to do it.

## Acknowledgements

Built for the MBRSC project on automated detection of building footprint change. The change detection architecture follows the coarse-to-fine and attention-based design used in recent remote sensing literature, with the LEVIR-CD dataset used for pretraining.
