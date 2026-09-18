## 1 -  Hậu xử lý

**Fill holes**: hiện có 3 hàm xử lý việc này

- FillHolesWithMeshlib: `meshlib.mrmeshpy.fillHole()` method

- FillHolesNicelyWithMeshlib: `meshlib.mrmeshpy.fillHoleNicely()` method. Xịn hơn cách đầu, kết quả "nice" hơn, tốn thời gian hơn.

- FillHolesWithCuMesh: dùng method `fill_holes()` có sẵn của đối tượng mesh. Có tham số "chu vi" giúp chỉ vá những lỗ nhỏ. Phù hợp tiền xử lý.

**Reconstruct**: dùng thuật toán **Dual Contouring** rebuild toàn bộ mesh theo độ phân giải mới từ dữ liệu vertices/faces hiện có, giúp bề mặt liền mạch, "mượt" hơn, có thể tự động "chữa" hole như tác dụng phụ của việc remesh (nếu độ phân giải không đổi thì vẫn rebuild mesh từ đầu, vẫn làm mượt, chuẩn hóa bề mặt)

- `CuMesh.remeshing.reconstruct_mesh_dc()`: bản thường, tam giác

- `CuMesh.remeshing.reconstruc_mesh_dc_quad()`: bản quad (tứ giác), ngon hơn

## 2 - Tối ưu hệ thống

RAM: nơi trung chuyển

VRAM: chứa dữ liệu GPU cần để tính toán

- Khi load model lên GPU: đã chiếm một phần VRAM

- Khi infer: chiếm thêm VRAM

Liệu có thể chạy song song đa luồng??

Nạp model là vào RAM, rồi mới đẩy lên VRAM của GPU để chạy??
