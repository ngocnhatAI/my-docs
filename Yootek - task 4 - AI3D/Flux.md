

![](/C:/Users/ngocn/AppData/Roaming/marktext/images/2026-09-18-12-17-05-image.png)

- **IdentityGuidance**: `img2img latent anchor`,trong 80% đầu quá trình denoise, nó kéo latent đang generate về gần latent của ảnh gốc. Strength càng cao, ảnh ra càng giống ảnh gốc về cấu trúc nhưng dễ giảm khả năng sáng tạo/đổi góc nhìn (trong case này mình đang muốn đổi sang góc 3/4).

- **IdentityFeatureTransfer**: hoạt động ở cấp cơ chế attention bên trong UNet — nó "mượn" đặc trưng (feature) từ model gốc vào các block cụ thể (0-23, tức gần như toàn bộ mạng) theo kiểu cosine similarity, chỉ áp dụng cho top 25% đặc trưng khớp nhất. Đây là ghim ở cấp texture/chất liệu/chi tiết bề mặt hơn là bố cục — giữ màu sắc, vân, kết cấu đặc trưng của chủ thể mà không ép cứng bố cục.

Tóm lại **IdentityGuidance** giữ bố cục/hình khối tổng thể, **IdentityFeatureTransfer** giữ chi tiết/chất liệu. Dùng khi bạn cần ảnh mới (góc khác, pose khác) nhưng vẫn nhận ra là "cùng một vật thể" cả về hình lẫn chất liệu.

Nếu chỉ chọn 1: Với mục tiêu là tạo ảnh turnaround/3/4 view lộ rõ cấu trúc ẩn để phục vụ dựng 3D thì ưu tiên **IdentityFeatureTransfer**, vì:

- Nó không ép cứng bố cục/latent gốc, cho phép model tự suy luận hình dạng/tách bộ phận theo đúng yêu cầu prompt (rất quan trọng để lộ cấu trúc 3D).
- Nó vẫn giữ màu sắc, chất liệu, đặc trưng nhận dạng — đáp ứng yêu cầu "preserving original colors, materials, and textures" trong prompt.

Ngược lại nếu bạn thấy ảnh ra bị "lệch dạng" quá nhiều so với ảnh gốc (sai tỷ lệ, method suy luận cấu trúc sai), lúc đó nên bật thêm **IdentityGuidance** với strength thấp (~0.2-0.3) chỉ để neo bố cục nhẹ, chứ không nên dùng 241 một mình vì nó dễ giữ nguyên góc nhìn gốc, cản trở việc đổi sang view 3/4 mà prompt yêu cầu.
