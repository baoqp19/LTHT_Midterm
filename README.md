# Midterm Project – Implement `ls(1)`

---

**Sinh viên thực hiện:** Phạm Quốc Bảo   
**Mã sinh viên:** 23IT018  
🔗Link truy cập GitHub: https://github.com/baoqp19/LTHT_Midterm

---

## 1. Giới thiệu

Dự án **`my_ls`** là phiên bản rút gọn và tự cài đặt của lệnh **`ls(1)`** trong hệ điều hành UNIX.  
Chương trình được viết hoàn toàn bằng ngôn ngữ **C**, nhằm mục tiêu giúp sinh viên hiểu sâu hơn về **hệ thống tập tin UNIX**, **thao tác với thư mục**, và **lập trình cấp hệ thống (system-level programming)**.  

---

## 2. Mục tiêu của project

- Hiểu cách hệ điều hành quản lý và truy cập hệ thống tập tin.  
- Rèn luyện kỹ năng thao tác với **struct `stat`**, **thư viện `<dirent.h>`**, và các hàm **system call**.  
- Tự cài đặt các tùy chọn cơ bản của `ls` như:
  - Liệt kê thư mục, tệp tin.
  - Hiển thị file ẩn (`-a`).
  - Hiển thị chi tiết (`-l`, `-h`).
  - Hiển thị inode, block, UID/GID, phân loại ký hiệu (`-i`, `-s`, `-n`, `-F`).
  - Sắp xếp theo thời gian, kích thước (`-t`, `-S`, `-r`).
  - Đệ quy thư mục con (`-R`).
  - Xử lý lỗi, thư mục rỗng, quyền truy cập.  
- Áp dụng mô hình **lập trình module hóa**, chia nhỏ project thành nhiều file `.c` và `.h`.

---

## 3. Mức độ hoàn thiện

| Tính năng | Trạng thái | Ghi chú |
|------------|-------------|---------|
| Liệt kê thư mục & tệp tin |  Hoàn thiện | Hiển thị chính xác nội dung |
| File ẩn (`-a`, `-A`) |  Hoàn thiện | Đúng hành vi UNIX |
| Hiển thị chi tiết (`-l`, `-h`, `-n`) |  Hoàn thiện | Có quyền, UID/GID, kích thước dễ đọc |
| Phân loại ký hiệu (`-F`) |  Hoàn thiện | Hiển thị `/`, `*`, `@` đúng chuẩn |
| Sắp xếp (`-S`, `-t`, `-r`, `-f`) |  Hoàn thiện | Đúng thứ tự, ổn định |
| Đệ quy (`-R`) |  Hoàn thiện | Hiển thị đầy đủ thư mục con |
| Xử lý lỗi truy cập, thư mục rỗng |  Hoàn thiện | Không segfault |
| Tích hợp Makefile |  Hoàn thiện | Biên dịch nhanh gọn |
| README và báo cáo |  Hoàn thiện | Phục vụ nộp bài & GitHub |

---

## 4. Cấu trúc thư mục dự án

```
myls/                # Thư mục gốc của dự án
├── include/         # Chứa các file header (.h)
│   ├── list_dir.h
│   ├── file_info.h
│   ├── options.h
│   └── utils.h
├── src/             # Chứa các file source (.c)
│   ├── main.c
│   ├── list_dir.c
│   ├── file_info.c
│   ├── options.c
│   └── utils.c
├── makefile         # File Makefile để build project
└── my_ls            # File thực thi sau khi build
```

---

## 5. Mô tả và phân tích các module chính

Dự án được chia thành nhiều module độc lập, mỗi module đảm nhiệm một vai trò cụ thể trong quy trình xử lý lệnh `ls`.  
Cách chia tách này giúp code dễ đọc, dễ bảo trì, và tuân thủ mô hình **modular programming**.

---

### 5.1 `main.c` — Điểm khởi đầu chương trình

**Chức năng:**  
- Đọc tham số dòng lệnh.  
- Gọi module `options.c` để xử lý flag.  
- Gọi `list_directory()` để bắt đầu hiển thị nội dung thư mục.  

