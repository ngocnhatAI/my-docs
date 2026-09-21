## 1 - Main workflow

- Image condition: image + CLIP Vision model

- Spartial structure: generator (DiT) + decoder (VAE)
  
  - Input: image condition
  
  - Output: Coordinates

- Shape: generator (DiT) + decoder (VAE)
  
  - shape_slat: (image condition, coordinates) + shape_DiT
  
  - mesh: shape_slat + shape_VAE (decoder) 

- Mesh postprocessing (fill holes, reconstruct, reduce faces,...)

- Texture: generator (DiT) + decoder (VAE)
  
  - texture_slat: (image condition, shape_slat) + texture_DiT
    
    - shape_slat: (image condition, coords) + shape_DiT (previous step)
    
    - shape_slat: mesh + shape_VAE (encoder)
  
  - texture: texture_slat + texture_VAE (decoder)

- Combine texture + mesh => Final model

## 2 - Main update

- Backend-option: flash-attn instead of spda => không nhanh hơn

- Image editting:
  
  - Prompt ngăn đổ bóng để tránh tô màu nhầm trắng => xám (case con thỏ)
  
  - Dùng opencv làm mượt ảnh. 
    
    - Trực giác: ảnh độ phân giải thấp cần ít faces xấp xỉ bề mặt => nhanh hơn.
    
    - Thực tế: mesh sinh ra nhiều faces hơn, lâu hơn (lol). 
  
  - Segmentation: dùng `inspyrenet `thay cho `rembg `(ngon hơn, bớt lẹm ảnh)

- Mesh/Texture resolution: image_cond và DiT_resolution phải khớp nhau
  
  - image_cond 512/1024: cùng backbone, khác mỗi số chiều.
  
  - shape_slat512 + con512 + text_dit_512: oke, bản base.
  
  - shape_slat512 + con1024 + text_dit_1024: ghép được. Khi sinh texture, shape là tham số độc lập, resolution không ảnh hưởng. 
  
  - shape_slat512 + con512 + text_dit_1024: mis-match.

## 

## 3 -  Mesh postprocessing

**Fill holes**: hiện có 3 hàm xử lý việc này

- FillHolesWithMeshlib: `meshlib.mrmeshpy.fillHole()` method

- FillHolesNicelyWithMeshlib: `meshlib.mrmeshpy.fillHoleNicely()` method. Xịn hơn cách đầu, kết quả "nice" hơn, tốn thời gian hơn.

- FillHolesWithCuMesh: dùng method `fill_holes()` có sẵn của đối tượng mesh. Có tham số "chu vi" giúp chỉ vá những lỗ nhỏ. Phù hợp tiền xử lý.

**Reconstruct**: dùng thuật toán **Dual Contouring** rebuild toàn bộ mesh theo độ phân giải mới từ dữ liệu vertices/faces hiện có, giúp bề mặt liền mạch, "mượt" hơn, có thể tự động "chữa" hole như tác dụng phụ của việc remesh (nếu độ phân giải không đổi thì vẫn rebuild mesh từ đầu, vẫn làm mượt, chuẩn hóa bề mặt)

- `CuMesh.remeshing.reconstruct_mesh_dc()`: bản thường, tam giác

- `CuMesh.remeshing.reconstruc_mesh_dc_quad()`: bản quad (tứ giác), ngon hơn

## 4 - System optimization

RAM: nơi trung chuyển

VRAM: chứa dữ liệu GPU cần để tính toán

- Khi load model lên GPU: đã chiếm một phần VRAM

- Khi infer: chiếm thêm VRAM

Liệu có thể chạy song song đa luồng??

Nạp model là vào RAM, rồi mới đẩy lên VRAM của GPU để chạy??

## 5 - Archieved:

Prompt: *Clean 3/4 perspective shot of the exact same subject as input image, preserving all true original colors, materials, and broad textures. Fully complete, entire single isolated object, no parts cut off, centered on solid neutral studio background with wide margins. Clear separation between distinct components and limbs revealing 3D form and geometry. Accurate anatomy and authentic proportions. Flat, shadowless, even studio lighting showing only true local colors without shadows, shading, or color cast. Simplified smooth surfaces, minimal micro-details, no floating elements, no artifacts.*

### 
