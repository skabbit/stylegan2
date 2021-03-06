# StyleGAN2 for practice

<p align='center'><img src='_out/palekh-512-1536x512-3x1.jpg' /></p>

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/eps696/stylegan2/blob/master/StyleGAN2_colab.ipynb)

This version of famous [StyleGAN2] is intended mostly for fellow artists, who rarely look at scientific metrics, but rather need a working creative tool. At least, this is what I use daily myself. 
Tested on Tensorflow 1.14, requires `pyturbojpeg` for JPG support. Sequence-to-video conversions require [FFMPEG]. For more explicit details refer to the original implementations. 

Notes about [StyleGAN2-ada]: 
1) It have shown smoother convergence (not faster though), but lower output variety (comparing to Diff Augmentation approach). Moreover, in my tests ada-version has failed on few-shot datasets (50~100 images), while Diff Aug succeeded there. So meanwhile i personally prefer this repo with Diff Augmentation training.
2) Nvidia has published [PyTorch-based StyleGAN2-ada], which is claimed to be up to 30% faster, works with flat folder datasets, and should be easier to tweak/debug than TF-based one. 
So here is **[such repo]**, adapted to the features below (custom generation, non-square RGBA data, etc.). 

## Features
* inference (image generation) in arbitrary resolution (finally with proper padding on both TF and Torch)
* **multi-latent inference** with split-frame or masked blending
* non-square aspect ratio support (auto-picked from dataset; resolution must be divisible by 2**n, such as 512x256, 1280x768, etc.)
* transparency (alpha channel) support (auto-picked from dataset)
* models mixing (SWA) and layers-blending (from [Justin Pinkney])
* freezing lower D layers for better finetuning (from [Freeze the Discriminator])