**Đoạn code tiêu biểu:**
```c
int main(int argc, char *argv[]) {
    Options opt = parse_options(argc, argv);
    const char *path = (opt.path) ? opt.path : ".";
    list_directory(path, opt);
    return 0;
}
```

**Phân tích:**  
- Hàm `parse_options()` xử lý toàn bộ tham số truyền vào (ví dụ `-alhR`).  
- Mặc định, nếu người dùng không chỉ định đường dẫn, chương trình sẽ liệt kê thư mục hiện tại `"."`.  
- Mọi xử lý hiển thị đều được ủy quyền cho module `list_dir.c`.  

---

### 5.2 `options.c` — Phân tích và lưu cờ tùy chọn (flags)

**Chức năng:**  
- Nhận mảng `argv[]` từ `main.c`.  
- Dò từng ký tự trong các flag để bật/tắt các tùy chọn trong struct `Options`.  

**Đoạn code tiêu biểu:**
```c
Options parse_options(int argc, char *argv[]) {
    Options opt = {0};
    for (int i = 1; i < argc; i++) {
        if (argv[i][0] == '-') {
            for (int j = 1; argv[i][j]; j++) {
                switch (argv[i][j]) {
                    case 'a': opt.show_all = 1; break;
                    case 'l': opt.long_format = 1; break;
                    case 'R': opt.recursive = 1; break;
                    case 'h': opt.human = 1; break;
                    case 't': opt.sort_time = 1; break;
                    default:  fprintf(stderr, "Unknown option: -%c\n", argv[i][j]);
                }
            }
        } else opt.path = argv[i];
    }
    if (!opt.path) opt.path = ".";
    return opt;
}
```

**Phân tích:**  
- Cấu trúc `Options` lưu các cờ như `-a`, `-l`, `-R` bằng biến boolean.  
- Cho phép kết hợp nhiều flag trong một tham số (ví dụ `-alh`).  
- Giúp chương trình tách biệt rõ phần “giao tiếp người dùng” và phần “xử lý hiển thị”.

---

### 5.3 `list_dir.c` — Liệt kê thư mục và duyệt đệ quy

**Chức năng:**  
- Mở thư mục (`opendir()`), đọc từng entry (`readdir()`),  
  bỏ qua hoặc hiển thị tùy theo flag `-a`, `-A`.  
- Nếu có `-R`, gọi đệ quy để xử lý thư mục con.  

**Đoạn code tiêu biểu:**
```c
void list_directory(const char *path, Options opt) {
    DIR *dir = opendir(path);
    if (!dir) { perror("ls"); return; }

    struct dirent *entry;
    while ((entry = readdir(dir)) != NULL) {
        if (!opt.show_all && entry->d_name[0] == '.') continue;
        print_file_info(path, entry->d_name, &opt);
    }

    if (opt.recursive) {
        rewinddir(dir);
        while ((entry = readdir(dir)) != NULL) {
            if (entry->d_type == DT_DIR &&
                strcmp(entry->d_name, ".") && strcmp(entry->d_name, "..")) {
                char subpath[512];
                snprintf(subpath, sizeof(subpath), "%s/%s", path, entry->d_name);
                printf("\n%s:\n", subpath);
                list_directory(subpath, opt);
            }
        }
    }

    closedir(dir);
}
```

**Phân tích:**  
- Sử dụng `struct dirent` để truy xuất tên và loại đối tượng trong thư mục.  
- Kiểm tra `entry->d_type` để xác định thư mục (`DT_DIR`) cho đệ quy.  
- Gọi `print_file_info()` để hiển thị chi tiết từng tệp tin.  
- Xử lý cẩn thận với `.` và `..` để tránh vòng lặp vô hạn khi đệ quy.  

---

### 5.4 `file_info.c` — Hiển thị thông tin chi tiết file

**Chức năng:**  
- Lấy thông tin file bằng `lstat()`.  
- In quyền truy cập, liên kết, UID, GID, kích thước, và thời gian.  
- Hỗ trợ hiển thị ký hiệu phân loại (`-F`).  

