# BÁO CÁO LAB-02: Device Driver & File System (Embedded Linux Systems)

## 1. Thông tin sinh viên và môi trường

| Mục | Giá trị |
|---|---|
| Họ tên / MSSV | Hà Anh Huy / SE203732 (suy ra từ tên thư mục Assignment 01 – hãy kiểm tra lại) |
| Kiến trúc mục tiêu | ARMv7-A, QEMU vexpress-a9 |
| Kernel | Linux 5.15.0 (`~/embedded_lab1/kernel/linux-5.15`) |
| Toolchain | arm-linux-gnueabihf-gcc 13.3.0 (Ubuntu 24.04) |
| Máy host | Ubuntu, Linux 7.0.0-34-generic |
| Kết quả LAB-01 dùng lại | `zImage`, `vexpress-v2p-ca9.dtb`, initramfs BusyBox |

## 2. Tóm tắt công việc đã thực hiện

| Phần | Nội dung | Trạng thái (theo bằng chứng trong thư mục) |
|---|---|---|
| A | Viết & cross-compile `lab2_driver.c` (char device `/dev/lab2`, major 240) | **Xong** – có `lab2_driver.ko` (ELF ARM, vermagic `5.15.0 SMP mod_unload ARMv7`) |
| B | `/proc/lab2_info` (seq_file + single_open) và 3 sysfs attribute `buffer_len`, `open_count`, `last_data` (DEVICE_ATTR_RO) | **Xong phần code**; chưa có log chạy |
| C | MTD nandsim + JFFS2 trên host | **Chưa thấy bằng chứng** – thư mục `mtd/` trống |
| D | BusyBox init auto-load module qua `rcS`, `test_driver.sh`, `test_procfs.sh`, đóng gói `initramfs_lab2.cpio.gz` | **Xong** – cpio chứa `inittab`, `rcS`, `lab2_driver.ko`, 2 script test |
| Log | `boot_log.txt`, `driver_test.txt`, `procfs_test.txt`, `mtd_jffs2.txt` | **Chưa có** (không có thư mục `logs/`) |

## 3. Sơ đồ kiến trúc driver

```
Userspace:  echo "hi" > /dev/lab2     cat /dev/lab2    cat /proc/lab2_info    cat /sys/class/lab2_class/lab2/*
                 │ write()               │ read()            │ read()                 │ read()
                 ▼                       ▼                   ▼                        ▼
Syscall  ───────────  system call interface (sys_write / sys_read / sys_open) ──────────────
                 ▼                       ▼                   ▼                        ▼
VFS      ── /dev/lab2 (c 240:0) ── inode → file_operations   procfs (proc_ops)   sysfs (kobject)
                 ▼                       ▼                   ▼                        ▼
Driver   lab2_write()             lab2_read()        lab2_proc_show()      buffer_len_show()
         copy_from_user ─┐        copy_to_user ◄─┐   (single_open/seq_file)  open_count_show()
                         ▼                       │          │                last_data_show()
                    device_buffer[1024] ◄────────┴──────────┴──────────────────┘
                    buffer_len, open_count   (ghi được bảo vệ bởi lab2_mutex)
```

Cấu trúc sysfs: `/sys/class/lab2_class/lab2/{buffer_len, open_count, last_data, dev, uevent, ...}`

Luồng write: user buffer → `copy_from_user()` → `device_buffer` (+ `buffer_len`) → hiển thị lại qua `/proc/lab2_info` và sysfs.

## 4. Chi tiết triển khai

**Phần A – Character device** (`driver/lab2_driver.c`)
- `register_chrdev(240, "lab2", &lab2_fops)`, `class_create` + `device_create` để tạo `/dev/lab2`.
- `lab2_open` tăng `open_count` (có mutex); `lab2_release` chỉ log; `lab2_read` dùng `copy_to_user` với offset; `lab2_write` cắt ở 1023 byte, `copy_from_user`, kết thúc bằng `'\0'`.
- Build: `make` trong `driver/` với `ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-`, KDIR trỏ tới kernel 5.15 của LAB-01, `-march=armv7-a`.

