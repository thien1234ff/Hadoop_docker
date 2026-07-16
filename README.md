# Triển Khai Thử Nghiệm Môi Trường Lưu Trữ Và Tính Toán Phân Tán Với Hadoop Trên Docker

[![Hadoop Version](https://img.shields.io/badge/Hadoop-3.2.1-blue.svg?style=flat-square&logo=apache-hadoop)](https://hadoop.apache.org/)
[![Docker Support](https://img.shields.io/badge/Docker-Supported-blue.svg?style=flat-square&logo=docker)](https://www.docker.com/)
[![License](https://img.shields.io/badge/License-Apache%202.0-orange.svg?style=flat-square)](LICENSE)

Dự án này là kết quả nghiên cứu và thực nghiệm thuộc đề tài **"Triển khai thử nghiệm môi trường lưu trữ và tính toán phân tán với Hadoop với Docker"** năm 2024 của sinh viên Khoa Công nghệ Thông tin, Trường Đại học Khoa học - Đại học Huế. 

Đề tài tập trung vào việc container hóa cụm Hadoop sử dụng Docker, đồng thời nghiên cứu, phân tích và đưa ra giải pháp khắc phục triệt để lỗi kết nối truyền nhận dữ liệu (Upload/Download) giữa Client ngoài host và các container DataNode trong môi trường Docker Network.

---

## 📖 Tổng Quan Đề Tài

Hệ thống tệp phân tán Hadoop (HDFS) sử dụng cơ chế truyền nhận dữ liệu trực tiếp giữa Client và các DataNode. Khi Client thực hiện thao tác Đọc/Ghi dữ liệu:
1. Client gửi yêu cầu đến **NameNode**.
2. **NameNode** trả về danh sách các **DataNode** chứa/lưu trữ block dữ liệu (kèm theo IP/Hostname nội bộ của DataNode đó).
3. Client kết nối trực tiếp đến **DataNode** thông qua thông tin được NameNode cung cấp để truyền nhận dữ liệu.

> [!WARNING]
> **Vấn đề gặp phải (Hadoop-before):**
> Trong môi trường Docker mặc định, các container DataNode nhận IP động và Hostname ngẫu nhiên (ví dụ: `2e5c090dead4`). Khi Client nằm ngoài Docker network (ví dụ chạy từ Docker Host), Client không thể phân giải được Hostname này hoặc không thể định tuyến tới IP nội bộ của DataNode, dẫn tới lỗi kết nối (Connection Refused / Host Unreachable) khi upload hoặc download file thông qua giao diện Web HDFS hoặc ứng dụng ngoài.

> [!TIP]
> **Giải pháp khắc phục (Hadoop-after):**
> Thiết lập cấu hình mạng cố định:
> - Định nghĩa dải mạng (Subnet Bridge) cố định cho cụm Hadoop trên Docker.
> - Cấu hình Hostname cố định (`namenode`, `datanode`) và gán IP tĩnh cho từng container.
> - Thực hiện cấu hình ánh xạ Hostname sang IP (trong file `hosts` của máy Client/Host) giúp Client phân giải trực tiếp địa chỉ các container và giao tiếp thông qua các Port được expose tương ứng.

---

## 📂 Cấu Trúc Thư Mục Dự Án

Thư mục chính của dự án bao gồm:

```text
├── Hadoop-before/            # Cấu hình Hadoop Docker mặc định (chưa khắc phục lỗi mạng)
├── Hadoop-after/             # Cấu hình Hadoop Docker đã sửa lỗi (sử dụng IP tĩnh & Hostname cố định)
├── BaoCaoTongKet_HadoopDocker_K46F.docx # File báo cáo tổng kết chi tiết của đề tài
├── hadoop_docker.pptx        # Slide thuyết trình báo cáo đề tài
└── README.md                 # Tài liệu hướng dẫn sử dụng (File này)
```

---

## 🛠️ Hướng Dẫn Cấu Hợp Và Triển Khai (`Hadoop-after`)

Để triển khai cụm Hadoop đã khắc phục lỗi kết nối, bạn thực hiện theo các bước sau:

### Bước 1: Cấu hình File `hosts` trên máy Client (Docker Host)

Để máy tính cá nhân của bạn có thể hiểu và kết nối được với các container bên trong Docker thông qua tên miền nội bộ, bạn cần ánh xạ IP tĩnh của các container vào file cấu hình hệ thống:

*   **Trên Windows:** Mở Notepad với quyền Administrator và chỉnh sửa file `C:\Windows\System32\drivers\etc\hosts`.
*   **Trên Linux/macOS:** Dùng Terminal và chạy lệnh `sudo nano /etc/hosts`.

Thêm các dòng cấu hình sau vào cuối file `hosts`:
```text
172.22.0.2 namenode
172.22.0.3 datanode
```

### Bước 2: Xem Cấu Hình Docker Compose (`Hadoop-after/docker-compose.yml`)

Trong cấu hình `Hadoop-after`, mạng `bigdatanetwork` được thiết lập dạng `bridge` với subnet cố định `172.22.0.0/16`:

```yaml
version: "3"

services:
  namenode:
    image: bde2020/hadoop-namenode:2.0.0-hadoop3.2.1-java8
    container_name: namenode
    hostname: namenode
    restart: always
    ports:
      - 9870:9870
      - 9000:9000
    volumes:
      - hadoop_namenode:/hadoop/dfs/name
    environment:
      - CLUSTER_NAME=test
    env_file:
      - ./hadoop.env
    networks:
      bigdatanetwork:
        ipv4_address: 172.22.0.2

  datanode:
    image: bde2020/hadoop-datanode:2.0.0-hadoop3.2.1-java8
    container_name: datanode
    hostname: datanode
    restart: always
    volumes:
      - hadoop_datanode:/hadoop/dfs/data
    ports:
      - 9864:9864
    environment:
      SERVICE_PRECONDITION: "namenode:9870"
    env_file:
      - ./hadoop.env
    networks:
      bigdatanetwork:
        ipv4_address: 172.22.0.3

volumes:
  hadoop_namenode:
  hadoop_datanode:

networks:
  bigdatanetwork:
    name: bigdatanetwork
    driver: bridge
    ipam:
      config:
        - subnet: 172.22.0.0/16
```

### Bước 3: Khởi động Cụm Hadoop

Di chuyển vào thư mục `Hadoop-after` và sử dụng Docker Compose để khởi chạy hệ thống:

```bash
cd Hadoop-after
docker-compose up -d
```

Lệnh trên sẽ tải các Docker images cần thiết và khởi chạy các dịch vụ ở chế độ chạy ngầm (`-d`). Bạn có thể kiểm tra danh sách container đang chạy bằng lệnh:
```bash
docker ps
```

---

## 🔍 Kiểm Tra Kết Nối Và Thao Tác HDFS

### 1. Truy cập Giao diện quản lý Web UI
Sau khi khởi động thành công, bạn mở trình duyệt và truy cập:
*   Giao diện NameNode: [http://localhost:9870/dfshealth.html](http://localhost:9870/dfshealth.html)
*   Giao diện DataNode: [http://localhost:9864/](http://localhost:9864/)

Tại đây, bạn có thể kiểm tra trạng thái hoạt động của cụm, không gian lưu trữ và danh sách các DataNode đang hoạt động.

### 2. Kiểm tra tính năng Upload File lên HDFS
Trước khi cấu hình mạng (`Hadoop-before`), việc tải file lên thông qua nút **"Upload"** trên giao diện Web UI (Utilities -> Browse the file system) sẽ thất bại vì trình duyệt của Client không thể gửi file trực tiếp đến DataNode.

Sau khi đã thực hiện cấu hình `/etc/hosts` và chạy bản `Hadoop-after`:
1. Truy cập [http://localhost:9870/dfshealth.html#tab-overview](http://localhost:9870/dfshealth.html#tab-overview).
2. Vào **Utilities** -> **Browse the file system**.
3. Chọn thư mục muốn tải lên (ví dụ: tạo thư mục mới hoặc thư mục gốc `/`).
4. Nhấn nút **Upload** và chọn một file bất kỳ từ máy của bạn.
5. File sẽ được upload thành công lên hệ thống HDFS phân tán!

### 3. Thao tác HDFS thông qua CLI (Dòng lệnh)
Bạn cũng có thể thao tác với HDFS bằng cách truy cập vào container NameNode:

```bash
# Đi vào container namenode
docker exec -it namenode bash

# Tạo thư mục mới trên HDFS
hdfs dfs -mkdir /test_dir

# Liệt kê các thư mục
hdfs dfs -ls /

# Tải tệp từ container lên HDFS
hdfs dfs -put /path/in/container/file.txt /test_dir/

# Tải tệp từ HDFS về container
hdfs dfs -get /test_dir/file.txt /path/to/save
```

---

## 👥 Thành Viên Thực Hiện Đề Tài

Đề tài được thực hiện bởi nhóm sinh viên **Lớp Công nghệ Thông tin K46F** - Trường Đại học Khoa học, Đại học Huế:

1.  **Hoàng Kim Thiên** (Chủ nhiệm đề tài)
2.  **Nguyễn Đức Dương**
3.  **Nguyễn Thế Quang**
4.  **Nguyễn Châu Ngọc Nhi**
5.  **Huỳnh Huy**
6.  **Võ Minh Quân**
7.  **Phan Thuỳ Dương**

**Cán bộ cố vấn khoa học:**
*   **Thạc sĩ Nguyễn Văn Trung** (Giảng viên Khoa CNTT - Trường Đại học Khoa học, Đại học Huế)

---

## 📄 Bản Quyền & Tài Liệu Tham Khảo
*   [Apache Hadoop Documentation](https://hadoop.apache.org/docs/stable/)
*   [Docker Documentation](https://docs.docker.com/)
*   [Big Data Europe GitHub Project](https://github.com/big-data-europe/docker-hadoop)
*   Tham khảo thảo luận lỗi kết nối trên [Big Data Europe Issue #157](https://github.com/big-data-europe/docker-hadoop/issues/157)
