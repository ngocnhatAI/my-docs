# Sự cố: Không SSH được vào server 192.168.1.57

> Ghi lại ngày 2026-09-23. Đã xử lý xong.

---

## TL;DR — Gặp lại thì làm gì

```powershell
ipconfig | Select-String "IPv4"
```

- Ra **`192.168.1.x`** → đúng mạng, SSH được: `ssh yootek`
- Ra **`10.10.x.x`** → **bắt nhầm access point**. Ngắt Wi-Fi, nối lại, kiểm tra lại. Lặp đến khi ra `192.168.1.x`.

**Tuyệt đối không đặt IP tĩnh.** Không giải quyết được gì, mà còn làm mất DHCP → văng mạng (xem phần Sai lầm bên dưới).

---

## Nguyên nhân thật

Văn phòng có **nhiều access point phát cùng tên SSID `YOOTEK HOLDINGS`**, nhưng chúng nối vào **hai mạng (VLAN) khác nhau**:

| BSSID (địa chỉ AP)  | Dải IP cấp ra  | Gateway       | Thấy server `.57`? |
| ------------------- | -------------- | ------------- | ------------------ |
| `c2:74:ad:2a:cf:45` | `10.10.x.x/16` | `10.10.0.1`   | ❌ Không            |
| `c0:74:ad:3a:d0:71` | `192.168.1.x`  | `192.168.1.1` | ✅ Có               |

Máy bám nhầm AP thuộc dải `10.10.x.x` → nằm khác mạng với server → ping timeout.

Cùng một tên Wi-Fi, nhưng vào nhầm AP là mất kết nối server. Nhìn tên SSID không phát hiện được — phải nhìn IP.

---

## Quá trình chẩn đoán

### 1. Xác định máy đang ở đâu

```powershell
Get-NetIPConfiguration | Format-List InterfaceAlias, IPv4Address, IPv4DefaultGateway
```

Kết quả lúc lỗi: Wi-Fi `10.10.2.245/16`, gateway `10.10.0.1`. Khác dải `192.168.1.x` của server → đây là manh mối đầu tiên.

### 2. Traceroute — khoanh vùng điểm chết

```powershell
tracert -d -h 5 -w 1000 192.168.1.57
```

```
1   110 ms   3 ms   5 ms  10.10.0.1
2     *       *      *    Request timed out.
```

Tới được gateway (hop 1), chết ở hop 2 → router không định tuyến sang mạng server.

### 3. Bằng chứng từng kết nối được

```powershell
Get-Content "$env:USERPROFILE\.ssh\known_hosts" | ForEach-Object { ($_ -split ' ')[0] }
```

Có sẵn `192.168.1.57`, `.101`, `.150` → trước đây vào được bình thường. Khẳng định đây là lỗi mạng, không phải sai cấu hình SSH.

### 4. Thử IP tĩnh — và loại trừ giả thuyết

Giả thuyết: nếu Wi-Fi cùng tầng vật lý (layer 2) với server, gán IP `192.168.1.x` sẽ tới được qua ARP.

```powershell
netsh interface ipv4 add address name="Wi-Fi" address=192.168.1.180 mask=255.255.255.0
ping -n 4 192.168.1.57
```

```
Reply from 192.168.1.180: Destination host unreachable.
```

**Đọc kỹ dòng này.** Phản hồi đến từ **chính máy mình** (`.180`), không phải từ router. Máy đã gửi ARP broadcast hỏi "ai là `.57`?" và **không ai trả lời**.

→ Kết luận: AP đang bám **không cùng mạng vật lý** với server. Giả thuyết sai, loại trừ dứt điểm.

### 5. Khôi phục DHCP → tình cờ nối lại đúng AP

Sau khi trả DHCP và kết nối lại, máy chọn AP khác (`c0:74:...`), nhận `192.168.1.34`, ping `.57` thông 2-4ms. Xong.

---

## Sai lầm đã mắc — nhớ để tránh

### `netsh add address` phá DHCP

Tưởng rằng lệnh này **thêm** IP phụ mà giữ nguyên DHCP. **Sai.** Nó chuyển cả interface sang chế độ static.

Hậu quả: khi xóa IP tĩnh đi, card không còn IP nào và cũng không tự xin DHCP nữa → rơi về APIPA `169.254.x.x` → **mất mạng hoàn toàn**.

Triệu chứng nhận biết:

```
Adapter Wi-Fi is not enabled for DHCP.
```

**Cách khôi phục** (cần PowerShell Admin):

```powershell
netsh interface ipv4 set address name="Wi-Fi" source=dhcp
netsh interface ipv4 set dnsservers name="Wi-Fi" source=dhcp
ipconfig /renew "Wi-Fi"
```

### Nhầm hotspot iPhone với mạng công ty

