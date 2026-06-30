# 05 · Edge Security & Routing (pfSense Firewall)

## Objective

Triển khai máy ảo **pfSense** đóng vai trò là Tường lửa (Firewall) và Bộ định tuyến trung tâm (Default Gateway) cho hệ thống Enterprise Private Cloud. 
Hệ thống sẽ được cấu hình kết nối ra mạng vật lý (WAN) và quản lý 3 vùng mạng nội bộ độc lập (VLAN 10, 20, 30) thông qua một cổng Trunking duy nhất.

---

## 1. Environment Strategy & IP Planning

pfSense sẽ kết nối 2 môi trường: Mạng vật lý ảo hóa của VMware và Mạng ảo lồng ghép của Proxmox.

| Giao diện pfSense | Bridge trên Proxmox | Phân vùng (Zone) | Địa chỉ IP Tĩnh | Chức năng |
| :--- | :--- | :--- | :--- | :--- |
| **WAN** (`vtnet0`) | `vmbr0` | External / Internet | `172.16.0.254/24` | Trỏ Gateway về `172.16.0.2` của VMware. |
| **LAN** (`vtnet1`) | `vmbr1` (VLAN Aware) | Trunking Port | *Không đặt IP* | Cổng vật lý ảo chở dữ liệu cho mọi VLAN. |
| **VLAN 20** (`vtnet1.20`) | Gắn trên `vtnet1` | Server Zone | `10.20.20.1/24` | Gateway cho Windows Server AD-DC01. |
| **VLAN 10** (`vtnet1.10`) | Gắn trên `vtnet1` | Client Zone | `10.10.10.1/24` | Gateway cho mạng nhân viên (HR, IT, Sales). |
| **VLAN 30** (`vtnet1.30`) | Gắn trên `vtnet1` | Storage Zone | `10.30.30.1/24` | Gateway cho TrueNAS. |

---

## 2. Tạo máy ảo pfSense trên Proxmox

Đảm bảo Proxmox đã sẵn sàng file ISO của pfSense.

1. Bấm **Create VM** trên giao diện Proxmox.
2. **General:** VM ID `254`, Name: `pfSense-FW`.
3. **OS:** Chọn ISO `pfSense-CE-*.iso`. Type: `Other`.
4. **System:** Để mặc định.
5. **Disks:** Storage `vm-data-pool`, Size `20 GB`.
6. **CPU / Memory:** `2 Cores`, `2048 MB` RAM.
7. **Network (Cổng WAN):** Chọn Bridge **`vmbr0`**, Model `VirtIO`. Bỏ trống VLAN Tag.
8. Nhấn **Finish**.
9. Vào máy ảo vừa tạo → **Hardware** → **Add** → **Network Device**.
10. Cấu hình cổng LAN Trunking: Chọn Bridge **`vmbr1`**, Model `VirtIO`. **Bỏ trống VLAN Tag** (Để pfSense tự bóc tách các gói tin VLAN).

---

## 3. Cài đặt Hệ điều hành pfSense

1. Khởi động VM (`Start`) và mở **Console**.
2. Khi hệ thống boot lên, nhấn phím `Enter` để Accept.
3. Chọn **Install pfSense**.
4. Keymap: Chọn **Continue with default keymap**.
5. Partitioning: Chọn **Auto (ZFS)** → Proceed with Installation → Stripe (No Redundancy) → Nhấn dấu Cách (Space) để chọn ổ đĩa cứng ảo → Nhấn `Enter`.
6. Quá trình cài đặt mất khoảng 1-2 phút. Xong xuôi, chọn **Reboot**.
*(Lưu ý: Tháo file ISO trong phần Hardware của Proxmox để VM boot vào ổ cứng).*

---

## 4. Cấu hình cơ bản qua Console (VLAN 20 & WAN)

Khi pfSense khởi động xong, màn hình Console sẽ hiển thị các Card mạng hiện có (vtnet0, vtnet1). Ta cần thiết lập mạng tối thiểu để máy ảo AD-DC01 có thể truy cập WebGUI.

### 4.1 Khai báo VLAN 20
1. Hệ thống hỏi *Should VLANs be set up now?* → Nhập `y` → `Enter`.
2. *VLAN Parent interface:* Nhập `vtnet1`.
3. *VLAN tag (1-4094):* Nhập `20`.
4. Nhấn `Enter` liên tục để kết thúc tạo VLAN.

### 4.2 Gán cổng (Assign Interfaces)
1. *WAN interface name:* Nhập `vtnet0`.
2. *LAN interface name:* Nhập `vtnet1.20` (Đây là Sub-interface phục vụ VLAN 20).
3. Nhập `y` để xác nhận.

