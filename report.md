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

We loaded the data and explored the classes, the frame and box sizes, the ignored regions, brightness, frame overlap and ran k-means on the box sizes to find typical anchor sizes. So the main conclusions we got:

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

### Models

#### Two-stage Anchor-based - Faster R-CNN

We used Faster R-CNN with a ResNet-50 backbone and a Feature Pyramid Network (FPN) from torchvision, pretrained on COCO and replaced its last layer with a new one for our 8 classes. It works in two stages, first the region proposal network suggests candidate boxes and then the second stage gives each box a class and corrects its position. We trained it with SGD, a learning rate of 0.01, warm-up and cosine decay. We tested three configurations and each of these experiments changed only one thing compared to the baseline:

- baseline: the default torchvision settings;
- more_proposals: twice as many candidate boxes were passed to the second stage, because our scenes are dense and some objects might never get a proposal;
- giou_loss: the GIoU loss instead of Smooth L1 for box positions, because for small boxes a shift of a few pixels changes the overlap a lot.

#### One-stage Anchor-based - RetinaNet

We used RetinaNet with the same backbone, pretrained weights and training settings as Faster R-CNN. RetinaNet has no proposal stage and classifies every anchor in the frame directly and it uses focal loss to deal with the huge number of background anchors. Since small objects and the many background anchors are the main difficulty for this model, our experiments focused on the anchors and the focal loss. We tested three configurations:

- baseline: the default anchors, which start at 32 px;
- data_anchors: we made the smallest anchors start at 13 px, based on k-means on our training boxes, because the default anchors covered only 49% of boxes under 32 px;
- focal_gamma3: focal loss with γ = 3 instead of 2, to give even less weight to easy background anchors.

Our first anchor attempt made the anchors smaller on every level, which helped small objects but hurt large ones, so in the final version we changed only the first level.

#### One-stage Anchor-free - YOLO11n

We used YOLO11n as a lightweight detector that predicts classes and boxes without a separate proposal stage. We tested three configurations:

- **baseline:** default training settings;
- **scale015:** reduced scale augmentation from 0.5 to 0.15 to preserve small objects;
- **box10:** increased box-loss weight to 10 to emphasize localization.

All three were evaluated using five-fold CV. We selected `scale015` based on OOF mAP and trained it in the full setup at 1280px for 8 epochs.

#### Query-based - RT-DETR-L

RT-DETR-L uses transformer object queries and set prediction instead of predefined anchor boxes. We tested whether this architecture could improve detection in dense drone scenes.

We first trained RT-DETR-L using the same reduced CV strategy as the other models: 960px images and 4 epochs. However, the baseline achieved only 0.0508 OOF mAP. We therefore tried longer training (8 epochs) and reduced the AdamW learning rate from 0.001 to 0.0002. The tuned version achieved 0.2251 OOF mAP.

We also fine-tuned the strongest checkpoint (Fold 4) on 10,999 instead of 2,221 training frames, keeping the same 1,019 validation images. This additional experiment used 4 epochs and learning rate 0.00005.

#### Our pipeline

Our final pipeline combines five tuned RT-DETR-L checkpoints using Weighted Boxes Fusion (WBF). Predictions were generated at 960px with confidence threshold 0.01. WBF merges overlapping same-class boxes using confidence-weighted coordinates.

We compared equal weights `[1,1,1,1,1]` with `[1,1,1,1,2]`, assigning additional weight to the strongest checkpoint, Fold 4. WBF used IoU threshold 0.55 and minimum score 0.02.

The weighted configuration achieved our best Public LB score of **0.28984**.

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

These experiments were trained with the reduced setup (described in validation design) and the metrics are out-of-fold over all 5 folds.

In cross-validation Faster R-CNN gave the best results and RetinaNet gave the worst. The tuning helped only a little and giou_loss, focal_gamma3 and box10 made the results even worse than the baseline.

**RT-DETR-L cross-validation results:**