**Đoạn code tiêu biểu:**
```c
void print_file_info(const char *path, const char *filename, Options opt) {
    struct stat st;
    char fullpath[512];
    snprintf(fullpath, sizeof(fullpath), "%s/%s", path, filename);
    if (lstat(fullpath, &st) < 0) return;

    if (opt.long_format) {
        print_permissions(st.st_mode);
        printf(" %3ld ", st.st_nlink);

        struct passwd *pw = getpwuid(st.st_uid);
        struct group *gr = getgrgid(st.st_gid);
        printf("%-8s %-8s %8ld ", 
               pw ? pw->pw_name : "?", 
               gr ? gr->gr_name : "?", 
               st.st_size);

        char timebuf[32];
        strftime(timebuf, sizeof(timebuf), "%b %d %H:%M", localtime(&st.st_mtime));
        printf("%s %s\n", timebuf, filename);
    } else printf("%s  ", filename);
}
```

**Phân tích:**  
- `lstat()` thay vì `stat()` để giữ nguyên thông tin symbolic link.  
- `getpwuid()` và `getgrgid()` ánh xạ UID/GID sang tên người dùng và nhóm.  
- Dùng `strftime()` để định dạng thời gian giống `ls -l` thật.  
- Nếu bật flag `-h`, kích thước sẽ được in theo đơn vị KB, MB qua hàm `human_readable()`.  

---

### 5.5 `utils.c` — Hàm tiện ích và xử lý phụ trợ

**Chức năng:**  
- Cung cấp các hàm hỗ trợ cho `list_dir.c` và `file_info.c`.  
- Bao gồm: nối chuỗi đường dẫn, so sánh để sắp xếp, chuyển kích thước sang định dạng dễ đọc.  

**Đoạn code tiêu biểu:**
```c
void human_readable(off_t size, char *buf) {
    const char *units[] = {"B","K","M","G"};
    double s = (double)size;
    int i = 0;
    while (s >= 1024 && i < 3) { s /= 1024; i++; }
    sprintf(buf, "%.1f%s", s, units[i]);
}
```

**Phân tích:**  
- Giúp hiển thị kích thước file thân thiện với người dùng (`ls -lh`).  
- Các hàm khác trong module này như `join_path()`, `compare_by_size()`… được sử dụng để sắp xếp khi có flag `-S` hoặc `-t`.  
- Giữ cho mã trong các module chính gọn gàng và tách biệt rõ phần xử lý logic phụ.  

---

### 5.6 `Makefile` — Quản lý quá trình biên dịch

**Chức năng:**  
- Tự động hóa việc biên dịch toàn bộ mã nguồn và tạo tệp thực thi.  

**Đoạn code tiêu biểu:**
```makefile
CC = gcc
CFLAGS = -Wall -Iinclude
OBJ = src/main.o src/utils.o src/list_dir.o src/file_info.o src/options.o

my_ls: $(OBJ)
	$(CC) $(CFLAGS) -o my_ls $(OBJ)

clean:
	rm -f src/*.o my_ls
```

**Phân tích:**  
- `-Iinclude` giúp trình biên dịch nhận diện tất cả header `.h` trong thư mục `include/`.  
- Cấu trúc Makefile gọn gàng, dễ hiểu và đáp ứng yêu cầu “project phải có Makefile”.  


---

## 6. Build và khởi động dự án

### Biên dịch chương trình
```bash
make
```

Sau khi chạy, file thực thi `my_ls` sẽ được tạo trong thư mục gốc.

### Chạy chương trình
```bash
./my_ls [OPTIONS] [PATH]
```

**Ví dụ:**
```bash
./my_ls
./my_ls -a
./my_ls -lh
./my_ls -R
./my_ls -alhR /etc
```

---

## 7. Minh họa các Flag

