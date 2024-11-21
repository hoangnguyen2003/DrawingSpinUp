# DrawingSpinUp: 3D Animation from Single Character Drawings (Siggraph Asia 2024)


<a href='https://arxiv.org/abs/2409.08615'><img src='https://img.shields.io/badge/arXiv-2310.12190-b31b1b.svg'></a> &nbsp;
<a href='https://lordliang.github.io/DrawingSpinUp/'><img src='https://img.shields.io/badge/Project-Page-Green'></a> &nbsp;
<a href='https://huggingface.co/papers/2409.08615'><img src='https://img.shields.io/badge/%F0%9F%A4%97%20Hugging%20Face-Page-blue'></a> &nbsp;
<a href='https://www.youtube.com/watch?v=tCkX6hLXaO4&t=3s'><img src='https://img.shields.io/badge/Youtube-Video-b31b1b.svg'></a> &nbsp;
<a href='https://openbayes.com/console/public/tutorials/7r7H1en6BiN'><img src='https://img.shields.io/badge/Demo-OpenBayes贝式计算-blue'></a>


![image](docs/static/images/teaser/fig_teaser.png)



## Install
Hardware: 
  - All experiments are run on a single RTX 2080Ti GPU.
Setup environment:
  - Python 3.8.0
  - PyTorch 1.13.1
  - Cuda Toolkit 11.6
  - Ubuntu 18.04
