**BÀI TOÁN:** Server công ty phải dùng mạng LAN mới ssh vào được. Desktop cá nhân dùng mạng công ty SSH được vào server. Muốn dùng laptop SSH nhà -> desktop -> GPU server (**desktop đóng vai trò jump host)**

```powershell
ssh -J user@desktop user@gpu-server
```

Hoặc cấu hình **\~/.ssh/config:**

```ssh
Host desktop
    HostName <IP-cua-desktop>
    User your_desktop_user

Host gpu
    HostName 192.168.x.x
    User your_server_user
    ProxyJump desktop
```

Sau đó ở nhà chỉ cần:

```powershell
ssh gpu
```

Cần phân biệt 2 IP khác nhau

* IP desktop (Tailscale)
* IP server (LAN)

**CÁCH THỰC HIỆN:** cài [Tailscale](https://tailscale.com/) trên laptop ở nhà + desktop công ty. 

**Bước 1:** Cài [Tailscale trên desktop](https://tailscale.com/download/). Sau khi cài mở PowerShell lấy IP **tailscale** của desktop

```powershell
tailscale ip -4
```

**Bước 2:** Bật SSH Server trên desktop

Mở PowerShell → Run as Administrator. Kiểm tra:

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'
```

Nếu thấy

```powershell
OpenSSH.Server~~~~0.0.1.0
State : NotPresent
```

Thì cài OpenSSH Server

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

Sau đó khởi động SSH Server -> set sshd tự động khi desktop khởi động -> kiểm tra trạng thái SSH Server

```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
Get-Service sshd
```

Nếu dòng cuối hiện như dưới đây là thành công

```powershell
Status   Name
------   ----
Running  sshd
```

**Bước 3:** Xác định username trên desktop (2 cách)

```powershell
whoami
$env:USERNAME
```

**Bước 4:** Cài Tailscale trên laptop ở nhà (đăng nhập cùng account với desktop)

Test thử ssh vào desktop (ping trước thử xem có thể kết nối được không)

```powershell
ping desktop_ip_tailscale
ssh desktop_user@desktop_ip_tailscale
```

Nếu thành công thì SSH thẳng GPU như đề cập đoạn đầu tiên bằng ProxyJump. Để kiểm tra (nếu **nvidia-smi** ra GPU là thành công)

```powershell
hostname
nvidia-smi
```

**NOTE**

Nếu có quyền **sudo** trên server, chỉ cần cài TailsCale lên Server, lấy **IP tailsace** rồi dùng laptop SSH thẳng vào server, không cần qua desktop trung gian.)

Khi chạy bước 2 trên desktop công ty nếu hiện như dưới, tức là desktop của bạn có thể ssh sang GPU server, nhưng laptop chưa thể ssh vào desktop window, nên chúng ta cần cài OpenSSH Server như bước 2 đã làm

```powershell
Name : OpenSSH.Client~~~~0.0.1.0 State : Installed
Name : OpenSSH.Server~~~~0.0.1.0 State : NotPresent
```

**TRƯỜNG HỢP BƯỚC 4 KHÔNG SSH ĐƯỢC DO SAI PASSWORD, DÙNG SSH-KEY**

**Tóm tắt:** tạo SSH trên laptop cá nhân rồi add vào desktop/server. Key giúp verify máy bạn là "nguồn tin cậy", có thể SSH vào server mà không cần password.

**Bước 1:** Tạo SSH key trên máy window

Tạo SSH key (uet-gpu là comment cho key, optional)

```powershell
ssh-keygen -t ed25519 -C "uet-gpu"
```

Enter liên tục để confirm lưu trữ key ở đường dẫn mặc định (thay admin = user\_laptop, tra cứu bằng **whoami**)

```powershell
C:\Users\admin\.ssh\id_ed25519
```

Kiểm tra với lệnh sau nên thấy

```powershell
dir C:\Users\admin\.ssh
dir $env:USERPROFILE\.ssh
```

- id\_ed25519     = private key, giữ bí mật

- id\_ed25519.pub = public key, copy được

Xem & copy public key (có dạng ssh-ed25519… uet-gpu.)

```powershell
type C:\Users\admin\.ssh\id_ed25519.pub
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

**Bước 2**: Trên desktop, đưa public key vào **authorized\_keys**. Nếu người dùng thuộc nhóm Administrators, Windows OpenSSH thường sử dụng **C:\\ProgramData\\ssh\\administrators\_authorized\_keys** thay vì **C:\\Users\\ngocn\\.ssh\\authorized\_keys**, trong case này đang ví dụ người dùng là Admin.

Kiểm tra người dùng có thuộc nhóm Admin hay không

```powershell
net localgroup Administrators
```

Tạo file administrators\_authorized\_keys (trong case này là admin)

```powershell
New-Item -ItemType File -Path "C:\ProgramData\ssh\administrators_authorized_keys" -Force
```

Sau đó mở file bằng notepad và dán public key vào -> Ctrl + S để lưu. 

```powershell
notepad "C:\ProgramData\ssh\administrators_authorized_keys"
```

Kiểm tra (hoặc dùng lệnh trên mở lại file notepad để kiểm tra)

```powershell
Get-Content "C:\ProgramData\ssh\administrators_authorized_keys"
```

Note: lệnh *notepad* để mở file ở trên có thể dùng để tạo file nếu file chưa tồn tại, tuy nhiên không nên sử dụng để tạo file mà nên dùng lệnh *New-Item -ItemTyope File -Path....*

Nguyên nhân khi tạo file bằng *notepad*, file sẽ tự động thêm extension txt *(administrators\_authorized\_keys.txt),* trong khi ta cần file không chứa extension txt. Khi tạo theo cách trên, vì file đã tồn tại nên khi lưu Notepad sẽ không tự thêm *.txt*

Hoặc có thể đổi tên để xóa đuôi .txt

```powershell
Rename-Item "old_file_path_with_txt" "new_file_path_without_txt"
```

**Bước 3:** Set Permission cho file (tắt thừa kế -> set Full Control -> kiểm tra)

```powershell
cacls "C:\ProgramData\ssh\administrators_authorized_keys" /inheritance:r
icacls "C:\ProgramData\ssh\administrators_authorized_keys" /grant "Administrators:F" "SYSTEM:F"
icacls "C:\ProgramData\ssh\administrators_authorized_keys"
```

Nếu dòng cuối hiện đại loại như này là thành công

```powershell
BUILTIN\Administrators:(F)
NT AUTHORITY\SYSTEM:(F)
```

Note: Trước khi chạy 2 lệnh đầu, khi chạy lệnh cuối để kiểm tra thì output của **icalcs chứa (I)(F)**, nghĩa là Administrators và SYSTEM có **F**ull Control, nhưng quyền đó được kế thừa **I**nheritance từ folder cha. Ta đã xóa các quyền kế thừa và cấp trực tiếp lại quyền (Full Control). 

Việc này không nên bỏ qua dù ban đầu đã có quyền Full Control, vì OpenSSH khá nhạy với permission; nếu file quá “mở”, nó có thể bỏ qua key.

**Bước 4**: Restart SSH và ssh lại GPU 

```powershell
Restart-Service sshd
Get-Service sshd
ssh desktop_user@desktop_ip_tailscale
ssh -J user@desktop user@gpu-server
```



**THÊM USER VÀO WORKSPACE**

Kiểm tra các thiết bị trong mạng

```powershell
tailscale status
```