### 4.3 Đặt IP Tĩnh
Từ Menu chính, chọn phím số `2` (Set interface IP address).
* **Cấu hình WAN (Số 1):** Không dùng DHCP (`n`), IP `172.16.0.254`, Subnet `24`, Gateway `172.16.0.2`. Không dùng IPv6, Không bật DHCP Server, Không revert HTTP.
* **Cấu hình LAN (Số 2):** Không dùng DHCP (`n`), IP `10.20.20.1`, Subnet `24`. Bỏ trống Gateway (Nhấn Enter). Không dùng IPv6, **KHÔNG** bật DHCP Server (sẽ dùng Windows Server thay thế).

---

## 5. Cấu hình Nâng cao qua WebGUI

Sử dụng máy ảo **AD-DC01** (Đã có IP tĩnh `10.20.20.10` và chung mạng VLAN 20).

1. Mở trình duyệt trên AD-DC01, truy cập: `https://10.20.20.1`. Đăng nhập với admin / pfsense.
2. Hoàn thành **Setup Wizard**:
   * Hostname: `pfsense` / Domain: `enterprise.local`.
   * Primary DNS Server: `8.8.8.8`
   * Timezone: `Asia/Ho_Chi_Minh`
   * Màn hình cấu hình WAN: Kéo xuống cuối, **Bỏ chọn** mục `Block RFC1918 Private Networks` (Để pfSense không chặn dải 172.16.x.x của VMware đi vào).
   * Đổi mật khẩu Admin và Reload.

---

## 6. Khai báo VLAN 10 và VLAN 30 trên pfSense

Giờ là lúc thêm vùng mạng cho Client và Storage trực tiếp trên giao diện Web.

### 6.1 Tạo VLAN
1. Điều hướng: **Interfaces** → **Assignments** → Tab **VLANs**.
2. Nhấn **Add**.
3. *Parent Interface:* Chọn `vtnet1`.
4. *VLAN Tag:* Nhập `10`. *Description:* `Client_VLAN10`. Nhấn **Save**.
5. Lặp lại bước 2-4 để tạo VLAN Tag `30`, Description `Storage_VLAN30`.

### 6.2 Gán VLAN thành Interface
1. Chuyển sang Tab **Interface Assignments**.
2. Ở ô *Available network ports*, chọn VLAN 10 vừa tạo và nhấn **Add**. Làm tương tự với VLAN 30.
3. Nhấn **Save**. (Lúc này hệ thống sinh ra 2 cổng mới tên là OPT1 và OPT2).

### 6.3 Cấu hình IP cho các VLAN mới
1. Bấm vào tên cổng **OPT1** (hoặc vào Interfaces → OPT1).
2. Tích chọn **Enable interface**.
3. Description: Đổi thành `CLIENT_ZONE`.
4. IPv4 Configuration Type: Chọn **Static IPv4**.
5. IPv4 Address: Nhập `10.10.10.1` / `24`.
6. Nhấn **Save** và **Apply Changes**.
7. Làm tương tự cho cổng **OPT2**: Đổi tên thành `STORAGE_ZONE`, IP tĩnh là `10.30.30.1` / `24`.

---

## 7. Thiết lập Firewall Rules (Quy tắc Tường lửa)

Mặc định, pfSense chặn mọi kết nối đi vào từ các cổng (trừ LAN - VLAN20 đang được phép đi ra Internet). Để Client Zone và Storage Zone hoạt động được, ta cần mở cổng cho chúng.

1. Điều hướng: **Firewall** → **Rules**.
2. Chọn Tab **CLIENT_ZONE**.
3. Nhấn **Add** (Mũi tên lên).
4. Thiết lập:
   * Action: `Pass`
   * Interface: `CLIENT_ZONE`
   * Protocol: `Any`
   * Source: `CLIENT_ZONE net`
   * Destination: `Any`
5. Nhấn **Save** và **Apply Changes**.
6. Lặp lại bước 2-5 đối với Tab **STORAGE_ZONE**. 

*(Quy tắc Any-to-Any này giúp các máy ở VLAN 10, 20, 30 ping thấy nhau và ra được Internet. Trong môi trường thực tế, ta sẽ siết chặt các rule này sau).*

---

## Result Check-list

* [x] pfSense định tuyến thành công kết nối ra Internet thông qua VMware NAT Network.
* [x] Đã cấu hình cổng Trunking `vtnet1` để vận chuyển nhiều VLAN đồng thời.
* [x] Các Gateway ảo (`10.20.20.1`, `10.10.10.1`, `10.30.30.1`) đã sẵn sàng.
* [x] Firewall đã mở khóa liên kết (Inter-VLAN Routing) để chuẩn bị đồng bộ hệ thống.