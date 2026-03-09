[English](/README.md) | [Русский](/README_ru.md)

## 📚 **ИНСТРУКЦИЯ: Идеальный стек ComfyUI WAN 2.2 для RDNA2 AMD GPUs (на примере RX 6700XT)**
*Март 2026*

Эта инструкция создана на основе **реального боевого опыта** и гарантирует стабильную работу ComfyUI + WanVideo 2.2 + SageAttention на **AMD RX 6700 XT** с **ROCm 7.1.1**.

## **Предвартиельные требования**
Данный стек был протестирован на следующей системе:
1. ОС: Ubuntu 25.10
2. Ядро: x86_64 Linux 6.18.0-061800-generic **(ВАЖНО: обязательно используте ядро не ниже 6.18, иначе у вас будут проблемы со стабильностью)**
3. CPU: AMD Ryzen 9 5900X
4. GPU: AMD Radeon RX 6700 XT 12 Gb
5. RAM: 32 Gb
6. Установлен Python 3.12.11
7. Установлен пакет git (`sudo apt install git`)

---
## **Часть 1. Фундамент: система и ROCm**

### 1.1 Установка драйвера с ROCm 7.1.1
```bash
# Добавьте репозиторий AMD
wget https://repo.radeon.com/amdgpu-install/7.1.1/ubuntu/noble/amdgpu-install_7.1.1.70101-1_all.deb
sudo apt install ./amdgpu-install_7.1.1.70101-1_all.deb
sudo apt update

# Установите ROCm (полная версия)
sudo amdgpu-install --usecase=rocm --no-dkms
## Примечание: очень важно наличие флага --no-dkms, так как DKMS не нужен, и для ядра linux >= 6.18 не существует сборки DKMS

# Проверьте установку
/opt/rocm/bin/rocminfo | grep -E "Marketing Name|gfx"
```
**Ожидаемый вывод:** `AMD Radeon RX 6700 XT` и `gfx1031`

---
## **Часть 2. Установка базового ComfyUI с использованием виртуального окружения**

### 2.1 Скачивание ComfyUI и активация venv
```bash
#Клонируем репозиторий
git clone https://github.com/Comfy-Org/ComfyUI.git

#Переходим к папке с ComfyUI
cd ComfyUI

#Создаём вирутальное окружение Python в папке venv
python3 -m venv venv

#Активируем вирутальное окружение
source venv/bin/activate

#Обновляем pip
pip install --upgrade pip
```


### 2.2 Установка PyTorch 2.10 для ROCm 7.1
```bash
pip install --index-url https://download.pytorch.org/whl/rocm7.1 torch==2.10.0 torchvision==0.25.0 torchaudio==2.10.0
```
### 2.3 Установка остальных зависимостей

В скачанном ComfyUI файл `requirements.txt` содержит лишние для нас пакеты torch для карт NVIDIA, нужно предотвратить их установку 
```bash
sed -i '/^torch\b/s/^/#/' requirements.txt
sed -i '/^torchvision\b/s/^/#/' requirements.txt
sed -i '/^torchaudio\b/s/^/#/' requirements.txt
```
Или отктройте в любом текстовом редакторе `requirements.txt` и закоментруйте строки:
```
...
comfyui-embedded-docs
#torch --> комментируем
torchsde --> не комментируем
#torchvision --> комментируем
#torchaudio --> комментируем
numpy>=1.25.0
...
```
Запускаем установку зависимотей
```bash
pip install -r requirements.txt
```
### 2.4 Проверка
```bash
python -c "import torch; print(f'PyTorch: {torch.__version__}'); print(f'CUDA available: {torch.cuda.is_available()}'); print(f'GPU: {torch.cuda.get_device_name(0)}')"
```
**Ожидаемый вывод:**
```
PyTorch: 2.10.0+rocm7.1
CUDA available: True
GPU: AMD Radeon RX 6700 XT
```

---
## **Часть 3. Установка SageAttention**
### 3.1 Скачиваем пропатченную версию под ROCm (необходим Triton>=3.6.0, что уже шёл в комплекте с PyTorch 2.10+ROCm7.1):

```bash
pip install https://github.com/guinmoon/SageAttention-Rocm7/releases/download/v1.0.6_rocm7/sageattention-1.0.6-py3-none-any.whl
```
### 3.2 Проверка
```bash
python -c "import sageattention; print('✅ SageAttention работает')"
```

---
## **Часть 4. Создание файла запуска ComfyUI**

