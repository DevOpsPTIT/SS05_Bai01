# Báo Cáo Thực Hành: Khôi Phục Commit Đã Mất Bằng Git Reflog

- **Bài tập**: Bài 1 - Khôi phục commit đã mất bằng Git Reflog (Session 05 / Git Intermediate & Advanced Recovery)
- **Đường dẫn nộp bài**: `SS05/Bai01/README.md`
- **GitHub Remote Repository**: `https://github.com/DevOpsPTIT/SS05_Bai01.git`

---

## 1. Bối cảnh & Mục tiêu

### Bối cảnh
Trong quá trình phát triển dự án phần mềm hoặc vận hành DevOps, lập trình viên có thể vô tình thực thi lệnh `git reset --hard HEAD~1` hoặc xóa/ghi đè nhánh làm việc. Lệnh này khiến cho commit quan trọng vừa tạo bị loại bỏ khỏi lịch sử làm việc của nhánh (`git log`), đồng thời làm sạch toàn bộ Working Directory và Staging Area. Do thông tin commit không còn hiển thị trong `git log`, người dùng rất dễ hoang mang và lầm tưởng dữ liệu mã nguồn đã bị xóa vĩnh viễn khỏi đĩa cứng.

### Mục tiêu
- **Hiểu rõ bản chất Git Reflog**: Nắm vững cơ chế theo dõi và ghi nhận mọi sự thay đổi của con trỏ `HEAD` trên repository cục bộ.
- **Truy vết tham chiếu bằng `git reflog`**: Tra cứu chính xác SHA-1 hash của commit bị rớt khỏi lịch sử làm việc.
- **Khôi phục 100% mã nguồn**: Đưa repository trở về đúng trạng thái ban đầu trước khi bị `reset --hard` mà **không cần viết lại mã nguồn thủ công**.
- **Xây dựng quy trình báo cáo chuẩn**: Trích xuất log thực tế chứng minh quy trình khôi phục thành công.

---

## 2. Thông Số Cấu Hình & Ràng Buộc

| Thông số / Cấu hình | Giá trị lựa chọn / Thao tác | Ghi chú / Ràng buộc |
| :--- | :--- | :--- |
| **Thư mục bài tập** | `SS05/Bai01` | Repository quản lý cục bộ |
| **Tệp tin thử nghiệm** | `feature.txt` | Nội dung: `Day la tinh nang quan trong` |
| **Commit cần khôi phục** | `"Them tinh nang quan trong"` | Commit bị mất sau lệnh hard reset |
| **Lệnh giả lập sự cố** | `git reset --hard HEAD~1` | Xóa commit khỏi `git log` & xóa tệp trong working tree |
| **Lệnh truy vết Reflog** | `git reflog` | Tìm hash commit cũ (VD: `0726fdc`) |
| **Lệnh khôi phục** | `git reset --hard <hash>` | Kéo HEAD & Working tree về commit bị mất |
| **Cơ chế khôi phục** | **Bắt buộc dùng Reflog** | Tuyệt đối không viết lại code thủ công |

---

## 3. Cơ Chế Hoạt Động Của Git Reflog vs Git Log

### 1. Phân biệt `git log` và `git reflog`

| Tiêu chí | `git log` | `git reflog` (Reference Log) |
| :--- | :--- | :--- |
| **Phạm vi quản lý** | Xem lịch sử các commit thuộc nhánh hiện tại (DAG hierarchy) | Ghi lại **tất cả lượt di chuyển của con trỏ `HEAD`** cục bộ |
| **Tác động của `reset --hard`** | Mất dấu commit bị lùi (không còn thuộc nhánh) | **Vẫn giữ lại nhật ký** thao tác commit & reset |
| **Tính riêng tư** | Chia sẻ công khai khi `push` lên Remote (GitHub/GitLab) | **Chỉ tồn tại cục bộ** tại máy cá nhân (không push lên remote) |
| **Thời gian lưu trữ** | Vĩnh viễn (cho đến khi commit bị xóa/rebase) | Mặc định 90 ngày (dành cho reachable) hoặc 30 ngày (unreachable) |

