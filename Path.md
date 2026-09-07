**pathlib.Path**

```python
from pathlib import Path

SCRIPT_DIR = Path(__file__).resolve().parent
new_file_path = SCRIPT_DIR / "test" / "test.py"


new_file_path.exists()   # Đường dẫn tồn tại có thể là file/folder
new_file_path.is_file()  # Đường dẫn tồn tại và là file
new_file_path.is_dir()   # Đường dẫn tồn tại và là folder

# Tạo folder (tạo luôn parent nếu chưa tồn tại)
new_file_path.parent.mkdir(parents=True, exist_oke=True)
# Tạo file, raise ERROR nếu folder chứa file chưa tồn tại
new_file_path.touch()    

new_file_path.unlink()   # Xóa file
new_file_path.rmdir()    # Xóa folder 

new_file_path.name     # test.py
new_file_path.stem     # test
new_file_path.suffix   # .py
new_file_path.parrent  # SCRIPT_DIR / "test"
```



**update Python package**

```bash
pip install torch
pip install -U torch
pip install torch=2.8.0
```



**UploadFile, File, Form, BackgroundTasks**

```python
from fastapi import File, UploadFile, Form, BackgroundTasks

# Khai báo
name: str = Form(...)
age: int = Form(...)
image: UploadFile = File(...)
background_tasks: BackgroundTasks

# UploadFile - object đại diện cho file được upload
image.file_name     # car.jpg
image.content_type  # image/jpg
image.file          # file-like object
image.read()        # read file


# BackgroundTasks
background_tasks.add_task(cleanup_function, para,)
```

Trong đó:

- str, UploadFile là Python type. 
  
  - UploadFile dùng cho bất kì file nào upload, bao gồm text, image,pdf,audio,...

- Trong FastAPI, nguồn trong HTTP Request dạng `multipart/form-data` thường gồm:
  
  - `File(...)` → phần dữ liệu là file upload.
  
  - `Form(...)` → phần dữ liệu là field thông thường của form, vd: int, str, bool,..

- `BackgroundTask` là object dùng để: Sau khi response đã được gửi cho client, hãy chạy thêm một hàm đơn lẻ nào đó ở background.