| **Lệnh** | **Ý nghĩa** | **Minh họa (Screenshot)** |
|-----------|--------------|-----------------------------|
| `./my_ls` | Liệt kê tệp và thư mục trong thư mục hiện tại |<img width="503" height="123" alt="image" src="https://github.com/user-attachments/assets/0eb27cc5-11d7-4307-8417-3badb13893a4" />|
| `./my_ls -a` | Hiển thị tất cả tệp (bao gồm tệp ẩn `.` và `..`) |<img width="671" height="113" alt="image" src="https://github.com/user-attachments/assets/683e7792-9ce2-4595-ab02-ed97156515c0" />|
| `./my_ls -A` | Giống `-a` nhưng bỏ qua `.` và `..` |<img width="523" height="117" alt="image" src="https://github.com/user-attachments/assets/6b251d11-d6cc-4db5-bebe-a88f1d3ebb42" />|
| `./my_ls -l` | Hiển thị chi tiết từng tệp: quyền, số liên kết, chủ sở hữu, nhóm, kích thước, thời gian |<img width="750" height="181" alt="image" src="https://github.com/user-attachments/assets/93fe700a-2b8a-4a98-b142-31ff5305572d" />|
| `./my_ls -lh` | Giống `-l` nhưng kích thước dễ đọc (KB, MB) |<img width="718" height="179" alt="image" src="https://github.com/user-attachments/assets/ae69ea49-fa97-4fbc-8b6f-a2b474e139ca" />|
| `./my_ls -R` | Đệ quy – liệt kê toàn bộ thư mục con | <img width="1116" height="244" alt="image" src="https://github.com/user-attachments/assets/0ddcc568-c9c4-420f-b5f3-409751f0d1c0" />|
| `./my_ls -F` | Phân loại ký hiệu: `/` (thư mục), `*` (thực thi), `@` (liên kết) |<img width="566" height="42" alt="image" src="https://github.com/user-attachments/assets/ecec88e9-4cde-4c45-8dcc-0970681723be" />|
| `./my_ls -t` | Sắp xếp theo thời gian chỉnh sửa (mới nhất trước) |<img width="507" height="85" alt="image" src="https://github.com/user-attachments/assets/6c496fdf-c731-4050-99b3-9b2c6d2c464a" />|
| `./my_ls -S` | Sắp xếp theo kích thước tệp (lớn nhất trước) |<img width="531" height="93" alt="image" src="https://github.com/user-attachments/assets/1a1213b7-bcea-4e6c-9dad-2f4c36e167f3" />|
| `./my_ls -r` | Đảo ngược thứ tự sắp xếp |<img width="515" height="94" alt="image" src="https://github.com/user-attachments/assets/0c05c3ec-aac5-421b-bed4-67208ecc98a5" />|
| `./my_ls -n` | Hiển thị UID/GID dạng số thay vì tên | <img width="404" height="108" alt="image" src="https://github.com/user-attachments/assets/28269a77-66cc-4861-99c3-dbc1d775e861" />|
| `./my_ls -is` | In thêm số inode và block của tệp |<img width="870" height="98" alt="image" src="https://github.com/user-attachments/assets/56d7280d-66ac-4013-b65a-411c7804f4a8" />|
| `./my_ls -alhR` | Kết hợp nhiều tùy chọn cùng lúc (ẩn + chi tiết + dễ đọc + đệ quy) |<img width="731" height="727" alt="image" src="https://github.com/user-attachments/assets/68c34078-d80e-41ec-aa7d-c7e2995ffcef" />
|

---

## 8. Kết luận

Dự án **my_ls** đã mô phỏng chính xác hành vi cơ bản của lệnh `ls` trên UNIX/Linux.  
Chương trình được chia module rõ ràng, có tính ổn định, không gặp lỗi phân đoạn (segfault), và dễ mở rộng cho các tính năng khác trong tương lai.  

Dự án thể hiện:
- Kỹ năng lập trình hệ thống (System Programming).  
- Hiểu biết về cấu trúc file, quyền truy cập, và thao tác thư mục.  
- Tuân thủ quy tắc viết code sạch, dễ bảo trì và có tính mở rộng.