**Phần B – procfs / sysfs**
- procfs: `proc_create("lab2_info", 0444, ...)` + `proc_ops` {`single_open`, `seq_read`, `seq_lseek`, `single_release`}. In tên, major, kích thước buffer, độ dài dữ liệu, số lần open, dữ liệu cuối.
- sysfs: `DEVICE_ATTR_RO` ×3 gom vào `attribute_group`, đăng ký bằng `sysfs_create_group()` trên `lab2_device->kobj`.

**Phần D – BusyBox init**
- `/init` mount proc/sysfs rồi `exec /sbin/init`; `inittab` chạy `::sysinit:/etc/init.d/rcS`.
- `rcS` mount proc/sys/tmpfs/devtmpfs, `insmod /lib/modules/5.15.0/lab2_driver.ko`, `mknod /dev/lab2 c 240 0`, `chmod 666`, in thông báo `[ LAB-02 ]`.
- Đóng gói: `output/initramfs_lab2.cpio.gz` (~1.1 MB).

## 5. Log thực hành

> **Phải bổ sung** (đề yêu cầu là bằng chứng bắt buộc). Lệnh gợi ý ở mục 10.

- Phần A: `modinfo`, `insmod`, `lsmod`, đọc/ghi `/dev/lab2`, `dmesg | grep lab2` → *chưa có*
- Phần B: `cat /proc/lab2_info` trước/sau write, toàn bộ sysfs → *chưa có*
- Phần C: `mtdinfo`, `flash_erase`, `nandwrite`, mount JFFS2, persistence → *chưa thực hiện*
- Phần D: boot log QEMU, output `test_driver.sh`, `test_procfs.sh` → *chưa có*

Chỉ có `modinfo` thật đã kiểm tra: `name: lab2_driver, version 1.0, license GPL, vermagic 5.15.0 SMP mod_unload ARMv7 p2v8`.

## 6. So sánh filesystem

| Tiêu chí | ext4 | JFFS2 | cramfs |
|---|---|---|---|
| Ghi được | Có | Có | Không (read-only) |
| Wear leveling | Không (cần FTL/eMMC bên dưới) | Có, tích hợp (log-structured) | Không cần (chỉ đọc) |
| Nén | Không | Có (zlib, lzo) | Có (zlib) |
| Thời gian mount/boot | Nhanh (journal) | Chậm khi flash lớn (quét toàn bộ) | Rất nhanh |
| Phù hợp | eMMC/SD/SSD | Raw NAND/NOR nhỏ | Rootfs cố định, firmware |

**Khuyến nghị cho IoT 32 MB NAND + OTA:** rootfs read-only nén (SquashFS/cramfs) + phân vùng ghi được JFFS2 (hoặc UBIFS) cho cấu hình/dữ liệu, dùng 2 slot rootfs (A/B) để OTA an toàn và rollback được. Với 32 MB, JFFS2 vẫn chấp nhận được về thời gian mount.

## 7. Trả lời câu hỏi phân tích (tóm tắt – cần viết đầy đủ khi nộp)

**Câu 1 – Vì sao không dùng `memcpy()`:** con trỏ user là địa chỉ ảo của tiến trình, có thể chưa được map, đã bị swap, hoặc trỏ vào vùng kernel (lỗ hổng bảo mật). `copy_to/from_user` kiểm tra quyền (`access_ok`), xử lý page-fault an toàn qua bảng exception và trả về số byte chưa copy được thay vì gây oops. Dùng `memcpy` có thể gây oops/panic hoặc cho user đọc/ghi bộ nhớ kernel tuỳ ý.

