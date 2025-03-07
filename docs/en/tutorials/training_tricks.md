# Tutorial 5: Training Tricks

MMSegmentation support following training tricks out of box.

## Different Learning Rate(LR) for Backbone and Heads

In semantic segmentation, some methods make the LR of heads larger than backbone to achieve better performance or faster convergence.

In MMSegmentation, you may add following lines to config to make the LR of heads 10 times of backbone.

```python
optimizer=dict(
    paramwise_cfg = dict(
        custom_keys={
            'head': dict(lr_mult=10.)}))
```

With this modification, the LR of any parameter group with `'head'` in name will be multiplied by 10.
You may refer to [MMCV doc](https://mmcv.readthedocs.io/en/latest/api.html#mmcv.runner.DefaultOptimizerConstructor) for further details.

## Online Hard Example Mining (OHEM)

We implement pixel sampler [here](https://github.com/open-mmlab/mmsegmentation/tree/master/mmseg/core/seg/sampler) for training sampling.
Here is an example config of training PSPNet with OHEM enabled.

```python
_base_ = './pspnet_r50-d8_512x1024_40k_cityscapes.py'
model=dict(
    decode_head=dict(
        sampler=dict(type='OHEMPixelSampler', thresh=0.7, min_kept=100000)) )
```

In this way, only pixels with confidence score under 0.7 are used to train. And we keep at least 100000 pixels during training. If `thresh` is not specified, pixels of top `min_kept` loss will be selected.

## Class Balanced Loss

For dataset that is not balanced in classes distribution, you may change the loss weight of each class.
Here is an example for cityscapes dataset.

```python
_base_ = './pspnet_r50-d8_512x1024_40k_cityscapes.py'
model=dict(
    decode_head=dict(
        loss_decode=dict(
            type='CrossEntropyLoss', use_sigmoid=False, loss_weight=1.0,
            # DeepLab used this class weight for cityscapes
            class_weight=[0.8373, 0.9180, 0.8660, 1.0345, 1.0166, 0.9969, 0.9754,
                        1.0489, 0.8786, 1.0023, 0.9539, 0.9843, 1.1116, 0.9037,
                        1.0865, 1.0955, 1.0865, 1.1529, 1.0507])))
```

`class_weight` will be passed into `CrossEntropyLoss` as `weight` argument. Please refer to [PyTorch Doc](https://pytorch.org/docs/stable/nn.html?highlight=crossentropy#torch.nn.CrossEntropyLoss) for details.

## Multiple Losses

For loss calculation, we support multiple losses training concurrently. Here is an example config of training `unet` on `DRIVE` dataset, whose loss function is `1:3` weighted sum of `CrossEntropyLoss` and `DiceLoss`:

```python
_base_ = './fcn_unet_s5-d16_64x64_40k_drive.py'
model = dict(
    decode_head=dict(loss_decode=[dict(type='CrossEntropyLoss', loss_name='loss_ce', loss_weight=1.0),
            dict(type='DiceLoss', loss_name='loss_dice', loss_weight=3.0)]),
    auxiliary_head=dict(loss_decode=[dict(type='CrossEntropyLoss', loss_name='loss_ce',loss_weight=1.0),
            dict(type='DiceLoss', loss_name='loss_dice', loss_weight=3.0)]),
    )
```

In this way, `loss_weight` and `loss_name` will be weight and name in training log of corresponding loss, respectively.

Note: If you want this loss item to be included into the backward graph, `loss_` must be the prefix of the name.

## Ignore specified label index in loss calculation

In default setting, `avg_non_ignore=False` which means each pixel counts for loss calculation although some of them belong to ignore-index labels.

For loss calculation, we support ignore index of certain label by `avg_non_ignore` and `ignore_index`. In this way, the average loss would only be calculated in non-ignored labels which may achieve better performance, and here is the [reference](https://github.com/open-mmlab/mmsegmentation/pull/1409). Here is an example config of training `unet` on `Cityscapes` dataset: in loss calculation it would ignore label 0 which is background and loss average is only calculated on non-ignore labels:

```python
_base_ = './fcn_unet_s5-d16_4x4_512x1024_160k_cityscapes.py'
model = dict(
    decode_head=dict(
        ignore_index=0,
        loss_decode=dict(
            type='CrossEntropyLoss', use_sigmoid=False, loss_weight=1.0, avg_non_ignore=True),
    auxiliary_head=dict(
        ignore_index=0,
        loss_decode=dict(
            type='CrossEntropyLoss', use_sigmoid=False, loss_weight=1.0, avg_non_ignore=True)),
    ))
```

## Efficient Receptive Field Search (RF-Next)

In semantic segmentation, existing methods construct models with hand-designed receptive fields in layers,
which may be suboptimal.
The RF-Next could effectively search for new receptive field combinations, which could achieve better performance than hand-designed patterns.

In MMSegmentation, you could add a `RFSearchHook` to enable receptive field searching:

```python
custom_hooks = [
    dict(
        type='RFSearchHook',
        mode='search',
        rfstructure_file=None,
        verbose=True,
        by_epoch=False,
        config=dict(
            search=dict(
                step=0,
                max_step=64001,
                search_interval=8000,
                exp_rate=0.5,
                init_alphas=0.01,
                mmin=1,
                mmax=64,
                num_branches=3,
                skip_layer=[]))
                # For the models with auxiliary heads,
                # we recommend skipping the layers used by auxiliary heads.
                # You can add these layers to `skip_layer`.
        ),
]
```

For detailed usage, please see [readme](../../../configs/rfnext/README.md).
