# NVIDIA Driver Recovery — Ubuntu 24.04

## 1. Thông tin máy

| Thành phần | Thông tin |
|---|---|
| OS | Ubuntu 24.04 LTS |
| GPU | NVIDIA GeForce RTX 4060 Max-Q / Mobile |
| GPU Architecture | AD107M |
| NVIDIA Driver | 595 |
| Kernel (lúc xảy ra lỗi) | 7.0.0-42-generic |
| CUDA (trước khi lỗi) | 13.2 |
| Driver type | NVIDIA Open Kernel Module |

## 2. Triệu chứng

Khi chạy:

```bash
nvidia-smi
```

Xuất hiện lỗi:

```text
NVIDIA-SMI has failed because it couldn't communicate
with the NVIDIA driver.
```

Kiểm tra GPU:

```bash
lspci | grep -i nvidia
```

GPU vẫn xuất hiện:

```text
01:00.0 VGA compatible controller:
NVIDIA Corporation AD107M [GeForce RTX 4060 Max-Q / Mobile]
```

Nhưng kiểm tra kernel module:

```bash
lsmod | grep nvidia
```

Không thấy các module chính như `nvidia`, `nvidia_uvm`, `nvidia_drm`.

Thử load module:

```bash
sudo modprobe nvidia
```

Nhận lỗi:

```text
modprobe: FATAL: Module nvidia not found
in directory /lib/modules/7.0.0-42-generic
```

## 3. Nguyên nhân

**NVIDIA Kernel Module không tồn tại hoặc chưa được build cho kernel Linux hiện tại.**

Khả năng xảy ra:

- Ubuntu cập nhật kernel mới.
- NVIDIA DKMS không build module cho kernel mới.
- NVIDIA DKMS package bị gỡ hoặc cài đặt không hoàn chỉnh.
- Thiếu kernel headers hoặc DKMS.

Luồng lỗi:

```text
Ubuntu Kernel Update
        |
        v
Kernel mới: 7.0.0-42-generic
        |
        v
NVIDIA Kernel Module không được build
        |
        v
Không có nvidia.ko trong /lib/modules/
        |
        v
modprobe nvidia thất bại
        |
        v
nvidia-smi không hoạt động
```

Lưu ý: GPU vẫn được PCIe nhận diện. Chưa có bằng chứng GPU bị hỏng phần cứng.

## 4. Quy trình kiểm tra

Khi gặp lỗi, chạy lần lượt:

### Bước 1: Kiểm tra GPU

```bash
lspci | grep -i nvidia
```

### Bước 2: Kiểm tra kernel

```bash
uname -r
```

### Bước 3: Kiểm tra NVIDIA module

```bash
lsmod | grep nvidia
```

### Bước 4: Kiểm tra DKMS

```bash
dkms status
```

Nếu báo `Command 'dkms' not found`, DKMS chưa được cài.

### Bước 5: Kiểm tra NVIDIA packages

```bash
dpkg -l | grep -E 'nvidia-driver|nvidia-dkms'
```

Chú ý trạng thái package:

- `ii`: Package đã cài.
- `rc`: Package đã bị gỡ, chỉ còn configuration files.

### Bước 6: Kiểm tra module của kernel hiện tại

```bash
ls /lib/modules/$(uname -r)/updates/dkms/ | grep nvidia
```

Nếu thư mục không tồn tại hoặc không có NVIDIA module, cần kiểm tra và khôi phục DKMS.

## 5. Cách khôi phục

**Chỉ áp dụng khi xác nhận đúng tình trạng NVIDIA DKMS bị thiếu như lần này.**

### Bước 1: Cập nhật package list

```bash
sudo apt update
```

### Bước 2: Cài DKMS và kernel headers

```bash
sudo apt install dkms linux-headers-$(uname -r)
```

### Bước 3: Kiểm tra phiên bản NVIDIA driver