Деактивирируем виртуальное окружение и выйдем из папки ComfyUI
```bash
deactivate
cd ..
```
Создайте файл `run_comfyui.sh`:
```bash
cat > run_comfyui.sh << 'EOF'
#!/bin/bash

cd ~/ComfyUI # !!! Укажите правильный путь до папки ComfyUI
source venv/bin/activate

#AMD GPU setting
export HIP_VISIBLE_DEVICES=0
export HSA_OVERRIDE_GFX_VERSION=10.3.0 #  Для RDNA2 всегда должна быть 10.3.0, для RDNA3 11.0.0

# TunableOp (rocBLAS only, no hipBLASLt)
export PYTORCH_TUNABLEOP_ENABLED=1 # Требуеют выключения hipBLASt, так как он не доступен для 6700 XT
export PYTORCH_TUNABLEOP_HIPBLASLT_ENABLED=0 # Выключение hipBLASt, использование rocBLAS вместо hipBLASt

# Triton and Attention
export TORCH_ROCM_AOTRITON_ENABLE_EXPERIMENTAL=1
export FLASH_ATTENTION_TRITON_AMD_ENABLE="TRUE"

# MIOpen
export MIOPEN_FIND_MODE=FAST

# Memory optimization
export PYTORCH_ALLOC_CONF=expandable_segments:False,max_split_size_mb:256 #для эксперимента можете попробовать значение expandable_segments:True
export HIP_DISABLE_AUTO_MEM_ALLOC=0
export HIP_ENABLE_SANITIZER=0

echo "Using Python: $(which python)"
python main.py --reserve-vram 1.0 --use-sage-attention --bf16-vae
EOF

chmod +x run_comfyui.sh
```
### Запуск
Запустите скрипт
```
./run_comfyui.sh
```
Проверьте, что всё работает: [http://127.0.0.1:8188/](http://127.0.0.1:8188/)

---
## **Часть 5. Настройка ComfyUI под WAN 2.2**
### 5.1 Установка кастомных нод
Перейдите в папку с кастомными нодами
```bash
cd ComfyUI/custom_nodes
```
Установите ComfyUI-Manager
```bash
git clone https://github.com/ltdrdata/ComfyUI-Manager.git
```
Установите AMD GPU Monitor
```bash
git clone https://github.com/iDAPPA/ComfyUI-AMDGPUMonitor.git
```
Запустите ComfyUI
```bash
cd ../..
./run_comfyui.sh
```
Установите ноды для workflow через Manager

- ComfyUI-WanVideoWrapper
- rgthree's ComfyUI Nodes
- ComfyUI Easy Use
- KJNodes for ComfyUI
- Crystools
- ComfyUI-VideoHelperSuite
- ComfyUI-mxToolkit
- ComfyMath

*Примчечание: При установке других нод не из списка, некоторые из них будут требовать numpy<2, или torch без ROCm. В таких случаях, нужно понизить версию numpy `pip install --upgrade "numpy<2"`, или заново установить torch для ROCm (п. 2.2)*

### 5.2 Скачивание workflow для WAN 2.2
Скачайте `I2V WAN 2.2 14B SVI (KJ).json` из репозитория в папку `ComfyUI/user/default/workflows` или просто перетащите файл в окно с открытым интерфейсом ComfyUI
Если Вы сохранили файл в папку `ComfyUI/user/default/workflows`, workflow появится с списке рабочих процессов слева. Если же Вы перетащили файл в окно, не забудьте его сохранить (Ctrl+S)
### 5.3 Скачивание моделей
Откройте workflow, скачайте необходимые модели из списка:


#### 🎬 WAN:
- [(CivitAI) DaSiWa-WAN 2.2 I2V 14B SynthSeduction v9 | Lightspeed | GGUF - Q4 High](https://civitai.com/models/2269796?modelVersionId=2555194)
- [(CivitAI) DaSiWa-WAN 2.2 I2V 14B SynthSeduction v9 | Lightspeed | GGUF - Q4 Low](https://civitai.com/models/2269796?modelVersionId=2555463)
#### 🖼️ WAN Vae:
- [(Hugging Face) Comfy-Org/Wan_2.2_ComfyUI_Repackaged - wan_2.1_vae.safetensors](https://huggingface.co/Comfy-Org/Wan_2.2_ComfyUI_Repackaged/resolve/main/split_files/vae/wan_2.1_vae.safetensors)
#### 📝 T5 Text encoder
- [(Hugging Face) Kijai/WanVideo_comfy - umt5-xxl-enc-fp8_e4m3fn.safetensors](https://huggingface.co/Kijai/WanVideo_comfy/resolve/main/umt5-xxl-enc-fp8_e4m3fn.safetensors?download=true)
#### 👁️ Tae for WAN 2.1 (for previews, optional)
- [(Hugging Face) Kijai/WanVideo_comfy - taew2_1.safetensors](https://huggingface.co/Kijai/WanVideo_comfy/blob/main/taew2_1.safetensors)
#### ♾️ SVI 2.0 Pro LoRA
- [(Hugging Face) Kijai/WanVideo_comfy - SVI_v2_PRO_Wan2.2-I2V-A14B_HIGH_lora_rank_128_fp16.safetensors](https://huggingface.co/Kijai/WanVideo_comfy/resolve/main/LoRAs/Stable-Video-Infinity/v2.0/SVI_v2_PRO_Wan2.2-I2V-A14B_HIGH_lora_rank_128_fp16.safetensors?download=true)
- [(Hugging Face) Kijai/WanVideo_comfy - SVI_v2_PRO_Wan2.2-I2V-A14B_LOW_lora_rank_128_fp16.safetensors](https://huggingface.co/Kijai/WanVideo_comfy/resolve/main/LoRAs/Stable-Video-Infinity/v2.0/SVI_v2_PRO_Wan2.2-I2V-A14B_LOW_lora_rank_128_fp16.safetensors?download=true)

### 📁 Структура папок

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

После скачивания моделей, обновите вкладку ComfyUI.

**Теперь можно запускать workflow!**

---
## **Часть 6. Ключевые настройки workflow для 12GB**

Workflow `I2V WAN 2.2 14B SVI (KJ)` использует ноды от KJ WanVideoWrapper для генерации длинных видео из изображения (I2V). Видео генерируется по сегментам указанной длины (слайдер Seconds), потом сегменты собриаются в общий видео-файл. Количество Extra Segments можно увеличивать

### 6.1 Рекомендуемые стартовые параметры
| Параметр | Значение |
|----------|----------|
| `Resize resolution (by short edge)` | `480` - размер изображение будет изменён под 480p (с делителем 32)|
| `FPS` | `16` - количество кадров в секнду, лучше оставить на 16. Если захочется увеличить FPS, рекомендую использовать интерполяцию перед нодой VideoComine|
| `Seconds per segment` | `3` - колчество секунд на сегмент, 3 генерируется достаточно быстро, 4 с натяжкой, 5 уже долго|

### 6.2 Model loader & Prompt
| Параметр | Значение |
|----------|----------|
| `High model\Low model` | Подойдёт квантизированная Wan 2.2 I2V 14B GGUF модель (Q4_K_M - Q6_K_M) |
| `T5 CLIP Encoder` | Любой T5 энкодер, но не scaled версия!|
| `Swap blocks` | `25-35` (чем больше значение, тем больше ОЗУ будет задейстовано в загрузке моделей)|

### 6.3 First segment
| Параметр | Значение |
|----------|----------|
| `Enable RifleXRope x2` | `true` (будет использована интерполяция для более быстрой генерации кадров) |
| `start_image_crop_positon` | `center` (если важный кусок изображения отрезается, поменяйте этот параметр) |

### 6.4 Extra segment
| Параметр | Значение |
|----------|----------|
| `Enable RifleXRope x2` | `true` (будет использована интерполяция для более быстрой генерации кадров) |
| `overlap` | `5` - количество кадров для перекрытия между сегментами, обеспечивает плавность перехода между сегментами, но урезает общее количество кадров |

---
## **Часть 7. Диагностика и мониторинг**

### 7.1 Проверка TunableOp (должен быть файл)
```bash
cat ComfyUI/tunableop_results0.csv | head
```
Посчитать количество строк в файле tunableop_results0.csv, если число увеличивается, значит TunableOp активен.
```bash
#В отдельном окне
watch -n 1 wc -l ComfyUI/tunableop_results0.csv
```
### 7.2 Если OOM
```bash
# Добавьте в скрипт запуска
export PYTORCH_ALLOC_CONF=expandable_segments:True
# И увеличьте --reserve-vram до 1.5
```

---
## **🎉 ФИНАЛ**

Всё готово! Ваша видеокарта теперь способна на **генерацию видео с 14B моделью Wan2.2**.
