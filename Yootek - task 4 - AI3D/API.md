**1 - Export workflow ra Python bằng extention comfy-ui-to-python extension**

Script tự tìm `ComfyUI` path bằng cách dò thư mục cha, nhưng vì thư mục hiện tại tên `ComfyUI_hunyuan` (không phải `ComfyUI`) nên cần set `COMFYUI_PATH`. 

```bash
conda activate comfy_hunyuan
export COMFYUI_PATH=/home/yootek/projects/ComfyUI_hunyuan
cd custom_nodes/comfyui-to-python-extension
python -m comfyui_to_python --input_file file.json --output_file file.py \-- queue_size 1
```

**2. Viết class API wrapper** 

Input: File workflow.py sau khi workflow.json, có thể chạy end-to-end không giao diện.

Output 1: Đưa toàn bộ workflow vào một hàm, gói vào class cùng vài helper function, fix conflict (một vài node chạy được trên UI nhưng code không chạy được)

Output 2:  Đóng API, gọi đến method trong class vừa build.

Kiểm thử: Chạy test bằng API docs.

Các bước đã sửa:

1. Import `torch` sau `bootstrap_comfyui_runtime()`, chi tiết phụ lục **A.1**

2. Kế thừa:
- Thêm logger

- Thêm class OptimizedMemory (static method)

- Thêm helper functions trong class chính (export mesh, upload R2, get save dir).

- Thêm hàm resize GLB

- Thêm hàm dọn dẹp GPU (thay cho node Free-GPU chỉ chạy được trên UI). **A.2**

- Thêm thống kê thời gian

- Thêm hàm `load_image` (thay cho node Load-Image chỉ chạy được ảnh có sẵn, nhận input là tên file)
4. Tính image-size bằng image.shape[0], [1], [2] (thay cho node Get-Image-Size chỉ chạy được trên UI)

5. Dùng if-else để rẽ nhánh Trellis/Hunyuan thay vì dùng `ExecutionBlocker` (UI-only)

6. Thu hẹp khoảng random-seed của nhánh Trellis (1 - 2^31)

7. Thêm hàm `fix-invalid-normals` xóa face có diện tích bằng 0 và các đỉnh không còn dùng (khi export sẽ bị gán NaN gây lỗi không đọc được file)

8. Thêm hàm main để test với args truyền vào (test API docs)

9. Kế thừa
- Logic API



Skill:

- Cách dùng hàm main để test với args truyền vào, dựng API đơn giản để test

- Cách dùng logger

- Cách dùng static method

- Khuyến khích hiểu sơ bộ workflow rồi nhờ AI làm hoàn toàn, coi như là lời giải mẫu rồi kế thừa? Nhưng làm sao để nhớ khi ta không tự tay làm, như cái cách mà ta đi dạy toán?

- Làm sao để thiết kế agent tự chạy tự fix nhỉ

- Thêm logs ở tất cả các bước để đo thời gian



**3. Viết file Modal wrapper**

Debug `workflow.py` để import được trong worker Modal

- **`import_custom_nodes()`**: thêm `nest_asyncio.apply()` trước `asyncio.run(...)`. 
  
  - Lý do: worker Modal là hàm `async` (`async def run_3d_generation_task`), tức đã có một event loop đang chạy. `asyncio.run()` bên trong một event loop khác sẽ ném lỗi "cannot be called from a running event loop". `nest_asyncio` cho phép lồng event loop.

- **`bootstrap_comfyui_runtime()`**: bỏ `comfy.options.enable_args_parsing()`. 
  
  - Lý do: nếu bật, ComfyUI sẽ `argparse.parse_args()`  trên `sys.argv` thật của tiến trình Modal container — chứa các tham số không phải của ComfyUI → có thể crash hoặc nhận nhầm giá trị. Tắt đi thì dùng args mặc định (giống cách
    `image_2_3d.py` cũ vốn đã làm).

Debug `modal.py`**(Image pin torch = 2.7.1)**

- `GLIBCXX_3.4.31' not found` (thiếu trong `libstdc++.so.6`)
  
  - Wheel `cumesh` (trong `wheels/Linux/Torch270`) được build bằng GCC 13, cần
    `GLIBCXX_3.4.32`. Ubuntu 22.04 base image chỉ có tới `3.4.30`.  
  
  - Fix: thêm bước `add-apt-repository ppa:ubuntu-toolchain-r/test` rồi nâng
    `libstdc++6`, verify bằng `strings ... | grep GLIBCXX_3.4.32`.

- `undefined symbol: c10::SymBool::guard_or_false` khi import `o_voxel`
  
  - Sau bước trên, bộ wheel (`cumesh`, `flex_gemm`, `nvdiffrast`, `o_voxel`) import được (torch270). Kiểm tra bằng `strings` trên các `.so`:  `o_voxel` được build với torch ≥ 2.8 
  - Fix: `pip install . --no-build-isolation --no-deps`, build từ source thay vì dùng wheel sẵn.  Dùng source `https://github.com/visualbruno/TRELLIS.2`

Nguyên tắc khi build Modal Image

- Không goi `.add_local_dir(".", "/root", copy=True)`ngay đầu chuỗi build, trước cả `apt_install`, `pip install`, compile CUDA extension. Modal build image theo layer có cache tuần tự: sửa bất kỳ dòng code Python nào (dù  không liên quan gì đến pip) → layer đó đổi → mọi layer phía sau bị build lại từ đầu, bao gồm cả việc tải hàng chục GB model và compile CUDA extension mất nhiều phút.

