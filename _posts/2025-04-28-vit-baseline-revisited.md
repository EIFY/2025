---
layout: distill
title: ViT Baseline Revisited # — Debugging ViT, RandAugment, and Inception Crop implementations
description: ViT (vision transformer) has emerged as a major model family for computer vision
  with the same architecture as that of dominant LLMs and performance matching or exceeding that of
  the CNN-based ResNet-like models. Shortly after the ICLR publication, a note was published to
  follow up with better performance of the smaller ViT-S/16 variant on the ImageNet-1k dataset. In
  our effort to reproduce that, we find inconsistencies among major implementations of ViT,
  RandAugment, and Inception crop that impact model performance. We achieve better performance with
  90 / 150 epoch training budget and call for better awareness of implementation discrepancies.
date: 2025-04-28
future: true
htmlwidgets: true
hidden: false

# Anonymize when submitting
authors:
  - name: Anonymous

# must be the exact same name as your blogpost
bibliography: 2025-04-28-vit-baseline-revisited.bib

# Add a table of contents to your post.
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly. 
#   - please use this format rather than manually creating a markdown table of contents.
toc:
  - name: Introduction
  - name: Implementation discrepancies
    subsections:
    - name: ViT initialization
    - name: RandAugment
    - name: Inception crop
    - name: Big Vision miscellaneous
  - name: Corrected reproduction
  - name: Implication
  - name: Conclusion
    subsections:
    - name: Replication guide

_styles: >
  r { color: Red }
  g { color: Green }
---

