## Bối cảnh

- Hiện chỉ có UnetLoaderGGUF và CLIPLoaderGGUF (text)

- Cần viết thêm VAELoaderGGUF, CLIPVisionLoaderGGUF

## Hướng tiếp cận

### 1 - UnetLoaderGGUF

`gguf.GGUFReader(path)` parse file `.gguf`, trả về object `reader` với properties: 

- `reader.tensors` — list các tensor info trong file (mỗi tensor có `.name`, `.shape`, `.data` (numpy array, mmap), `.tensor_type` là kiểu quant như F32/F16/Q4_K...).

- `reader.fields` — dict tên các metadata field (kiến trúc, tokenizer config, v.v.).

- `reader.get_field(name)` — lấy 1 metadata field cụ thể, trả về object có `.types` (kiểu GGUF value) và `.parts`/`.data` (dữ liệu thô, phải tự decode).

`mmap` là cơ chế read-only (non-writable). 

- Thay vì đọc file vào RAM bằng `read()`, `mmap` "trỏ" vào file chứ chưa copy vào RAM. Khi code truy cập, OS mới thực sự nạp phần dữ liệu ấy.

Khi quantize, dữ liệu được lưu theo block-quantization, mỗi "block" gồm nhiều giá trị được nén chung, nên buffer thô không cùng `shape` với tensor gốc. 

- Vì vậy cần lưu shape gốc để sau `dequantize` reshape về shape gốc.
