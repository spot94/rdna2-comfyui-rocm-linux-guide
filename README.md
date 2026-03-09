[English](/README.md) | [Русский](/README_ru.md)

## 📚 GUIDE: The Perfect ComfyUI WAN 2.2 Stack for RDNA2 AMD GPUs (using RX 6700XT as an example)

*March 2026*

This guide is based on real-world experience and guarantees stable operation of ComfyUI + WanVideo 2.2 + SageAttention on an AMD RX 6700 XT (gfx1031) with ROCm 7.1.1.

## Prerequisites

This stack has been tested on the following system:

- **OS:** Ubuntu 25.10
- **Kernel:** x86_64 Linux 6.18.0-061800-generic (**IMPORTANT:** You must use kernel version 6.18 or higher, otherwise you will encounter stability issues)
- **CPU:** AMD Ryzen 9 5900X
- **GPU:** AMD Radeon RX 6700 XT 12 Gb
- **RAM:** 32 Gb
- **Python 3.12.11** installed
- **git** package installed (`sudo apt install git`)

## Part 1. The Foundation: System and ROCm

### 1.1 Installing the Driver with ROCm 7.1.1
```bash
# Add the AMD repository
wget https://repo.radeon.com/amdgpu-install/7.1.1/ubuntu/noble/amdgpu-install_7.1.1.70101-1_all.deb
sudo apt install ./amdgpu-install_7.1.1.70101-1_all.deb
sudo apt update

# Install ROCm (full version)
sudo amdgpu-install --usecase=rocm --no-dkms
## Note: The --no-dkms flag is very important. DKMS is not needed, and there is no DKMS build available for kernel >= 6.18.

# Verify the installation
/opt/rocm/bin/rocminfo | grep -E "Marketing Name|gfx"
```
Expected output: `AMD Radeon RX 6700 XT` and `gfx1031`

## Part 2. Installing Base ComfyUI Using a Virtual Environment

### 2.1 Downloading ComfyUI and Activating venv
```bash
# Clone the repository
git clone https://github.com/Comfy-Org/ComfyUI.git

# Navigate to the ComfyUI folder
cd ComfyUI

# Create a Python virtual environment in the venv folder
python3 -m venv venv

# Activate the virtual environment
source venv/bin/activate

# Upgrade pip
pip install --upgrade pip
```

### 2.2 Installing PyTorch 2.10 for ROCm 7.1
```bash
pip install --index-url https://download.pytorch.org/whl/rocm7.1 torch==2.10.0 torchvision==0.25.0 torchaudio==2.10.0
```

### 2.3 Installing Other Dependencies

The downloaded ComfyUI `requirements.txt` file contains torch packages for NVIDIA cards that are unnecessary for us. We need to prevent their installation.
```bash
sed -i '/^torch\b/s/^/#/' requirements.txt
sed -i '/^torchvision\b/s/^/#/' requirements.txt
sed -i '/^torchaudio\b/s/^/#/' requirements.txt
```
Alternatively, open `requirements.txt` in any text editor and comment out the lines:
```
...
comfyui-embedded-docs
#torch --> comment this line
torchsde --> do not comment this
#torchvision --> comment this line
#torchaudio --> comment this line
numpy>=1.25.0
...
```

Run the dependency installation:
```bash
pip install -r requirements.txt
```

### 2.4 Verification
```bash
python -c "import torch; print(f'PyTorch: {torch.__version__}'); print(f'CUDA available: {torch.cuda.is_available()}'); print(f'GPU: {torch.cuda.get_device_name(0)}')"
```
Expected output:
```
PyTorch: 2.10.0+rocm7.1
CUDA available: True
GPU: AMD Radeon RX 6700 XT
```

## Part 3. Installing SageAttention

### 3.1 Download the version patched for ROCm (requires Triton>=3.6.0, which already came with PyTorch 2.10+ROCm7.1):
```bash
pip install https://github.com/guinmoon/SageAttention-Rocm7/releases/download/v1.0.6_rocm7/sageattention-1.0.6-py3-none-any.whl
```

### 3.2 Verification
```bash
python -c "import sageattention; print('✅ SageAttention is working')"
```

## Part 4. Creating the ComfyUI Launch Script