*Edited and expanded from [EIFY/mup-vit/README.md](https://github.com/EIFY/mup-vit).*

## Introduction

The dawn of the deep learning era is marked by the "Imagenet moment", when Alexnet won the
ILSVRC-2012 competition for classifying the ImageNet-1k dataset <d-cite key="10.1145/3065386"></d-cite>.
Since then, the field of CV (computer vision) had been dominated by deep CNNs (convolutional neural networks),
especially ResNet-like ones featuring residual connections <d-cite key="7780459"></d-cite>. In 2020,
however, Dosovitskiy et al. proposed vision transformer (ViT) <d-cite key="dosovitskiy2020image"></d-cite>,
which uses transformer architecture that is already dominating the field of language modeling for CV.
With competitive performance, ViT is now widely adapted not only for CV tasks, but also as a component
for vision-language models <d-cite key="radford2021learning"></d-cite>.

<div class="caption">
    <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/model_scheme.png" class="img-fluid" width="auto" height="auto">
</div>
<div class="caption">
    ViT architecture. Images are divided into square patches, which are then linearly projected into
    tokens. After adding position embedding, the tokens are fed to the pre-norm transformer encoder.
    In the original architecture, an extra classification token is added to the input sequence and
    its output is used for classification ("token pooling"). Figure from <d-cite key="dosovitskiy2020image"></d-cite>.
</div>

Since transformer encoder is permutation-invariant, ViT is considered to have weaker inductive bias
and relies more on model regularization or data augmentation, especially at smaller scale <d-cite key="steiner2022how"></d-cite>.
It is therefore particularly notable that Beyer et al. published a short note claiming that ViT-S/16
can achieve better metrics on the ImageNet-1k dataset than ResNet-50 <d-cite key="beyer2022better"></d-cite>.
With manageable compute requirement and open-sourced Big Vision repo <d-cite key="big_vision"></d-cite>
as the reference implementation, we decide to reproduce it in the PyTorch <d-cite key="Ansel_PyTorch_2_Faster_2024"></d-cite>
ecosystem as the entry point to ViT research.

| Model         | Layers        | Width | MLP        | Heads           | Params  |
|:-------------:|:-------------:|:-----:|:-------------:|:-------------:|:-----:|
| ViT-Ti<d-cite key="pmlr-v139-touvron21a"></d-cite>| 12 | 192 | 768 | 3 | 5.8M |
| ViT-S<d-cite key="pmlr-v139-touvron21a"></d-cite>| 12 | 384 | 1536 | 6 | 22.2M |
| ViT-B<d-cite key="dosovitskiy2020image"></d-cite>| 12 | 768 | 3072 | 12 | 86M |
| ViT-L<d-cite key="dosovitskiy2020image"></d-cite>| 24 | 1024 | 4096 | 16 | 307M |

<div class="caption">
    A few common ViT sizes. <d-cite key="beyer2022better"></d-cite> and this blogpost exclusively
    focuses on ViT-S/16, a variant of ViT-S with patch size $16 \times 16$. Table from <d-cite key="steiner2022how"></d-cite>.
</div>

## Implementation discrepancies

The variant of ViT-S/16 used in <d-cite key="beyer2022better"></d-cite> differs from the original ViT:

1. Instead of token pooling, average of all the output tokens ("global average-pooling", GAP) <d-cite key="9711302"></d-cite> is fed to the MLP head for classification.
2. Fixed 2D sin-cos position embeddings <d-cite key="9711302"></d-cite> is used instead of learned position embeddings.

The model is then trained with Inception crop <d-cite key="7298594"></d-cite>, random horizontal flips,
RandAugment <d-cite key="NEURIPS2020_d85b63ef"></d-cite>, and Mixup <d-cite key="zhang2018mixup"></d-cite>.
Furthermore, there is a variant that replaces the MLP head with a single linear layer that we initially
focus on since it is even simpler and makes "no significant difference" <d-cite key="beyer2022better"></d-cite>.
Sometimes called "simple ViT" <d-cite key="vit-pytorch"></d-cite>, it turns out that there is no up-to-date implementation in PyTorch readily
available<d-footnote>simple_vit.py from vit-pytorch does not support modern attention kernels e.g.
FlashAttention <d-cite key="dao2022flashattention"></d-cite>. The simple_flash_attn_vit.py variant
does but is heavily defensive and resorts to the functional F.scaled_dot_product_attention() for
backward compatibility. </d-footnote>, so we decide to implement our own.

### ViT initialization

It turns out that even building simple ViT from built-in modules from PyTorch requires reinitializing
most of the parameters to match that of the Big Vision reference implementation, including:

1. `torch.nn.MultiheadAttention`

    * `in_proj_weight`: In the most common use case when the value dimension is equal to query and key dimension, their projection matrices are combined into `in_proj_weight` whose
    initial values are [set with `xavier_uniform_()`](https://github.com/pytorch/pytorch/blob/aafb3deaf1460764432472a749d625f03570a53d/torch/nn/modules/activation.py#L1112).
    Likely unintentionally, this means that the values are sampled from uniform distribution $\mathcal{U}(-a, a)$ where $$a = \sqrt{\frac{3}{2 \text{ hidden_dim}}}$$ instead of $$\sqrt{\frac{3}{\text{hidden_dim}}}$$<d-footnote>This is not the only case where the combined QKV projection matrix misleads shape-aware optimizer and parameter initialization (<a href="https://x.com/kellerjordan0/status/1844820920780947800">1</a>, <a href="https://github.com/graphcore-research/unit-scaling/blob/1edf20543439e042cc314667b348d9b4c4480e23/unit_scaling/_modules.py#L479">2</a>). It may be worth reexamining whether this truly leads to better performance and whether it is worth it.</d-footnote>.

    * `out_proj.weight`:  Furthermore, the output projection is [initialized as `NonDynamicallyQuantizableLinear`](https://github.com/pytorch/pytorch/blob/e62073d7997c9e63896cb5289ffd0874a8cc1838/torch/nn/modules/activation.py#L1097). Just like [`torch.nn.Linear`](https://pytorch.org/docs/stable/generated/torch.nn.Linear.html), its initial weights are sampled from uniform distribution $\mathcal{U}(-a, a)$ where $$a = \sqrt{\frac{1}{\text{hidden_dim}}}$$ instead of $$\sqrt{\frac{3}{\text{hidden_dim}}}$$ <d-footnote>See <a href="https://github.com/pytorch/pytorch/issues/57109#issuecomment-828847575">pytorch/pytorch#57109 (comment)</a> for the origin of this discrepancy.</d-footnote>.

    To conform with [`jax.nn.initializers.xavier_uniform()` used by the reference implementation](https://github.com/google-research/big_vision/blob/ec86e4da0f4e9e02574acdead5bd27e282013ff1/big_vision/models/vit.py#L93), both are re-initialized with samples from uniform distribution $\mathcal{U}(-a, a)$ where $$a = \sqrt{\frac{3}{\text{hidden_dim}}}$$ <d-footnote>Note that the standard deviation of uniform distribution $\mathcal{U}(-a, a)$ is $$\frac{a}{\sqrt{3}}$$ So this is also the correct initialization for preserving the input scale assuming that the input features are uncorrelated.</d-footnote>.

2. `torch.nn.Conv2d`: Linear projection of flattened patches can be done with 2D convolution, namely
   [`torch.nn.Conv2d` initialized with `in_channels = 3`, `out_channels = hidden_dim`, and both `kernel_size`
   and `stride` set to `patch_size` in PyTorch](https://pytorch.org/docs/stable/generated/torch.nn.Conv2d.html).
   `torch.nn.Conv2d`, however, defaults to weight and bias initialization with uniform distribution
   $\mathcal{U}(-a, a)$ where $$a = \sqrt{\frac{1}{\text{fan_in}}}$$ instead of Big Vision's Lecun normal
   (truncated normal) initialization <d-cite key="klambauer2017self"></d-cite> for weight and zero-init for bias.
   Furthermore, [PyTorch's own `nn.init.trunc_normal_()`](https://pytorch.org/docs/stable/nn.init.html#torch.nn.init.trunc_normal_)
   doesn't take the effect of truncation on standard deviation into account unlike [the JAX implementation](https://github.com/google/jax/blob/1949691daabe815f4b098253609dc4912b3d61d8/jax/_src/nn/initializers.py#L334), so we have to implement the correction ourselves:

   <div class="caption">
     <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/truncated_normal.png" class="img-fluid" width="auto" height="auto">
   </div>

   <div class="caption">
     Unit normal distribution has standard deviation $\sigma = 1$, but after truncating at $\pm 2$
     the standard deviation is reduced to $\sigma = 0.880$. To restore unit standard deviation, one
     needs to sample $\mathcal{N}(0, \sigma = \frac{1}{.880})$ instead and truncate at $\pm 2 \sigma$.
   </div>

3. `torch.nn.Linear` for the classification head: Specifically for the classification head, Big Vision
   usually zero-init both the weight and bias for the linear layer, including the ViT-S/16 in question.
   Notably, neither [`simple_vit.py`](https://github.com/lucidrains/vit-pytorch/blob/141239ca86afc6e1fe6f4e50b60d173e21ca38ec/vit_pytorch/simple_vit.py#L108) nor [`simple_flash_attn_vit.py`](https://github.com/lucidrains/vit-pytorch/blob/141239ca86afc6e1fe6f4e50b60d173e21ca38ec/vit_pytorch/simple_flash_attn_vit.py#L162) from vit-pytorch does this.

After fixing 1-3, we verify that all of the per-layer summary statistics including minimum, maximum,
mean, and standard deviation of the 21,974,632 model parameters at initialization match that of the
Big Vision reference implementation.

### RandAugment

Following <d-cite key="beyer2022better"></d-cite>, we train the model with batch size 1024, training
budget 90 epochs = round(1281167 / 1024 * 90) = 112603 steps, 10000 warm-up steps, cosine learning
rate decay, 1e-3 maximum learning rate, 1e-4 decoupled weight decay <d-cite key="adamw-decoupling-blog"></d-cite>
(0.1 in PyTorch's parameterization before multiplying by the learning rate), AdamW optimizer,
gradient clip with [`max_norm = 1.0`](https://pytorch.org/docs/stable/generated/torch.nn.utils.clip_grad_norm_.html), Inception
crop with 5%-100% of the area of the original image ("scale"), random horizontal flips,
RandAugment <d-cite key="NEURIPS2020_d85b63ef"></d-cite> with [`num_ops = 2` and `magnitude = 10`](https://pytorch.org/vision/main/generated/torchvision.transforms.v2.RandAugment.html),
and Mixup <d-cite key="zhang2018mixup"></d-cite> with [`alpha = 0.2`](https://pytorch.org/vision/main/generated/torchvision.transforms.v2.MixUp.html).
We also notice that the Big Vision implementation "normalizes" the image with mean = std = (0.5, 0.5, 0.5)
([`value_range(-1, 1)`](https://github.com/google-research/big_vision/blob/46b2456f54b9d4f829d1925b78943372b376153d/big_vision/configs/vit_s16_i1k.py#L52))
and uses [the same neutral RGB value as the fill value for RandAugment](https://github.com/google-research/big_vision/blob/46b2456f54b9d4f829d1925b78943372b376153d/big_vision/pp/archive/autoaugment.py#L676)
and make sure that our reproduction conforms to this training setup. Our reproduction attempt reaches
76.91% top-1 ImageNet-1k validation set accuracy vs. 76.7% for the Head: MLP → linear ablation after
90 epochs as given in <d-cite key="beyer2022better"></d-cite>. However, the training loss and gradient
L2 norm comparison suggests that they are not equivalent:

<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/gradient_L2_norm_comparison.png" class="img-fluid" width="auto" height="auto">
</div>

It turns out that "RandAugment" in torchVision <d-cite key="torchvision2016"></d-cite> is quite different
from "RandAugment" in Big Vision. RandAugment in torchVision, following the description in the paper <d-cite key="NEURIPS2020_d85b63ef"></d-cite>,
randomly selects one of the following 14 transforms:

||||
|:-------------:|:-------------:|:-----:|
|Identity | AutoContrast | Equalize|
|Rotate | Solarize | Color|
|Posterize | Contrast | Brightness|
|Sharpness | ShearX | ShearY|
|TranslateX | TranslateY||

RandAugment in Big Vision, however, [forked](https://github.com/google-research/big_vision/blob/46b2456f54b9d4f829d1925b78943372b376153d/big_vision/pp/archive/autoaugment.py#L20)
from the EfficientNet reference implementation and has a lineup that consists of 16 transforms:

||||
|:-------------:|:-------------:|:-----:|
|AutoContrast | Equalize | Rotate|
|Solarize | Color | Posterize|
|Contrast | Brightness | Sharpness|
|ShearX | ShearY | TranslateX|
|TranslateY | Invert | Cutout|
|SolarizeAdd|||

with "Identity" no-op removed and "Invert", "Cutout", and "SolarizeAdd" added to the list. Moreover,
its implementation of the Contrast transform is broken. In short: [the `mean` here](https://github.com/google-research/big_vision/blob/01edb81a4716f93a48be43b3a4af14e29cdb3a7f/big_vision/pp/autoaugment.py#L209-L213)

{% highlight python %}
  # Compute the grayscale histogram, then compute the mean pixel value,
  # and create a constant image size of that value.  Use that as the
  # blending degenerate target of the original image.
  hist = tf.histogram_fixed_width(degenerate, [0, 255], nbins=256)
  mean = tf.reduce_sum(tf.cast(hist, tf.float32)) / 256.0
{% endhighlight %}

is supposed to be the mean pixel value, but as it is it's just summing over the histogram (therefore
equal to height * width), divided by 256. The correct implementation should be

{% highlight python %}
  image_height = tf.shape(image)[0]
  image_width = tf.shape(image)[1]
  # Compute the grayscale histogram, then compute the mean pixel value,
  # and create a constant image size of that value.  Use that as the
  # blending degenerate target of the original image.
  hist = tf.histogram_fixed_width(degenerate, [0, 255], nbins=256)
  mean = tf.reduce_sum(
      tf.cast(hist, tf.float32) * tf.linspace(0., 255., 256)) / float(image_height * image_width)
{% endhighlight %}

or [as suggested by @yeqingli, just call `tf.image.adjust_contrast()` for the whole function](https://github.com/tensorflow/models/pull/11219#discussion_r1792502043):

{% highlight python %}
def contrast(image: tf.Tensor, factor: float) -> tf.Tensor:
  return tf.image.adjust_contrast(image, factor)
{% endhighlight %}

We can experimentally confirm and visualize this using a calibration grid. For example, given the
$224 \times 224$ grid below that consists of (192, 64, 64) and (64, 192, 192) squares:

<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/color_grid.png" class="img-fluid" width="auto" height="auto">
</div>

`contrast(tf_color_tile, 1.9)` before the fix returns a grid with (188, 0, 0) and (0, 188, 188) squares:
<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/broken_result.png" class="img-fluid" width="auto" height="auto">
</div>

After the fix, `contrast(tf_color_tile, 1.9)` returns a grid with (249, 6, 6) and (6, 249, 249) squares instead:
<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/correct_result.png" class="img-fluid" width="auto" height="auto">
</div>

Another problematic transform is the Solarize transform, [whose augmentation strength varies unintuitively with `magnitude` and is in danger of uint8 overflow](https://github.com/google-research/big_vision/issues/110).
It just happens to behave as expected with [`randaug(2, 10)`](https://github.com/google-research/big_vision/blob/ec86e4da0f4e9e02574acdead5bd27e282013ff1/big_vision/configs/vit_s16_i1k.py#L57) in our case.

As for the discrepancy between torchVision's RandAugment and Big Vision's RandAugment, we can also
use a calibration grid to confirm and visualize. Given the following $224 \times 224$ black & white
calibration grid:

<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/calibration_grid.png" class="img-fluid" width="auto" height="auto">
</div>

We apply both versions of `RandAugment(2, 10)` 100000 times ([notebook](https://github.com/EIFY/mup-vit/blob/e35ab281acb88f669b18555603d9187a194ccc2f/notebooks/RandAugmentOnCalibrationGrid.ipynb)) to gather the stats. All of the resulting pixels remain colorless (i.e. for RGB values (r, g, b), r == g == b) so
we can sort them from black to white into a spectrum ([notebook](https://github.com/EIFY/mup-vit/blob/e35ab281acb88f669b18555603d9187a194ccc2f/notebooks/GradientVisual.ipynb)).
For the following $2000 \times 200$ spectra, pixels are sorted top-down, left-right, and each pixel represents 224 * 224 * 100000 / (2000 * 200) = 112 * 112 pixels of the aggregated output, amounting to 1/4 of one output image. In case one batch of 12544 pixels happens to be of different values, I took the average. Here is the spectrum of torchvision's `RandAugment(2, 10)`:

<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/torch_vision_randaugment_2_10.png" class="img-fluid" width="auto" height="auto">
</div>

Here is the spectrum of torchvision's `RandAugment(2, 10, fill=[128] * 3)`. We can see that it just shifts the zero-padding part of the black to the (128, 128, 128) neutral gray:

<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/torch_vision_randaugment_2_10_mid_fill.png" class="img-fluid" width="auto" height="auto">
</div>

And here is the spectrum of big_vision's `randaug(2, 10)`:

<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/big_vision_randaugment_2_10.png" class="img-fluid" width="auto" height="auto">
</div>

We end up subclassing torchvision's [`v2.RandAugment`](https://pytorch.org/vision/main/generated/torchvision.transforms.v2.RandAugment.html)([notebook](https://github.com/EIFY/mup-vit/blob/e35ab281acb88f669b18555603d9187a194ccc2f/notebooks/RandAugmentCalibration.ipynb)) in order to
remove and add transforms to match the lineup of Big Vision's RandAugment. We use a variety of
calibration grids to make sure that all transforms give results within $\pm 1$ of the RGB values given by the Big Vision
counterpart, except the Contrast transform for which we decide against replicating the bug. Even with
that exception, the near-replication of big_vision's `randaug(2, 10)` results in near-identical spectrum:

<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/torch_vision_randaugment17_2_10.png" class="img-fluid" width="auto" height="auto">
</div>

Before we move on: credit to [@tobiasmaier for first noticing the contrast bug and submitting a PR to fix it](https://github.com/tensorflow/tpu/pull/764).
Apparently, some CV people are aware of the bug and the discrepancy between the RandAugment
implemenations. To our best knowledge however, these issues are not publcally documented anywhere.

{% twitter https://twitter.com/giffmana/status/1798978504689938560 %}
{% twitter https://twitter.com/wightmanr/status/1799170879697686897 %}

### Inception crop

Model trained with the near-replication of Big Vision's `randaug(2, 10)` for 90 epochs
[reaches 77.27% top-1 validation set accuracy and the gradient L2 norm looks the same, but the training loss still differs](https://api.wandb.ai/links/eify/8d0wix47):

<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/inception_crop_l2_grads_discrepancy.png" class="img-fluid" width="auto" height="auto">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/inception_crop_training_loss_discrepancy.png" class="img-fluid" width="auto" height="auto">
</div>

It turns out that besides the default min scale ([8%](https://pytorch.org/vision/main/generated/torchvision.transforms.v2.RandomResizedCrop.html)
vs. [5%](https://github.com/google-research/big_vision/blob/01edb81a4716f93a48be43b3a4af14e29cdb3a7f/big_vision/pp/ops_image.py#L199)),
the "Inception crop" implemented as torchvision [`v2.RandomResizedCrop()`](https://pytorch.org/vision/main/generated/torchvision.transforms.v2.RandomResizedCrop.html)
is not the same as calling [`tf.slice()`](https://www.tensorflow.org/api_docs/python/tf/slice) with the bbox returned by
[`tf.image.sample_distorted_bounding_box()`](https://www.tensorflow.org/api_docs/python/tf/image/sample_distorted_bounding_box)
as it is commonly done in JAX / TF:

1. They both rejection-sample the crop, but [`v2.RandomResizedCrop()`](https://pytorch.org/vision/main/generated/torchvision.transforms.v2.RandomResizedCrop.html)
is hardcoded to try 10 times while [`tf.image.sample_distorted_bounding_box()`](https://www.tensorflow.org/api_docs/python/tf/image/sample_distorted_bounding_box)
defaults to 100 attempts.
2. [`v2.RandomResizedCrop()`](https://pytorch.org/vision/main/generated/torchvision.transforms.v2.RandomResizedCrop.html) samples the aspect ratio uniformly in log space,
[`tf.image.sample_distorted_bounding_box()`](https://www.tensorflow.org/api_docs/python/tf/image/sample_distorted_bounding_box) samples uniformly in linear space.
3. [`v2.RandomResizedCrop()`](https://pytorch.org/vision/main/generated/torchvision.transforms.v2.RandomResizedCrop.html) samples the area cropped uniformly while
[`tf.image.sample_distorted_bounding_box()`](https://www.tensorflow.org/api_docs/python/tf/image/sample_distorted_bounding_box) samples the crop height uniformly given the aspect ratio and area range.
4. If all attempts fail, [`v2.RandomResizedCrop()`](https://pytorch.org/vision/main/generated/torchvision.transforms.v2.RandomResizedCrop.html) at least crops the image to make sure that the aspect ratio falls within range before resizing. [`tf.image.sample_distorted_bounding_box()`](https://www.tensorflow.org/api_docs/python/tf/image/sample_distorted_bounding_box) just returns the whole image to be resized.

We can verify this by taking stats of the crop sizes given the same image. Here is the density plot of
(h, w) returned by `v2.RandomResizedCrop.get_params(..., scale=(0.05, 1.0), ratio=(3/4, 4/3))`,
given an image of (height, width) = (256, 512), N = 10000000:

<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/torch_hw_counts.png" class="img-fluid" width="auto" height="auto">
</div>

There is a total of 14340 crop failures resulting in a bright pixel at the bottom right, but otherwise
the density is roughly uniform. In comparison, here is what `tf.image.sample_distorted_bounding_box(..., area_range=[0.05, 1])` returns:

<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/tf_hw_counts.png" class="img-fluid" width="auto" height="auto">
</div>

While cropping never failed, we can see clearly that it's oversampling smaller crop areas, as if there
were light shining from top-left ([notebook](https://github.com/EIFY/mup-vit/blob/e35ab281acb88f669b18555603d9187a194ccc2f/notebooks/InceptionCropStats.ipynb)).
We replicate `tf.image.sample_distorted_bounding_box()`'s sampling logic in PyTorch and rerun the experiment by
training a model with it for 90 epochs. At last, both the gradient L2 norm and the training loss match:

<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/inception_crop_l2_grads_match.png" class="img-fluid" width="auto" height="auto">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/inception_crop_training_loss_match.png" class="img-fluid" width="auto" height="auto">
</div>

This true reproduction model reaches 76.94% top-1 validation set accuracy after 90 epochs. In summary:

| Model | Top-1 val acc. |
|:-------------:|:-------------:|
| Reference | 76.7% |
| Same ViT init. in PyTorch | 76.91% |
| + Same RandAugment | **77.27%** |
| + Same Inception Crop | 76.94% |

Before we move on: Which implementation is correct? While the reference implementation of the Inception crop
is not publicly available to our best knowledge, here is the relevant excerpt from the paper <d-cite key="szegedy2015going"></d-cite>:

<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/Inception_crop_paper_excerpt.png" class="img-fluid" width="auto" height="auto">
</div>

While there is some ambiguity in whether aspect ratio should be sampled uniformly in log space or
linear space, it is clear from the description that crop size should be sampled uniformly in crop area.

### Big Vision miscellaneous

Finally <d-footnote>Chronologically, these experiments were run before the true reproduction.</d-footnote>,
there are a few idiosyncrasies of the Big Vision reference implementation that either are impractical
or do not make sense to replicate. Here we run experiments using the Big Vision repo to test their
effects on model training.

1. In <d-cite key="beyer2022better"></d-cite> only the first 99% of the training data is used for training while the
remaining 1% is used for minival "to encourage the community to stop selecting design choices on the validation (de-facto test) set". This however is
difficult to replicate with [`torchvision.datasets`](https://pytorch.org/vision/main/datasets.html)
since [`datasets.ImageNet()`](https://pytorch.org/vision/main/generated/torchvision.datasets.ImageNet.html)
is ordered by class label, unlike [tfds](https://www.tensorflow.org/datasets/overview) whose ordering
is somewhat randomized:

    ```python
    import tensorflow_datasets as tfds
    ds = tfds.builder('imagenet2012').as_dataset(split='train[99%:]')
    from collections import Counter
    c = Counter(int(e['label']) for e in ds)
    >>> len(c)
    999
    >>> max(c.values())
    27
    >>> min(c.values())
    3
    ```
Naively trying to do the same with [`torchvision.datasets`](https://pytorch.org/vision/main/datasets.html)
prevents the model from learning the last few classes and results in [near-random performance on the minival](https://api.wandb.ai/links/eify/3ju0jben):
the model only learns the class that happened to stride across the first 99% and the last 1%.
Instead of randomly selecting 99% of the training data or copying the 99% slice given by tfds, we
just fall back to training on 100% of the training data.

2. The reference implementation sets [`shuffle_buffer_size = 250_000`](https://github.com/google-research/big_vision/blob/46b2456f54b9d4f829d1925b78943372b376153d/big_vision/configs/vit_s16_i1k.py#L49)
which is only 20% of the training set, so the training data is not fully shuffled. To fit the data in
a TensorBook, we need to further reduce it to `150_000` and [revive the `utils.accumulate_gradient()`](https://github.com/EIFY/big_vision/commit/e2f74c170d926ab73846ceb9b0d9aad2aa5814af)
function to train with gradient accumulation. 1 and 2 result in 76.74% top-1 validation set accuracy.

3. We fix the Contrast transform bug [described above](#randaugment).

4. [Inadvertently the validation set is anti-aliased](https://x.com/giffmana/status/1808394534424138049)
because [the `area` resize method always anti-aliases](https://www.tensorflow.org/api_docs/python/tf/image/resize),
but not the training set. We change the resize method to `bilinear` with anti-aliasing for both like
the default for [`v2.Reize()`](https://pytorch.org/vision/main/generated/torchvision.transforms.v2.Resize.html#torchvision.transforms.v2.Resize)
and [`v2.RandomResizedCrop()`](https://pytorch.org/vision/main/generated/torchvision.transforms.v2.RandomResizedCrop.html)
in torchvision.

5. [`tf.io.decode_jpeg()` by default lets the system decide the JPEG decompression algorithm](https://www.tensorflow.org/api_docs/python/tf/io/decode_jpeg). Specifying `dct_method="INTEGER_ACCURATE"` makes it [behave like the PIL/cv2/PyTorch counterpart](https://github.com/google-research/big_vision/blob/01edb81a4716f93a48be43b3a4af14e29cdb3a7f/big_vision/pp/ops_image.py#L39) (see also some quick test at the end of the [notebook](https://github.com/EIFY/mup-vit/blob/e35ab281acb88f669b18555603d9187a194ccc2f/notebooks/RandAugmentCalibration.ipynb)). [We expose this option](https://github.com/EIFY/big_vision/commit/a31822116a9377d6f6dbfbd78372964ed48d8b9a) and test it in our experiment. 1-5 result in 76.87% top-1 validation set accuracy.

6. [`optax.scale_by_adam()` supports the unusual option of using a different dtype for the 1st order accumulator, `mu_dtype`](https://optax.readthedocs.io/en/latest/api/transformations.html#optax.scale_by_adam) and the reference implementation set it to [`bfloat16`](https://github.com/google-research/big_vision/blob/46b2456f54b9d4f829d1925b78943372b376153d/big_vision/configs/vit_s16_i1k.py#L80) instead of `float32` like the rest of the model. Changing it back to `float32` in addition to 1-5 results in [76.77% top-1 validation set accuracy](https://api.wandb.ai/links/eify/dr9b8q4w).

7. Lastly, we test whether [fully shuffling the training set](https://www.tensorflow.org/api_docs/python/tf/data/Dataset#fully_shuffling_all_the_data)
helps model performance. We set [`config.input.shuffle_buffer_size = 1281167`](https://github.com/EIFY/big_vision/blob/5adab5c5985f0cd9b2e5fd887a58c062866ab092/big_vision/configs/vit_s16_i1k_8_gpu.py#L50)
and train a model on a 8x A100-SXM4-40GB instance on [Lambda](https://lambdalabs.com/) with 1-6 but
no gradient accumulation. The model reaches [76.85% top-1 validation set accuracy](https://api.wandb.ai/links/eify/huigfbka).

8. There is one discrepancy that we choose to ignore: In the Big Vision implementation of the Inception crop
[max (aspect) ratio is set to 1.33](https://github.com/google-research/big_vision/blob/46b2456f54b9d4f829d1925b78943372b376153d/big_vision/pp/ops_image.py#L200),
but in [`v2.RandomResizedCrop()` of torchvision](https://pytorch.org/vision/main/generated/torchvision.transforms.v2.RandomResizedCrop.html) it's set to 4.0/3.0,
with difference well above floating point precision. We don't find it relevant however, and it should be the latter as originally described.

In summary:

| Model | Top-1 val acc. |
|:-------------:|:-------------:|
| Reference | 76.7% |
| 100% training set, smaller shuffle buffer, grad. acc. | 76.74% |
| + contrast() fix, consistent anti-aliasing, accurate JPEG decode | **76.87%** |
| + fp32 1st order acc. | 76.77% |
| + full shuffle, no grad. acc. | 76.85% |

The metrics suggest that none of them has any effect.

## Corrected reproduction

To further assess the effect of different implementations, we reproduce the full ablation results
with 90, 150, and 300 epochs of training budget from <d-cite key="beyer2022better"></d-cite> on a
8x A100-SXM4-40GB instance but with the torchvision Inception crop, [`v2.RandomResizedCrop()`](https://pytorch.org/vision/main/generated/torchvision.transforms.v2.RandomResizedCrop.html).
Here is the relevant part of Table 1 from <d-cite key="beyer2022better"></d-cite>:

| Model | 90ep |  150ep | 300ep |
|:-------------:|:-------------:|:-------------:|:-------------:|
| Our improvements | 76.5 | 78.5 | 80.0 |
| no RandAug+MixUp | 73.6 | 73.7 | 73.7 |
| Posemb: sincos2d → learned | 75.0 | 78.0 | 79.6 |
| Batch-size: 1024 → 4096 | 74.7 | 77.3 | 78.6 |
| Global Avgpool → [cls] token | 75.0 | 76.9 | 78.2 |
| Head: MLP → linear | 76.7 | 78.6 | 79.8 |

Here are our results training the same models with the correct Inception crop. We call their main
approach "better baseline" and we round to the same figure to avoid making them seem more accurate
than they are:

| Model | 90ep |  150ep | 300ep |
|:-------------:|:-------------:|:-------------:|:-------------:|
| Better baseline | 76.8 <g>+0.3</g> | 78.8 <g>+0.3</g> | 79.6 <r>-0.4</r> |
| no RandAug+MixUp | 73.4 <r>-0.2</r> | 73.5 <r>-0.2</r> | 73.8 <g>+0.1</g> |
| Posemb: sincos2d → learned | 76.2 <g>+1.2</g> | 77.7 <r>-0.3</r> | 79.6 <g>+0.0</g> |
| Batch-size: 1024 → 4096 | 75.6 <g>+0.9</g> | 77.8 <g>+0.5</g> | 78.3 <r>-0.3</r> |
| Global Avgpool → [cls] token | 75.3 <g>+0.3</g> | 77.1 <g>+0.2</g> | 77.5 <r>-0.5</r> |
| Head: MLP → linear | 76.8 <g>+0.1</g> | 78.1 <r>-0.5</r> | 78.9 <r>-0.9</r> |

We first notice that there is quite a bit of variation: In particular, 76.8(3)% for 90ep Head: MLP → linear
is low compared to our previous result 77.27%. This reminds us of our ["grafting" experiment](https://github.com/EIFY/big_vision/tree/grafted)
of training a PyTorch model on the Big Vision data pipelines that results in [76.38% top-1 validation set accuracy](https://wandb.ai/eify/mup-vit/reports/torch-on-big-vision-input3--Vmlldzo4NTMzMTU4).
If we believe that they are all from their respective distributions, then the Inception crop implementation
means the difference between 76.38-76.87 and 76.83-77.27, with variation of $\pm 0.2$ in each range and overlapping extrema.
Regardless, the trend seems to be that the correct Inception crop benefits lower training budget
experiments but shows more mixed results for higher training budget. One plausible explanation
is that oversampling crops with smaller area amounts to stronger augmentation, which tends to benefit
models in the long run <d-footnote>One might be tempted to try curriculum learning with increasing augmentation strength, but early experiments (<a href="https://github.com/EIFY/mup-vit/tree/curriculum">source</a>) result in <a href="https://api.wandb.ai/links/eify/ylnhm3or">76.65-77.35 for 90ep Head: MLP → linear</a>, almost the same range.</d-footnote>. We can also see this by [comparing better baseline with no RandAug+MixUp](https://api.wandb.ai/links/eify/3joh568g):

<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/augmentation_tortoise_and_hare.png" class="img-fluid" width="auto" height="auto">
</div>

As a setup that is already under-augmented, no RandAug+MixUp itself does not benefit from the
correct Inception crop even with 90ep training budget. We also suspect that it may be possible to
compensate the Inception crop implementation discrepancy by adjusting augmentation strength otherwise,
either through parameters of Inception crop (scale and ratio) or other parts of the augmentation pipeline
(RandAugment and Mixup).

## Implication

In the broad sense, the discrepancies in parameter initialization we highlight are likely relevant to
most of the models written in PyTorch and JAX, as long as they use the default

* [`torch.nn.MultiheadAttention`](https://pytorch.org/docs/stable/generated/torch.nn.MultiheadAttention.html) / [`flax.linen.MultiHeadDotProductAttention`](https://flax.readthedocs.io/en/v0.5.3/_autosummary/flax.linen.MultiHeadDotProductAttention.html)
* [`torch.nn.Linear`](https://pytorch.org/docs/stable/generated/torch.nn.Linear.html) / [`flax.linen.Dense`](https://flax.readthedocs.io/en/v0.5.3/_autosummary/flax.linen.Dense.html)
* [`torch.nn.Conv2d`](https://pytorch.org/docs/stable/generated/torch.nn.Conv2d.html) and other `torch.nn._ConvNd` subclasses / [`flax.linen.Conv`](https://flax.readthedocs.io/en/v0.5.3/_autosummary/flax.linen.Conv.html), or
* [`torch.nn.init.trunc_normal_()`](https://pytorch.org/docs/stable/nn.init.html#torch.nn.init.trunc_normal_) / [`jax.nn.initializers.variance_scaling` with `distribution="truncated_normal"`](https://jax.readthedocs.io/en/latest/_autosummary/jax.nn.initializers.variance_scaling.html).

However, we don't know which of these discrepancies matter, if any. In the narrow sense, we have
shown that the discrepancies in RandAugment and Inception crop implementations impact model performance.
Some CV people are aware of the RandAugment issues, but to our best knowledge the Inception crop
discrepancies are not documented anywhere. Given its deep root in training deep image models, it has
potentially broad implications. Tracking down the commit histories, we find that the schism between
the implementations goes back 8 years, almost as old as the paper <d-cite key="szegedy2015going"></d-cite> itself:

* [`sample_distorted_bounding_box()` has remained unchanged](https://github.com/tensorflow/tensorflow/commit/f3a77378b4c056e76691c5eba350a022c11e00d4).
* Hardcoded 10 sampling attempts and uniform crop area sampling ([1 and 3 listed here](#inception-crop)) [have been the same for `RandomSizedCrop()`](https://github.com/pytorch/vision/commit/d9b8d003d282904461d30e60b9e13ed2f74f3bc6).

Below are some of the papers that we know are affected, the context in which the TF implementation
of Inception crop is mentioned, and its current Google Scholar citation counts:

| Paper | Context |  Cited By |
|:-------------:|:-------------:|:-------------:|
| A Simple Framework for Contrastive Learning of Visual Representations <d-cite key="chen2020simple"></d-cite>  | [`tf.image.sample_distorted_bounding_box()` call in the repo](https://github.com/google-research/simclr/blob/383d4143fd8cf7879ae10f1046a9baeb753ff438/data_util.py#L285) | 20235 |
| Big Self-Supervised Models are Strong Semi-Supervised Learners <d-cite key="chen2020big"></d-cite>  | [`tf.image.sample_distorted_bounding_box()` call in the repo](https://github.com/google-research/simclr/blob/383d4143fd8cf7879ae10f1046a9baeb753ff438/tf2/data_util.py#L279) | 2498 |
| Scaling Vision Transformers <d-cite key="zhai2022scaling"></d-cite>  | See [`inception_crop(224)` in the reference config](https://github.com/google-research/big_vision/blob/46b2456f54b9d4f829d1925b78943372b376153d/big_vision/configs/proj/scaling_laws/train_vit_g.py#L49) in Big Vision | 1149 |
| How to train your ViT? Data, Augmentation, and Regularization in Vision Transformers <d-cite key="steiner2022how"></d-cite> | 'The code for full reproduction of model training is available at <https://github.com/google-research/big_vision>.' (...) 'The images are pre-processed by Inception-style cropping (36) and random horizontal flipping.' | 650 |
| Better plain ViT baselines for ImageNet-1k <d-cite key="beyer2022better"></d-cite> | <https://github.com/google-research/big_vision>, 'All experiments use “inception crop”' | 108 |
| Getting ViT in Shape: Scaling Laws for Compute-Optimal Model Design <d-cite key="alabdulmohsin2024getting"></d-cite> | 'We use the big_vision codebase [10, 9] for conducting experiments in this project' and `inception_crop` can be found in the hyper-parameters settings in the appendix. | 26 |
| FlexiViT: One Model for All Patch Sizes <d-cite key="beyer2023flexivit"></d-cite> | `inception_crop` can be found in config.json downloaded from the link in the [flexivit project README](https://github.com/google-research/big_vision/tree/46b2456f54b9d4f829d1925b78943372b376153d/big_vision/configs/proj/flexivit) in Big Vision. | 82 |
| Patch n' Pack: NaViT, a Vision Transformer for any Aspect Ratio and Resolution <d-cite key="dehghani2024patch"></d-cite> | 'NaViT is implemented in JAX [21] using the FLAX library [22] and built within Scenic [23].' (...) 'Second, we apply inception-style cropping [35] with a fixed minimum area of 50%'. While the source code of NaViT isn't available, [Scenic relies on `tf.image.sample_distorted_bounding_box()` for Inception crop](https://github.com/search?q=repo%3Agoogle-research%2Fscenic%20sample_distorted_bounding_box&type=code) and [this call](https://github.com/google-research/scenic/blob/0340172a1ffa97a2cdb02adde7ea6d0ea66e539c/scenic/dataset_lib/dataset_utils.py#L735) might be what it uses. | 51 |

It is out of scope to retrain these models with the correct Inception crop, but we can compare the
90ep Head: MLP → linear models we trained with either the correct uniform area sampling:

* "Fixed": the model trained on a single RTX 3080 Laptop (77.27% top-1)
* "Fixed Parallel": the model trained on 8x A100-SXM4-40GB (76.83% top-1)

or the TF-like uniform crop height sampling:

* "Repro": the true reproduction model (76.94% top-1)
* "Grafted": the PyTorch model trained on the Big Vision data pipelines (76.38% top-1)

Loosely inspired by guided backpropagation <d-cite key="DBLP:journals/corr/SpringenbergDBR14"></d-cite>,
we compute the gradient of the class logit on the images from the ImageNet-1k validation set, sum over
gradients of the 3 (RGB) channels per pixel, and then zero-out the negative ones. Finally, we measure
how evenly the positive gradient is distributed among the $224^2$ pixels of the validation image center
crop by the entropy ([notebook](https://github.com/EIFY/mup-vit/blob/51bcdc77ea6e26ceef049e96f59702a9d9ef15d1/notebooks/gather_stats.ipynb)).
We hypothesize that since the correct Inception crop tends to let the model see more of the image,
How focused the model's attention (in the colloquial sense here) may differ. For reference and sanity
check, the maximum possible entropy here is $\log(224^2) = 10.82$:

<div class="caption">
  <img src="/2025/assets/img/2025-04-28-vit-baseline-revisited/gradient_entropy.png" class="img-fluid" width="auto" height="auto">
</div>

Tentatively we find that to be the case. There is quite a bit of variation between the replications
just like the evaluation metric, but models trained with the correct uniform area sampling tend to
have more focused attention with lower positive gradient entropy. This is interesting and not necessary
what one might have expected. As a plausible explanation, we hypothesize that perhaps the smaller crops
of TF-like Inception crop force the model to learn different localized features in each epoch, so
when the model sees the validation set center crop it is paying more evenly distributed attention.

Now, how would the models trained for the highly-cited papers above including SimCLR, SoViT-400M,
and FlexViT behave differently if they were trained with the correct Inception crop?

## Conclusion

We wish that things just worked — The same models, the same layers, the same initializations, and the
same data augmentations mean the same things across ecosystems — but that is not the world we live in.
To get closer to that world, all of us need to be more proactive in documenting discrepancies and
fixing bugs in existing implementations.

And our own repo should be no exception.

### Replication guide

1. [Install PyTorch](https://pytorch.org/), including torchvision
2. Prepare the [ILSVRC 2012 ImageNet-1k dataset](https://pytorch.org/vision/main/generated/torchvision.datasets.ImageNet.html)
3. `git clone https://github.com/EIFY/mup-vit.git`
4. `pip install wandb` or make changes to use your preferred logging platform
5. Depending on your machine:
   * Multi-node, multi-GPU:
    The code is intended to support but currently not tested on a multi-node, multi-GPU setup.
   * Single-node, multi-GPU:
    Say the ImagNet-1k dataset is at `/data/ImageNet/`. Just point `MUPVIT_MAIN` to `main.py` in the repo and run the bash script `baseline_revisited.sh`:

    ```bash
    #!/bin/bash

    MUPVIT_MAIN=~/Downloads/mup-vit/main.py
    PYTHON=torchrun
    N_WORKERS=80
    N_THREADS=124

    for EPOCH in 90 150 300
    do
        NUMEXPR_MAX_THREADS=$N_THREADS $PYTHON $MUPVIT_MAIN /data/ImageNet/ --workers $N_WORKERS --multiprocessing-distributed --epochs $EPOCH --batch-size 1024 --torchvision-inception-crop --mlp-head --report-to wandb --name better-baseline-${EPOCH}ep
    done
    # (...)
    ```

    Remove `--torchvision-inception-crop` for faithful replication of <d-cite key="beyer2022better"></d-cite>, i.e. with TF-like Inception crop.
    Experimentally step/s hits a plateau with 64-112 workers on a Lambda 8x A100-SXM4-40GB instance but the `no-randaug-no-mixup` experiment is
    still faster (17h30m vs. 1d for 300 epochs), suggesting that the GPUs are sometimes waiting for data augmentation. Adjust both `N_WORKERS`
    and `N_THREADS` for your machiine.

   * Single-node, single-GPU:
    Replace `torchrun` with `python` and remove `--multiprocessing-distributed`. You may need to use gradient accumulation to make sure $$\left( \frac{\text{batch-size}}{\text{accum-freq}} \right)$$ samples fit in the GPU RAM:

    ```bash
    #!/bin/bash

    MUPVIT_MAIN=~/Downloads/mup-vit/main.py
    PYTHON=python
    N_WORKERS=4
    ACC_FREQ=8

    for EPOCH in 90 150 300
    do
        $PYTHON $MUPVIT_MAIN /data/ImageNet/ --workers $N_WORKERS --epochs $EPOCH --batch-size 1024 --accum-freq $ACC_FREQ --torchvision-inception-crop --mlp-head --report-to wandb --name better-baseline-${EPOCH}ep
    done
    # (...)
    ```

    Warning: On a single GPU these experiments will likely take a long time.