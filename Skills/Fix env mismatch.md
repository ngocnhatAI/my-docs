## 1 - Nhận diện lỗi

Quy tắc: dòng cuối cùng mới là lỗi thật sự, ví dụ:

```
File ".../torch/library.py", line 496, in impl
    if torch._C._dispatch_has_kernel_for_dispatch_key(
RuntimeError: operator torchvision::nms does not exist
```

Dấu hiệu nhận biết:

- `does not exist.....` 

- `undefined symbol: .....` 

- `OSError: Could not load this library.....`

## 2. Quy trình chẩn đoán

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

## 3. Install package tương thích

### 3.1 - Với package thuộc Pytorch index

Nguồn đáng tin cậy nhất là index riêng của PyTorch

```bash
# Liệt kê torch version có sẵn cho 1 CUDA build cụ thể
pip index versions torch --index-url https://download.pytorch.org/whl/cu128Quan trọng: **luôn dùng cùng một `--index-url` (cùng cuXXX) cho cả 3 package trong 1 lệnh cài** để pip resolver tự chọn bản khớp nhau:
```

Dùng cùng một `--index-url` (cùng cuXXX) cho các package trong 1 lệnh cài để pip resolver tự chọn bản khớp nhau (cố định torch, các package còn lại tương thích theo)

```bash
pip install "torch==2.7.1" -U torchvision torchaudio xformers \
  --index-url https://download.pytorch.org/whl/cu128
```

### 3.2 - Với package không thuộc PyTorch index (flash_attn,...)

Đây là các package bên thứ 3, biên dịch sẵn riêng cho từng tổ hợp (python version + torch version + cuda version + cxx11abi).

Ví dụ (`python 3.12`, `torch 2.7`, `cu12`, `cxx11abiTRUE`) chính là "chìa khóa" để tìm đúng wheel. Các project như flash-attention đặt tên file wheel theo đúng format này, ví dụ:

```
flash_attn-2.8.3.post1+cu12torch2.7cxx11abiTRUE-cp312-cp312-linux_x86_64.whl
```

Đọc tên file: `cu12` (CUDA 12), `torch2.7`, `cxx11abiTRUE`, `cp312` (Python 3.12) — khớp cả 4 giá trị ở trên thì chắc chắn dùng được, không cần build từ source.

Cách tìm: vào trang **Releases** trên GitHub của package đó (không phải PyPI, vì các package native/CUDA nặng thường không upload hết lên PyPI do giới hạn dung lượng), tìm asset có tên khớp 4 tiêu chí.

**NOTE**: với trường hợp torch/torchaudio... đã cài xong xuôi, muốn cài thêm một package mới. Cứ cài như bình thường, xong dùng `pip show...` để kiểm tra torch có bị thay đổi không. 

Nếu torch bị thay đổi (gây lỗi), cài lại với constrant:

```bash
pip install some-new-package -c path-to-constrant-txt-file

# constraints.txt
torch==2.7.1
torchvision==0.22.1
torchaudio==2.7.1
```

## 4. Gỡ và cài lại — thứ tự an toàn

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

## Tóm tắt quy trình

1. **Đọc traceback từ dòng `Error` cuối lên** — phân loại: `does not exist`/`undefined symbol` = version/ABI mismatch.
2. **Tìm anchor version** — `pip show` các package native để đọc build tag (`+torch271`, `+pt27cu128`...) suy ra chuẩn ban đầu.

1. **Cài đồng bộ trong 1 lệnh** với cùng `--index-url`; luôn đọc cảnh báo `incompatible` cuối log để bắt domino tiếp theo (như xformers, cumesh...).
2. **Verify bằng import thực tế**, không chỉ tin version string — vì version đúng mà symbol vẫn có thể thiếu nếu package đó build riêng (flash_attn, cumesh) thì cần khớp thêm `cxx11abi` + `cp3XX` khi tìm wheel bên thứ 3.
