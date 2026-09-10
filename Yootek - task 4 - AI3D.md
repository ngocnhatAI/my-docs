## 1 - Principles

- No remove, no ovtk. Just focus on core modules to connect, improve pipeline

## 2 - Summary

- main.py: entry point khởi động server

- nodes.py: định nghĩa toàn bộ node built-in 

- comfy_extras/: định nghĩa các node phụ trợ (video, audio, masks,...)

- models/: chứa toàn bộ checkpoint/model weight

- image_2_3d.py: định nghĩa logic pipeline lõi cho one-image-to-3D (không qua HTTP)

- hunyuan_api_3.py: FastAPI wrapper quanh logic trên + upload R2 + multi-view generation 

### 2.1 - custom_nodes/: chứa các node liên quan đến 3D

| custom_nodes/                                            | Vai trò                                                                                                                                                            |
|:-------------------------------------------------------- |:------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ComfyUI-Hunyuan3DWrapper                                 | Wrapper gốc cho model Hunyuan3D — sinh mesh + paint texture.                                                                                                       |
| ComfyUI-Hunyuan3d-2-1                                    | Wrapper cho phiên bản model Hunyuan3D-2.1mới hơn                                                                                                                   |
| ComfyUI-UltraShape1                                      | Refine mesh 3D thô dựa theo ảnh gốc — bước hậu xử lý ngay sau khi generate mesh, tăng chi tiết hình học.                                                           |
| ComfyUI-Free-GPU                                         | Giải phóng VRAM/RAM giữa các bước                                                                                                                                  |
| ComfyUI_UltimateSDUpscale                                | Upscale ảnh bằng SD — dùng để tăng chất lượng ảnh input hoặc texture multiview trước khi bake.                                                                     |
| comfyui-kjnodes<br/>ComfyUI_essentials<br/>ComfyLiterals | Bộ node tiện ích cơ bản (image processing, math, mask ops,..)                                                                                                      |
| comfyui-to-python-extension                              | Convert workflow JSON → script Python — có thể đã được dùng để tạo image_2_3d.py từ one_image_2_3d.json. Hữu ích nếu bạn cần tái tạo/refactor code từ workflow UI. |

### 2.2 - workflows/

- Các workflow .json chỉ chạy được trên UI. Muốn expose qua API phải chuyển workflow từ .json => .py qua extension `comfyui-to-python-extension`

- Cách build workflow
  
  - Kéo-thả node trên giao diện web ComfyUI (có thể copy node từ workflow khác). Sau khi chạy thử thành công, lưu, ComfyUI sẽ tự sinh file .json
  
  - Dùng `Group/Subgraph` để thu gọn 1 cụm node thành một hộp duy nhất
  
  - Dùng Note node để ghi chú giải thích từng cụm, tránh quên sau này

- Node install: Trên UI, nút **"Manager"** → **"Install Custom Nodes"** → hiện danh sách search được hàng nghìn node từ cộng đồng
  
  - Danh sách lấy từ:[custom-node-list.json](https://github.com/ltdrdata/ComfyUI-Manager) — hoặc local `custom_nodes/comfyui-manager/custom-node-list.json`(update nếu cần)

| Workflow                               | Mục đích                                                                   |
| -------------------------------------- | -------------------------------------------------------------------------- |
| flux_1_kontext_dev_basic.json          | Chỉnh sửa ảnh bằng Flux.1 Kontext (ảnh+prompt)                             |
| flux 2.json                            | Chỉnh sửa ảnh bằng Flux.2 Dev (ảnh+prompt)                                 |
| hunyuan_2-1.json                       | Ảnh đơn → 3D dùng model Hunyuan3D-2.1. Pipeline đơn giản để test model mới |
| Hunyuan 3d Multiple View Ai Verse.json | Hunyuan 3d Multiple View Ai Verse.json                                     |
| hunyuan+ultrashape.json                | Ảnh đơn → mesh thô (Hunyuan3D) → refine bằng UltraShape → texture/export   |
| hy3d_example_01.json                   | Workflow mẫu cơ bản gốc từ ComfyUI-Hunyuan3DWrapper                        |
| one_image_2_3d.json                    | Ảnh đơn → Flux Kontext tiền xử lý ảnh → mesh → texture                     |

## 3 - Generate 3D

B1: Ảnh 2D được đưa qua một mạng trích xuất đặc trưng (như **CLIP** hoặc **DINO**) để lấy **Feature Embedding** (vector ngữ cảnh).

B2: Diffusion Transformer (DiT) bắt đầu từ một khối **Nhiễu ngẫu nhiên trong không gian 3D (3D Noisy Latent)**, dưới sự hướng dẫn của Feature Embedding từ ảnh 2D, DiT tiến hành khử nhiễu từng bước để tạo ra một **3D Clean Latent** hoàn chỉnh.

B3: 3D-VAE Decoder nhận 3D Clean Latent và decode nó thành một trường biểu diễn 3D liên tục (thường là **SDF - Signed Distance Field**, **NeRF**, hoặc **3D Gaussians**).

B4: Trích xuất Mesh. Do VAE chỉ giải nén ra mật độ khối / trường khoảng cách (SDF), hệ thống cần thêm một thuật toán hình học (như **Marching Cubes**) để quét bề mặt đẳng trị và xuất ra file lưới tam giác (**Mesh .obj / .glb**).

> **Tóm tắt luồng xử lý:**
> 
> `Ảnh 2D` $\rightarrow$ `Image Feature` (Condition)
> 
> `3D Noise` + `Condition` $\xrightarrow{\text{DiT}}$ `3D Latent` $\xrightarrow{\text{3D-VAE}}$ `SDF / NeRF Field` $\xrightarrow{\text{Marching Cubes}}$ `Mesh (3D)`

## 4 - Tối ưu

Hướng1: guidance latent space (không có model nào hỗ trợ text)

Huong2: sinh multi-view image (ảnh multiview từ AI làm tăng sai số tích lũy)

Hướng 3: sử dụng model image-2-image chỉnh ảnh, giảm góc khuất, giảm bớt yêu cầu nội suy cho model 3D

- **Trellis**
  
  - Kết hợp Text Prompt để định hình các chi tiết bị che

- **Direct3D (DiT 3D Native):**
  
  - Sử dụng trực tiếp DiT để sinh tri-plane latents từ ảnh đơn và text.

- **CLAY (Continuous Latent 3D via DiT):**
  
  - Hỗ trợ conditioning linh hoạt bằng cả Text, Image hoặc Voxel sketches.

- **LGM / CRM kết hợp DiT (Triplane-based DiT):**
  
  - Dòng mô hình mã hóa ảnh 2D sang không gian tri-plane/Gaussian latent bằng Transformer blocks, cho phép inject thêm Text prompt.

- **Shap-E (OpenAI) / Biến thể DiT:**
  
  - Hỗ trợ native chế độ Image-to-3D có text conditioning

- **TripoSR**

- **Hunyuan3D**

**Di chuyển bằng Group (Nhóm):** Nhấp giữ chuột trái vào **tiêu đề của khung Group** để di chuyển toàn bộ các node nằm bên trong khung đó.

**Debug Tensor Shape** → `essentials/utilities`: in ra tensor shape của ảnh

Vì sao trong input Hy3DGenerateMesh, image lấy từ ảnh gốc sau resize + mask thay vì ảnh sau khi mask: vì ảnh sau khi mask có chiều 1x518x518x4, phải là 1x518x518x3, có lẽ thừa chiều alpha (độ trong suốt)

? Model phải wraper? - để thống nhất input, output => readdy to connect

? Add nodes ở đâu

Hướng: guidance latent space
