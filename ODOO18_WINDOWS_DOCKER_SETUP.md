# Windows 로컬 + Git 소스 clone + Docker로 Odoo 18 개발환경 구축 가이드

이 문서는 Windows 환경에서 Odoo 18을 Git 소스로 관리하면서 Docker 기반으로 개발 환경을 안정적으로 구성하는 방법을 정리한 가이드입니다.

## 1) 사전 설치 (최초 1회)

관리자 권한 PowerShell에서 아래를 실행합니다.

```powershell
winget install --id Git.Git -e
winget install --id Docker.DockerDesktop -e
```

설치 후 Docker Desktop을 실행하고, 아래로 설치 확인:

```powershell
git --version
docker --version
docker compose version
```

---

## 2) 작업 폴더 생성 + Odoo 소스 clone

```powershell
New-Item -ItemType Directory -Force C:\workspace\odoo-local | Out-Null
Set-Location C:\workspace\odoo-local

git clone --depth 1 --branch 18.0 https://github.com/odoo/odoo.git
New-Item -ItemType Directory -Force .\config,.\custom-addons | Out-Null
```

---

## 3) Dockerfile 생성

파일 경로: `C:\workspace\odoo-local\Dockerfile`

```powershell
@'
FROM python:3.11-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    libxml2-dev \
    libxslt1-dev \
    libldap2-dev \
    libsasl2-dev \
    libjpeg-dev \
    zlib1g-dev \
    libffi-dev \
    libssl-dev \
    postgresql-client \
    wkhtmltopdf \
    && rm -rf /var/lib/apt/lists/*

COPY odoo/requirements.txt /tmp/requirements.txt
RUN pip install --no-cache-dir -U pip setuptools wheel && \
    pip install --no-cache-dir -r /tmp/requirements.txt

WORKDIR /opt/odoo
'@ | Set-Content -Path .\Dockerfile -Encoding ASCII
```

---

## 4) docker-compose.yml 생성

파일 경로: `C:\workspace\odoo-local\docker-compose.yml`

```powershell
@'
services:
  db:
    image: postgres:16
    container_name: odoo18-db
    environment:
      POSTGRES_USER: odoo
      POSTGRES_PASSWORD: odoo
      POSTGRES_DB: postgres
    volumes:
      - odoo-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U odoo -d postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  odoo:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: odoo18-web
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "8069:8069"
    volumes:
      - ./odoo:/opt/odoo
      - ./config/odoo.conf:/etc/odoo/odoo.conf
      - ./custom-addons:/mnt/extra-addons
      - odoo-web-data:/var/lib/odoo
    command: python /opt/odoo/odoo-bin -c /etc/odoo/odoo.conf

volumes:
  odoo-db-data:
  odoo-web-data:
'@ | Set-Content -Path .\docker-compose.yml -Encoding ASCII
```

---

## 5) odoo.conf 생성

파일 경로: `C:\workspace\odoo-local\config\odoo.conf`

```powershell
@'
[options]
admin_passwd = admin
db_host = db
db_port = 5432
db_user = odoo
db_password = odoo
addons_path = /opt/odoo/addons,/mnt/extra-addons
data_dir = /var/lib/odoo
http_port = 8069
'@ | Set-Content -Path .\config\odoo.conf -Encoding ASCII
```

---

## 6) 실행

```powershell
Set-Location C:\workspace\odoo-local
docker compose up -d --build
docker compose ps
docker compose logs -f odoo
```

브라우저 접속:

- `http://localhost:8069`

초기 DB 생성 예시:

- Master Password: `admin`
- Database Name: `odoo18_dev`
- Email / Password: 원하는 계정 정보 입력

---

## 7) 개발 시 자주 쓰는 명령

```powershell
# 중지
docker compose down

# 재시작
docker compose up -d

# 로그 확인
docker compose logs -f odoo

# 특정 모듈 업데이트 (예: my_module)
docker compose exec odoo python /opt/odoo/odoo-bin -c /etc/odoo/odoo.conf -d odoo18_dev -u my_module --stop-after-init

# DB/볼륨까지 완전 초기화
docker compose down -v
```

---

## 8) Git 커스텀 모듈 연동 개발 루틴

`custom-addons`에 외부 모듈 저장소를 clone하고 Odoo에서 바로 읽도록 구성합니다.

```powershell
cd C:\workspace\odoo-local\custom-addons
git clone https://github.com/<org>/<repo1>.git
git clone https://github.com/<org>/<repo2>.git
```

`odoo.conf`의 `addons_path`는 아래처럼 유지:

```ini
addons_path = /opt/odoo/addons,/mnt/extra-addons
```

`docker-compose.yml` 볼륨 마운트 확인:

```yaml
volumes:
  - ./custom-addons:/mnt/extra-addons
```

코드 수정 후 반영 루틴:

```powershell
docker compose exec odoo python /opt/odoo/odoo-bin -c /etc/odoo/odoo.conf -d odoo18_dev -u <module_name> --stop-after-init
docker compose restart odoo
```

---

## 9) 문제 해결 체크리스트

- `docker compose up` 실패:
  - Docker Desktop이 실행 중인지 확인
  - 포트 `8069` 점유 여부 확인
- 모듈이 앱 목록에 안 보임:
  - 모듈 폴더에 `__manifest__.py` 존재 확인
  - Apps에서 `Update Apps List` 또는 모듈 업데이트 명령 실행
- 권한/의존성 오류:
  - 컨테이너 로그 확인: `docker compose logs -f odoo`

---

이 구성을 사용하면 Odoo 소스는 Git으로 버전 관리하고, 실행 환경은 Docker로 고정하여 로컬 개발 재현성을 높일 수 있습니다.
