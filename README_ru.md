## 📚 **ФИНАЛЬНАЯ ИНСТРУКЦИЯ: Идеальный стек для RDNA AMD GPUs (на примере RX 6700XT)**
*Март 2026*

Эта инструкция создана на основе **реального боевого опыта** и гарантирует стабильную работу ComfyUI + WanVideo + SageAttention на **AMD RX 6700 XT** с **ROCm 7.1.1**.

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
```
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
```
sed -i '/^torch\b/s/^/#/' requirements.txt
sed -i '/^torchvision\b/s/^/#/' requirements.txt
sed -i '/^torchaudio\b/s/^/#/' requirements.txt
sed -i '/^torchsde\b/s/^/#/' requirements.txt

```
Или отктройте в любом текстовом редакторе `requirements.txt` и закоментруйте строки:
```
...
comfyui-embedded-docs
#torch --> комментируем
#torchsde --> комментируем
#torchvision --> комментируем
#torchaudio --> комментируем
numpy>=1.25.0
...
```
Запускаем установку зависимотей
```
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
```
deactivate
cd ..
```
Создайте файл `run_comfyui.sh`:
```bash
cat > run_comfyui.sh << 'EOF'
#!/bin/bash

cd ~/ComfyUI # !!! Укажите правильный путь до папки ComfyUI
source venv/bin/activate

#ROCm paths
export ROCM_PATH=/opt/rocm
export HIP_PATH=$ROCM_PATH
export PATH=$ROCM_PATH/bin:$ROCM_PATH/llvm/bin:$PATH

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
Проверьте, что всё работает

---