Few operation formats ::
* Windows batch-files, described below (if you're on Windows with powerful GPU)
* local Jupyter notebook (for non-Windows platforms)
* [Colab notebook] (max ease of use, requires Google drive)

also, from [Data-Efficient GANs] ::
* differential augmentation for fast training on small datasets (~100 images)
* support of custom CUDA-compiled TF ops and (slower) Python-based reference ops

also, from [Aydao] ::
* funky "digression" technique (added inside the network for ~6x speed-up)
* cropping square models to non-square aspect ratio (experimental)

also, from [Peter Baylies] and [skyflynil] ::
* non-progressive configs (E,F) with single-scale datasets
* raw JPG support in TFRecords dataset (auto-picked from source images)
* conditional support (labels)
* vertical mirroring augmentation

## Training

* Put your images in `data` as subfolder. Ensure they all have the same color channels (monochrome, RGB or RGBA).  
If needed, crop square fragments from `source` video or directory with images (feasible method, if you work with patterns or shapes, rather than compostions):
```
 multicrop.bat source 512 256 
```
This will cut every source image (or video frame) into 512x512px fragments, overlapped with 256px shift by X and Y. Result will be in directory `source-sub`, rename it as you wish. Non-square dataset should be prepared separately.

* Make TFRecords dataset from directory `data/mydata`:
```
 prepare_dataset.bat mydata
```
This will create file `mydata-512x512.tfr` in `data` directory (if your dataset resolution is 512x512). Images without alpha channel will be stored directly as JPG (dramatically reducing file size). For conditional model split the data by subfolders (`mydata/1`, `mydata/2`, ..) and add `--labels` option.

* Train StyleGAN2 on prepared dataset:
```
 train.bat mydata
```
This will run training process, according to the settings in `src/train.py` (check and explore those!!). If there's no TFRecords file from the previous step, it will be created at this point. Results (models and samples) are saved under `train` directory, similar to original Nvidia approach. Only newest configs E and F are used in this repo (default is F; set `--config E` if you face OOM issue). 

Please note: we save both compact models (containing only Gs network for inference) as `<dataset>-...pkl` (e.g. `mydata-512-0360.pkl`), and full models (containing G/D/Gs networks for further training) as `snapshot-...pkl`. The naming is for convenience only, it does not affect the operations anymore (as the arguments are stored inside the models).

For small datasets (100x images instead of 10000x) one should add `--d_aug` option to use [Differential Augmentation] for more effective training. 
Length of the training is defined by `--lod_kimg X` argument (training duration per layer/LOD). Network with base resolution 1024px will be trained for 20 such steps, for 512px - 18 steps, et cetera. Reasonable `lod_kimg` value for full training from scratch is 300-600, while for finetuning in `--d_aug` mode 20-40 is sufficient. One can override this approach, setting total duration directly with `--kimg X`.

* Resume training on `mydata` dataset from the last saved model at `train/000-mydata-512-f` directory:
```
 train_resume.bat mydata 000-mydata-512-f
```

* Uptrain (finetune) trained model `ffhq-512.pkl` on new data:
```
 train_resume.bat newdata ffhq-512.pkl --finetune
```
There's no need to go for exact steps in this case, you may stop when you're ok with the results (it's better to set low `lod_kimg` to follow the progress). Again, `--d_aug` would greatly enhance training here. There's also `--freezeD` option, supposedly enhancing finetuning on similar data.

## Generation

Results (frame sequences and videos) are saved by default under `_out` directory.

* Test the model in its native resolution:
```
 gen.bat ffhq-1024.pkl
```

* Generate custom animation between random latent points (in `z` space):
```
 gen.bat ffhq-1024 1920-1080 100-20
```
This will load `ffhq-1024.pkl` from `models` directory and make a 1920x1080 px looped video of 100 frames, with interpolation step of 20 frames between keypoints. Please note: omitting `.pkl` extension would load custom network, effectively enabling arbitrary resolution, multi-latent blending, etc. Using filename with extension will load original network from PKL (useful to test foreign downloaded models). There are `--cubic` and `--gauss` options for animation smoothing, and few `--scale_type` choices. Add `--save_lat` option to save all traversed dlatent `w` points as Numpy array in `*.npy` file (useful for further curating).

* Generate more various imagery:
```
 gen.bat ffhq-1024 3072-1024 100-20 -n 3-1
```
This will produce animated composition of 3 independent frames, blended together horizontally (like the image in the repo header). Argument `--splitfine X` controls boundary fineness (0 = smoothest). 
Instead of simple frame splitting, one can load external mask(s) from b/w image file (or folder with file sequence):
```
 gen.bat ffhq-1024 1024-1024 100-20 --latmask _in/mask.jpg
```
Arguments `--digress X` would add some animated funky displacements with X strength (by tweaking initial const layer params). Arguments `--trunc X` controls truncation psi parameter, as usual. 

**NB**: Windows batch-files support only 9 command arguments; if you need more options, you have to edit batch-file itself.

* Project external images onto StyleGAN2 model dlatent points (in `w` space):
```
 project.bat ffhq-1024.pkl photo
```
The result (found dlatent points as Numpy arrays in `*.npy` files, and video/still previews) will be saved to `_out/proj` directory. 

* Generate smooth animation between saved dlatent points (in `w` space):
```
 play_dlatents.bat ffhq-1024 dlats 25 1920-1080
```
This will load saved dlatent points from `_in/dlats` and produce a smooth looped animation between them (with resolution 1920x1080 and interpolation step of 25 frames). `dlats` may be a file or a directory with `*.npy` or `*.npz` files. To select only few frames from a sequence `somename.npy`, create text file with comma-delimited frame numbers and save it as `somename.txt` in the same directory (check examples for FFHQ model). You can also "style" the result: setting `--style_dlat blonde458.npy` will load dlatent from `blonde458.npy` and apply it to higher layers, producing some visual similarity. `--cubic` smoothing and `--digress X` displacements are also applicable here. 

* Generate animation from saved point and feature directions (say, aging/smiling/etc for faces model) in dlatent `w` space:
```
 play_vectors.bat ffhq-1024.pkl blonde458.npy vectors_ffhq
```
This will load base dlatent point from `_in/blonde458.npy` and move it along direction vectors from `_in/vectors_ffhq`, one by one. Result is saved as looped video. 

## Tweaking models

* Strip G/D networks from a full model, leaving only Gs for inference:
```
 model_convert.bat snapshot-1024.pkl 
```
Resulting file is saved with `-Gs` suffix. It's recommended to add `-r` option to reconstruct the network, saving necessary arguments with it. Useful for foreign downloaded models.

* Add or remove layers (from a trained model) to adjust its resolution for further finetuning:
```
 model_convert.bat snapshot-256.pkl --res 512
```
This will produce new model with 512px resolution, populating weights on the layers up to 256px from the source snapshot (the rest will be initialized randomly). It also can decrease resolution (say, make 512 from 1024). Note that this effectively changes number of layers in the model. 

This option works with complete (G/D/Gs) models only, since it's purposed for transfer-learning (resulting model will contain either partially random weights, or wrong `ToRGB` params). 

* Crop resolution of a trained model:
```
 model_convert.bat snapshot-1024.pkl --res 1024-768
```
This will produce working non-square 1024x768 model. Opposite to the method above, this one doesn't change layer count. This is experimental feature (as stated by the author @Aydao), also using some voluntary logic, so works only with basic resolutions.

* Combine lower layers from one model with higher layers from another:
```
 models_blend.bat model1.pkl model2.pkl <res> <level>
```
`<res>` is resolution, at which the models are switched (usually 32/64/128); `<level>` is 0 or 1.

* Mix few models by stochastic averaging all weights:
```
 models_mix.bat models_dir
```
This would work properly only for models from one "family", i.e. uptrained (finetuned) from the same original model. 

## Credits

StyleGAN2: 
Copyright � 2019, NVIDIA Corporation. All rights reserved.
Made available under the [Nvidia Source Code License-NC]
Original paper: http://arxiv.org/abs/1912.04958

Other contributions:
follow the links in the descriptions.

[Nvidia Source Code License-NC]: <https://nvlabs.github.io/stylegan2/license.html>
[StyleGAN2]: <https://github.com/NVlabs/stylegan2>
[StyleGAN2-ada]: <https://github.com/NVlabs/stylegan2-ada>
[PyTorch-based StyleGAN2-ada]: <https://github.com/NVlabs/stylegan2-ada-pytorch>
[such repo]: <https://github.com/eps696/stylegan2ada>
[Peter Baylies]: <https://github.com/pbaylies/stylegan2>
[Aydao]: <https://github.com/aydao/stylegan2-surgery>
[Justin Pinkney]: <https://github.com/justinpinkney/stylegan2/blob/master/blend_models.py>
[skyflynil]: <https://github.com/skyflynil/stylegan2>
[Data-Efficient GANs]: <https://github.com/mit-han-lab/data-efficient-gans>
[Differential Augmentation]: <https://github.com/mit-han-lab/data-efficient-gans>
[Freeze the Discriminator]: <https://arxiv.org/abs/2002.10964>
[FFMPEG]: <https://ffmpeg.org/download.html>
[Colab notebook]: <https://colab.research.google.com/github/eps696/stylegan2/blob/master/StyleGAN2_colab.ipynb>