| Experiment | OOF mAP@100 | AP50@500 | AP_small@500 |
|---|---:|---:|---:|
| Baseline (4 epochs) | 0.0508 | 0.1046 | 0.0208 |
| Tuned (8 epochs) | **0.2251** | **0.4304** | **0.0821** |

The tuned RT-DETR-L achieved substantially higher OOF mAP. However, the baseline and tuned experiments differed in both training budget and validation fold composition, so this comparison does not isolate the effect of hyperparameter tuning.

Final model results:

| Model | mAP | AP50 | AP_small | Public LB | Private LB |
|---|---|---|---|---|---|
| Faster R-CNN (more_proposals) | 0.2317 | 0.4577 | 0.1041 | 0.2391 | 0.2049 |
| RetinaNet (data_anchors) | 0.1823 | 0.3479 | 0.0676 | 0.1912 | 0.1545 |
| YOLO11n (scale015) | 0.2109 | 0.3943 | 0.1000 | 0.1643 | 0.1718 |

After full training Faster R-CNN reached a mAP of 0.2317 and AP_small of 0.1041, while RetinaNet reached only 0.1823 and 0.0676 and Faster R-CNN needed about 87 ms per frame on a T4 GPU. Most errors came from small and far objects and from classes that look alike from above, such as van and car, as we expected from the data analysis.

Faster R-CNN, RetinaNet, and YOLO11n were evaluated on Fold 0 after full training. The RT-DETR-L experiments used a different validation setup, so their validation metrics are presented separately.

**RT-DETR-L additional experiments and Kaggle results:**

| Model | Validation mAP@100 | Public LB |
|---|---:|---:|
| Tuned Fold 4 | 0.2864 | 0.2870 |
| More Data Fold 4 | 0.2585 | 0.2759 |
| Equal 5-Fold WBF | — | 0.2882 |
| **Weighted 5-Fold WBF** | — | **0.2898** |

The More Data experiment reduced validation mAP by 0.0279 compared with the original Fold 4 checkpoint, when evaluated using the same procedure. Increasing the number of training frames therefore did not improve this model.

The ensemble achieved higher public leaderboard scores than the individual Fold 4 submission. Weighted WBF performed better than equal weighting and became our final selected solution.

The validation mAP values refer to Fold 4, whereas the ensemble scores come from the Kaggle public leaderboard. They are not directly interchangeable.

## Final model and run instructions


Our selected solution is the five-model RT-DETR-L Weighted Boxes Fusion ensemble.

**Environment:**
- Python, PyTorch and NVIDIA Tesla T4 GPU
- `ultralytics==8.4.202`
- `ensemble-boxes==1.0.9`
- `pycocotools`, NumPy, pandas, OpenCV
- Random seed: 42

**Reproduction steps:**

1. Prepare the VisDrone dataset with eight object classes and video-level folds.
2. Run `notebooks/RT-DETR-L/rt-detr-l-tuned-5-fold-wbf-ensamble.ipynb` to train the five RT-DETR-L checkpoints.
3. Save the `best.pt` checkpoint from each fold.
4. Attach the competition test dataset and all five checkpoints to the WBF notebook.
5. Run the ensemble inference notebook to generate predictions for 3,811 test images.
6. Submit `submission.csv` to Kaggle.

The final submission uses weights `[1, 1, 1, 1, 2]`.

The output CSV contains `image_id` and `PredictionString`. Each prediction consists of `class confidence x y width height`, with bounding-box coordinates in original image pixels.

Datasets and trained weights are not stored in GitHub. The notebooks use Kaggle Inputs and external checkpoints, so input paths must be configured when reproducing the experiments in a different environment.


## Conclusion

Our data analysis showed that most objects are very small, and this was the main problem for all models. Among the models we compared under the same protocol, Faster R-CNN was the best and RetinaNet the weakest, and tuning gave only small gains. Our best result came from an ensemble of five RT-DETR-L models with Weighted Boxes Fusion, which scored 0.2898 on the public leaderboard. RT-DETR-L was trained and tuned with a different setup, so we couldn't compare its validation scores with the other models, but we could compare all of them on the leaderboard, where it was clearly the best.
