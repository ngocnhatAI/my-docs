# Deploy Hunyuan3D / Trellis2 workflow lên Modal.com — Nhật ký & tri thức

File này tổng hợp toàn bộ quá trình tìm hiểu, quyết định và sửa lỗi khi đưa
workflow ComfyUI (`z_image_2_3d_combine_api.py`) lên chạy trên Modal.com qua
[zhunyuan_modal.py](zhunyuan_modal.py). Mục đích: người đọc sau (kể cả người
không tham gia đoạn chat gốc) hiểu được **vì sao** code hiện tại trông như vậy,
không chỉ đọc code.

---

## 1. Bối cảnh ban đầu: `zhunyuan_modal.py` làm gì

File gốc đóng gói **toàn bộ thư mục ComfyUI local** (bao gồm `custom_nodes/`)
thành một Modal App gồm 2 Function:

- **`run` (web, không GPU)**: FastAPI app, nhận ảnh qua HTTP, lưu tạm, rồi
  `spawn()` một task chạy nền chứ không tự sinh 3D.
- **`run_3d_generation_task` (worker, GPU L40S)**: chạy thật workflow ComfyUI
  bằng cách import module Python (dạng "export ComfyUI workflow ra script
  Python" của `comfyui-to-python-extension`), gọi thẳng các node qua
  `NODE_CLASS_MAPPINGS["TênNode"]()` — không cần UI ComfyUI, không cần
  PromptServer thật.

Cơ chế cốt lõi: mọi custom node ComfyUI đều được nạp bằng
`nodes.init_extra_nodes()` (giả lập một `PromptServer` tối thiểu), sau đó
gọi trực tiếp class node như gọi hàm Python bình thường. Đây là lý do vì sao
"file .py xuất ra từ workflow" chạy độc lập được, không cần ComfyUI server.

**Kết luận ban đầu quan trọng:** hai `@app.function` (`run` và
`run_3d_generation_task`) **không hàm nào thừa** — chúng đóng vai trò
client-facing (nhẹ, luôn sẵn sàng) và heavy-compute (GPU, chỉ bật khi có
việc). Tách ra để không phải giữ GPU sống chờ HTTP.

---

## 2. Quyết định chuyển sang workflow mới

User có sẵn workflow mới `z_image_2_3d_combine_api.py` (khác với
`image_2_3d.py` cũ), pipeline phức tạp hơn nhiều:

```
Flux2-Klein (GGUF) chỉnh sửa/tăng cường ảnh đầu vào
        │
        ▼
Qwen3-VL (AILab_QwenVL) phân loại vật thể trong ảnh → "trellis" | "hunyuan"
        │
        ├── nhánh "trellis" → Trellis2 (microsoft/visualbruno fork) sinh mesh + texture
        │
        └── nhánh "hunyuan" → Hunyuan3DWrapper (mesh) + paint model (texture multiview)
        │
        ▼
Export GLB, scale, upload lên Cloudflare R2 → trả `modelUrl`
```

**Quyết định đã chốt qua AskUserQuestion:**

1. Chạy trên Modal (không chạy local-only).
2. Chỉ hỗ trợ **single image** — bỏ nhánh multi-view cũ
   (`mv_image_2_3d`). Nếu client gửi nhiều ảnh, ưu tiên lấy ảnh `front`,
   không có thì lấy ảnh đầu tiên trong dict.

**Nguyên tắc sửa code xuyên suốt (theo yêu cầu user, đã lưu ở
[keep-old-lines-commented.md](/home/yootek/.claude/projects/-home-yootek-projects-ComfyUI-hunyuan/memory/keep-old-lines-commented.md)):**
mọi dòng bị sửa/xoá đều giữ lại dạng comment ngay phía trên/dưới, không xoá
hẳn. Áp dụng nhất quán trong toàn bộ quá trình.

---

## 3. Sửa `z_image_2_3d_combine_api.py` để import được trong worker Modal

Workflow này vốn được thiết kế để **chạy độc lập** (`python
z_image_2_3d_combine_api.py` tự mở server uvicorn ở cổng 8887). Để dùng làm
module import trong worker Modal (`from z_image_2_3d_combine_api import
OptimizedImageTo3DAPI`), phải sửa 3 chỗ:

1. **`import_custom_nodes()`**: thêm `nest_asyncio.apply()` trước
   `asyncio.run(init_extra_nodes())`. Lý do: worker Modal là hàm `async`
   (`async def run_3d_generation_task`), tức đã có một event loop đang chạy.
   `asyncio.run()` bên trong một event loop khác sẽ ném lỗi "cannot be
   called from a running event loop". `nest_asyncio` cho phép lồng event
   loop.
2. **`bootstrap_comfyui_runtime()`**: comment dòng
   `comfy.options.enable_args_parsing()`. Lý do: nếu bật, ComfyUI sẽ
   `argparse.parse_args()` trên `sys.argv` thật của tiến trình Modal
   container — chứa các tham số không phải của ComfyUI → có thể crash hoặc
   nhận nhầm giá trị. Tắt đi thì dùng args mặc định (giống cách
   `image_2_3d.py` cũ vốn đã làm).
3. **Đoạn cuối file** (`_hunyuan_api = ...; app = create_app(...);
   uvicorn.run(...)`): bọc vào `if __name__ == "__main__":`. Lý do: nếu để
   ở top-level, **chỉ import module thôi** cũng khởi động ngay một server
   uvicorn và treo tiến trình — phá hỏng worker Modal (worker không cần
   server nội bộ, nó là một Function được gọi trực tiếp). Sau khi bọc,
   chạy trực tiếp file (`python z_image_2_3d_combine_api.py`) vẫn hoạt động
   y như cũ để test local.

---

## 4. Thiết kế lại Modal Image — tại sao phải suy nghĩ về thứ tự layer

### 4.1. Vấn đề với cấu trúc cũ

Bản gốc gọi `.add_local_dir(".", "/root", copy=True)` **ngay ở đầu** chuỗi
build, trước cả `apt_install`, `pip install`, compile CUDA extension. Modal
build image theo layer có cache tuần tự: sửa bất kỳ dòng code Python nào
(kể cả một file không liên quan gì đến pip) → layer đó đổi → **mọi layer
phía sau bị build lại từ đầu**, bao gồm cả việc tải hàng chục GB model và
compile CUDA extension mất nhiều phút.

### 4.2. Nguyên tắc sắp xếp lại (áp dụng nhất quán)

> Đặt cái **ít đổi nhất** lên trên, cái **đổi thường xuyên nhất** (code
> Python) xuống **cuối cùng**, và dùng `copy=False` (mount runtime) cho
> code để sửa code không kích hoạt build lại bất kỳ layer nào.

Thứ tự cuối cùng trong `zhunyuan_modal.py`:

1. `apt_install` (build tools, OpenGL/Mesa) — gần như không đổi.
2. `.env(...)` — biến môi trường CUDA_HOME, TORCH_CUDA_ARCH_LIST, COMFYUI_PATH.
3. `mkdir` thư mục input/output/temp + `ln -s /models /root/models` (xem mục 5).
4. Copy **chỉ các file `requirements*.txt`** (không phải cả thư mục) rồi
   pip install torch pin cứng + mọi requirements của các custom node cần
   dùng, luôn kèm `-c constrant.txt` để khoá version torch.
5. Copy source 2 extension CUDA/C++ của Hunyuan3DWrapper
   (`custom_rasterizer`, `differentiable_renderer`) rồi compile tại build
   time.
6. Copy wheel sẵn của Trellis2 (`wheels/Linux/Torch270`), cài `--no-deps`.
7. Nâng `libstdc++6`, build `o_voxel` từ source, pin `onnxruntime-gpu`,
   pin `rembg` (xem mục 6 — các bước vá lỗi, cố tình đặt **gần cuối** để
   không phải build lại các bước pip nặng phía trên khi phải vá thêm).
8. **Cuối cùng**: `.add_local_dir(".", "/root", ignore=[...], copy=False)`
   — mount code, không bao giờ "bake" vào image. Ignore các thư mục dữ liệu
   nặng/sinh runtime (`models/`, `input/`, `output/`, `cache/`, `user/`,
   `.git/`, `__pycache__/`) và ignore luôn 2 thư mục extension đã compile ở
   bước 5 (để bản `.so` build sẵn ở máy local không đè lên bản compile
   trong image), cũng như các custom node không dùng đến trong workflow
   mới (comfyui-manager, ComfyUI-UltraShape1, UltimateSDUpscale,
   comfyui-to-python-extension, was-node-suite-comfyui, bản
   Hunyuan3d-2-1.disabled) — giảm cold-start-node-scan và giảm khả năng
   "IMPORT FAILED" gây nhiễu log.

**Hệ quả thực tế đã quan sát:** sau khi cấu trúc lại, một lần deploy chỉ
sửa Python code mất **16–27 giây** (build cache toàn bộ, chỉ mount lại
code), so với build đầy đủ ban đầu tốn nhiều phút.

---

## 5. Model không "bake" vào image mà dùng Modal Volume

**Quyết định:** models (~42GB) không tải bằng `wget`/`snapshot_download`
trong lúc build image (cách cũ), mà đẩy vào một **Modal Volume** riêng
(`hunyuan3d-models`), mount ở `/models`, rồi trong image tạo symlink
`/root/models → /models`.

**Lý do:**

- Image nhỏ, build nhanh, không phải build lại khi đổi model.
- Volume có thể update qua `modal volume put` mà không cần deploy lại.
- Các model mà node tự tải lúc runtime (rembg, delight/paint model...)
  cũng nên rơi vào Volume để cache được giữ qua các lần cold start, thay vì
  container mới tải lại từ đầu mỗi lần. → đã set `HF_HOME=/models/.hf_cache`
  và `U2NET_HOME=/models/.u2net` trong image.

**10 model bắt buộc** (danh sách final, dùng để kiểm tra tồn tại trước khi
chạy — xem mục 8):

```
unet/flux-2-klein-9b-Q8_0.gguf
text_encoders/Qwen3-8B-Q8_0.gguf
vae/flux2-klein-9b-vae.safetensors
diffusion_models/hunyuan3d-dit-v2-mini-fast/model.fp16.safetensors
diffusers/hunyuan3d-paint-v2-0
visualbruno/TRELLIS.2-4B-FP8/pipeline_fp8.json
facebook/dinov3-vitl16-pretrain-lvd1689m/model.safetensors
microsoft/TRELLIS-image-large/ckpts/ss_dec_conv3d_16l8_fp16.safetensors
microsoft/TRELLIS.2-4B/reconviagen_pipeline.json
LLM/Qwen-VL/Qwen3-VL-4B-Instruct-FP8
```

Lệnh upload dùng `modal volume put --force hunyuan3d-models <local> <remote>`.
`--force` quan trọng vì Volume có thể đã chứa bản tải dở do node tự
`snapshot_download` khi chạy lỗi giữa chừng.

**Lưu ý đặc biệt:** thư mục `microsoft/TRELLIS.2-4B/` (chứa
`reconviagen_pipeline.json`) **bắt buộc phải tồn tại trước**, vì node
`Trellis2LoadModel` (`ComfyUI-Trellis2/nodes.py`) gọi
`shutil.copyfile()` vào đường dẫn này và sẽ ném `FileNotFoundError` nếu
thư mục cha chưa có — đây chính là một trong các lỗi thực tế gặp phải (xem
mục 8, vòng lặp lỗi #3).

---

## 6. Chuỗi lỗi thực tế gặp phải khi build/chạy — nguyên nhân & cách vá

Đây là phần quan trọng nhất để hiểu "tại sao code trông như vậy": mỗi lần
deploy phát hiện một lỗi mới, phải đọc log (`modal app logs`), xác định
nguyên nhân gốc, rồi vá **ở đúng vị trí trong image để tận dụng cache**.

### Lỗi #1 — `KeyError: 'was-ns'` / thư mục không tồn tại

Code cũ tham chiếu `custom_nodes/was-ns`, nhưng thư mục thực tế trên máy đã
đổi tên thành `custom_nodes/was-node-suite-comfyui` (was-ns đã bị xoá theo
git status). → Xoá lệnh `cd .../was-ns && pip install...` khỏi image.

### Lỗi #2 — `add_local_dir(".")` copy luôn 171GB `models/`

Modal không tự đọc `.gitignore`. Phải truyền `ignore=[...]` tường minh khi
gọi `add_local_dir`. Đã liệt kê rõ trong mục 4.2.

### Lỗi #3 (vòng đầu) — `KeyError: 'Trellis2LoadModel'`

Triệu chứng bề mặt: code gọi `NODE_CLASS_MAPPINGS["Trellis2LoadModel"]`
nhưng key không tồn tại. **Đây không phải lỗi gốc** — chỉ là hệ quả của
việc node Trellis2 **import thất bại** (ComfyUI nuốt lỗi import, chỉ log
"IMPORT FAILED" rồi chạy tiếp, khiến lỗi thật bị che mất). Phải đọc log
worker (`modal app logs`) mới thấy traceback thật.

**Cách chẩn đoán chuẩn được rút ra:** mọi `KeyError` về tên node đều là
dấu hiệu "đi tìm dòng IMPORT FAILED trong log", không phải lỗi thiếu đăng
ký node.

Bên dưới `KeyError` này thực chất là **3 lỗi con liên tiếp**, mỗi lỗi lộ ra
sau khi lỗi trước được vá:

**#3a — `GLIBCXX_3.4.31' not found` (thiếu trong `libstdc++.so.6`)**
Wheel `cumesh` (trong `wheels/Linux/Torch270`) được build bằng GCC 13, cần
`GLIBCXX_3.4.32`. Ubuntu 22.04 base image chỉ có tới `3.4.30`. Máy local
không gặp vì env conda có `libstdc++` riêng, mới hơn bản hệ thống.
→ Vá: thêm bước `add-apt-repository ppa:ubuntu-toolchain-r/test` rồi nâng
`libstdc++6`, verify bằng `strings ... | grep GLIBCXX_3.4.32`.

**#3b — `undefined symbol: c10::SymBool::guard_or_false` khi import
`o_voxel`**
Sau khi #3a được vá, `cumesh` import được nhưng `o_voxel` (cùng bộ wheel
Torch270) lại lỗi symbol không tồn tại. Kiểm tra bằng `strings` trên các
`.so`: chỉ riêng `o_voxel` trong bộ wheel này được build với **torch ≥
2.8**, trong khi image pin torch 2.7.1 (giống local). 3 wheel còn lại
(`cumesh`, `flex_gemm`, `nvdiffrast`) không có symbol này nên chạy được.
Máy local không dùng bản wheel `o_voxel` này — nó build `o_voxel` **từ
source** của repo TRELLIS.2.
→ Vá: clone source, build từ source thay vì dùng wheel sẵn, `pip install .
--no-build-isolation --no-deps` (đặt `--no-deps` vì `o-voxel/pyproject.toml`
khai `cumesh`/`flex_gemm` phụ thuộc bản `@ git+...` — nếu không có
`--no-deps` thì pip báo xung đột version với 2 wheel đã cài ở bước trước).

Thử lần đầu dùng source **microsoft/TRELLIS.2** (`git checkout
75fbf018...`) → build được, nhưng khi chạy thật lộ ra lỗi tiếp theo (#3c).

**#3c — `ImportError: cannot import name 'tiled_flexible_dual_grid_to_mesh'
from 'o_voxel.convert'`**
Bản `microsoft/TRELLIS.2` gốc **không có** hàm này —
`ComfyUI-Trellis2/trellis2/models/sc_vaes/fdg_vae.py` cần nó. README của
node `ComfyUI-Trellis2` có ghi rõ: dùng fork riêng
**`visualbruno/TRELLIS.2`** cho `o_voxel`/`cumesh`/`flex_gemm`. Đã verify
bằng cách so sánh nội dung file `flexible_dual_grid.py` giữa fork
visualbruno và bản đang cài ở env local (`comfy_hunyuan`) — **giống hệt
nhau**, xác nhận đây đúng là source mà local đang dùng.
→ Vá cuối cùng: đổi remote clone sang
`https://github.com/visualbruno/TRELLIS.2.git`, pin ở commit
`65d1e13b4a92296036044df0633242bb9e95abf6`.

### Lỗi #4 — Volume model trống → `FileNotFoundError`

Sau khi Trellis2 import thành công, lỗi tiếp theo là thiếu file thật:
`/root/models/microsoft/TRELLIS.2-4B/reconviagen_pipeline.json` không tồn
tại — vì lúc đó **chưa từng chạy** `modal volume put` để đẩy model lên
Volume (kế hoạch có ghi nhưng bước thực thi bị bỏ sót). → Chạy upload 10
model (mục 5), verify bằng `modal volume ls` đối chiếu dung lượng file với
bản local (ví dụ `flux-2-klein-9b-Q8_0.gguf` = 9.3 GiB khớp cả hai bên).

### Lỗi #5 — `onnxruntime` yêu cầu `libcublasLt.so.13` (CUDA 13)

Không làm task fail (chỉ log warning), nhưng khiến rembg (tách nền ảnh)
chạy trên **CPU thay vì GPU** — chậm hơn. Bản `onnxruntime-gpu` mới nhất
được cài mặc định cần CUDA 13; image build trên CUDA 12.8. → Pin
`onnxruntime-gpu==1.22.0` (bản build cho CUDA 12, giống local).

### Lỗi #6 (thực ra không phải lỗi) — `KeyError` do dùng nhầm profile Modal

User hỏi "nếu trước dùng tài khoản khác thì sao" → phát hiện máy có **7
profile Modal** lưu sẵn trong `~/.modal.toml`
(`yootek-labs`, `yootek-pro`, `chukhoa2002`, `nguyenkhanh3200175`,
`khanhchu20204569`, `sequense`, `hiepchip318`), và sau đó user đổi sang
profile `ngocnhat1209` (không có trong danh sách gốc — tự thêm). **Bài học
quan trọng:** Volume, Dict, App đều **thuộc về từng workspace riêng biệt**,
không chia sẻ giữa các profile. Nếu `modal volume put` chạy trên profile A
nhưng `modal deploy` chạy trên profile B, worker sẽ thấy Volume rỗng — dễ
nhầm với "lỗi upload" trong khi thực ra là nhầm tài khoản. Luôn
`modal profile current` trước khi làm bất kỳ thao tác Modal nào.

### "Lỗi" #7 — Model tách nền đổi mặc định, tải 1GB mỗi cold start

Không phải bug, nhưng ảnh hưởng UX: `rembg` bản mới nhất (được cài tự do,
không pin version) đổi model mặc định từ `u2net` (176MB) sang `bria-rmbg`
(~1GB), tải với tốc độ ~4-7MB/s trong container → mất gần 3 phút. Máy local
dùng `rembg==2.0.67` nên vẫn là `u2net`. User yêu cầu pin để kết quả giống
hệt local. → Pin `rembg==2.0.67` trong requirements, đặt sau `onnxruntime`
để không phải build lại các bước pip nặng phía trước.

### Lỗi #8 — `POST ... 500` không có traceback trong log

Xảy ra khi script test tự động dừng container cũ rồi **chỉ chờ 5 giây** đã
gửi request tiếp — request rơi đúng lúc container đang bị `container stop`
cắt ngang giữa chừng (không phải lỗi code). → Bài học quy trình: sau khi
`modal container stop`, phải **poll `modal container list` đến khi danh
sách trống hẳn** rồi mới gửi request mới; không dừng container khi đang có
task chạy dở.

---

## 7. Kết quả cuối cùng — pipeline đã chạy thành công end-to-end

Test với `input/motobike.jpg` (nhánh dự kiến: Trellis2, vì đây là vật thể
cơ khí phức tạp):

- Task hoàn tất với `status: completed`, có `modelUrl` trỏ tới GLB trên
  Cloudflare R2, verify bằng `curl -I` trả về `HTTP/2 200`,
  `content-type: model/gltf-binary`.
- Lần đầu (cold, chưa cache rembg): tổng thời gian ~9 phút 18 giây (bao
  gồm cold start + nạp toàn bộ model từ Volume lần đầu + tải rembg 1GB
  lúc đó còn là bria-rmbg).
- Sau khi pin rembg về u2net: tổng thời gian giảm còn ~5 phút 38 giây
  (176MB thay vì 1GB, và không có cảnh báo rơi về CPU nữa).

**Chưa test:** nhánh "hunyuan" (ảnh nhân vật/con vật, ví dụ
`input/tiger.jpg` hoặc `input/anime.png`) — được đề xuất nhưng chưa thực
hiện trong đoạn chat này.

---

## 8. Cơ chế báo lỗi chủ động đã thêm vào `get_hunyuan_api()`

Sau khi hiểu rằng lỗi import node bị ComfyUI nuốt thành "IMPORT FAILED"
im lặng, và lỗi thiếu model chỉ lộ ra giữa chừng workflow (tốn thời gian
chờ), đã chủ động thêm 2 lớp kiểm tra **fail-fast** ngay khi khởi tạo
API, trước khi chạy workflow:

1. **Kiểm tra custom node đã nạp** — check 7 node đại diện
   (`Trellis2LoadModel`, `UnetLoaderGGUF`, `CLIPLoaderGGUF`,
   `IdentityFeatureTransfer`, `TransparentBGSession+`, `Hy3DModelLoader`,
   `DownloadAndLoadHy3DPaintModel`) có mặt trong `NODE_CLASS_MAPPINGS`
   không. Thiếu thì raise ngay với thông báo trỏ người đọc log tìm dòng
   "IMPORT FAILED", thay vì để lỗi hiện ra dưới dạng `KeyError` khó hiểu ở
   giữa workflow.
2. **Kiểm tra 10 model bắt buộc tồn tại trên `/root/models`** (danh sách ở
   mục 5) — thiếu thì raise `RuntimeError` liệt kê rõ file/thư mục nào
   thiếu trong Volume, thay vì để lỗi rơi vào giữa chừng generation (tốn
   nhiều phút compute GPU trước khi mới biết là thiếu model).

Đây là pattern nên giữ khi mở rộng thêm workflow khác: luôn validate
"đủ điều kiện chạy" trước khi tốn tài nguyên GPU.

---

## 9. Vận hành & chi phí Modal — những gì cần nhớ

- **Cấu trúc App:** 1 App = nhiều Function, **mỗi Function chạy trong
  container riêng**. Thấy 2 `Container ID` dưới cùng 1 `App ID` là bình
  thường — một cái là `run` (web, không GPU, bật khi có HTTP request), một
  cái là `run_3d_generation_task` (worker GPU, bật khi web `spawn()` task).
  Không phải lỗi hay trùng lặp.
- **`scaledown_window=1000`**: container giữ ấm ~16-17 phút sau request
  cuối để tránh cold start cho request kế tiếp — nhưng **thời gian chờ này
  vẫn tính tiền compute** (đặc biệt tốn với container GPU L40S).
- **Không có container nào chạy → không tính tiền compute.** Muốn dừng
  hẳn tạm thời: `modal container stop <id>` cho từng container (chỉ dừng
  khi task đã `completed`/`failed`, không dừng khi đang chạy dở). App vẫn
  ở trạng thái `deployed`, URL vẫn hoạt động, gọi API tiếp thì Modal tự
  bật lại container — không cần deploy lại.
- **Không dùng `modal app stop`** để "tạm nghỉ" — lệnh này gỡ hẳn app,
  API sẽ ngừng trả lời cho tới khi `modal deploy` lại.
- Muốn giảm tiền chờ về lâu dài: hạ `scaledown_window` xuống (đánh đổi:
  cold start xảy ra thường xuyên hơn nếu request thưa).
- **Volume và Dict thuộc về từng Modal profile/workspace riêng** — luôn
  `modal profile current` trước khi `deploy`/`volume put` để chắc chắn
  thao tác đúng tài khoản, tránh tình trạng "upload một nơi, deploy một
  nơi khác" (đã từng xảy ra thực tế, xem lỗi #6 mục 6).
- **Đọc log/trạng thái task không cần user dán tay:** có thể tự
  `modal app logs <app-name> --timestamps`, `modal dict get
  hunyuan-3d-task-status <task_id>`, `modal container list --json` để
  debug trực tiếp.

---

## 10. Danh sách file liên quan

| File                                                               | Vai trò                                                                                                                                  |
| ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| [zhunyuan_modal.py](zhunyuan_modal.py)                             | Định nghĩa Modal App, Image, 2 Function (`run`, `run_3d_generation_task`)                                                                |
| [z_image_2_3d_combine_api.py](z_image_2_3d_combine_api.py)         | Workflow Python export từ ComfyUI (Flux2 → QwenVL → Trellis2/Hunyuan)                                                                    |
| [hunyuan_api.py](hunyuan_api.py)                                   | FastAPI routes cũ (tham khảo, không còn được `run()` sử dụng trực tiếp sau khi đổi sang workflow mới — cần double-check nếu tái sử dụng) |
| [image_2_3d.py](image_2_3d.py)                                     | Workflow cũ (Hunyuan-only, có nhánh multi-view) — không còn dùng trong luồng chính nhưng giữ lại tham khảo                               |
| `custom_nodes/ComfyUI-Trellis2/`                                   | Node Trellis2, cần wheel/source riêng cho `o_voxel`, `cumesh`, `flex_gemm`                                                               |
| `custom_nodes/ComfyUI-QwenVL/`                                     | Node `AILab_QwenVL` dùng phân loại ảnh                                                                                                   |
| `constrant.txt`                                                    | Pin `torch==2.7.1`, dùng làm file `-c` (constraint) cho mọi lệnh pip trong image                                                         |
| `/home/yootek/.claude/plans/ch-y-c-workfflow-crystalline-rabin.md` | Plan gốc đã được duyệt (Context, việc cần sửa ở phần A/B, Volume model, Verification)                                                    |

---

## 11. Việc còn tồn đọng / gợi ý cho lần sau

- Chưa test nhánh "hunyuan" (ảnh nhân vật/con vật) end-to-end trên Modal.
- `hunyuan_api.py` (routes cũ) có thể không còn tương thích 100% với
  `OptimizedImageTo3DAPI` mới — nếu có ý định dùng lại, cần đối chiếu lại
  tham số.
- Có thể cân nhắc hạ `scaledown_window` của worker GPU nếu tần suất dùng
  thưa, để giảm chi phí chờ.
- Nếu Trellis2/Hunyuan3DWrapper có bản cập nhật mới, cần re-verify lại
  toàn bộ chuỗi lỗi #3 (GLIBCXX / o_voxel torch version / fork source) vì
  đây là các vấn đề rất đặc thù theo đúng version wheel tại thời điểm build.