```bash
dpkg -l | grep -E 'nvidia-driver|nvidia-dkms'
ubuntu-drivers devices
```

Xác định đúng phiên bản driver và loại module đang sử dụng.

### Bước 4: Cài NVIDIA DKMS tương ứng

Trong lần lỗi đã ghi nhận, driver được khuyến nghị là `595-open`.

Lệnh khôi phục dự kiến:

```bash
sudo apt install nvidia-dkms-595-open
```

Lưu ý:
- Kiểm tra phiên bản package trước khi cài.
- Đảm bảo DKMS tương thích với driver đang cài.
- Nếu APT báo conflict hoặc muốn gỡ/thay đổi nhiều NVIDIA packages, dừng lại và kiểm tra trước.
- Không tự ý purge toàn bộ NVIDIA/CUDA.

### Bước 5: Kiểm tra DKMS

```bash
dkms status
```

Mong đợi module NVIDIA có trạng thái `installed` cho kernel hiện tại.

### Bước 6: Load NVIDIA module

```bash
sudo modprobe nvidia
```

### Bước 7: Kiểm tra GPU

```bash
nvidia-smi
```

Nếu hoạt động, kiểm tra thêm:

```bash
lsmod | grep nvidia
```

### Bước 8: Khởi động lại nếu cần

```bash
sudo reboot
```

Sau khi khởi động:

```bash
nvidia-smi
```

## 6. Những việc KHÔNG nên làm

Khi gặp lỗi này, không nên vội vàng:

```bash
sudo apt purge nvidia*
```

Không nên:
- Cài lại CUDA Toolkit ngay lập tức.
- Xóa toàn bộ NVIDIA packages khi chưa xác định nguyên nhân.
- Cài driver phiên bản mới hơn một cách tùy tiện.
- Thay đổi CUDA, PyTorch hoặc DeepStream environment khi chưa cần thiết.

Mục tiêu là khôi phục NVIDIA kernel module, không phải cài lại toàn bộ môi trường AI.

## 7. Checklist kiểm tra nhanh lần sau

Copy toàn bộ block dưới đây và chạy khi `nvidia-smi` bị lỗi:

```bash
echo "========== OS / KERNEL =========="
uname -a

echo "========== GPU =========="
lspci | grep -i nvidia

echo "========== NVIDIA MODULE =========="
lsmod | grep nvidia

echo "========== DKMS =========="
dkms status

echo "========== NVIDIA PACKAGES =========="
dpkg -l | grep -E 'nvidia-driver|nvidia-dkms'

echo "========== NVIDIA MODULE FILES =========="
find /lib/modules/$(uname -r) -name 'nvidia.ko*' 2>/dev/null

echo "========== MODPROBE =========="
sudo modprobe nvidia

echo "========== NVIDIA-SMI =========="
nvidia-smi
```

## 8. Prompt gửi AI lần sau

Nếu gặp lại lỗi, gửi file này cùng output các lệnh kiểm tra và sử dụng prompt:

> Máy Ubuntu 24.04 của tôi sử dụng NVIDIA RTX 4060 Mobile.
>
> Tôi từng gặp lỗi NVIDIA kernel module không được build cho kernel mới sau khi cập nhật Ubuntu.
>
> Hãy đọc tài liệu README.md và phân tích tình trạng hiện tại của máy dựa trên output tôi cung cấp.
>
> Hướng dẫn tôi khôi phục NVIDIA driver từng bước, ưu tiên giữ nguyên CUDA Toolkit, PyTorch, Docker và DeepStream.
>
> Không tự ý purge hoặc cài lại toàn bộ driver. Mỗi bước hãy giải thích ngắn gọn và chờ tôi gửi output trước khi chuyển sang bước tiếp theo.

---

**Mục tiêu cuối cùng:** `nvidia-smi` hoạt động bình thường, NVIDIA kernel module được load và driver tương thích với kernel Linux hiện tại.