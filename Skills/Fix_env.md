## ERROR 1 - ENV mismatch

Dấu hiệu nhận biết:

- `torchaudio does not exist.....` 

- `undefined symbol: .....` 

- `OSError: Could not load this library.....`

### 1 - Chẩn đoán

1. Xác định mỏ neo: CUDA driver của GPU, torch version
2. Kiểm tra version drift (lệch phiên bản) giữa torch và các package phụ thuộc vào nó 

```bash
# Bước 0: Xác định CUDA driver
nvidia-smi

# Bước 1: xác định  torch version + CUDA build của nó
python -c "import torch; print(torch.__version__, torch.version.cuda)"

# Bước 2: liệt kê MỌI package có native extension (biên dịch C++/CUDA)
pip list | grep -iE "torch|cuda|flash|cumesh|xformers|rasterizer"
pip list | grep -iE "^torch" (only start with torch)
```

- i (ignore): không phân biệt chữ hoa chữ thường

- E (extended regex): cho phép dùng ký tự đặc biệt như `|` (OR) để mở rộng

### 2 - Install package tương thích

### a - Với package thuộc Pytorch index

Nguồn đáng tin cậy nhất là index riêng của PyTorch

```bash
# Liệt kê torch version có sẵn cho 1 CUDA build cụ thể
pip index versions torch --index-url https://download.pytorch.org/whl/cu128
```

Dùng cùng một `--index-url` (cùng cuXXX) cho các package trong 1 lệnh cài để pip resolver tự chọn bản khớp nhau (cố định torch, các package còn lại tương thích theo)

```bash
pip install "torch==2.7.1" -U torchvision torchaudio xformers \
  --index-url https://download.pytorch.org/whl/cu128
```

### b - Với package không thuộc PyTorch index (flash_attn,...)

Là package bên thứ 3, download từ trang Releases trên Github của package đó,  được biên dịch sẵn cho từng tổ hợp (python version + torch version + cuda version). Ví dụ: 

```
flash_attn-2.8.3.post1+cu12torch2.7cxx11abiTRUE-cp312-cp312-linux_x86_64.whl
```

Đọc tên file: `cu12` (CUDA 12), `torch2.7`,  `cp312` (Python 3.12) — khớp cả 3 giá trị ở trên thì chắc chắn dùng được, không cần build từ source.

**NOTE**: với trường hợp torch/torchaudio... đã cài xong xuôi, muốn cài thêm một package mới. Cứ cài như bình thường, xong dùng `pip show...` để kiểm tra torch có bị thay đổi không. 

Nếu torch bị thay đổi (gây lỗi), cài lại với constrant:

```bash
pip install some-new-package -c path-to-constrant-txt-file

# constraints.txt
torch==2.7.1
torchvision==0.22.1
torchaudio==2.7.1
```

## 3. Gỡ và cài lại — thứ tự an toàn

```bash
# 1. Luôn xem trước cái gì đang cài, version nào
pip show <pkg>

# 2. Cài đè trực tiếp (pip tự uninstall bản cũ) — KHÔNG cần uninstall thủ công trước
pip install "torch==2.7.1" --index-url https://download.pytorch.org/whl/cu128
```

Sau khi cài xong, luôn có bước verify riêng — đừng chỉ tin "Successfully installed":

```bash
# verify version đúng, or pip show.....
python -c "import torch, torchvision, torchaudio; print(torch.__version__, torchvision.__version__, torchaudio.__version__)"

# verify import KHÔNG lỗi (quan trọng hơn version, vì version đúng mà vẫn có thể lỗi symbol)
python -c "from torchvision.ops import nms; print('ok')"
python -c "import cumesh; print('ok')"
```



## ERROR 2 - Thiếu CUDA Toolkit headers

```bash
fatal error: cusparse.h: No such file or directory
```

Nguyên nhân: CUDA Toolkit thiếu header `.h` để complie extension C++/CUDA

**Bước 1: Kiểm tra xem header có tồn tại ở đâu đó không**

```bash
find / -name "cusparse.h" 2>/dev/null
```

**Bước 2a: Nếu tìm thấy, ví dụ**

```
/usr/local/lib/python3.12/dist-packages/nvidia/cusparse/include/cusparse.hKiểm tra:
```

Nếu có, thêm đường dẫn đó vào `CPATH`  trước khi build:

```bash
# Gom include path của tất cả package nvidia-cu12 đã cài qua pip
export CPATH=$(python3 -c "
import glob, os
paths = glob.glob('/usr/local/lib/python3.12/dist-packages/nvidia/*/include')
print(':'.join(paths))
")
echo "CPATH=$CPATH"

# Build lại
/workspace/runpod-slim/ComfyUI/.venv-cu128/bin/python -m pip install . --no-build-isolation

```

**Bước 2b: Nếu không tìm thấy ở đâu cả**

Thiếu CUDA Toolkit header, cần cài bổ sung CUDA Toolkit, ví dụ

```bash
apt-get install cuda-toolkit-12-8
```
