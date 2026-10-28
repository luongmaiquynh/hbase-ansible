# Cài ZooKeeper + HBase bằng Ansible

Tự động hóa phần cài đặt trong tài liệu "HBase và Hadoop: cài đặt, kiến trúc và vận hành".
Dùng cho server một node `<hostname>` và mở rộng được cho cụm nhiều node (mẫu trong `inventory/hosts.ini`).

Phạm vi: ZooKeeper 3.8.7 và HBase 2.6.7-hadoop3. **Không cài Java và Hadoop**
(Java dùng chung, Hadoop do người khác quản lý); playbook chỉ kiểm tra và đọc cấu hình của chúng.

## Cấu trúc

```
hbase-ansible/
├── ansible.cfg
├── inventory/hosts.ini        # danh sách server theo nhóm
├── group_vars/all.yml         # phiên bản, đường dẫn, port
├── files/                     # đặt tarball ở đây
├── site.yml                   # cài + cấu hình (không bật dịch vụ)
├── start.yml                  # bật: ZooKeeper -> chờ HDFS -> HMaster -> RegionServer
├── stop.yml                   # tắt: RegionServer -> HMaster -> (ZooKeeper)
├── verify.yml                 # kiểm tra tiến trình, status, fsck
└── roles/
    ├── common/                # kiểm tra Java, tạo thư mục
    ├── zookeeper/             # giải nén, zoo.cfg, zookeeper-env.sh, myid
    └── hbase/                 # giải nén, hbase-env.sh, hbase-site.xml, link cấu hình Hadoop
```

## Chuẩn bị (trên máy chạy Ansible, ví dụ MacBook)

```bash
pip3 install ansible-core          # hoặc: brew install ansible
cd hbase-ansible
ssh-copy-id -p 22 ubuntu@<IP>       # đăng nhập SSH không cần mật khẩu
ansible all -m ping                          # phải trả về "pong"
```

Tarball được lấy theo thứ tự:

1. Server đã có sẵn trong `/opt/hbase/` → dùng luôn.
2. Chưa có và `download_on_server: true` (mặc định) → server tự tải từ `apache_mirror`.
3. Chưa có và `download_on_server: false` → chép từ thư mục `files/` trên máy chạy Ansible
   (dùng cho server không có Internet; đặt tarball vào `files/` trước).

`dlcdn.apache.org` chỉ giữ bản mới nhất. Nếu tải báo 404, đặt
`apache_mirror: "https://archive.apache.org/dist"` trong `group_vars/all.yml`.

## Chạy

```bash
ansible-playbook site.yml -K --check --diff   # chạy thử, xem sẽ thay đổi gì
ansible-playbook site.yml -K                  # cài đặt (-K: hỏi mật khẩu sudo)
ansible-playbook start.yml                    # bật dịch vụ (HDFS phải đang chạy)
ansible-playbook verify.yml                   # kiểm tra
ansible-playbook stop.yml                     # tắt HBase, giữ ZooKeeper
ansible-playbook stop.yml -e stop_zookeeper=true   # tắt cả ZooKeeper
```

Chạy lại `site.yml` nhiều lần không gây hại: bước nào đã làm thì bỏ qua (idempotent).
File cấu hình bị ghi đè sẽ có bản sao lưu `*.~` kèm thời gian trong cùng thư mục.

## Tùy chỉnh thường gặp

| Muốn | Sửa trong `group_vars/all.yml` |
| --- | --- |
| Đổi phiên bản | `zk_version`, `hbase_version` |
| HDFS ở địa chỉ khác | `hbase_rootdir` (khớp `fs.defaultFS`) |
| Hadoop ở thư mục khác | `hadoop_home` |
| Kiểm tra checksum | bỏ `#` ở `zk_sha512`, `hbase_sha512` và điền mã |
| Dùng ZooKeeper nhúng của HBase | `hbase_manages_zk: true` và bỏ nhóm `[zookeeper]` khỏi start |

## Lưu ý

- Playbook không động vào HDFS: `start.yml` chỉ chờ HDFS thoát safe mode.
- Với cụm nhiều node: số node ZooKeeper phải lẻ (3 hoặc 5); máy đầu tiên trong `[hbase_master]` là Active.
- Áp dụng lên server đang chạy (<hostname>): `site.yml` chỉ ghi đè file cấu hình (có backup),
  cần `stop.yml` rồi `start.yml` để cấu hình mới có hiệu lực.
