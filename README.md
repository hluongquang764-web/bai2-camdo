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
KhachHang (1) ──────< HopDongCam >────── (1) TaiSan
│
│ (1)
▼
ThanhToan (nhiều)
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

📸 **Ảnh thiết kế CSDL viết tay:**

![CSDL viết tay](images/11_csld_viet_tay.jpg)

---

## 2. Cấu trúc thư mục Docker
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

📸 **Ảnh cấu trúc thư mục:**

![Cấu trúc thư mục](images/1_cautruc_thu_muc.png)

---

## 3. 📄 Nội dung các file cấu hình

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

📸 **Ảnh Dockerfile:**

![Dockerfile](images/2_dockerfile.png)

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

📸 **Ảnh kết quả build Docker:**

![Docker build](images/3_docker_compose_up.png)

### Bước 3: Khởi tạo Django project

```bash
docker-compose run --rm django django-admin startproject camdo .
docker-compose run --rm django python manage.py startapp quanly
```

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

### Bước 6: Chạy migrate và tạo superuser

```bash
docker-compose exec django python manage.py makemigrations
docker-compose exec django python manage.py migrate
docker-compose exec django python manage.py createsuperuser
```

📸 **Ảnh kết quả migrate:**

![Migrate](images/4_migrate.png)

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

📸 **Trang Django Admin:**

![Admin](images/5_admin_trang_chu.png)

📸 **Form thêm hợp đồng (FK hiển thị dạng dropdown):**

![Form thêm hợp đồng](images/6_admin_them_hopdong.png)

### Trang con nợ đến hạn (local)

📸 **Trang con nợ local:**

![Con nợ local](images/7_trang_con_no_local.png)

### Trang con nợ đến hạn (public qua Cloudflare Tunnel)

📸 **Trang con nợ public:**

![Con nợ public](images/8_trang_con_no_public.png)

### phpMyAdmin - Kiểm tra FK lưu ID

📸 **Danh sách bảng trong camdo_db:**

![phpMyAdmin tables](images/9_phpmyadmin_tables.png)

📸 **Bảng hopdongcam - FK lưu ID (khach_hang_id=1, tai_san_id=1):**

![FK lưu ID](images/10_phpmyadmin_fk.png)

---

## 6. 🌐 Cloudflare Tunnel

Sử dụng Cloudflare Tunnel để public ứng dụng ra internet:

- **Tunnel name:** luongquangha-tunnel
- **Domain:** luongquangha.io.vn
- **Service:** http://172.27.253.21:8000

**Kết quả:**
- 🌍 Trang con nợ: https://luongquangha.io.vn
- 🔐 Trang Admin: https://luongquangha.io.vn/admin

---

## 7. 📚 Tổng kết

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
