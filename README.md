# bai2-camdo
#  Bài tập 02 - TEE0421: Quản lý Tiệm Cầm Đồ bằng Django

**Sinh viên:** Lương Quang Hà  
**MSSV:** K225480106010  
**Lớp:** 58KTPM  
**Deadline:** 23h59 ngày 09/05/2026  

---

##  Yêu cầu bài tập

- Thiết kế CSDL cho hệ thống quản lý tiệm cầm đồ
- Sử dụng Docker trên Ubuntu để chạy MariaDB, phpMyAdmin, Django
- Tạo trang Admin cho phép thêm/sửa/xóa dữ liệu
- Tạo trang liệt kê con nợ đến hạn chưa trả bằng Jinja2 template
- Public kết quả qua Cloudflare Tunnel

---

## 1.  Thiết kế CSDL

Hệ thống quản lý tiệm cầm đồ gồm 4 bảng chính:

### Sơ đồ quan hệ

```
┌─────────────┐         ┌─────────────┐
│  KhachHang  │         │   TaiSan    │
└──────┬──────┘         └──────┬──────┘
       │ 1                     │ 1
       │                       │
       └──────────┬────────────┘
                  │ N
         ┌────────┴────────┐
         │   HopDongCam    │
         └────────┬────────┘
                  │ 1
                  │ N
         ┌────────┴────────┐
         │   ThanhToan     │
         └─────────────────┘
```
### Chi tiết các bảng

**Bảng KhachHang**
| Trường | Kiểu | Ghi chú |
|---|---|---|
| id | INT PK AI | Khóa chính |
| ho_ten | VARCHAR(100) | Họ tên khách |
| so_dien_thoai | VARCHAR(15) | SĐT |
| dia_chi | TEXT | Địa chỉ |
| so_cmnd | VARCHAR(20) | CMND/CCCD (unique) |
| ngay_tao | DATETIME | Tự động tạo |

**Bảng TaiSan**
| Trường | Kiểu | Ghi chú |
|---|---|---|
| id | INT PK AI | Khóa chính |
| ten_tai_san | VARCHAR(200) | Tên tài sản |
| mo_ta | TEXT | Mô tả |
| tinh_trang | VARCHAR(100) | Tình trạng |
| gia_tri_dinh_gia | DECIMAL(15,2) | Giá trị định giá |

**Bảng HopDongCam** *(bảng trung tâm)*
| Trường | Kiểu | Ghi chú |
|---|---|---|
| id | INT PK AI | Khóa chính |
| khach_hang_id | FK → KhachHang | Khóa ngoại |
| tai_san_id | FK → TaiSan | Khóa ngoại |
| so_tien_vay | DECIMAL(15,2) | Số tiền vay |
| lai_suat_thang | DECIMAL(5,2) | Lãi suất %/tháng |
| ngay_vay | DATE | Ngày vay |
| ngay_dao_han | DATE | Ngày đáo hạn |
| trang_thai | VARCHAR(30) | dang_cam/da_chuoc/qua_han/da_ban |
| ghi_chu | TEXT | Ghi chú |

**Bảng ThanhToan**
| Trường | Kiểu | Ghi chú |
|---|---|---|
| id | INT PK AI | Khóa chính |
| hop_dong_id | FK → HopDongCam | Khóa ngoại |
| ngay_thanh_toan | DATE | Ngày thanh toán |
| so_tien | DECIMAL(15,2) | Số tiền |
| loai | VARCHAR(20) | tra_lai/chuoc_do |
| ghi_chu | TEXT | Ghi chú |

<img width="1922" height="2560" alt="image" src="https://github.com/user-attachments/assets/470e162b-309d-47b5-8a52-210c3f72e6bc" />


---

## 2. Cấu trúc thư mục Docker
```
camdo/
├── docker-compose.yml
├── django/
│   ├── Dockerfile
│   └── requirements.txt
└── app/
    ├── manage.py
    ├── camdo/
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    └── quanly/
        ├── models.py
        ├── views.py
        ├── admin.py
        └── templates/
            └── quanly/
                └── home.html
```

<img width="980" height="592" alt="image" src="https://github.com/user-attachments/assets/b80a230c-8e68-4bcb-810d-064057b17c42" />


---

## 3.  Nội dung các file cấu hình

### Dockerfile

```dockerfile
FROM python:3.11-slim
LABEL maintainer="luongha"
WORKDIR /app
RUN apt-get update && apt-get install -y \
    gcc \
    default-libmysqlclient-dev \
    pkg-config \
    curl \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```
Dockerfile
<img width="973" height="596" alt="image" src="https://github.com/user-attachments/assets/ae72c477-27aa-40d4-8917-2838f0bf32aa" />

docker-compose.yml:
<img width="1303" height="787" alt="image" src="https://github.com/user-attachments/assets/ad9872f0-624c-4908-80a7-ed22601131d6" />