Deactivate the virtual environment and exit the ComfyUI folder:
```bash
deactivate
cd ..
```

Create the file `run_comfyui.sh`:
```bash
cat > run_comfyui.sh << 'EOF'
#!/bin/bash

cd ~/ComfyUI # !!! Specify the correct path to your ComfyUI folder
source venv/bin/activate

# AMD GPU settings
export HIP_VISIBLE_DEVICES=0
export HSA_OVERRIDE_GFX_VERSION=10.3.0 # For RDNA2, this must always be 10.3.0; for RDNA3 it's 11.0.0

# TunableOp (rocBLAS only, no hipBLASLt)
export PYTORCH_TUNABLEOP_ENABLED=1 # Requires disabling hipBLASLt, as it's not available for RX 6700 XT
export PYTORCH_TUNABLEOP_HIPBLASLT_ENABLED=0 # Disable hipBLASLt, use rocBLAS instead

# Triton and Attention
export TORCH_ROCM_AOTRITON_ENABLE_EXPERIMENTAL=1
export FLASH_ATTENTION_TRITON_AMD_ENABLE="TRUE"

# MIOpen
export MIOPEN_FIND_MODE=FAST

# Memory optimization
export PYTORCH_ALLOC_CONF=expandable_segments:False,max_split_size_mb:256 # You can experiment with expandable_segments:True
export HIP_DISABLE_AUTO_MEM_ALLOC=0
export HIP_ENABLE_SANITIZER=0

echo "Using Python: $(which python)"
python main.py --reserve-vram 1.0 --use-sage-attention --bf16-vae
EOF

chmod +x run_comfyui.sh
```

### Launch
Run the script:
```bash
./run_comfyui.sh
```

Check if everything is working: http://127.0.0.1:8188/

## Part 5. Configuring ComfyUI for WAN 2.2

### 5.1 Installing Custom Nodes

Navigate to the custom nodes folder:
```bash
cd ComfyUI/custom_nodes
```

Install ComfyUI-Manager:
```bash
git clone https://github.com/ltdrdata/ComfyUI-Manager.git
```

Install AMD GPU Monitor:
```bash
git clone https://github.com/iDAPPA/ComfyUI-AMDGPUMonitor.git
```

Launch ComfyUI:
```bash
cd ../..
./run_comfyui.sh
```

Install the workflow nodes via Manager:
*   ComfyUI-WanVideoWrapper
*   rgthree's ComfyUI Nodes
*   ComfyUI Easy Use
*   KJNodes for ComfyUI
*   Crystools
*   ComfyUI-VideoHelperSuite
*   ComfyUI-mxToolkit
*   ComfyMath

**Note:** When installing other nodes not on this list, some may require numpy<2, or torch without ROCm support. In such cases, you need to downgrade numpy (`pip install --upgrade "numpy<2"`) or reinstall torch for ROCm (see part 2.2).

### 5.2 Downloading the Workflow for WAN 2.2

Download the `I2V WAN 2.2 14B SVI (KJ Wrapper).json` file from the repository into the `ComfyUI/user/default/workflows` folder, or simply drag and drop the file into the ComfyUI browser window.

If you saved the file to the `ComfyUI/user/default/workflows` folder, the workflow will appear in the workflow list on the left. If you dragged the file into the window, don't forget to save it (Ctrl+S).

### 5.3 Downloading Models

Open the workflow and download the required models from the list:

#### 🎬 WAN (choose your quantization):
*   [DasiwaWAN22I2V14BSynthseduction_q4High.gguf](<Link Placeholder>)
*   [DasiwaWAN22I2V14BSynthseduction_q4Low.gguf](<Link Placeholder>)

#### 🖼️ WAN Vae:
*   [wan_2.1_vae.safetensors](<Link Placeholder>)

#### 📝 T5 Text encoder:
*   [umt5-xxl-enc-fp8_e4m3fn.safetensors](<Link Placeholder>)

#### 👁️ Tae for WAN 2.1 (for previews, optional):
*   [taew2_1.safetensors](<Link Placeholder>)