Có lúc máy nhận `172.20.10.12`, mask `255.255.255.240`, gateway `172.20.10.1`.

**Dải `172.20.10.x/28` là đặc trưng của hotspot iPhone** — Apple luôn dùng đúng dải này. Thấy IP kiểu này nghĩa là đang bám hotspot điện thoại, không phải Wi-Fi công ty.

---

## Bảng tra IP nhanh

| IP nhận được  | Nghĩa là gì         | Vào được server?               |
| ------------- | ------------------- | ------------------------------ |
| `192.168.1.x` | Đúng AP, đúng mạng  | ✅                              |
| `10.10.x.x`   | Nhầm AP (khác VLAN) | ❌ Ngắt/nối lại Wi-Fi           |
| `172.20.10.x` | Hotspot iPhone      | ❌ Đổi sang Wi-Fi công ty       |
| `169.254.x.x` | APIPA — DHCP hỏng   | ❌ Khôi phục DHCP (lệnh ở trên) |

---

## Lệnh hay dùng

```powershell
# IP hiện tại
ipconfig | Select-String "IPv4"

# Đang bám AP nào (xem BSSID)
netsh wlan show interfaces | Select-String "SSID|BSSID|Signal"

# Chi tiết IP + gateway
Get-NetIPConfiguration | Format-List InterfaceAlias, IPv4Address, IPv4DefaultGateway

# Server có phản hồi ARP không
arp -a | Select-String "192.168.1"

# Khoanh vùng điểm chết
tracert -d -h 5 192.168.1.57
```

---

## Thông tin môi trường

**Server:** `192.168.1.57`, user `yootek`, MAC `10-7c-61-d2-00-5d`

**SSH alias đã có sẵn** trong `~/.ssh/config` — dùng `ssh yootek`, khỏi gõ IP:

```
Host yootek
  HostName 192.168.1.57
  User yootek

Host pc_anh_npthang
  HostName 192.168.1.101
  User npthang
```

**Wi-Fi:**

- `YOOTEK HOLDINGS` — mạng công ty, **có 2 AP khác VLAN** (đây là gốc rễ vấn đề)
- `YOOTEK HOLDINGS VIP` — mạng cá nhân, **không vào được server**

**Tailscale:** đang chạy, IP máy này `100.100.39.118`. Server `.57` **chưa cài** — nếu cài thì SSH được từ mọi nơi, khỏi phụ thuộc AP nào.

---

## Nên đề xuất với IT

Vấn đề gốc nằm ở hạ tầng, không sửa được từ máy cá nhân:

1. **Hai AP cùng SSID nhưng khác VLAN** → đề nghị tách tên SSID riêng, hoặc gộp về một VLAN. Đây là cấu hình dễ gây rối cho mọi người, không riêng mình.
2. **Hoặc** mở route giữa `10.10.0.0/16` và `192.168.1.0/24`.
3. **Hoặc** cài Tailscale lên server — giải pháp chủ động nhất, không cần chờ IT.

### Mẫu tin nhắn báo leader

> Em đã tìm ra nguyên nhân: Wi-Fi YOOTEK HOLDINGS có 2 access point cùng tên nhưng khác VLAN. AP `c2:74:ad:2a:cf:45` cấp dải 10.10.x.x (không thấy server), AP `c0:74:ad:3a:d0:71` cấp dải 192.168.1.x (thấy server). Lúc máy em bắt nhầm AP đầu thì mất kết nối server, ping báo Destination host unreachable do ARP không có phản hồi. Đặt IP tĩnh không giải quyết được vì khác mạng vật lý. Nhờ anh đề xuất IT tách SSID riêng cho 2 mạng, hoặc gộp chung VLAN ạ.

---

## Bài học

1. **Ping timeout trước hết là kiểm tra mình đang ở mạng nào** — `ipconfig` trước khi đụng bất cứ thứ gì.
2. **Cùng tên Wi-Fi không có nghĩa cùng mạng.** Phải xem BSSID và dải IP.
3. **"Destination host unreachable" từ chính IP của mình** = ARP không ai trả lời = khác mạng vật lý. Khác hẳn "Request timed out" (có thể do firewall).
4. **IP tĩnh không sửa được lỗi khác mạng vật lý.** Gói tin không tới được đích thì đặt IP gì cũng vô nghĩa.
5. **Chỉ dẫn chung chung như "fix IP tĩnh 192.168..." thì hỏi lại cho rõ** — IP nào, gateway nào, nối mạng nào. Đoán mò IP còn có thể gây xung đột với máy khác.
6. **`known_hosts` và `~/.ssh/config` là manh mối tốt** — cho biết trước đây từng kết nối được, giúp khoanh vùng lỗi mới phát sinh.