**Câu 2 – Vì sao cần mutex:** nhiều tiến trình mở `/dev/lab2` có thể `write()` đồng thời (kể cả 1 CPU do preemption). Không có mutex, hai `copy_from_user` xen kẽ làm trộn nội dung `device_buffer`, `buffer_len` có thể khớp dữ liệu của tiến trình kia → chuỗi rách, độ dài sai, `'\0'` sai chỗ. Dùng mutex (có thể ngủ) vì `copy_from_user` có thể sleep khi page-fault; spinlock thì không được.

**Câu 3 – procfs vs sysfs:** procfs vốn dành cho thông tin tiến trình, file mới tuỳ ý, không có cấu trúc. sysfs gắn với mô hình thiết bị (kobject), mỗi file một giá trị, tự gỡ khi device bị huỷ. Trong bài: thống kê tổng hợp nhiều dòng (`/proc/lab2_info`) hợp với procfs; từng giá trị đơn (`buffer_len`, `open_count`, `last_data`) đúng chuẩn sysfs.

**Câu 4 – JFFS2 và erase block:** NAND chỉ đổi bit 1→0 khi ghi theo page; muốn 0→1 phải xoá cả erase block, nên không ghi đè tại chỗ được. JFFS2 ghi node mới nối tiếp (log-structured), đánh dấu node cũ obsolete, garbage collector gom và erase block cũ; phân tán erase đều → wear leveling.

**Câu 5:** chưa tóm tắt – xem Mục VI của đề.

## 8. Khó khăn gặp phải và cách giải quyết

> Cần điền theo trải nghiệm thực tế. Hai điểm có dấu vết trong mã nguồn:
1. **Biên dịch module cho ARM:** phải ép `-march=armv7-a` và trỏ KDIR vào kernel 5.15 đã build ở LAB-01 để vermagic khớp, nếu không `insmod` báo *Invalid module format*.
2. **Tạo `/dev/lab2`:** initramfs BusyBox không có udev nên rcS dùng `mknod /dev/lab2 c 240 0` với major cố định 240.

## 9. Đánh giá thẳng: những chỗ còn thiếu so với đề

1. **Thiếu comment bắt buộc** trong `lab2_driver.c`: copy_to/from_user vs memcpy, mutex, seq_file/`single_open`, `DEVICE_ATTR_RO`/kobject, mục đích từng hàm.
2. **Phần C chưa làm** (25 điểm): `mtd/` trống.
3. **Chưa có `logs/` và PDF báo cáo**; `configs/` trống.
4. **Lỗi nhỏ trong code:** `lab2_read`, `lab2_proc_show`, các `*_show` đọc `device_buffer`/`buffer_len` không giữ mutex (race với write); `lab2_open` in `open_count` sau khi nhả mutex; nên dùng `sysfs_emit` thay `sprintf`; `lab2_exit` không kiểm tra `lab2_device`.
5. Cấu trúc nộp bài cần đổi thành `MSSV_HoTen_Lab02/` và nén `.tar.gz`.

## 10. Cách tạo log còn thiếu (gợi ý)

```bash
# Phần D – boot QEMU, lưu log
qemu-system-arm -M vexpress-a9 -m 256M -nographic \
  -kernel ~/embedded_lab1/output/zImage -dtb ~/embedded_lab1/output/vexpress-v2p-ca9.dtb \
  -initrd ~/embedded_lab2/output/initramfs_lab2.cpio.gz \
  -append "console=ttyAMA0 rdinit=/init" | tee logs/boot_log.txt

# Phần C – trên HOST (sudo)
sudo modprobe nandsim first_id_byte=0x20 second_id_byte=0xaa third_id_byte=0x00 fourth_id_byte=0x15
mtdinfo -a; sudo flash_erase /dev/mtd0 0 0
# mkfs.jffs2 -r dir -o img.jffs2 -e 0x4000 ; nandwrite -p /dev/mtd0 img.jffs2
# mount -t jffs2 mtdblock0 /mnt ; kiểm tra persistence sau umount/mount
```
