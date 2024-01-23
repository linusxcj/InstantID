# InstantID
<a href='https://instantid.github.io/'><img src='https://img.shields.io/badge/Project-Page-green'></a> 
<a href='https://arxiv.org/abs/2401.07519'><img src='https://img.shields.io/badge/Technique-Report-red'></a> 
<a href='https://huggingface.co/papers/2401.07519'><img src='https://img.shields.io/static/v1?label=Paper&message=Huggingface&color=orange'></a> 
<a href='https://huggingface.co/spaces/InstantX/InstantID'><img src='https://img.shields.io/badge/%F0%9F%A4%97%20Hugging%20Face-Spaces-blue'></a> 
[![Replicate](https://replicate.com/zsxkib/instant-id/badge)](https://replicate.com/zsxkib/instant-id)

**InstantID : Zero-shot Identity-Preserving Generation in Seconds**

InstantID is a new state-of-the-art tuning-free method to achieve ID-Preserving generation with only single image, supporting various downstream tasks.

<img src='assets/applications.png'>

## Demos

### Stylized Synthesis

<p align="center">
  <img src="assets/0.png">
</p>

### Comparison with Previous Works

<p align="center">
  <img src="assets/compare-a.png">
</p>

Comparison with existing tuning-free state-of-the-art techniques. InstantID achieves better fidelity and retain good text editability (faces and styles blend better).

<p align="center">
  <img src="assets/compare-c.png">
</p>

Comparison with pre-trained character LoRAs. We don't need multiple images and still can achieve competitive results as LoRAs without any training.

<p align="center">
  <img src="assets/compare-b.png">
</p>

Comparison with InsightFace Swapper (also known as ROOP or Refactor). However, in non-realistic style, our work is more flexible on the integration of face and background.


## Download

You can directly download the model from [Huggingface](https://huggingface.co/InstantX/InstantID).
You also can download the model in python script:

```python
from huggingface_hub import hf_hub_download
hf_hub_download(repo_id="InstantX/InstantID", filename="ControlNetModel/config.json", local_dir="./checkpoints")
hf_hub_download(repo_id="InstantX/InstantID", filename="ControlNetModel/diffusion_pytorch_model.safetensors", local_dir="./checkpoints")
hf_hub_download(repo_id="InstantX/InstantID", filename="ip-adapter.bin", local_dir="./checkpoints")
```

If you cannot access to Huggingface, you can use [hf-mirror](https://hf-mirror.com/) to download models.
```python
export HF_ENDPOINT=https://hf-mirror.com
huggingface-cli download --resume-download InstantX/InstantID --local-dir checkpoints
```

For face encoder, you need to manually download via this [URL](https://github.com/deepinsight/insightface/issues/1896#issuecomment-1023867304) to `models/antelopev2` as the default link is invalid. Once you have prepared all models, the folder tree should be like:

```
  .
  ├── models
  ├── checkpoints
  ├── ip_adapter
  ├── pipeline_stable_diffusion_xl_instantid.py
  └── README.md
```

## Usage

```python
# !pip install opencv-python transformers accelerate insightface
import diffusers
from diffusers.utils import load_image
from diffusers.models import ControlNetModel

import cv2
import torch
import numpy as np
from PIL import Image

from insightface.app import FaceAnalysis
from pipeline_stable_diffusion_xl_instantid import StableDiffusionXLInstantIDPipeline, draw_kps

# prepare 'antelopev2' under ./models
app = FaceAnalysis(name='antelopev2', root='./', providers=['CUDAExecutionProvider', 'CPUExecutionProvider'])
app.prepare(ctx_id=0, det_size=(640, 640))

# prepare models under ./checkpoints
face_adapter = f'./checkpoints/ip-adapter.bin'
controlnet_path = f'./checkpoints/ControlNetModel'

# load IdentityNet
controlnet = ControlNetModel.from_pretrained(controlnet_path, torch_dtype=torch.float16)

base_model = 'wangqixun/YamerMIX_v8'  # from https://civitai.com/models/84040?modelVersionId=196039
pipe = StableDiffusionXLInstantIDPipeline.from_pretrained(
    base_model,
    controlnet=controlnet,
    torch_dtype=torch.float16
)
pipe.cuda()

# load adapter
pipe.load_ip_adapter_instantid(face_adapter)
```

Then, you can customized your own face images

```python
# load an image
face_image = load_image("./examples/yann-lecun_resize.jpg")

# prepare face emb
face_info = app.get(cv2.cvtColor(np.array(face_image), cv2.COLOR_RGB2BGR))
face_info = sorted(face_info, key=lambda x:(x['bbox'][2]-x['bbox'][0])*x['bbox'][3]-x['bbox'][1])[-1]  # only use the maximum face
face_emb = face_info['embedding']
face_kps = draw_kps(face_image, face_info['kps'])

# prompt
prompt = "film noir style, ink sketch|vector, male man, highly detailed, sharp focus, ultra sharpness, monochrome, high contrast, dramatic shadows, 1940s style, mysterious, cinematic"
negative_prompt = "ugly, deformed, noisy, blurry, low contrast, realism, photorealistic, vibrant, colorful"

# generate image
pipe.set_ip_adapter_scale(0.8)
image = pipe(
    prompt,
    image_embeds=face_emb,
    image=face_kps,
    controlnet_conditioning_scale=0.8,
).images[0]
```

## Start a local gradio demo
Run the following command:

```python
python gradio_demo/app.py
```

## Usage Tips
- For higher similarity, increase the weight of controlnet_conditioning_scale (IdentityNet) and ip_adapter_scale (Adapter).
- For over-saturation, decrease the ip_adapter_scale. If not work, decrease controlnet_conditioning_scale.
- For higher text control ability, decrease ip_adapter_scale.
- For specific styles, choose corresponding base model makes differences.


## Resources

### ComfyUI
1. [ZHO-ZHO-ZHO/ComfyUI-InstantID](https://github.com/ZHO-ZHO-ZHO/ComfyUI-InstantID)
2. [huxiuhan/ComfyUI-InstantID](https://github.com/huxiuhan/ComfyUI-InstantID)

## Acknowledgements
- Our work is highly inspired by [IP-Adapter](https://github.com/tencent-ailab/IP-Adapter) and [ControlNet](https://github.com/lllyasviel/ControlNet). Thanks for their great works!
- Thanks [ZHO-ZHO-ZHO](https://github.com/ZHO-ZHO-ZHO), [huxiuhan](https://github.com/huxiuhan) for supporting ComfyUI.
- Thanks to the HuggingFace team for their generous GPU support!

## Disclaimer
This project is released under [Apache License](https://github.com/InstantID/InstantID?tab=Apache-2.0-1-ov-file#readme) and aims to positively impact the field of AI-driven image generation. Users are granted the freedom to create images using this tool, but they are obligated to comply with local laws and utilize it responsibly. The developers will not assume any responsibility for potential misuse by users.

## Cite
If you find InstantID useful for your research and applications, please cite us using this BibTeX:

```bibtex
@article{wang2024instantid,
  title={InstantID: Zero-shot Identity-Preserving Generation in Seconds},
  author={Wang, Qixun and Bai, Xu and Wang, Haofan and Qin, Zekui and Chen, Anthony},
  journal={arXiv preprint arXiv:2401.07519},
  year={2024}
}
