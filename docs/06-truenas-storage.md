# 06 · Enterprise Storage (OpenMediaVault)

## Objective

Triển khai **OpenMediaVault (OMV)** làm hệ thống lưu trữ tập trung (File Server) hạng nhẹ cho doanh nghiệp thay thế cho TrueNAS. 
Mục tiêu là tối ưu hóa tài nguyên phần cứng, tích hợp OMV vào **Active Directory (AD-DC01)** và thực hiện phân quyền truy cập tập tin (SMB) theo đúng phòng ban (HR, IT, Sales) đã cấu hình trong Phase 3.

---

## Environment & Network Strategy

Máy chủ OMV sẽ nằm trong vùng **Storage Zone (VLAN 30)**, tách biệt hoàn toàn với vùng máy chủ và vùng người dùng.

| Component | Value | Note |
| :--- | :--- | :--- |
| **VM Name** | `OMV-Storage` | Hệ điều hành lưu trữ (dựa trên Debian) |
| **Network** | Bridge `vmbr1`, **VLAN Tag: 30** | Thuộc vùng mạng STORAGE|
| **Static IP** | `10.30.30.10/24` | Gateway: `10.30.30.1` (pfSense)|
| **DNS Server** | `10.20.20.10` | Trỏ về AD-DC01 để xác thực tên miền|

---

## Step 1 · Create OMV VM on Proxmox

Truy cập giao diện **Proxmox Web UI** và nhấn **Create VM**.

### 1.1 Cấu hình máy ảo cơ bản

*   **General:** VM ID: `300`, Name: `OMV-Storage`.
*   **OS:** Chọn kho lưu trữ `local`, chọn file ISO `openmediavault_*.iso`.
*   **System:** Giữ mặc định (SeaBIOS/Q35).
*   **CPU:** Cores: `2` (OMV hoạt động mượt mà chỉ với 1-2 Cores).
*   **Memory:** `2048 MB` (Chỉ cần 2GB RAM là đủ, tối ưu hơn rất nhiều so với 8GB của TrueNAS)[cite: 6].
*   **Network:** Bridge: `vmbr1`, Model: `VirtIO`, **VLAN Tag: 30**.

### 1.2 Cấu hình Ổ cứng (Disks)

OMV yêu cầu tách biệt ổ OS và ổ Data. Bạn cần tạo 2 ổ cứng ảo:
1.  **Ổ OS (Cài hệ điều hành):** Storage: `vm-data-pool`, Size: **`10 GB`**.
2.  **Ổ Data (Chứa dữ liệu):** Sau khi hoàn thành tạo VM, vào tab **Hardware** của máy ảo -> **Add** -> **Hard Disk**. Chọn Storage `vm-data-pool`, Size: **`100 GB`**.

---

## Step 2 · Install OpenMediaVault

1.  Nhấn **Start** máy ảo và mở màn hình **Console**.
2.  Tại menu boot, chọn **Install** và nhấn Enter.
3.  Thực hiện các bước thiết lập cơ bản:
    *   **Ngôn ngữ/Khu vực:** English / Other -> Asia -> Vietnam.
    *   **Hostname:** `omv`
    *   **Domain name:** `enterprise.local`
    *   **Root password:** Đặt mật khẩu quản trị hệ thống (VD: `P@ssw0rd123`).
4.  Hệ thống sẽ tự động format phân vùng `10GB` và cài đặt hệ điều hành.
5.  Khi có thông báo hoàn tất, chọn **Continue** để khởi động lại máy ảo. (Nhớ tháo file ISO ra khỏi tab Hardware).

---

## Step 3 · Cấu hình IP Tĩnh (Static IP)

Khi OMV khởi động xong, màn hình Console màu đen sẽ hiện ra dòng chữ `omv login:`.

1.  Đăng nhập bằng user `root` và mật khẩu bạn vừa đặt ở Bước 2.
2.  Gõ lệnh sau để mở menu cấu hình mạng:
    `omv-firstaid`
