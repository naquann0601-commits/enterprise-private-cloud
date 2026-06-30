# Enterprise Core Infrastructure Deployment on Proxmox VE 🏢

Kho lưu trữ (repository) này tài liệu hóa quá trình triển khai và cấu hình hệ thống hạ tầng CNTT cốt lõi cho một doanh nghiệp SMB (vừa và nhỏ) trên nền tảng ảo hóa Proxmox VE. Mục tiêu của dự án là xây dựng một môi trường hoạt động ổn định, tập trung vào tính bảo mật, khả năng quản trị tập trung và lưu trữ dữ liệu an toàn.

**Lưu ý quan trọng:** Các cấu hình và thiết lập trong dự án này đóng vai trò như một mô hình chuẩn (template) cho môi trường SMB hoặc Lab. Cần điều chỉnh cẩn thận cho phù hợp với phần cứng, mạng và yêu cầu bảo mật thực tế của doanh nghiệp trước khi đưa vào môi trường Production.

---

## 🎯 Mục tiêu dự án (Goals)

* Xây dựng và vận hành hạ tầng CNTT cốt lõi cho doanh nghiệp SMB trên nền tảng ảo hóa.
* Đảm bảo tính bảo mật và cách ly hệ thống mạng một cách chặt chẽ.
* Triển khai hệ thống quản trị người dùng và thiết bị tập trung.
* Cung cấp giải pháp lưu trữ dữ liệu an toàn, tin cậy có phân quyền chi tiết.
* Xây dựng quy trình sao lưu và vận hành chuẩn chỉ.

---

## 🛠️ Công nghệ cốt lõi sử dụng (Core Technologies Used)

* **Proxmox VE**: Nền tảng ảo hóa Hypervisor (Single Node, tối ưu hóa tài nguyên).
* **pfSense**: Firewall, Router, NAT, DHCP, quản lý VLAN.
* **Windows Server 2022**: Active Directory, DNS, File Server tập trung.
* **TrueNAS**: Hệ thống lưu trữ ZFS, NFS/SMB, tích hợp AD.
* **Windows 10/11**: Máy khách (Client Management, GPO Policy, Domain Join).

---

## ✨ Điểm nổi bật / Tính năng chính (Key Features/Highlights)

* **Nền tảng Ảo hóa linh hoạt**: Tối ưu hóa Linux Bridge và Storage Pool trên Proxmox VE, ứng dụng Nested Virtualization để mô phỏng môi trường Server thực tế.
* **Vành đai mạng (Network Perimeter) an toàn**: Dùng pfSense làm Gateway toàn hệ thống, phân đoạn VLAN để cách ly vùng Management, Server và User.
* **Quản trị tập trung (Centralized Management)**: Triển khai AD DS, thiết lập GPO để hạn chế quyền, map ổ đĩa mạng và quản lý cập nhật đồng bộ.
* **Lưu trữ chuyên nghiệp & Tích hợp sâu**: Cấu hình RAID và Dataset trên TrueNAS, tích hợp xác thực với Active Directory để phân quyền SMB cho từng phòng ban (HR/IT/Sales).
* **Vận hành & Bảo trì (O&M)**: Tự động hóa quy trình Backup/Restore cho VM và thiết lập hệ thống giám sát, phân tích logs sự cố.

---

## 🏛️ Cấu trúc Repository (Repository Structure)

```text
enterprise-infra-proxmox/
├── README.md
├── docs/
│   ├── phase1-proxmox-setup.md
│   ├── phase2-network-pfsense.md
│   ├── phase3-active-directory.md
│   ├── phase4-truenas-storage.md
│   └── phase5-maintenance-backup.md
├── configs/
│   ├── pfsense/
│   │   └── pfsense-config-backup.xml
│   └── truenas/
│       └── smb_acls.conf
└── scripts/
    └── backup/
        └── proxmox-vm-backup.sh
```
## 🚀 Getting Started / Configuration

Dự án được triển khai theo từng giai đoạn nhằm mô phỏng quy trình xây dựng và vận hành hạ tầng **Enterprise Private Cloud** trong môi trường thực tế.

Toàn bộ hướng dẫn chi tiết được đặt trong thư mục `docs/`.

---

### Phase 1 · Virtualization Platform Deployment

Triển khai nền tảng ảo hóa với **Proxmox VE**.

Các nội dung bao gồm:

* Cài đặt Proxmox VE trên VMware
* Kích hoạt Nested Virtualization
* Cấu hình Linux Bridge
* Thiết lập Storage Pool
* Chuẩn bị môi trường Lab

Tài liệu:

```text
docs/01-installation.md
docs/02-network.md
docs/03-storage.md
```

---

### Phase 2 · Network Perimeter

Thiết lập lớp mạng trung tâm cho toàn bộ hạ tầng.

Các nội dung bao gồm:

* Triển khai pfSense Gateway
* Thiết lập VLAN Segmentation
* Phân tách vùng Management / Server / User
* Routing và Network Isolation

Tài liệu:

```text
docs/05-network-pfsense.md
```

---

### Phase 3 · Centralized Identity Management (Active Directory)

Triển khai hệ thống quản trị tập trung bằng **Windows Active Directory**.

Các nội dung bao gồm:

* Cài đặt Windows Server
* Triển khai Active Directory Domain Services
* Cấu hình DNS nội bộ
* Tạo User / Group
* Quản lý Group Policy (GPO)

Tài liệu:

```text
docs/04-ad.md
```

---

### Phase 4 · Enterprise Storage (TrueNAS)

Xây dựng hệ thống lưu trữ chuyên nghiệp phục vụ Private Cloud.

Các nội dung bao gồm:

* Cài đặt TrueNAS
* Thiết lập Dataset
* Cấu hình RAID
* Tích hợp SMB
* Kết nối Active Directory

Phân quyền truy cập theo từng phòng ban:

```text
HR
IT
Sales
```

Tài liệu:

```text
docs/06-truenas-storage.md
```

---

### Phase 5 · Operations & Maintenance

Xây dựng quy trình vận hành và bảo trì hệ thống.

Các nội dung bao gồm:

* Backup / Restore VM
* Snapshot Management
* Monitoring Dashboard
* Log Analysis
* Troubleshooting

Tài liệu:

```text
docs/07-backup.md
docs/08-monitoring.md
docs/09-disaster-recovery.md
```

---

## Notes

* Cấu trúc thư mục có thể thay đổi tùy theo quá trình triển khai thực tế.
* Các tài liệu trong `docs/` được thiết kế độc lập để có thể thực hiện từng giai đoạn riêng biệt.
* Dự án ưu tiên khả năng mở rộng để triển khai thêm Cluster, HA hoặc Hybrid Cloud trong tương lai.
