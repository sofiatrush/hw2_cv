---
title: Homework 2 -- Multiple Object Video Detection
author: Sofia Trush, Viktoria Khimiak
course: Artificial Intelligence · UCU
date: 2026-11-16
---

## Introduction

In this homework our task was to predict bounding boxes and classes of 8 object types for every object in each frame of drone video clips. We had 56 training and 7 validation videos with annotations and 48 unlabelled test clips.

## Methodology and motivation

### Data preparation

We loaded the data and explored the classes, the frame and box sizes, the ignored regions, brightness and frame overlap. So the main conclusions we got:

1. Most objects are very small. When frames are resized to 640 px, half of the objects are smaller than 16 px, at 1280 px about 10%
2. Objects at the bottom of the frame have bigger boxes than objects at the top, because the drone films at an angle
3. The mean brightness of frames ranges from 46 to 185, so there are both dark and very bright scenes
4. The classes are very unbalanced. Buses are under 1% of the objects and trucks about 3%
5. Neighbouring frames are very similar, so there is no need to use every frame.

Based on this, we decided to resize the images to 1280 px, take every 5th frame from each video, paint the ignored regions grey and mark them with iscrowd = 1. We also added these augmentations:

- only horizontal flip
- stronger brightness, contrast and gamma changes to cover the different lighting
- zoom-in with a random crop
- copy-paste of the rare classes, where trucks and buses from other frames are pasted into the image.

### Validation design

We combined the train and validation videos and split them into 5 folds, ensuring that all frames from one video are in the same fold. We did the validation in two steps. First, we compared different configurations of each model with 5-fold cross-validation using a reduced setup with 960 px images, every 10th frame, 4 epochs and computed one out-of-fold score from the predictions of all folds. Then we took the configuration of each model that gave the best result and trained it again with the full setup with 1280 px images, every 5th frame and 8 epochs, this time on folds 1-4 and evaluated it on fold 0.

## Method

### Models

#### Two-stage Anchor-based 

#### One-stage Anchor-based

#### One-stage Anchor-free - YOLO11n

We used YOLO11n as a lightweight one-stage anchor-free detector. Unlike two-stage models, YOLO predicts object classes and bounding boxes directly, without a separate region proposal stage.

We selected the nano version to establish a computationally efficient baseline and investigate how training configurations affect detection performance. Experiments were conducted using five-fold cross-validation with video-level separation to avoid leakage between training and validation frames.

We tested three configurations:

- Baseline: default training configuration.

- scale015: reduced scale augmentation from 0.5 to 0.15 to preserve the appearance and relative sizes of small objects.

- box10: increased the bounding-box loss weight to 10 to emphasize localization accuracy.

The cross-validation experiments used 960px images and four training epochs. The best configuration was then trained in the full setup at 1280px for eight epochs.

#### Query-based - RT-DETR-L

We used RT-DETR-L, a real-time transformer-based object detector that formulates detection as a set-prediction problem. Instead of relying on predefined anchor boxes, the model uses object queries to predict classes and bounding boxes.

We selected RT-DETR-L to investigate whether a query-based architecture could outperform the lightweight YOLO detector on challenging drone footage containing small and densely distributed objects.

Our tuned experiment used pretrained RT-DETR-L weights, the AdamW optimizer, an initial learning rate of 0.0002, an image size of 960px, and eight training epochs per fold.

We trained five models using video-level cross-validation, preserving the separation between training and validation sequences.

We also tested additional fine-tuning on a larger dataset. The training set increased from 2,221 to 10,999 frames, while the same 1,019 validation images were retained. This experiment used four additional epochs and a learning rate of 0.00005.

#### Our pipeline

## Results and analysis

Experiment results:

| Model | Experiment | OOF mAP | OOF AP50 | OOF AP_small |
|---|---|---|---|---|
| Faster R-CNN | baseline | 0.1782 | 0.3679 | 0.0481 |
| Faster R-CNN | more_proposals | **0.1811** | 0.3789 | 0.0513 |
| Faster R-CNN | giou_loss | 0.1700 | 0.3580 | 0.0443 |
| RetinaNet | baseline | 0.0813 | 0.1723 | 0.0154 |
| RetinaNet | data_anchors | **0.0827** | 0.1727 | 0.0161 |
| RetinaNet | focal_gamma3 | 0.0711 | 0.1513 | 0.0119 |
| YOLO11n | baseline | 0.1470 | 0.2897 | 0.0403 |
| YOLO11n | scale015 | **0.1484** | **0.2946** | 0.0445 |
| YOLO11n | box10 | 0.1428 | 0.2833 | **0.0455** |
| RT-DETR-L | tuned 5-fold CV | **0.2281** | — | — |

These experiments were trained with the reduced setup (described in validation design) and the metrics are out-of-fold over all 5 folds.

Final model results:

| Model | mAP | AP50 | AP_small | Public LB | Private LB |
|---|---|---|---|---|---|
| Faster R-CNN (more_proposals) | 0.2317 | 0.4577 | 0.1041 | 0.2391 | 0.2049 |
| RetinaNet (data_anchors) | 0.1823 | 0.3479 | 0.0676 | 0.1912 | 0.1545 |

The final models were trained with the full setup on folds 1-4 and evaluated on fold 0.

Images go into the `images/` folder:

<!-- ![Training loss](images/loss.png) -->

## Final model and run instructions

## Conclusion

> Summarize your findings here.