3.  Dùng phím mũi tên chọn **1 Configure network interface**.
4.  Chọn card mạng (VD: `ens18` hoặc `eth0`).
5.  Hệ thống hỏi cấu hình IPv4/DHCP:
    *   *Do you want to configure IPv4?* -> Chọn **Yes**.
    *   *Do you want to use DHCP?* -> Chọn **No**.
    *   *IP Address:* Nhập `10.30.30.10`.
    *   *Netmask:* Nhập `255.255.255.0`.
    *   *Gateway:* Nhập `10.30.30.1`.
    *   *DNS Server:* Nhập `10.20.20.10` (Trỏ về máy ảo AD-DC01).
    *   *IPv6:* Chọn **No**.
6.  Chờ hệ thống lưu cấu hình. 

---

## Step 4 · Truy cập WebUI và Mount Ổ dữ liệu

1.  Từ trình duyệt trên máy **AD-DC01** (VLAN 20), truy cập vào địa chỉ: `http://10.30.30.10`.
2.  Đăng nhập với tài khoản mặc định của giao diện Web OMV:
    *   Username: `admin`
    *   Password: `openmediavault` (Hệ thống sẽ yêu cầu đổi mật khẩu sau khi đăng nhập).
3.  **Khởi tạo ổ 100GB:**
    *   Vào menu **Storage -> File Systems**.
    *   Nhấn dấu **+ (Create)**, chọn ổ đĩa 100GB (`/dev/sdb`), định dạng là **EXT4**, nhấn Save.
    *   Sau khi tạo xong, chọn ổ đĩa đó và nhấn nút **Play (Mount)** để gắn ổ đĩa vào hệ thống.
    *   *Lưu ý: Luôn nhấn nút check màu vàng (Apply) ở góc trên cùng để lưu thay đổi.*

---

# Step 5 · Tích hợp Active Directory (Join Domain)

Để **OpenMediaVault (OMV)** nhận diện người dùng và nhóm từ hệ thống **Windows Server Active Directory**, thực hiện cấu hình thủ công bằng dòng lệnh thay vì sử dụng Plugin Web như các phiên bản cũ.

---

## 5.1 Chuẩn bị môi trường mạng

### Đồng bộ thời gian (Time Synchronization)

Vào:

```text
System → Date & Time
```

Thiết lập:

| Thuộc tính  | Giá trị       |
| ----------- | ------------- |
| Time Server | `10.20.20.10` |

> Đây là địa chỉ IP của máy **AD-DC01**.
> Chênh lệch thời gian giữa Domain Controller và OMV có thể khiến quá trình Join Domain thất bại.

---

### Cấu hình DNS Domain

Vào:

```text
Network → Interfaces
```

Chỉnh sửa card mạng và đảm bảo:

| Thuộc tính | Giá trị       |
| ---------- | ------------- |
| DNS Server | `10.20.20.10` |

> DNS phải trỏ về Domain Controller để OMV phân giải được tên miền nội bộ.

---

## 5.2 Join Domain bằng dòng lệnh (Console)

Mở **Console** hoặc **SSH** vào máy ảo OMV bằng quyền:

```bash
root
```

### Cài đặt các gói cần thiết

```bash
apt-get install -y winbind libpam-winbind libnss-winbind
```

Các gói này cho phép Linux giao tiếp với **Windows Active Directory**.

---

### Chỉnh sửa cấu hình Samba

Mở file:

```bash
nano /etc/samba/smb.conf
```

Tìm tới phần:

```ini
[global]
```

Bổ sung các dòng cấu hình sau ngay bên dưới:

```ini
server role = member server
security = ads
realm = ENTERPRISE.LOCAL
workgroup = ENTERPRISE
```

Lưu file:

```text
Ctrl + O → Enter → Ctrl + X
```

---

### Thực hiện Join Domain

Chạy lệnh:

```bash
net ads join -U Administrator
```

Hệ thống sẽ yêu cầu nhập mật khẩu của tài khoản **Administrator** trên Windows Server.

Nếu xuất hiện thông báo:

```text
Joined 'OMV' to dns domain 'enterprise.local'
```

→ Quá trình tích hợp Domain đã thành công.

---

