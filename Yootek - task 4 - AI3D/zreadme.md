## Kiến trúc

- 1 App Modal = 2 Function, **mỗi Function 1 container riêng**:

  - `run` (web, không GPU): FastAPI, nhận request, `spawn()` task nền.

  - `run_3d_generation_task` (worker, GPU): chạy workflow thật.

- Thấy 2 Container ID dưới cùng 1 App ID trong `modal container list` là

  bình thường, không phải lỗi.

- Cơ chế chạy workflow không cần UI ComfyUI: `nodes.init_extra_nodes()`

  nạp mọi custom node vào `NODE_CLASS_MAPPINGS`, sau đó gọi thẳng

  `NODE_CLASS_MAPPINGS["TênNode"]()` như hàm Python bình thường.

## Sửa script export để import được trong worker (async)

3 chỗ luôn phải sửa khi lấy 1 file "export ra .py" của ComfyUI làm module

import (thay vì chạy trực tiếp):

1. Thêm `nest_asyncio.apply()` trước `asyncio.run(init_extra_nodes())` —

   worker Modal là `async def`, đã có event loop chạy sẵn, `asyncio.run()`

   lồng vào sẽ lỗi.

2. Tắt `comfy.options.enable_args_parsing()` — nếu bật, ComfyUI sẽ

   `argparse` trên `sys.argv` thật của container, dễ crash/nhận nhầm.

3. Bọc đoạn `uvicorn.run(...)` cuối file vào `if __name__ == "__main__":`

   — nếu không, **chỉ import module** cũng tự mở server và treo worker.

## Thứ tự layer trong Modal Image — nguyên tắc cache

Modal build image theo layer tuần tự, sửa 1 layer thì mọi layer **phía

sau** bị build lại. Nguyên tắc: **ít đổi → trên, hay đổi (code) → dưới

cùng**, và mount code bằng `copy=False` để sửa code không kích hoạt build

lại gì cả. Thứ tự đã áp dụng:

1. `apt_install`, biến môi trường CUDA

2. Copy riêng từng file `requirements*.txt` (không copy cả thư mục) → pip

   install, luôn kèm `-c constrant.txt` để khoá version torch

3. Copy source các extension CUDA/C++ cần compile → compile tại build time

4. Các bước vá lỗi phát sinh (đặt gần cuối để không kéo theo build lại pip)

5. **Cuối cùng**: `add_local_dir(".", ..., copy=False, ignore=[...])` —

   mount code, ignore data nặng (`models/`, `input/`, `output/`, `.git/`,

   `__pycache__/`) và ignore custom node không dùng tới (giảm cold start,

   giảm "IMPORT FAILED" gây nhiễu log)

Hiệu quả: sau khi sắp lại, deploy chỉ sửa Python code mất **16–27s** thay

vì build lại toàn bộ.

## Model dùng Modal Volume, không bake vào image

- Tạo Volume riêng (`hunyuan3d-models`), mount `/models`, symlink

  `/root/models → /models`. Đổi model không cần build lại image.

- Set `HF_HOME` và `U2NET_HOME` trỏ vào Volume để model node tự tải lúc

  runtime cũng được cache qua các lần cold start.

- Upload bằng `modal volume put --force <volume> <local> <remote>`.

  `--force` vì Volume có thể đã có bản tải dở do node tự tải khi lỗi giữa

  chừng.

- Trước khi chạy, code tự check các file/thư mục model bắt buộc có tồn

  tại không (`os.path.exists`), thiếu thì raise rõ ràng luôn — tránh tốn

  GPU compute rồi mới biết thiếu model.

## Checklist debug lỗi custom node (rút từ thực tế)

**Nguyên tắc chung:** ComfyUI nuốt lỗi import node, chỉ log dòng "IMPORT

FAILED" trong `nodes.py`'s scan rồi chạy tiếp. Mọi `KeyError` về tên node

