# Bài 2: Khởi tạo User thường và thiết lập đặc quyền quản trị (Sudoers)

## Mục tiêu
- Tạo tài khoản người dùng thường `devops` để làm việc hằng ngày thay vì dùng `root`, theo nguyên tắc **Least Privilege** (đặc quyền tối thiểu).
- Cấp quyền quản trị cho `devops` thông qua nhóm `sudo`.
- Sao chép SSH Key từ `root` sang `devops` và phân quyền đúng để `devops` đăng nhập được bằng key.

## Môi trường

| Thông tin | Giá trị |
|---|---|
| Máy chủ | VPS AZVPS – Ubuntu 22.04.5 LTS |
| Hostname | `wean2607` |
| IPv4 | `160.187.229.88` |
| User mới | `devops` (thuộc nhóm `sudo`) |
| Private key trên máy cá nhân | `%USERPROFILE%\.ssh\id_ed25519` |

---

## Các bước thực hiện

> Các lệnh từ Bước 1 đến Bước 4 được chạy trên VPS bằng tài khoản `root`, sau khi đã đăng nhập bằng lệnh `ssh -i %USERPROFILE%\.ssh\id_ed25519 root@160.187.229.88`.

### Bước 1 – Tạo user `devops`

```bash
adduser devops
```

- Đặt mật khẩu cho `devops` (nhập 2 lần). Mật khẩu này dùng khi chạy lệnh `sudo`.
- Các thông tin Full Name, Room Number… để trống bằng cách nhấn Enter, cuối cùng xác nhận `Y`.
- `adduser` tự tạo thư mục home `/home/devops` và nhóm riêng `devops`.

### Bước 2 – Thêm `devops` vào nhóm `sudo`

```bash
usermod -aG sudo devops
groups devops
```

- `-aG` nghĩa là **thêm** (append) user vào nhóm `sudo` mà không xoá các nhóm hiện có.
- Trên Ubuntu, file `/etc/sudoers` mặc định có dòng `%sudo ALL=(ALL:ALL) ALL`, nên mọi thành viên nhóm `sudo` đều có quyền quản trị.

Kết quả thực tế:

```text
root@wean2607:~# groups devops
devops : devops sudo
```

### Bước 3 – Sao chép SSH Key từ `root` sang `devops`

```bash
rsync --archive --chown=devops:devops ~/.ssh /home/devops/
```

- `--archive` giữ nguyên cấu trúc thư mục và quyền của file.
- `--chown=devops:devops` chuyển chủ sở hữu sang `devops`. Nếu thiếu bước này, SSH sẽ từ chối key vì file vẫn thuộc quyền `root`.

### Bước 4 – Phân quyền thư mục `.ssh` và file `authorized_keys`

```bash
chmod 700 /home/devops/.ssh
chmod 600 /home/devops/.ssh/authorized_keys
ls -la /home/devops/.ssh
```

| Đối tượng | Quyền | Ý nghĩa |
|---|---|---|
| `/home/devops/.ssh` | `700` (`drwx------`) | Chỉ `devops` được đọc, ghi và truy cập thư mục |
| `/home/devops/.ssh/authorized_keys` | `600` (`-rw-------`) | Chỉ `devops` được đọc và ghi |

SSH (cơ chế `StrictModes`) sẽ **từ chối đăng nhập bằng key** nếu các file này có quyền rộng hơn mức trên, hoặc không thuộc sở hữu của user.

Kết quả thực tế:

```text
root@wean2607:~# ls -la /home/devops/.ssh
total 12
drwx------ 2 devops devops 4096 Aug  3  2022 .
drwxr-x--- 3 devops devops 4096 Oct  1 21:07 ..
-rw------- 1 devops devops  106 Oct  1 20:36 authorized_keys
```

![Tạo user, thêm nhóm sudo và phân quyền](images/01-create-user.png)

### Bước 5 – Kiểm tra từ máy cá nhân

Thoát khỏi phiên `root` bằng lệnh `exit`, rồi đăng nhập lại bằng `devops`:

```bat
ssh -i %USERPROFILE%\.ssh\id_ed25519 devops@160.187.229.88
```

Sau khi vào được máy chủ, chạy:

```bash
whoami
sudo whoami
```

---

## Log Terminal kiểm tra

```text
C:\Users\ADMIN>ssh -i %USERPROFILE%\.ssh\id_ed25519 devops@160.187.229.88
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-198-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Thu Oct  1 09:09:50 PM +07 2026

  System load:  0.0                Processes:             93
  Usage of /:   26.2% of 19.59GB   Users logged in:       0
  Memory usage: 10%                IPv4 address for eth0: 160.187.229.88
  Swap usage:   0%

0 updates can be applied immediately.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

devops@wean2607:~$ whoami
devops
devops@wean2607:~$ sudo whoami
[sudo] password for devops:
root
devops@wean2607:~$
```

Phân tích kết quả:
- Lệnh SSH bằng `devops` vào thẳng máy chủ, **không bị hỏi mật khẩu đăng nhập**, chứng tỏ SSH key đã được sao chép và phân quyền đúng.
- `whoami` trả về `devops`: phiên làm việc đang chạy với quyền user thường.
- `sudo whoami` hỏi mật khẩu của `devops` rồi trả về `root`: user `devops` đã có quyền quản trị thông qua nhóm `sudo`.

![Đăng nhập devops và sudo whoami](images/02-ssh-devops-sudo.png)

## Đối chiếu yêu cầu

| Yêu cầu | Kết quả |
|---|---|
| Tạo user `devops` | ✅ `adduser devops` |
| Thêm `devops` vào nhóm `sudo` | ✅ `groups devops` cho ra `devops sudo` |
| Sao chép SSH Key từ `root` sang `devops` | ✅ `rsync --archive --chown=devops:devops` |
| Phân quyền `.ssh` = 700, `authorized_keys` = 600 | ✅ Xem kết quả `ls -la` ở Bước 4 |
| SSH bằng `devops` thành công | ✅ Đăng nhập bằng key, không cần mật khẩu |
| `sudo whoami` trả về `root` | ✅ Hỏi mật khẩu của `devops`, sau đó in ra `root` |
