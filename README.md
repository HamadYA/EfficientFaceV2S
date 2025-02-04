
# EfficientFaceV2S

## Introduction
- This repository contains the official implementation of EfficientFaceV2S face recognition models.

![EfficientFaceV2S](/images/img.png)

## Results
  - `IJBB` and `IJBC` are scored at `TAR@FAR=1e-4`
  - **Download this version of the LFW bechmark [lfw.bin](https://github.com/leondgarse/Keras_insightface/releases/download/v1.0.0/lfw.bin), follow the Datasets and Data Preparation, download the weights, and then modify and run evaluation.ipynb. Note: Set flip=True inside the code.**
    
  - **EfficientFaceV2S**

  | Model | Training Dataset | lfw (%)      | cfp_fp (%)   | agedb_30 (%) | IJBB     | IJBC     |
  | -------------- | ----- | -------- | -------- | -------- | -------- | -------- |
  | [EfficientFaceV2S](https://github.com/HamadYA/EfficientFaceV2S/releases/download/v1/EfficientFaceV2S.h5) | MS1MV3 | 99.85 | 99.37 | 98.50 | 96.12 | 97.42 |
  
***

## Installation
To run EfficientFaceV2S, please install the following packages:

    1. Python 3.9.12 64-bit
    2. TensorFlow 2.8.0 or above (CUDA compatible GPU needed for GPU training)
    3. Keras 2.8.0 or above
    4. keras_cv_attention_models
    5. glob2
    6. pandas
    7. tqdm
    8. sklearn
    9. scikit-image

- To install [keras_cv_attention_models](https://github.com/leondgarse/keras_cv_attention_models), please run pip install keras_cv_attention_models.
- Both Linux and Windows OS are supported.

- **All results in the paper are generated using `Ubuntu 20.04.05 LTS`, `Miniconda`, `Tensorflow 2.8.0` with `cuda==11.2` `cudnn==8.1`**.
- Please follow [Step-by-step instructions Tensorflow installation guide](https://www.tensorflow.org/install/pip)

## Datasets and Data Preparation
- The training and testing datasets (`LFW` `CFP-FP` `AgeDB-30` included inside the training datasets) can be downloaded from the following URLs:
    1. [MS1M-ArcFace - MS1MV2](https://github.com/deepinsight/insightface/tree/master/recognition/_datasets_) 
    2. [MS1M-RetinaFace - MS1MV3](https://github.com/deepinsight/insightface/tree/master/recognition/_datasets_)

- Download and extract the training dataset in the datasets directory.
- The extracted folder contains .bin files of the training and testing datasets.
- To convert .bin files of the training and testing datasets, please run the following commands:
 - **[prepare_data.py](prepare_data.py)** script, Extract data from mxnet record format to `folders`.
    ```sh
    # Convert `datasets/faces_emore` to `datasets/faces_emore_112x112_folders`
    python prepare_data.py -D datasets/faces_emore
    
    # Convert evaluating bin files
    python prepare_data.py -D datasets/faces_emore -T lfw.bin cfp_fp.bin agedb_30.bin
    ```

  - **Training dataset Required** is a `folder` including `person folders`, each `person folder` including multi `face images`. Format like
    ```sh
    .               # dataset folder
    ├── 0           # person folder
    │   ├── 100.jpg # face image
    │   ├── 101.jpg # face image
    │   └── 102.jpg # face image
    ├── 1           # person folder
    │   ├── 111.jpg
    │   ├── 112.jpg
    │   └── 113.jpg
    ├── 10
    │   ├── 707.jpg
    │   ├── 708.jpg
    │   └── 709.jpg
    ```
  - **Evaluating bin files** include jpeg image data pairs, and a label indicating if it's a same person, so there are double images than labels
    ```sh
    #    bins   | issame_list
    img_1 img_2 | True
    img_3 img_4 | True
    img_5 img_6 | False
    img_7 img_8 | False
    ```
    Image data in bin files like `CFP-FP` `AgeDB-30` is not compatible with `tf.image.decode_jpeg`, we need to reformat it, which is done by `-T` parameter.
    ```py
    ''' Throw error if not reformated yet '''
    ValueError: Can't convert non-rectangular Python sequence to Tensor.
    ```

## Project Structure
- **Training Modules**
    - [train.py](train.py) contains a `Train` class. It uses a `scheduler` to connect different `loss` / `optimizer` / `epochs`. The basic function is simply `basic_model` --> `build dataset` --> `add output layer` --> `add callbacks` --> `compile` --> `fit`.
    - [data.py](data.py) loads image data as `tf.dataset` for training. `Triplet` dataset is different from others.
    - [evals.py](evals.py) contains evaluating callback using `bin` files.
    - [losses.py](losses.py) contains `arcface` / `cosface` loss functions.
    - [myCallbacks.py](myCallbacks.py) contains my other callbacks, like saving model / learning rate adjusting / save history.
    - [IJB_evals.py](IJB_evals.py) evaluates model accuracy using [insightface/evaluation/IJB/](https://github.com/deepinsight/insightface/tree/master/evaluation/IJB) datasets.
    - [eval_folder.py](eval_folder.py) Run model evaluation on any custom dataset folder, which is in the same format with Training dataset.
    - [augment.py](augment.py) including implementation of `RandAug` and `AutoAug`.
- **Other Modules**
    - [data_drop_top_k.py](data_drop_top_k.py) create dataset after trained with [Sub Center ArcFace](#sub-center-arcface) method.
    - [plot_parameters.py](plot_parameters.py) Used to plot Figure 8 in the paper.

## Basic Training
  - We included all training scripts in the paper in [training_scripts.ipynb](training_scripts.ipynb)
  - **Training example** `train.Train` is mostly functioned as a scheduler.
    ```py
    import os
    from tensorflow import keras
    import losses, train, models
    import tensorflow as tf
    import keras_cv_attention_models
    from keras_cv_attention_models import efficientnet
    import tensorflow_addons as tfa

    data_path = 'datasets/faces_emore_112x112_folders'
    eval_paths = ['datasets/faces_emore/lfw.bin', 'datasets/faces_emore/cfp_fp.bin', 'datasets/faces_emore/agedb_30.bin']

    basic_model = efficientnet.EfficientNetV2S(input_shape=(112, 112, 3), num_classes=0, drop_connect_rate=0.2, pretrained="imagenet", first_strides=1)
basic_model = models.buildin_models(basic_model, dropout=0.2, emb_shape=512, output_layer='F', bn_momentum=0.9, bn_epsilon=1e-5, add_pointwise_conv=True, pointwise_conv_act="swish", scale=True, use_bias=False)

    tt = train.Train(data_path, eval_paths=eval_paths,
    save_path='EfficientFaceV2S.h5',
    basic_model=basic_model, model=None, lr_base=0.01, lr_decay=0.5, lr_decay_steps=50, lr_min=1e-6, lr_warmup_steps=3,
    batch_size=512, random_status=100, eval_freq=1, output_weight_decay=1)

optimizer = tfa.optimizers.AdamW(learning_rate=1e-2, weight_decay=5e-4)    sch = [
    {"loss": losses.AdaFaceLoss(scale=16), "epoch": 4, "optimizer": optimizer},
    {"loss": losses.AdaFaceLoss(scale=32), "epoch": 3},
    {"loss": losses.AdaFaceLoss(scale=64), "epoch": 46},
]
tt.train(sch, 0)
    ```
    May use `tt.train_single_scheduler` controlling the behavior more detail.
  - **Model** basically containing two parts:
    - **Basic model** is layers from `input` to `embedding`.
    - **Model** is `Basic model` + `bottleneck` layer, like `softmax` / `arcface` layer. For triplet training, `Model` == `Basic model`. For combined `loss` training, it may have multiple outputs.
  - **Saving strategy**
    - **Model** will save the latest one on every epoch end to local path `./checkpoints`, name is specified by `train.Train` `save_path`.
    - **basic_model** will be saved monitoring on the last `eval_paths` evaluating `bin` item, and save the best only.
  - **train.Train model parameters** including `basic_model` / `model`. Combine them to initialize model from different sources. Sometimes may need `custom_objects` to load model.
    | basic_model                                                     | model           | Used for                                    |
    | --------------------------------------------------------------- | --------------- | ------------------------------------------- |
    | model structure                                                 | None            | Scratch train                               |
    | basic model .h5 file                                            | None            | Continue training from a saved basic model  |
    | None for 'embedding' layer or layer index of basic model output | model .h5 file  | Continue training from last saved model     |
    | None for 'embedding' layer or layer index of basic model output | model structure | Continue training from a modified model     |
    | None                                                            | None            | Reload model from "checkpoints/{save_path}" |

  - **Scheduler** is a list of dicts, each containing a training plan. For detials please refer to [keras_insightface](https://github.com/leondgarse/Keras_insightface)
    
  - **Restore training from break point**
    ```py
    from tensorflow import keras
    import losses, train
    data_path = 'datasets/faces_emore_112x112_folders'
    eval_paths = ['datasets/faces_emore/lfw.bin', 'datasets/faces_emore/cfp_fp.bin', 'datasets/faces_emore/agedb_30.bin']
    tt = train.Train(data_path, 'EfficientFaceV2S.h5', eval_paths, model='./checkpoints/EfficientFaceV2S.h5',
                    batch_size=128, lr_base=0.01, lr_decay=0.5, lr_decay_steps=50, lr_min=1e-6, lr_warmup_steps=3,
    batch_size=512, random_status=100, eval_freq=1, output_weight_decay=1)

    sch = [
      # {"loss": losses.AdaFaceLoss(scale=16), "epoch": 6, "optimizer": optimizer},
        {"loss": losses.AdaFaceLoss(scale=32), "epoch": 8},
    {"loss": losses.AdaFaceLoss(scale=64), "epoch": 46},
    ]
    tt.train(sch, initial_epoch=15)
    ```
  - We included all **evaluation** scripts in [evaluation.ipynb](evaluation.ipynb)
  - **Evaluation** example
    ```py
    import evals
    basic_model = keras.models.load_model('checkpoints/EfficientFaceV2S.h5', compile=False)
    ee = evals.eval_callback(basic_model, 'datasets/faces_emore/lfw.bin', batch_size=256, PCA_acc=True)
    ee.on_epoch_end(0)
    ```
    For training process, default evaluating strategy is `on_epoch_end`. Setting an `eval_freq` greater than `1` in `train.Train` will also **add** an `on_batch_end` evaluation.
    ```py
    # Change evaluating strategy to `on_epoch_end`, as long as `on_batch_end` for every `1000` batch.
    tt = train.Train(data_path, 'EfficientFaceV2S.h5', eval_paths, basic_model=basic_model, eval_freq=1000)
    ```
## Other Basic Functions and Parameters
  - **train.Train output_weight_decay** controls `L2 regularizer` value added to `output_layer`.
    - `0` for None.
    - `(0, 1)` for specific value, actual added value will also divided by `2`.
    - `>= 1` will be value multiplied by `L2 regularizer` value in `basic_model` if added.
  - **train.Train random_status** controls data augmentation weights.
    - `-1` will disable all augmentation.
    - `0` will apply `random_flip_left_right` only.
    - `1` will also apply `random_brightness`.
    - `2` will also apply `random_contrast` and `random_saturation`.
    - `3` will also apply `random_crop`.
    - `>= 100` will apply `RandAugment` with `magnitude = 5 * random_status / 100`, so `random_status=100` means using `RandAugment` with `magnitude=5`.
  - **train.Train random_cutout_mask_area** set ratio of randomly cutout image bottom `2/5` area, regarding as ignoring mask area.
  - **train.Train partial_fc_split** set a int number like `2` / `4`, will build model and dataset with total classes split in `partial_fc_split` parts. Works also on a single GPU. Supports `ArcFace` loss family like `ArcFace` / `AirFaceLoss` / `CosFaceLoss` / `MagFaceLoss`.
  - **models.buildin_models** is mainly for adding output feature layer `GDC` to a backbone model. The first parameter `stem_model` can be:
    - String can be printed by `models.print_buildin_models()`.
    - **models.add_l2_regularizer_2_model** will add `l2_regularizer` to `dense` / `convolution` layers, or set `apply_to_batch_normal=True` also to `PReLU` / `BatchNormalization` layers. The actual added `l2` value is divided by `2`.
    ```py
    # Will add keras.regularizers.L2(5e-4) to `dense` / `convolution` layers.
    basic_model = models.add_l2_regularizer_2_model(basic_model, 1e-3, apply_to_batch_normal=False)
    ```
  - **Gently stop** is a callback to stop training gently. Input an `n` and `<Enter>` anytime during training, will set training stop on that epoch ends.
  - **My history**
    - This is a callback collecting training `loss`, `accuracy` and `evaluating accuracy`.
    - On every epoch end, backup to the path `save_path` defined in `train.Train` with suffix `_hist.json`.
    - Reload when initializing, if the backup `<save_path>_hist.json` file exists.
    - The saved `_hist.json` can be used for plotting using `plot.py`.
  - **eval_folder.py** is used for test evaluating accuracy on custom test dataset:
    ```sh
    python ./eval_folder.py -d {DATA_PATH} -m {BASIC_MODEL.h5}
    ```
    Or create own test bin file which can be used in `train.Train` `eval_paths`:
    ```sh
    python ./eval_folder.py -d {DATA_PATH} -m {BASIC_MODEL.h5} -B {BIN_FILE.bin}
    ```
## Learning rate
  - `train.Train` parameters `lr_base` / `lr_decay` / `lr_decay_steps` / `lr_warmup_steps` set different decay strategies and their parameters.
  - `tt.lr_scheduler` can also be used to set learning rate scheduler directly.
    ```py
    tt = train.Train(...)
    import myCallbacks
    tt.lr_scheduler = myCallbacks.CosineLrSchedulerEpoch(lr_base=1e-3, first_restart_step=16, warmup_steps=3)
    ```
  - **lr_decay_steps** controls different decay types.
    - Default is `Exponential decay` with `lr_base=0.001, lr_decay=0.05`.
    - For `CosineLrScheduler`, `steps_per_epoch` is set after dataset been inited.
    - For `CosineLrScheduler`, default value of `cooldown_steps=1`, means will train `1 epoch` using `lr_min` before each restart.

    | lr_decay_steps | decay type                                       | mean of lr_decay_steps    | mean of lr_decay |
    | -------------- | ------------------------------------------------ | ------------------------- | ---------------- |
    | <= 1           | Exponential decay                                |                           | decay_rate       |
    | > 1            | Cosine decay, will multiply with steps_per_epoch | first_restart_step, epoch | m_mul            |
    | list           | Constant decay                                   | lr_decay_steps            | decay_rate       |

    ```py
    # lr_decay_steps == 0, Exponential
    tt = train.Train(..., lr_base=0.001, lr_decay=0.05, ...)
    # 1 < lr_decay_steps, Cosine decay, first_restart_step = lr_decay_steps * steps_per_epoch
    # restart on epoch [16 * 1 + 1, 16 * 3 + 2, 16 * 7 + 3] == [17, 50, 115]
    tt = train.Train(..., lr_base=0.001, lr_decay=0.5, lr_decay_steps=16, lr_min=1e-7, ...)
    # 1 < lr_decay_steps, lr_min == lr_base * lr_decay, Cosine decay, no restart
    tt = train.Train(..., lr_base=0.001, lr_decay=1e-4, lr_decay_steps=24, lr_min=1e-7, ...)
    # lr_decay_steps is a list, Constant
    tt = train.Train(..., lr_base=0.1, lr_decay=0.1, lr_decay_steps=[3, 5, 7, 16, 20, 24], ...)
    ```
## Mixed precision float16
  - [Tensorflow Guide - Mixed precision](https://www.tensorflow.org/guide/mixed_precision)
  - Enable `Mixed precision` at the beginning of all functional code by
    ```py
    keras.mixed_precision.set_global_policy("mixed_float16")
    ```
  - In most training case, it will have a `~2x` speedup and less GPU memory consumption.

## Optimizers
  - **SGDW / AdamW** [tensorflow_addons AdamW](https://www.tensorflow.org/addons/api_docs/python/tfa/optimizers/AdamW).
    ```py
    # !pip install tensorflow-addons
    !pip install tfa-nightly

    import tensorflow_addons as tfa
    optimizer = tfa.optimizers.SGDW(learning_rate=0.1, weight_decay=5e-4, momentum=0.9)
    optimizer = tfa.optimizers.AdamW(learning_rate=0.001, weight_decay=5e-5)
    ```
    `weight_decay` and `learning_rate` should share the same decay strategy. A callback `OptimizerWeightDecay` will set `weight_decay` according to `learning_rate`.
    ```py
    opt = tfa.optimizers.AdamW(weight_decay=5e-5)
    sch = [{"loss": keras.losses.CategoricalCrossentropy(label_smoothing=0.1), "centerloss": True, "epoch": 60, "optimizer": opt}]
    ```
    - **RAdam / Lookahead / Ranger optimizer** [tensorflow_addons RectifiedAdam](https://www.tensorflow.org/addons/api_docs/python/tfa/optimizers/RectifiedAdam).
    ```py
    # Rectified Adam,a.k.a. RAdam, [ON THE VARIANCE OF THE ADAPTIVE LEARNING RATE AND BEYOND](https://arxiv.org/pdf/1908.03265.pdf)
    optimizer = tfa.optimizers.RectifiedAdam()
    # SGD with Lookahead [Lookahead Optimizer: k steps forward, 1 step back](https://arxiv.org/pdf/1907.08610.pdf)
    optmizer = tfa.optimizers.Lookahead(keras.optimizers.SGD(0.1))
    # Ranger [Gradient Centralization: A New Optimization Technique for Deep Neural Networks](https://arxiv.org/pdf/2004.01461.pdf)
    optmizer = tfa.optimizers.Lookahead(tfa.optimizers.RectifiedAdam())
    ```

# Subcenter ArcFace
  - [Original MXNet Subcenter ArcFace](https://github.com/deepinsight/insightface/tree/master/recognition/subcenter_arcface)
  - [PDF Sub-center ArcFace: Boosting Face Recognition by Large-scale Noisy Web Faces](https://ibug.doc.ic.ac.uk/media/uploads/documents/eccv_1445.pdf)
  - Please refer to [keras_insightface](https://github.com/leondgarse/Keras_insightface) for usage.
  
# Evaluating on IJB datasets
  - [IJB_evals.py](IJB_evals.py) evaluates model accuracy using [insightface/evaluation/IJB/](https://github.com/deepinsight/insightface/tree/master/recognition/_evaluation_/ijb) datasets.
  - If any error occured from using the below scripts, kindly refer to [evaluation.ipynb](evaluation.ipynb). **Note: We can run the models on Google Colaboratory due to dependency issues**.
  - In case placing `IJB` dataset `IJB/IJB_release`, basic usage will be:
    ```sh
    # Test mxnet model, default scenario N0D1F1
    python IJB_evals.py -m 'checkpoints/EfficientFaceV2S.h5' -d IJB/IJB_release -L

    # Test keras h5 model, default scenario N0D1F1
    python IJB_evals.py -m 'checkpoints/EfficientFaceV2S.h5' -d IJB/IJB_release -L

    # `-B` to run all 8 tests N{0,1}D{0,1}F{0,1}
    python IJB_evals.py -m 'checkpoints/EfficientFaceV2S.h5' -d IJB/IJB_release -B -L

    # `-N` to run 1N test
    python IJB_evals.py -m 'checkpoints/EfficientFaceV2S.h5' -d IJB/IJB_release -N -L

    # `-E` to save embeddings data
    python IJB_evals.py -m 'checkpoints/EfficientFaceV2S.h5' -d IJB/IJB_release -E
    # Then can be restored for other tests, add `-E` to save again
    python IJB_evals.py -R IJB_result/checkpoints/EfficientFaceV2S_IJBB.npz -d IJB/IJB_release -B

    # Plot result only, this needs the `label` data, which can be saved using `-L` parameter.
    # Or should provide the label txt file.
    python IJB_evals.py --plot_only IJB/IJB_release/IJBB/result/*100*.npy IJB/IJB_release/IJBB/meta/ijbb_template_pair_label.txt
    ```
  - See `-h` for detail usage.
    ```sh
    python IJB_evals.py -h
    ```
***

# Evaluating on MegaFace datasets
  - Download megaface testsuite from [baiducloud](https://pan.baidu.com/s/1Vdxc2GgbY8wIW0hVcObIwg)(code:0n6w) or [gdrive](https://drive.google.com/file/d/1KBwp0U9oZgZj7SYDXRxUnnH7Lwvd9XMy/view?usp=sharing). The official devkit is also included.
  - Prepare the environment. OpenCV 2.4 is required by the official devkit, for convenience, you can download it from [BaiduCloud](https://pan.baidu.com/s/1By4yIds0hEnw6_Ihh75R5w) or [GoogleDrive](https://drive.google.com/open?id=1Ifjj6zJQaXzuggr0tVcaMe21F7hd1PZk) and unzip to ``/usr/local/lib/opencv2.4``.
  - Edit and call ``run.sh`` to evaluate your face recognition model performance.

**If errors**
  - Run gen_megaface.ipynb or gen_megaface.py **After making appropriate modifications if needed, i.e., provide model path; or move the EfficientFaceV2S directory to be a part of the MegaFace testsuite**
  - Run run_remove_noises.sh
  - Run run_megaface.sh
  - Run run_megaface_refined.sh
***

# Acknowledgement
  This project includes code and ideas from the following sources:
  
  - [keras_insightface](https://github.com/leondgarse/Keras_insightface)
  - [keras_cv_attention_models](https://github.com/leondgarse/keras_cv_attention_models)
  
  We would like to thank [leondgarse](https://github.com/leondgarse) for their valuable contributions to the field and for sharing their work with the community. Their ideas and code have been instrumental in the development of this project and we are grateful for the opportunity to build upon their work.
  ***

## Citation
If you use EfficientFaceV2S (or any part of this code in your research), please cite the following:

```
Alansari, Mohamad and Alnuaimi, Khaled and Ganapathi, Dr. Iyyakutti Iyappan and Alansari, Sara and Javed, Sajid and Shoufan, Abdulhadi and Zweiri, Yahya and Werghi, Naoufel, Efficientfacev2s: A Lightweight Model and a Benchmarking Approach for Drone-Captured Face Recognition. Available at SSRN: https://ssrn.com/abstract=4698437 or http://dx.doi.org/10.2139/ssrn.4698437

```
and
  - **BibTeX**
    ```bibtex
    @misc{leondgarse,
      author = {Leondgarse},
      title = {Keras Insightface},
      year = {2022},
      publisher = {GitHub},
      journal = {GitHub repository},
      doi = {10.5281/zenodo.6506949},
      howpublished = {\url{https://github.com/leondgarse/Keras_insightface}}
    }
    ```
  - **Latest DOI**: [![DOI](https://zenodo.org/badge/229437028.svg)](https://zenodo.org/badge/latestdoi/229437028)
***

  - **BibTeX**
    ```bibtex
    @misc{leondgarse,
      author = {Leondgarse},
      title = {Keras CV Attention Models},
      year = {2022},
      publisher = {GitHub},
      journal = {GitHub repository},
      doi = {10.5281/zenodo.6506947},
      howpublished = {\url{https://github.com/leondgarse/keras_cv_attention_models}}
    }
    ```
  - **Latest DOI**: [![DOI](https://zenodo.org/badge/391777965.svg)](https://zenodo.org/badge/latestdoi/391777965)
***
