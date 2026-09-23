**1 - Export workflow ra Python bằng extention comfy-ui-to-python extension**

Script tự tìm `ComfyUI` path bằng cách dò thư mục cha, nhưng vì thư mục hiện tại tên `ComfyUI_hunyuan` (không phải `ComfyUI`) nên cần set `COMFYUI_PATH`. 

```bash
conda activate comfy_hunyuan
export COMFYUI_PATH=/home/yootek/projects/ComfyUI_hunyuan
cd custom_nodes/comfyui-to-python-extension
python -m comfyui_to_python --input_file file.json --output_file file.py \-- queue_size 1
```



**2. Viết class API wrapper** 

Xóa  `build_workflow()` và các biến liên quan (không sử dụng)

Thêm hàm dọn dẹp GPU: 

- <mark>Chưa thực sự giải phóng vram model đã load</mark>

Static method: hay

Thêm Class Optimized Memory, các hàm get_save_folder... trong class chính (giữ nguyên logic)

Cách dùng logger

Cách upload to cloudflare

Cách dùng time đo thời gian

Thêm hàm reszie và tự export, upload lên clourflare.

Lỗi torch đặt sớm quá, bị mis-match backend

chỗ gpu đang đặt 9998, sẽ lỗi nhưng k dừng pipeline (vì khi chạy file.py này mình không mở node nào cả)

Thay FreeGPU bằng unload_model (từ vram xuống ram)

==

Wrap API vào luôn trong file

Có minitest cuối luôn, quá yêu

Có lỗi nên phải dùng hàm load image riêng

Dùng hàm getsize riêng (vì lỗi) 

Blocker chỉ chạy trong comfy, code bị lỗi



Đóng gói thành 1 class có:

- `__init__` nhận `args` (device, cache_path, cấu hình storage/cloud)
- 1+ method xử lý chính, input là bytes ảnh/dữ liệu, output là dict `{success, message, ...path/url, stats}`
- Helper: `gen_save_folder()`, `export_mesh()`/export kết quả, upload lên cloud storage (R2/S3/MinIO) nếu cần trả URL công khai

**3. Viết `hunyuan_api.py` (phần đang thiếu)** File `hunyuan_modal.py` import `create_app(task_store, run_3d_generation_task)` từ đây nhưng bạn chưa có file này. Cần viết Flask/FastAPI app với ít nhất:

- Endpoint POST nhận ảnh/input, tạo `task_id`, gọi `.spawn()` vào Modal function chạy nền, lưu status ban đầu vào `task_store`
- Endpoint GET `/status/{task_id}` đọc từ `task_store`
- Endpoint GET `/download/{...}` trả file kết quả (nếu không dùng cloud URL)

**4. Viết file Modal wrapper (như `hunyuan_modal.py`)**

- Đổi tên App, `task_store` Dict name
- Sửa Docker image: `apt_install` các lib hệ thống cần cho pipeline của bạn, `run_commands` cài đúng requirements, tải đúng model weights bạn dùng (thay URL HuggingFace)
- Sửa `get_hunyuan_api()` để import class ở bước 2 thay vì `OptimizedHunyuan3DAPI`
- Sửa `run_3d_generation_task` để gọi đúng method(s) của bạn thay vì `image_2_3d`/`mv_image_2_3d`
- Cấu hình GPU (`gpu="L40S"` hay loại khác), `timeout`, `memory`, `volumes` theo nhu cầu

**5. Cấu hình secrets/env** Modal cần các biến môi trường (CF_ENDPOINT, CF_ACCESS_KEY...) — thiết lập qua `modal secret create` hoặc `Secret.from_name(...)` thay vì chỉ đọc `.env` (file `.env` không tồn tại trong container trừ khi bạn add vào image hoặc dùng Modal Secret — đây là điểm cần chú ý vì cách hiện tại dùng `load_dotenv()` sau khi `add_local_dir` copy `.env` vào, khá rủi ro nếu `.env` chứa secret thật).

**6. Deploy và test**

```bash
modal deploy hunyuan_modal.py
```

Sau đó test endpoint qua `curl`/Postman: POST ảnh → nhận `task_id` → poll status → lấy URL/file kết quả.