# Step 6 · Tạo Shared Folders và phân quyền SMB

## 6.1 Tạo Shared Folders cơ bản trên OMV

Vào:

```text
Storage → Shared Folders → +
```

Tạo lần lượt các thư mục:

* `HR_Data`
* `IT_Data`
* `Sales_Data`

Lưu trên ổ dữ liệu **60GB (EXT4)** đã mount.

---

### Thiết lập quyền thư mục gốc

Trong mục **Permissions**:

| Đối tượng     | Quyền        |
| ------------- | ------------ |
| Administrator | Read / Write |
| Users         | Read / Write |
| Others        | No Access    |

→ Nhấn **Save** → **Apply**

> Bỏ qua hoàn toàn tab **Privileges** trên giao diện Web.

---

## 6.2 Kích hoạt dịch vụ SMB/CIFS

Vào:

```text
Services → SMB/CIFS → Settings
```

Thiết lập:

| Thuộc tính | Giá trị      |
| ---------- | ------------ |
| Enable     | ✓            |
| Workgroup  | `ENTERPRISE` |

→ Nhấn **Save**

---

### Publish Shared Folder

Chuyển sang tab:

```text
Shares → +
```

Lần lượt publish:

* `HR_Data`
* `IT_Data`
* `Sales_Data`

→ Nhấn **Save** → **Apply**

---

### Khởi động lại dịch vụ (Khuyến nghị)

Mở Console và chạy:

```bash
systemctl restart smbd nmbd winbind
```

Dịch vụ sẽ nạp cấu hình mới nhất.

---

## 6.3 Ép quyền Active Directory từ Windows Server

Trên máy **AD-DC01**:

1. Nhấn **Win + R**
2. Nhập:

```text
\\10.30.30.10
```

Đăng nhập bằng:

```text
Administrator@enterprise.local
```

(nếu được yêu cầu)

---

### Gán quyền cho thư mục HR

* Chuột phải **HR_Data**
* Chọn **Properties**
* Mở tab **Security**
* Chọn **Edit → Add**
* Nhập:

```text
HR_Group
```

* Nhấn **Check Names**
* Tick quyền:

```text
Modify (Read / Write)
```

→ Nhấn **OK**

---

### Lặp lại cho các phòng ban khác

| Thư mục    | Nhóm AD       |
| ---------- | ------------- |
| IT_Data    | `IT_Group`    |
| Sales_Data | `Sales_Group` |

---

# Step 7 · Verification & Mapping

Trên máy **Client** *(hoặc mở File Explorer mới)*:

Truy cập:

```text
\\10.30.30.10
```

Hệ thống sẽ yêu cầu xác thực.

Đăng nhập bằng:

| Thuộc tính | Giá trị                     |
| ---------- | --------------------------- |
| Username   | `hr_user1@enterprise.local` |
| Password   | Mật khẩu của user           |

---

## Kiểm thử quyền truy cập (Test Access)

### Kiểm thử thư mục HR_Data

Thực hiện:

* Tạo file `.txt`
* Hoặc copy một file bất kỳ

Kết quả mong đợi:

✅ Cho phép ghi dữ liệu
✅ Lưu thành công

---

### Kiểm thử thư mục IT_Data hoặc Sales_Data

Thử mở hoặc ghi dữ liệu.

Kết quả mong đợi:

❌ Hiển thị thông báo:

```text
Access Denied
```

Do tài khoản đang đăng nhập không thuộc nhóm phòng ban tương ứng.

---

Sau khi hoàn tất bước kiểm thử, hệ thống NAS đã tích hợp thành công với **Active Directory**, hỗ trợ xác thực tập trung và phân quyền theo phòng ban.

---

## Result Check-list

* [x] OpenMediaVault đã được cài đặt và vận hành nhẹ nhàng, tối ưu tài nguyên ảo hóa.
* [x] Tích hợp thành công vào Domain `enterprise.local`.
* [x] Đã thiết lập Shared Folders phân cấp theo phòng ban (HR, IT, Sales).
* [x] Phân quyền SMB sử dụng Security Groups của Active Directory đã hoạt động.