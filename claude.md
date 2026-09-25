Nên lưu vào **`CLAUDE.md`**. Claude Code tự đọc file này ở đầu mỗi phiên, trước khi làm việc.

## 1. Chọn vị trí

| File                        | Phạm vi                     | Khi nào dùng                                                                |
| --------------------------- | --------------------------- | --------------------------------------------------------------------------- |
| `~/.claude/CLAUDE.md`       | Mọi project trên máy        | **Hợp nhất với bạn**, vì "chỉ hướng dẫn, không làm hộ" là thói quen cá nhân |
| `<project>/CLAUDE.md`       | Một project, commit lên git | Quy ước chung cho cả team                                                   |
| `<project>/CLAUDE.local.md` | Một project, chỉ mình bạn   | Thiết lập riêng cho project này, nên thêm vào `.gitignore`                  |

Có thể dùng cả ba cùng lúc, Claude sẽ nạp tất cả. Để xem file nào đang được nạp và mở ra sửa, gõ lệnh `/memory` trong Claude Code.

## 2. Mẫu nội dung (dạng tổng quát)

```markdown
# Cách làm việc với tôi

## Chế độ: hướng dẫn, không làm hộ
- KHÔNG tự sửa file, KHÔNG chạy lệnh thay đổi hệ thống, trừ khi tôi nói rõ "làm đi".
- Chỉ đưa ra: các bước cần làm, lý do, và cú pháp lệnh ở dạng tổng quát.
  Ví dụ: `git checkout -b <tên-nhánh>`, không điền sẵn giá trị cụ thể.
- Được đọc code để giải thích, nhưng chỉ trích đoạn liên quan.
- Cuối mỗi câu trả lời: gợi ý cách tôi tự kiểm tra kết quả.

## Tự cập nhật
- Khi tôi nói "từ giờ…", "luôn…", "đừng bao giờ…", "nhớ là…":
  cập nhật mục phù hợp trong file này, rồi báo lại một dòng đã thêm gì.
- Không xoá quy tắc cũ khi chưa hỏi tôi.

## Quy tắc bổ sung
<!-- Claude sẽ thêm vào đây -->
```

## 3. Cập nhật tự động

**Cách chính:** dùng mục "Tự cập nhật" trong mẫu trên. Mỗi lần bạn nói "từ giờ…", Claude sẽ dùng công cụ Edit để ghi vào file. Nếu không muốn bị hỏi quyền mỗi lần sửa, thêm quyền vào `~/.claude/settings.json`:

```json
{ "permissions": { "allow": ["Edit(~/.claude/CLAUDE.md)"] } }
```

**Có sẵn thêm:** Claude Code đã có bộ nhớ tự động theo từng project, lưu tại `~/.claude/projects/<tên-project>/memory/`. Claude tự ghi vào đó khi bạn góp ý cách làm việc. Tuy vậy, bộ nhớ này chỉ áp dụng cho một project, nên quy tắc cốt lõi vẫn nên để trong `~/.claude/CLAUDE.md`.

**Lựa chọn thay thế:** Claude Code có sẵn output style **"Learning"**, gần giống ý bạn: nó hướng dẫn và để bạn tự viết phần quan trọng. Bạn chọn style này qua `/config`. Nếu muốn tự viết style riêng, tạo file `~/.claude/output-styles/<tên>.md`, bắt đầu bằng phần frontmatter `name:` và `description:`, rồi viết quy tắc bên dưới. Output style thay đổi hẳn cách Claude trả lời, mạnh hơn so với CLAUDE.md.

**Không cần hook:** hook chỉ chạy lệnh shell cố định, không hiểu được câu "từ giờ…" của bạn là một quy tắc mới.

Theo đúng tinh thần bạn muốn, tôi chưa tạo file nào. Nếu cần tôi tạo sẵn `~/.claude/CLAUDE.md` theo mẫu trên, hãy nói "làm đi".