#### ♾️ SVI 2.0 Pro LoRA:
*   [SVI_v2_PRO_Wan2.2-I2V-A14B_HIGH_lora_rank_128_fp16.safetensors](<Link Placeholder>)
*   [SVI_v2_PRO_Wan2.2-I2V-A14B_LOW_lora_rank_128_fp16.safetensors](<Link Placeholder>)

### 📁 Folder Structure
```
/path_to_ComfyUI/
├── models/
│   ├── unet/
│   │   ├── DasiwaWAN22I2V14BSynthseduction_q4High.gguf
│   │   └── DasiwaWAN22I2V14BSynthseduction_q4Low.gguf
│   ├── text_encoders/
│   │   └── umt5-xxl-enc-fp8_e4m3fn.safetensors
│   ├── vae/
│   │   └── wan_2.1_vae.safetensors
│   ├── vae_approx/
│   │   └── taew2_1.safetensors
│   └── lora/
│       ├── SVI_v2_PRO_Wan2.2-I2V-A14B_HIGH_lora_rank_128_fp16.safetensors
│       └── SVI_v2_PRO_Wan2.2-I2V-A14B_LOW_lora_rank_128_fp16.safetensors
```

After downloading the models, refresh the ComfyUI tab in your browser.

Now you can run the workflow!

## Part 6. Key Workflow Settings for 12GB VRAM

The `I2V WAN 2.2 14B SVI (KJ)` workflow uses nodes from KJ WanVideoWrapper to generate long videos from an image (I2V). The video is generated segment by segment (according to the "Seconds" slider), and then the segments are combined into a final video file. The number of "Extra Segments" can be increased.

### 6.1 Recommended Starting Parameters

| Parameter                | Value | Description                                                                          |
| ------------------------ | ----- | ------------------------------------------------------------------------------------ |
| **Resize resolution (by short edge)** | 480   | The image will be resized to 480p (using a divisor of 32).                             |
| **FPS**                  | 16    | Frames per second. It's best to leave this at 16. To increase FPS, use interpolation before the `VideoCombine` node. |
| **Seconds per segment**  | 3     | Number of seconds per segment. 3 generates relatively quickly, 4 is manageable, 5 is slow. |

### 6.2 Model Loader & Prompt

| Parameter                | Value | Description                                                                                           |
| ------------------------ | ----- | ----------------------------------------------------------------------------------------------------- |
| **High model/Low model** |  DaSiWa-WAN 2.2 I2V 14B SynthSeduction v9 Q4 | Use a quantized Wan 2.2 I2V 14B GGUF model (Q4_K_M - Q6_K_M).                                         |
| **T5 CLIP Encoder**      | umt5-xxl-enc-fp8_e4m3fn | Any T5 encoder, but **not** the scaled version!                                                        |
| **Swap blocks**          | 25-35 | The higher the value, the more RAM will be used during model loading to potentially reduce loading time/swapping. |

### 6.3 First Segment

| Parameter                       | Value | Description                                                  |
| ------------------------------- | ----- | ------------------------------------------------------------ |
| **Enable RifleXRope x2**        | true  | Enables interpolation for faster frame generation.           |
| **start_image_crop_position**   | center | If an important part of the image is being cropped, change this parameter. |

### 6.4 Extra Segment

| Parameter                       | Value | Description                                                                                               |
| ------------------------------- | ----- | --------------------------------------------------------------------------------------------------------- |
| **Enable RifleXRope x2**        | true  | Enables interpolation for faster frame generation.                                                           |
| **overlap**                     | 5     | Number of overlapping frames between segments. Ensures smooth transitions but reduces the total frame count. |

## Part 7. Diagnostics and Monitoring

### 7.1 Checking TunableOp (the file should exist)
```bash
cat ComfyUI/tunableop_results0.csv | head
```
Count the number of lines in `tunableop_results0.csv`. If the number increases, TunableOp is active.
```bash
# In a separate terminal window
watch -n 1 wc -l ComfyUI/tunableop_results0.csv
```

### 7.2 If OOM (Out of Memory) Occurs
```bash
# Add this to your launch script
export PYTORCH_ALLOC_CONF=expandable_segments:True
# And increase --reserve-vram to 1.5
```

## 🎉 FINALE

Everything is ready! Your graphics card is now capable of generating video with the 14B Wan2.2 model.
