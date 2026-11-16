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

### Validation design

## Method

### Models

#### Two-stage Anchor-based 

#### One-stage Anchor-based

#### One-stage Anchor-free

#### Query-based

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
