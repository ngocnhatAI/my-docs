Clone về chỉ có "name" + "model glb"

\- Thumbnail: là ảnh jpeg, có được bằng cách chụp file .glb

\- Description: được tạo ra từ Thumbnail (genAI)

\- Dense embedding được tạo ra từ description dùng để đo similarity (model: AITeamVN/Vietnamese\_Embedding\_v2)



Task: description đang sai => dense embedding sai => recommendation tuất

Sửa description, nguyên nhân (chẩn đoán):

\- Ảnh thumbnail tự chụp vật thể không ở giữa

\- Ảnh thumbnail tự chụp vật thể bị mất thông tin do góc chụp khuất



Giải pháp:

1. **Kiểm soát chặt để góc chụp bao quát vật thể, đảm bảo không bị lệch, quá to (thiếu thông tin), quá nhỏ (thừa thông tin)**
2. **Chụp đa dạng góc để tránh góc chết, khoảng 4 view / GLB**

Dùng multi-image image-to-text để sinh description thay vì ghép 4 ảnh thành 1 ảnh hoặc chạy 1 GLB 4 lần.

Vào tag task: chọn Image-to-Text hoặc Image-Text-to-Text

3. **Prompt chặt hơn. Ví dụ:**

Only describe attributes directly visible in the images.

Do not infer brand, usage, material, style or other properties

unless visually supported.

For uncertain attributes, output "unknown".

**4. Hybrid recommendation**

Lưu cả image embedding và description\_embedding, tính similarity khi recommendation dựa vào cả hai score

**5. Cross-check dữ liệu cũ bằng image-text embedding**

image -> image encoder -> image\_embedding

text > text encoder -> text\_embedding

=> Tính cosine(image\_embedding, text\_embedding), nếu thấp thì thumbnail và descriptioni có khả năng không khớp.

Nên dùng model image-text encoder để image/text nằm trong cùng latent space như CLIP/SigLIP-style model.

Nếu dùng một image encoder và  một text encoder độc lập bật kỳ thì hai vector chưa chắc cùng không gian để cosine trực tiếp.

**6. Sử dụng 2 VLM khác family để kiểm chứng caption (sinh mới).**

image -> VLM A -> description A

image -> VLM B -> description B

Kiểm tra các chỉ số

cosine(desc A, desc B): text-encoder

cosine(image, desc A): vision-text encoder

cosine(image, desc B): vision-text encoder

Vd: Nếu chỉ số đầu cao, hai chỉ số sau thấp, thì là cả hai model đều đồng thuận sai

**Từ khóa hf:** vision/image/text + encoder/embedding/retrieval, CLIP, image-text retrieval,..

**Nhược điểm:** các model image encoder không hỗ trợ tốt tiếng việt. Nhìn chung vẫn cần dùng VLM sinh text từ ảnh làm trung gian để sinh vector embed.