ở giữa workflow (`NODE_CLASS_MAPPINGS["X"]`) đều là **hệ quả**, không phải

lỗi gốc → luôn đọc `modal app logs` tìm traceback thật gần dòng "IMPORT

FAILED", đừng sửa theo triệu chứng bề mặt.

Các lỗi cụ thể đã gặp khi mang node có compile native (CUDA/C++) sang

container khác:

| Triệu chứng | Nguyên nhân | Cách vá |

|---|---|---|

| `GLIBCXX_3.4.31' not found` | Wheel build bằng GCC mới hơn libstdc++ của base image | Nâng `libstdc++6` qua PPA `ubuntu-toolchain-r/test`, verify bằng `strings libstdc++.so.6 \| grep GLIBCXX_x.x.x` |

| `undefined symbol: c10::...` khi import 1 package torch-extension | Wheel đó build với version torch khác version đang pin | Build lại từ source đúng version thay vì dùng wheel sẵn |

| `ImportError: cannot import name X` dù source "đúng repo" | Nhiều fork tồn tại, code cần hàm chỉ có ở 1 fork cụ thể | Đọc README của node xem có ghi rõ dùng fork nào; so sánh file nguồn với bản cài ở local (nếu local chạy được) để xác nhận đúng fork |

| Task fail giữa chừng vì thiếu file model | Volume chưa upload / thiếu thư mục cha mà code có `shutil.copyfile` vào | Check tồn tại file trước khi chạy; xem code node có write file vào path nào không, tạo sẵn thư mục cha |

| Log không lỗi nhưng chậm bất thường | Package không pin version, bản mới đổi default (model default, provider default...) tải file nặng mỗi cold start | Pin version giống môi trường đã chạy ổn (local), so log 2 bên |

| `POST` 500 không traceback ngay sau khi vừa `container stop` | Request rơi vào container đang bị dừng giữa chừng | Poll `container list` tới khi rỗng hẳn rồi mới gửi request; không dừng container khi có task đang chạy |

## Modal profile / workspace

- Volume, Dict, App **thuộc riêng từng profile**, không chia sẻ. Nhầm

  profile giữa lúc `volume put` và lúc `deploy` → worker thấy Volume rỗng,

  dễ nhầm là lỗi upload.

- Luôn `modal profile current` trước khi làm bất cứ thao tác gì.

- Máy có thể có nhiều profile lưu sẵn trong `~/.modal.toml` — kiểm tra

  trước khi làm việc trên máy lạ.

## Vận hành / chi phí

- Container rảnh vẫn tính tiền trong khoảng `scaledown_window` (giữ ấm

  sau request cuối, tránh cold start). Không container nào chạy = không

  tính tiền compute.

- Dừng tạm: `modal container stop <id>` (chỉ khi task đã xong). App vẫn

  `deployed`, gọi API tiếp thì tự bật lại, không cần deploy lại.

- Đừng dùng `modal app stop` để "nghỉ tạm" — nó gỡ hẳn app.

- Có thể tự debug không cần user dán log: `modal app logs <app> --timestamps`,

  `modal dict get <dict-name> <key>`, `modal container list --json`.

## File liên quan trong dự án này

- [zhunyuan_modal.py](zhunyuan_modal.py) — Modal App/Image/Function

- [z_image_2_3d_combine_api.py](z_image_2_3d_combine_api.py) — workflow chính đang dùng

- `constrant.txt` — pin torch, dùng làm `-c` constraint cho mọi lệnh pip

- `custom_nodes/ComfyUI-Trellis2/` — ví dụ điển hình node cần build native, xem README trước khi đoán fork

## Việc tồn đọng

- Chưa test nhánh "hunyuan" của workflow (mới test nhánh "trellis").

- Nếu custom node cập nhật version, cần re-check lại toàn bộ phần

  GLIBCXX/torch-version/fork-source vì rất đặc thù theo version tại thời

  điểm build.