- Đặt cái ít đổi nhất lên trên, cái đổi thường xuyên nhất (code Python) xuống cuối cùng, và dùng `copy=False` (mount runtime) cho code để sửa code không kích hoạt build lại bất kỳ layer nào.

- Model không "bake" vào image mà dùng Modal Volume,mount ở `/models`, rồi trong image tạo symlink `/root/models → /models`. Lý do
  
  - Image nhỏ, build nhanh`modal volume put`

Base commands

```python
# Login and Verify current user
modal token new --profile <your-modal-username>
modal profile current

#  Create and Upload to Modal Volumn
model volume create <modal-volume-name>
modal volume put --force <modal-volume-name> <local> <remote> --force
# ex: modal volume put hunyuan3d-models models/unet/flux-2-klein-9b-Q8_0.gguf /unet/flux-2-klein-9b-Q8_0.gguf

# Deploy
modal deploy file.py

# Immediately stop
modal container list
modal container stop <container-id>
```

## Appendix A

### A.1 - Lỗi torch bị import quá sớm

- Khi import `torch` đầu file, lúc được nạp torch nhận backend mặc định `native`

- Sau đó `bootstrap_comfyui_runtime()` gọi `import cuda_malloc`.File này đặt `PYTORCH_CUDA_ALLOC_CONF=backend:cudaMallocAsync.`

- Đến khi CUDA khởi tạo, torch đọc lại biến môi trường và thấy `cudaMallocAsync`, giá trị này khác với lúc nạp nên bị lỗi.

Cách sửa

- Import torch lại ngay sau `bootstrap_comfyui_runtime()`, để biến môi trường được đặt trước khi torch nạp.

- Hoặc đặt `export PYTORCH_CUDA_ALLOC_CONF=backend:native` trước khi chạy.

Lưu ý:

- `Trellis2LoadModel` tự đặt `os.environ["PYTORCH_CUDA_ALLOC_CONF"] = "expandable_segments:True"`. Đây cùng loại biến đã gây lỗi allocator lúc trước. Lúc node này chạy thì CUDA đã khởi tạo xong nên thường không sao.

### A.2 - Giải phóng Vram GPU

- Phần lớn các loader chỉ nạp model vào RAM, hoặc chưa nạp gì cả. Chỉ khi sử dụng, model mới được load từ RAM/SSD lên VRAM. 

| Loader                                    | RAM   | VRAM (nạp sẵn)              |
| ----------------------------------------- | ----- | --------------------------- |
| Trellis2LoadModel                         | Không | Không (với `low_vram=True`) |
| Hy3DModelLoader                           | Có    | Có (với chế độ `HIGH_VRAM`) |
| DownloadAndLoadHy3DPaintModel             | Có    | Không                       |
| UnetLoaderGGUF, CLIPLoaderGGUF, VAELoader | Có    | Không                       |
| TransparentBGSession+                     | ----  | Có                          |

- `unload`: chuyển các model do ComfyUI quản lý từ GPU về RAM, gồm Unet/CLIP/VAE-Loader
  
  - `gc.collect` giải phóng object không còn ai tham chiếu (cả VRAM và RAM)
  
  - `soft_empty_cache` trả VRAM lại cho hệ thống

```python
comfy.model_management.unload_all_models()
gc.collect()
model_management.soft_empty_cache()
```

- `unload` chỉ giải phóng VRAM, model được lưu ở RAM. Muốn giải phóng cả RAM, `del` mọi biến tham chiếu đến model =>  `gc.collect` => `soft_empty_cache`. 
  
  - Tuy nhiên với API chạy nhiều lần, nên chuyển về RAM để nhường VRAM cho bước sau. Tránh load lại model từ đĩa tốn thời gian.

- `torch.cuda.empty_cache()` và `torch.cuda.soft_empty_cache()`
  
  - `empty_cache`: dọn sạch VRAM, tốc độ chậm, dùng khi chuyển task hoàn toàn
  
  - `soft_empty_cache`: dọn từ từ, tốc độ nhanh, dùng trong vòng lặp train/infer

### A.3 - Static method

1. **Instance method (`self`)**

```python
class Foo:
    def __init__(self, name):
        self.name = name

    def instance_method(self):
        return f"Hello, {self.name}"
```

- Gọi qua instance, dùng khi hàm cần truy cập dữ liệu riêng của từng object (mỗi instance có `name` khác nhau). Gọi qua class thì phải tự truyền instance vào:

```python
f = Foo("An")
f.instance_method() 

Foo.instance_method(f)    
```

2. **Class method (`@classmethod`, `cls`)**
- Dùng phổ biến nhất cho **factory method** — hàm tạo ra instance theo cách khác:

```python
class Foo:
    def __init__(self, name):
        self.name = name

    @classmethod
    def class_method(cls, new_name):
        return cls(new_name)   # cls ở đây chính là Foo

f = Foo.class_method("An")
```

- Gọi qua class hoặc instance đều được, kết quả như nhau:

```python
Foo.class_method()       
f = Foo()
f.class_method()           
```

3. **Static method (`@staticmethod`)**

```python
class Foo:
    @staticmethod
    def static_method(x, y):
        return x + y
```

- Gọi qua class hoặc instance đều được (khuyến khích gọi qua class). Về bản chất là hàm thường, chỉ được "nhét" vào trong class cho gọn, để nhóm các hàm liên quan lại một chỗ (namespace)

```python
Foo.static_method(1, 2)    # → 3, cách khuyến khích
f = Foo()
f.static_method(1, 2)      # → 3, vẫn chạy được nhưng không cần tạo f làm gì
```