models.py:
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/5ac01323-c91c-4c9e-be96-834250e96f73" />

### requirements.txt
Django==4.2.11
mysqlclient==2.2.4
Pillow==10.3.0
python-decouple==3.8
### docker-compose.yml

```yaml
services:
  db:
    image: mariadb:10.11
    container_name: camdo_db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: camdo_db
      MYSQL_USER: camdo_user
      MYSQL_PASSWORD: camdo_pass
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - camdo_net

  django:
    build:
      context: ./django
    container_name: camdo_django
    restart: always
    ports:
      - "8000:8000"
    volumes:
      - ./app:/app
    environment:
      DB_NAME: camdo_db
      DB_USER: camdo_user
      DB_PASSWORD: camdo_pass
      DB_HOST: db
      DB_PORT: "3306"
    depends_on:
      - db
    networks:
      - camdo_net

volumes:
  db_data:

networks:
  camdo_net:
    driver: bridge
```

---
<img width="1303" height="790" alt="image" src="https://github.com/user-attachments/assets/e6f95182-60c5-4a46-a99c-a3ded570de1e" />


## 4. ⚙️ Các bước thực hiện

### Bước 1: Tạo thư mục và file cấu hình

```bash
mkdir -p ~/camdo/django
mkdir -p ~/camdo/app
cd ~/camdo
nano ~/camdo/django/Dockerfile
nano ~/camdo/django/requirements.txt
nano ~/camdo/docker-compose.yml
```

### Bước 2: Build và khởi động Docker

```bash
cd ~/camdo
docker-compose up -d --build
```

<img width="1302" height="787" alt="image" src="https://github.com/user-attachments/assets/c7d773ea-8758-4076-9e28-5053b042e65b" />

### Bước 3: Khởi tạo Django project

```bash
docker-compose run --rm django django-admin startproject camdo .
docker-compose run --rm django python manage.py startapp quanly
```
<img width="1298" height="793" alt="image" src="https://github.com/user-attachments/assets/c5de6e46-6284-4a3c-b2cc-4f5aabb0b77d" />

### Bước 4: Cấu hình settings.py

Sửa `ALLOWED_HOSTS`, `DATABASES`, `INSTALLED_APPS`, `CSRF_TRUSTED_ORIGINS`:

```python
ALLOWED_HOSTS = ['*']

INSTALLED_APPS = [
    ...
    'quanly',
]

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'camdo_db',
        'USER': 'camdo_user',
        'PASSWORD': 'camdo_pass',
        'HOST': 'db',
        'PORT': '3306',
    }
}

CSRF_TRUSTED_ORIGINS = [
    'https://luongquangha.io.vn',
]
```
<img width="1033" height="537" alt="image" src="https://github.com/user-attachments/assets/aae2ac70-270a-408b-ba6d-eb18a5e9a02a" />
ALLOWED_HOSTS và CSRF:
<img width="1299" height="784" alt="image" src="https://github.com/user-attachments/assets/337116c8-6e20-4474-aa9e-f6e1eee9e66a" />

### Bước 5: Tạo models.py

```python
from django.db import models

class KhachHang(models.Model):
    ho_ten = models.CharField(max_length=100)
    so_dien_thoai = models.CharField(max_length=15)
    dia_chi = models.TextField()
    so_cmnd = models.CharField(max_length=20, unique=True)
    ngay_tao = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.ho_ten

class TaiSan(models.Model):
    ten_tai_san = models.CharField(max_length=200)
    mo_ta = models.TextField()
    tinh_trang = models.CharField(max_length=100)
    gia_tri_dinh_gia = models.DecimalField(max_digits=15, decimal_places=2)

    def __str__(self):
        return self.ten_tai_san

class HopDongCam(models.Model):
    TRANG_THAI = [
        ('dang_cam', 'Đang cầm'),
        ('da_chuoc', 'Đã chuộc'),
        ('qua_han', 'Quá hạn'),
        ('da_ban', 'Đã bán'),
    ]
    khach_hang = models.ForeignKey(KhachHang, on_delete=models.CASCADE)
    tai_san = models.ForeignKey(TaiSan, on_delete=models.CASCADE)
    so_tien_vay = models.DecimalField(max_digits=15, decimal_places=2)
    lai_suat_thang = models.DecimalField(max_digits=5, decimal_places=2)
    ngay_vay = models.DateField()
    ngay_dao_han = models.DateField()
    trang_thai = models.CharField(max_length=20, choices=TRANG_THAI, default='dang_cam')
    ghi_chu = models.TextField(blank=True)

    def __str__(self):
        return f"HD {self.id} - {self.khach_hang}"

class ThanhToan(models.Model):
    LOAI = [
        ('tra_lai', 'Trả lãi'),
        ('chuoc_do', 'Chuộc đồ'),
    ]
    hop_dong = models.ForeignKey(HopDongCam, on_delete=models.CASCADE)
    ngay_thanh_toan = models.DateField()
    so_tien = models.DecimalField(max_digits=15, decimal_places=2)
    loai = models.CharField(max_length=20, choices=LOAI)
    ghi_chu = models.TextField(blank=True)

    def __str__(self):
        return f"TT {self.id} - {self.hop_dong}"
```
<img width="978" height="514" alt="image" src="https://github.com/user-attachments/assets/83254ca8-fd5e-4c21-a459-b90a42c24eec" />
<img width="979" height="513" alt="image" src="https://github.com/user-attachments/assets/a1345a4f-7c86-4a76-a37c-e2f1c642f869" />
<img width="977" height="514" alt="image" src="https://github.com/user-attachments/assets/9d2fb846-267d-4ade-aaeb-cf63b9bdcec0" />