### 2. Sơ đồ minh họa cơ chế cứu dữ liệu của Reflog

```
[Trạng thái ban đầu]
Commit A (b98cce3) <-- Commit B (0726fdc - Them tinh nang quan trong) [HEAD -> master]

[Sau khi chạy git reset --hard HEAD~1]
Commit A (b98cce3) [HEAD -> master]
    \
     `--> Commit B (0726fdc) (Trở thành Dangling/Orphaned Commit - Không hiển thị trong git log)

[Tra cứu git reflog]
Reflog ghi nhận: HEAD@{1} chính là commit 0726fdc!

[Sau khi chạy git reset --hard 0726fdc]
Commit A (b98cce3) <-- Commit B (0726fdc - Them tinh nang quan trong) [HEAD -> master]  <-- ĐÃ KHÔI PHỤC!
```

---

## 4. Các Bước Thực Hiện Chi Tiết

### Bước 1: Khởi tạo dự án & Tạo commit tính năng quan trọng

Khởi tạo kho lưu trữ Git cục bộ, tạo tệp `feature.txt` và đóng gói commit:

```bash
# Khởi tạo repository
git init

# Tạo tệp tin chứa mã nguồn quan trọng
echo "Day la tinh nang quan trong" > feature.txt

# Đưa tệp vào Staging Area và thực hiện Commit
git add feature.txt
git commit -m "Them tinh nang quan trong"
```

*Kết quả*: Git tạo commit mới với thông điệp `"Them tinh nang quan trong"` (Mã hash ví dụ: `0726fdc`).

---

### Bước 2: Giả lập sự cố xóa nhầm commit (`git reset --hard`)

Thực hiện lùi lịch sử commit về phía trước 1 bước và xóa sạch Working Directory:

```bash
git reset --hard HEAD~1
```

*Kết quả Terminal*: `HEAD is now at b98cce3 Initial commit`. Tệp `feature.txt` bị xóa hoàn toàn khỏi thư mục làm việc.

---

### Bước 3: Kiểm tra xác nhận commit bị mất qua `git log`

Chạy lệnh kiểm tra lịch sử commit:

```bash
git log --oneline
```

*Kết quả Terminal*:
```console
b98cce3 Initial commit
```
> **Đánh giá**: Commit `"Them tinh nang quan trong"` đã không còn xuất hiện trong cây lịch sử `git log`.

---

### Bước 4: Truy vết mã Hash của commit bị mất bằng `git reflog`

Chạy lệnh xem nhật ký thao tác con trỏ `HEAD`:

```bash
git reflog
```

*Kết quả Terminal*:
```console
b98cce3 HEAD@{0}: reset: moving to HEAD~1
0726fdc HEAD@{1}: commit: Them tinh nang quan trong
b98cce3 HEAD@{2}: commit (initial): Initial commit
```

> **Phân tích Reflog**:
> - Dòng `HEAD@{0}` thể hiện hành động `reset` vừa thực hiện.
> - Dòng `HEAD@{1}` cho thấy trạng thái trước đó chính là commit `0726fdc` chứa thông điệp `"Them tinh nang quan trong"`.

---

### Bước 5: Khôi phục commit bằng `git reset --hard`

Sử dụng mã SHA-1 hash tìm được từ Reflog để đưa nhánh làm việc hiện tại quay lại commit đã mất:

```bash
git reset --hard 0726fdc
```
*(Hoặc có thể dùng tham chiếu Reflog: `git reset --hard HEAD@{1}`)*

*Kết quả Terminal*:
```console
HEAD is now at 0726fdc Them tinh nang quan trong
```

---

## 5. Trích Xuất Log Terminal Chứng Minh (Verification Logs)

### 1. Kiểm tra lại lịch sử commit (`git log --oneline`)

```console
$ git log --oneline
0726fdc Them tinh nang quan trong
b98cce3 Initial commit
```
> **Đánh giá**: Lịch sử commit đã khôi phục lại đầy đủ commit `"Them tinh nang quan trong"`.

---

### 2. Kiểm tra nội dung tệp tin đã khôi phục (`feature.txt`)

```console
$ Get-Content feature.txt
Day la tinh nang quan trong
```
> **Đánh giá**: Tệp `feature.txt` đã xuất hiện trở lại trong thư mục làm việc với đúng nội dung ban đầu.

---

### 3. Kiểm tra nhật ký Reflog đầy đủ (`git reflog`)

```console
$ git reflog
0726fdc HEAD@{0}: reset: moving to 0726fdc
b98cce3 HEAD@{1}: reset: moving to HEAD~1
0726fdc HEAD@{2}: commit: Them tinh nang quan trong
b98cce3 HEAD@{3}: commit (initial): Initial commit
```
> **Đánh giá**: Toàn bộ quá trình từ lúc tạo commit -> lùi commit -> khôi phục commit đều được Reflog ghi lại rõ ràng, minh bạch.

---

### 4. Kiểm tra trạng thái làm việc (`git status`)

```console
$ git status
On branch master
nothing to commit, working tree clean
```
> **Đánh giá**: Cây làm việc sạch sẽ, không có lỗi hay tệp tin rác phát sinh.

---

## 6. Các Phương Pháp Khôi Phục Commit Nâng Cao Với Reflog

Trong thực tế vận hành dự án, ngoài việc dùng `git reset --hard`, lập trình viên có thể áp dụng các chiến lược khôi phục linh hoạt khác dựa vào kết quả của `git reflog`:

### Phương pháp 1: Khôi phục trực tiếp trên nhánh hiện tại (`git reset --hard`)
- **Cú pháp**: `git reset --hard <commit-hash>`
- **Ưu điểm**: Nhanh chóng, khôi phục ngay lập tức cả HEAD và Working Directory.
- **Trường hợp áp dụng**: Khi chắc chắn muốn đè trạng thái hiện tại về commit cũ.

### Phương pháp 2: Khôi phục sang một nhánh mới an toàn (`git checkout -b`)
- **Cú pháp**: `git checkout -b feature-recovered <commit-hash>`
- **Ưu điểm**: An toàn tuyệt đối, không làm ảnh hưởng đến nhánh hiện tại. Dễ dàng kiểm tra mã nguồn trước khi merge lại.
- **Trường hợp áp dụng**: Khi đang phân vân hoặc cần so sánh mã nguồn giữa 2 trạng thái.

### Phương pháp 3: Bốc commit bị mất vào nhánh hiện tại (`git cherry-pick`)
- **Cú pháp**: `git cherry-pick <commit-hash>`
- **Ưu điểm**: Giữ nguyên các commit mới tạo sau đó và chỉ áp dụng đúng thay đổi của commit bị mất.
- **Trường hợp áp dụng**: Khi đã tạo thêm nhiều commit mới sau khi lỡ tay xóa commit cũ.

---

## 7. Kết Luận & Bài Học Kinh Nghiệm

1. **Git không bao giờ xóa dữ liệu ngay lập tức**: Tất cả các commit sau khi được tạo ra đều được lưu trong Git Object Database. Ngay cả khi bị gỡ khỏi nhánh (`git reset`), commit đó vẫn tồn tại dưới dạng *dangling commit* và có thể khôi phục qua Reflog.
2. **`git reflog` là "Phao cứu sinh" cục bộ**: Reflog là công cụ cực kỳ mạnh mẽ giúp truy vết lại mọi hành động sai lầm như `reset --hard`, `rebase` hỏng, hoặc vô tình xóa nhánh (`git branch -D`).
3. **Bài học an toàn**:
   - Trước khi thực hiện các lệnh nguy hiểm (`reset --hard`, `rebase`, `stash drop`), nên kiểm tra kỹ trạng thái làm việc.
   - Thường xuyên push các nhánh làm việc quan trọng lên Remote Server để có thêm lớp sao lưu thứ hai.