Install the required packages:
```sh
conda create -n drawingspinup python=3.8
conda activate drawingspinup
pip install torch==1.13.1+cu116 torchvision==0.14.1+cu116 torchaudio==0.13.1 --extra-index-url https://download.pytorch.org/whl/cu116
pip install -r requirements.txt
# tiny-cuda-nn
pip install git+https://github.com/NVlabs/tiny-cuda-nn/#subdirectory=bindings/torch
# python-mesh-raycast
git clone https://github.com/cprogrammer1994/python-mesh-raycast
cd python-mesh-raycast
python setup.py develop
```
Clone this repository and download our 120 processed character drawings and reconstructed 3D characters from [preprocessed.zip](https://portland-my.sharepoint.com/:u:/g/personal/jzhou67-c_my_cityu_edu_hk/EVNRuFdeNrhFt0qkifwCuCwBIhJWSfEke5KvYF_hk91FcQ?e=FtLauE) (a tiny subset of [Amateur Drawings Dataset](https://github.com/facebookresearch/AnimatedDrawings)). Of course you can prepare your own image: a 512x512 character drawing 'texture.png' with its foreground mask 'mask.png'.

```sh
git clone https://github.com/LordLiang/DrawingSpinUp.git
cd DrawingSpinUp
# download blender for frame rendering
wget https://download.blender.org/release/Blender3.3/blender-3.3.1-linux-x64.tar.xz
tar -xvf blender-3.3.1-linux-x64.tar.xz
# install trimesh for blender's python
wget https://bootstrap.pypa.io/get-pip.py
./blender-3.3.1-linux-x64/3.3/python/bin/python3.10 get-pip.py
./blender-3.3.1-linux-x64/3.3/python/bin/python3.10 -m pip install trimesh
cd dataset/AnimatedDrawings
# download preprocessed.zip and put it here
unzip preprocessed.zip
cd ../..
```

## Try A Toy
For convenience, here we offer an example *ff7ab74a67a443e3bda61e69577f4e80* with two retargeted animation files. you can directly run the following scripts to generate two stylized animations. 
```sh
dataset
  └── AnimateDrawings
      └── preprocessed
          └── ff7ab74a67a443e3bda61e69577f4e80
              ├── mesh
              │   ├── fbx_files
              │   │   ├── rest_pose.fbx
              │   │   ├── dab.fbx
              │   │   └── jumping.fbx
              │   └── it3000-mc512-f50000_c_r_s_cbp.obj
              └── char
                  ├── ffc_resnet_inpainted.png
                  ├── mask.png
                  ├── texture.png
                  └── texture_with_bg.png

```
The 'mesh/fbx_files/rest_pose.fbx' is the rigged character generated by Mixamo.
The 'mesh/fbx_files/dab.fbx' and 'mesh/fbx_files/jumping.fbx' are two retargeted animation files. 
```sh
cd 3_style_translator
# only for headless rendering, skip this if you have a GUI
# please check the value because it may be different for you system
export DISPLAY=:1 
# render keyframe pair for training
python run_render.py --uid ff7ab74a67a443e3bda61e69577f4e80
# The default rendering engine is 'BLENDER_EEVEE' and you can change it to 'CYCLES' by:
python run_render.py --uid ff7ab74a67a443e3bda61e69577f4e80 --engine_type CYCLES
# stage1 training
python train_stage1.py --uid ff7ab74a67a443e3bda61e69577f4e80
# stage2 training
python train_stage2.py --uid ff7ab74a67a443e3bda61e69577f4e80
# render frames for inference
python run_render.py --uid ff7ab74a67a443e3bda61e69577f4e80 --test
# inference
python test_stage1.py --uid ff7ab74a67a443e3bda61e69577f4e80
python test_stage2.py --uid ff7ab74a67a443e3bda61e69577f4e80
# generate GIF animation
python gif_writer.py --uid ff7ab74a67a443e3bda61e69577f4e80
cd ..
```

![image](docs/results/gallery/ff7ab74a67a443e3bda61e69577f4e80.gif)

## Step by Step
### Step-1: Contour Removal
We use [FFC-ResNet](https://github.com/advimman/lama) as the backbone to predict the contour region of a given character drawing. 
For model training, you can refer to the original repo.
For training image rendering, see [1_lama_contour_remover/bicar_render_codes](1_lama_contour_remover/bicar_render_codes) which are borrowed from [Wonder3D](https://github.com/xxlong0/Wonder3D/tree/main/render_codes).
Here we focus on inference. Download our pretrained contour removal models from [experiments.zip](https://portland-my.sharepoint.com/:u:/g/personal/jzhou67-c_my_cityu_edu_hk/Ed6BaAAWgIhGqIMjaju_v4kB_K-DIFGu1bQ7zM3CbQMrTw?e=KaltGi).
```sh
cd 1_lama_contour_remover
# download experiments.zip and put it here
unzip experiments.zip
python predict.py
cd ..
```
### Step-2: Textured Character Generation
Firstly please download the pretrained [isnet](https://xuebinqin.github.io/dis/index.html) model ([isnet_dis.onnx](https://huggingface.co/stoned0651/isnet_dis.onnx/resolve/main/isnet_dis.onnx)) for background removal of generated multi-view images.
```sh
cd 2_charactor_reconstructor
mkdir dis_pretrained
cd dis_pretrained
wget https://huggingface.co/stoned0651/isnet_dis.onnx/resolve/main/isnet_dis.onnx
cd ..
```
Then generate multi-view images and fuse them into a textured character.
```sh
# multi-view image generation
python mv.py --uid YOUR_EXAMPLE_ID
# textured character reconstruction
python recon.py --uid YOUR_EXAMPLE_ID
cd ..
```
### Step-3: Stylized Contour Restoration

#### 1) Rigging

Once we get the textured character, we use [Mixamo](https://www.mixamo.com) to rig it automatically and download the rigged character in rest pose as 'mesh/fbx_files/rest_pose.fbx'. 

#### 2) Retargeting

Then we can directly retarget a Mixamo motion (e.g., jumping) onto the rigged character online and download the character with animation as 'mesh/fbx_files/jumping.fbx'. We can also use [rokoko-studio-live-blender](https://github.com/Rokoko/rokoko-studio-live-blender) to retarget a 3D motion (e.g., *.bvh, *.fbx) onto the rigged character offline to generate the animation fbx file.

#### 3) Rendering & Training & Inference

We need to train a model for each sample. Once trained, the model can be applied directly to any new animation frames without further training.

```sh
cd 3_style_translator
# for headless rendering
export DISPLAY=:1
# render keyframe pair for training
python run_render.py --uid YOUR_EXAMPLE_ID
# stage1 training
python train_stage1.py --uid YOUR_EXAMPLE_ID
# stage2 training
python train_stage2.py --uid YOUR_EXAMPLE_ID
# render frames for inference
python run_render.py --uid YOUR_EXAMPLE_ID --test
# inference
python test_stage1.py --uid YOUR_EXAMPLE_ID
python test_stage2.py --uid YOUR_EXAMPLE_ID
# generate GIF animation
python gif_writer.py --uid YOUR_EXAMPLE_ID
cd ..
```

## Acknowledgements
We have intensively borrow codes from the following repositories. Many thanks to the authors for sharing their codes.
- [LaMa](https://github.com/advimman/lama)
- [Wonder3D](https://github.com/xxlong0/Wonder3D)
- [Few-Shot-Patch-Based-Training](https://github.com/OndrejTexler/Few-Shot-Patch-Based-Training)

## Citation
If you find this repository useful in your project, please cite the following work. :)
```
@inproceedings{zhou2024drawingspinup,
  author    = {Zhou, Jie and Xiao, Chufeng and Lam, Miu-Ling and Fu, Hongbo},
  title     = {DrawingSpinUp: 3D Animation from Single Character Drawings},
  booktitle = {SIGGRAPH Asia 2024 Conference Papers},
  year      = {2024},
}
```





