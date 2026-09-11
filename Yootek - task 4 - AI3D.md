## Tóm tắt

- Bài toán: Cải thiện chất lượng đầu ra cho pipeline one-image-to-3D

- Thực trạng: mô hình 3D bị méo, dính, sai thông tin cơ bản (ví dụ thỏ có 3 chân)

- Phương pháp: tiền xử lý ảnh bằng một model image-to-image, tạo ảnh mới sạch hơn, ít góc khuất hơn, giảm tải áp lực nội suy cho mô hình image-to-3D



## Giai đoạn 1: Mì ăn liền

### 1 - Chạy thử các workflow sử dụng ComfyUI Web

- ComfyUI là framework rất mạnh, hỗ trợ xây rất nhiều workflow chỉ bằng kéo thả.

- Công việc chính: sử dụng **Node Manager** để download **custom node**, chạy các workflows có sẵn trong các package đó rồi so sánh.
  
  - Bản chất của nút Download trên web là clone + pip install requirements. Một vài nodes cần tự tải thêm thư viện mới chạy được

- Sau khi chỉnh sửa **workflow**, khi bấm lưu thì ComfyUI sẽ tự cập nhật file .json. Từ file .json có thể xuất ra file .py (tự động). Với file .py ta có thể wrap thêm API.

### 2 - Tóm tắt pipeline image-to-3D

3D = mesh + texture. Tóm tắt pipeline: 

- image => CLIP => Feature Embedding (Condition)

- Condition => **DiT** => mesh latent space => **vae decode** = > mesh

- Condition + mesh => **DiT** => texture latent space => **vae decode** => texture

Bước 1: Ảnh 2D được đưa qua Image-encoder (CLIP,...) để lấy Feature Embedding.

Bước 2: DiT bắt đầu từ 3D noisy latent, dưới sự hướng dẫn của Feature Embedding từng bước khử nhiểu để tạo latent space cho mesh/texture.

Bước 3: 3D-VAE decode để tạo mesh/texture, kết hợp lại ta được vật thể 3D hoàn chỉnh

### 3 - Các hướng tối ưu

Hướng 1: Sử dụng text prompt để guidance 3D-latent-space. Hướng này còn rất mới, chỉ dừng ở phương pháp nghiên cứu, chưa có model hỗ trợ (tìm hiểu sau)

Hướng 2: Dùng AI sinh multi-view images từ ảnh ban đầu. Sau đó sinh 3D với pipeline multi-images-to-3D. Không khả thi do ảnh multi-view từ AI chưa chắc đảm bảo nhất quán, có thể làm tăng sai số tích lũy.

Hướng 3: sử dụng model image-2-image chỉnh ảnh, tạo ảnh mới sạch hơn, hạn chế tối đa góc khuất, chia sẻ gánh nặng nội suy vùng khuất cho model 3D.

- Hướng 3.1: Chỉnh sửa ảnh gốc dựa theo prompt. 
  
  - Condition = Text prompt, Latent space = Ref Image.

- Hướng 3.2: Sinh ảnh mới hoàn toàn từ ảnh gốc và text
  
  - Condition = Text prompt + Ref Image, Latent space = Noise.

Ở đây chọn hướng 3.2, vì cần ảnh mới thật sạch và hạn chế vùng khuất, gần như là sinh ảnh mới. Hơn nữa model 3D ưu tiên đúng về cấu trúc, không cần giống hệt vật thể gốc.

### 4 - Kết quả thực nghiệm

Quá trình được thực nghiệm trên các workflow. Các workflows đều sử dụng mặc định, dùng AI để phân loại nhanh ý nghĩa workflow, tránh phải tự đọc/thử toàn bộ.

- Trellis 2: độ chi tiết tương đối cao nhưng chỉ chạy được bản FB8 latent-dim=512, các bản khác cao hơn đều không chạy được, kể cả bản FB8 latent-dim=1024 (cấu hình max là FB16 latent-dim=1024). Cũng có tự tải bản IN8 nhưng chưa wrapper nên không thể dùng.

- Flux 2: sử dụng workflow mặc định + node **gguf** để chạy (ref: unsloth)

- Hunyuan 2, 2.1: cũng tạm ổn

Nhìn chung Hunyuan 2 khá ổn, nhưng với case con thỏ thì đuôi dài như đuôi cáo. Hunyuan2.1 và Trellis 2 tránh được lỗi đuôi cáo, nhưng chạy cũng quá chậm.

### 5 - Note

**Debug Tensor Shape** → `essentials/utilities`: Node in ra tensor shape của ảnh

File safetensors gồm nhiều sub-models, có thể phân rã thành các file độc lập







Thiết kế model được lazy-load từng phần (chỉ load lên GPU đúng lúc cần, có thể unload sau khi dùng xong stage)

Thiết kế cơ chế low_vram (chunk từng phần theo chunk_size)

Giải phóng VRAM chủ động