### Bước 6: Chạy migrate và tạo superuser

```bash
docker-compose exec django python manage.py makemigrations
docker-compose exec django python manage.py migrate
docker-compose exec django python manage.py createsuperuser
```

<img width="978" height="510" alt="image" src="https://github.com/user-attachments/assets/4260a153-cad0-4c70-bb6e-2bc9fa018635" />


### Bước 7: Tạo views.py và template

**views.py:**
```python
from django.shortcuts import render
from .models import HopDongCam
from django.utils import timezone

def home_page(request):
    hom_nay = timezone.now().date()
    con_no = HopDongCam.objects.filter(
        ngay_dao_han__lte=hom_nay,
        trang_thai__in=['dang_cam', 'qua_han']
    ).select_related('khach_hang', 'tai_san')
    context = {
        'con_no': con_no,
        'hom_nay': hom_nay,
    }
    return render(request, 'quanly/home.html', context)
```

**home.html** sử dụng cú pháp Jinja2 của Django để hiển thị danh sách con nợ đến hạn.

---

## 5. ✅ Kết quả

### Trang Admin - Thêm/Sửa/Xóa dữ liệu

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/9e7b3970-9fb1-416b-a729-0a624c1170c0" />


📸 **Form thêm hợp đồng (FK hiển thị dạng dropdown):**

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/642fcbd1-13e4-413d-b50f-2216e97fc691" />


### Trang con nợ đến hạn (local)

📸 **Trang con nợ local:**

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/dbbb02f9-d737-49f7-bfbc-7a7aedd294a3" />


### Trang con nợ đến hạn (public qua Cloudflare Tunnel)

📸 **Trang con nợ public:**

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a23f87ef-7b4a-48b1-8afc-32b4ba6fa425" />

### phpMyAdmin - Kiểm tra FK lưu ID

📸 **Danh sách bảng trong camdo_db:**
<img width="1920" height="1078" alt="image" src="https://github.com/user-attachments/assets/403eef74-c972-4ac6-af4f-6776993827a9" />


📸 **Bảng hopdongcam - FK lưu ID (khach_hang_id=1, tai_san_id=1):**

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/61a3160d-13b4-4f86-a8ef-a6c451fce12d" />

---

## 6. 🌐 Cloudflare Tunnel

Sử dụng Cloudflare Tunnel để public ứng dụng ra internet:

- **Tunnel name:** luongquangha-tunnel
- **Domain:** luongquangha.io.vn
- **Service:** http://172.27.253.21:8000
  luongquangha-tunnel đang HEALTHY
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/268f2ad8-ca9c-409e-a2cb-f424c750cfa2" />
<img width="1905" height="1064" alt="image" src="https://github.com/user-attachments/assets/5afc2203-4e97-46a7-b5d1-0c28d4498c39" />
<img width="1907" height="1074" alt="image" src="https://github.com/user-attachments/assets/7352b956-bc47-4e58-b1dc-6978ff528654" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/6ca2826e-df49-41cf-bc30-da70a45643d7" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/d04aef62-557e-4021-bc65-a6b1765612a2" />

**Kết quả:**
-  Trang con nợ: https://luongquangha.io.vn
  <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/f1bce65f-3aa1-492a-82ed-6b4891d2c671" />

-  Trang Admin: https://luongquangha.io.vn/admin
  <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/53cb1b10-19c0-490a-b06f-372cfa224258" />


---

## 7. Tổng kết

| Yêu cầu | Kết quả |
|---|---|
| Thiết kế CSDL | ✅ 4 bảng với quan hệ FK |
| MariaDB container | ✅ Docker container |
| phpMyAdmin | ✅ Port 8082 |
| Django từ Dockerfile | ✅ Build thành công |
| Kết nối Django ↔ MariaDB | ✅ settings.py |
| Trang Admin | ✅ Thêm/sửa/xóa, FK dropdown |
| Trang home_page Jinja2 | ✅ Con nợ đến hạn |
| Cloudflare Tunnel | ✅ luongquangha.io.vn |
