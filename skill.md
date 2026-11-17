---
name: vsphere8-authentik-ldaps-mfa
description: >
  Kiến trúc Giải pháp Zero-Trust & Thiết kế Hệ thống Thực tế bảo vệ VMware vSphere 8.0 U3
  chống Ransomware: Triển khai Authentik IdP trên Ubuntu 24.04 LTS, tích hợp LDAPS (636)
  đa domain AD1 (corp.vn) và AD2 (cloud.local), đồng bộ thuộc tính Mobile Phone từ AD,
  cưỡng chế QR Code TOTP MFA (Microsoft Authenticator / Google Authenticator), SSO vCenter 8.0 U3 (443),
  chính sách mật khẩu an toàn và bảo mật SSH Port 22 (PPK + MFA).
target_local_path: C:\VFF8\MFA\skill.md
version: 1.0.0
---

# Kiến Trúc Bảo Mật & Triển Khai MFA Toàn Diện cho VMware vSphere 8.0 U3

> **Bối cảnh & Mục tiêu**: Khắc phục triệt để các lỗ hổng tầng OS/Appliance và xác thực đơn yếu tố (nguyên nhân chính dẫn đến các cuộc tấn công mã hóa dữ liệu / Ransomware tương tự kịch bản NAS Synology bị khai thác OS CVE hoặc lộ mật khẩu admin). Tài liệu này thiết kế mô hình **Zero-Trust Identity & Management Isolation** cho cụm **VMware vSphere 8.0 U3**, đảm bảo mọi phiên truy cập quản trị đều phải đi qua **Vá lỗ hổng mới nhất + Xác thực đa yếu tố (MFA QR Code) + Cách ly mạng quản trị (Bastion PPK+MFA)**.

---

## 1. Phân Tích Rủi Ro & Tổng Quan Kiến Trúc Giải Pháp

### 1.1. Bài học từ sự cố Ransomware (NAS OS CVE) áp dụng sang vSphere 8.0 U3
| Bề mặt tấn công | Kịch bản Ransomware khai thác | Biện pháp triệt tiêu trong thiết kế này |
| :--- | :--- | :--- |
| **OS / Appliance CVE** | Chưa nâng cấp build bảo mật mới nhất cho OS/Web Admin dẫn tới RCE. | Patch **VCSA 8.0 U3** & **ESXi 8.0 U3** lên bản build mới nhất; bật auto security patch trên **Ubuntu 24.04 LTS**. |
| **vCenter Web UI (443)** | Đăng nhập chỉ dùng Username/Password, không có MFA; lộ mật khẩu Domain Admin. | Chuyển SSO sang **Authentik OIDC Federation**; ép buộc **TOTP QR (MS/Google Authenticator)** kết hợp kiểm tra số `mobile` từ AD. |
| **LDAP Cleartext (389)** | Nghe lén hoặc giả mạo kết nối LDAP giữa ứng dụng và Domain Controller. | Cấm port 389; bắt buộc **LDAPS (TCP 636)** xác thực chứng chỉ **Enterprise Root CA**. |
| **SSH Port 22 & ESXi Shell** | Attacker bật SSH hoặc dùng key/mật khẩu bị lộ để chạy script mã hóa VMFS/VMDK. | **Tắt SSH mặc định** trên VCSA & ESXi; bật **ESXi Lockdown Mode**; bắt buộc đi qua **Bastion (PPK Ed25519 + TOTP MFA)**. |
| **Break-Glass Accounts** | Lạm dụng `root` hoặc `Administrator@vsphere.local` làm tài khoản trực vận hành hàng ngày. | Tách biệt hoàn toàn: Daily Ops dùng AD + MFA; `root` và `Administrator@vsphere.local` khóa trong PAM Vault + SIEM Alert. |

### 1.2. Kiến trúc Luồng Xác thực Thực tế (Identity & Network Flow)

```text
[Admin Browser / SSH Client]
   │
   ├──► (1) HTTPS 443 ──► [vCenter 8.0 U3 (vcenter.corp.vn / 10.10.10.10)]
   │                             │ (OIDC Auth Redirect)
   │                             ▼
   │                      [Authentik IdP on Ubuntu 24.04 (sso.corp.vn / 10.10.20.15)]
   │                             │
   │                             ├──► (2) LDAPS 636 (TLS + Root CA) ──► [AD-DC1: corp.vn (10.10.20.11)]
   │                             ├──► (3) LDAPS 636 (TLS + Root CA) ──► [AD-DC2: cloud.local (10.10.30.11)]
   │                             │      └─ Verify Password + Sync 'mobile' & 'memberOf'
   │                             │
   │                             └──► (4) MFA Enrollment / Validation Flow
   │                                    ├─ Policy Check: AD 'mobile' attribute must exist
   │                                    └─ TOTP QR Code ──► [MS Authenticator / Google Authenticator]
   │
   └──► (5) SSH Port 22 (PPK Ed25519 + TOTP MFA) ──► [Mgmt Bastion (10.10.99.10)]
                                                             │
                                                             ├──► VCSA VAMI 5480 / SSH 22 (Break-glass only)
                                                             └──► ESXi DCUI / SSH (Stopped by default + Lockdown Mode)
```

### 1.3. Điểm Khác Biệt Kỹ Thuật Quan Trọng trên vSphere 8.0 U3
- **Tại sao KHÔNG cấu hình vCenter nối thẳng AD LDAPS để làm MFA?**
  Giao thức LDAP/LDAPS chuẩn trên vCenter chỉ thực hiện `Bind(username, password)`. Nó **không có giao diện** để hiển thị QR Code hay prompt nhập mã OTP từ MS/Google Authenticator.
- **Giải pháp chuẩn Production**:
  Sử dụng **vCenter 8.0 U3 Identity Provider Federation (OIDC)** kết nối với **Authentik**. Authentik đóng vai trò **Identity Broker**: kết nối tới cả **AD1 (`corp.vn`)** và **AD2 (`cloud.local`)** qua **LDAPS 636**, kiểm tra thuộc tính `mobile`, hiển thị **QR Code TOTP** cho người dùng tự đăng ký bằng điện thoại, sau đó trả OIDC Token về cho vCenter 8.0 U3.

---

## 2. Kế Hoạch Chuẩn Bị Triển Khai (Prerequisites & Matrix)

### 2.1. Thông số Hạ tầng & Quy hoạch Mạng
| Thành phần | FQDN | IP / Subnet | OS / Version | Vai trò |
| :--- | :--- | :--- | :--- | :--- |
| **vCenter Server** | `vcenter.corp.vn` | `10.10.10.10` | VCSA 8.0 U3 (Latest Build) | SSO Service Provider (OIDC) |
| **Authentik IdP** | `sso.corp.vn` | `10.10.20.15` | Ubuntu 24.04 LTS + Docker | External IdP & LDAPS Broker |
| **AD1 Domain Controller** | `dc1.corp.vn` | `10.10.20.11` | Windows Server AD DS (`corp.vn`) | Primary Directory (LDAPS 636) |
| **AD2 Domain Controller** | `dc2.cloud.local` | `10.10.30.11` | Windows Server AD DS (`cloud.local`) | Secondary Directory (LDAPS 636) |
| **Bastion / Jump Host** | `bastion.corp.vn` | `10.10.99.10` | Hardened Linux / Windows | Cổng duy nhất vào 5480 / 22 |
| **Management VLAN** | — | `10.10.10.0/24` | Isolated VLAN | Mạng quản trị cô lập |

### 2.2. Ma trận Firewall Rules Bắt Buộc
| Source | Destination | Port / Protocol | Mục đích |
| :--- | :--- | :--- | :--- |
| Trusted Admin / Bastion | `10.10.10.10` (vCenter) | `443/TCP` | vSphere Client Web UI |
| Trusted Admin / Bastion | `10.10.20.15` (Authentik) | `443/TCP` | Authentik SSO Portal & QR Enrollment |
| `10.10.10.10` (vCenter) | `10.10.20.15` (Authentik) | `443/TCP` | OIDC Token Exchange & JWKS Fetch |
| `10.10.20.15` (Authentik) | `10.10.20.11`, `10.10.30.11` | `636/TCP` (LDAPS) | Đồng bộ User/Group/Mobile & Bind Auth |
| `10.10.99.10` (Bastion ONLY) | `10.10.10.10`, ESXi, `10.10.20.15` | `22/TCP`, `5480/TCP` | SSH PPK+MFA & VAMI Patching (Cấm từ IP khác) |
| All Hosts (VCSA, ESXi, AD, Ubuntu) | Internal NTP Server | `123/UDP` | **Bắt buộc**: Lệch giờ > 30s sẽ lỗi mã TOTP |

### 2.3. Chuẩn bị Active Directory (AD1 `corp.vn` & AD2 `cloud.local`)
1. **Kích hoạt LDAPS 636**: Cài đặt **AD CS (Enterprise CA)** hoặc cấp chứng chỉ Server Authentication cho `dc1.corp.vn` và `dc2.cloud.local`. Kiểm tra từ Linux:
   ```bash
   openssl s_client -connect 10.10.20.11:636 -showcerts </dev/null
   openssl s_client -connect 10.10.30.11:636 -showcerts </dev/null
   ```
2. **Xuất chứng chỉ Root CA**: Lưu file `ad1-root-ca.crt` và `ad2-root-ca.crt` (định dạng Base-64 PEM).
3. **Tạo Service Account Bind (Read-Only)**:
   - Trên AD1 & AD2, tạo user `svc_authentik_ldap` (chỉ thuộc nhóm `Domain Users`, mật khẩu 24+ ký tự, không hết hạn hoặc xoay vòng tự động, chỉ cấp quyền đọc thuộc tính cơ bản + `mobile`).
4. **Cập nhật thuộc tính `mobile` và `mail` trên AD1, AD2**:
   - Trong **ADUC** (hoặc PowerShell), điền số điện thoại chuẩn **E.164** (ví dụ `+84901234567`) vào trường **Mobile** (`mobile`) cho toàn bộ nhân sự được phép truy cập vSphere 8:
   ```powershell
   Set-ADUser -Identity "nguyen.vana" -MobilePhone "+84901234567" -EmailAddress "nguyen.vana@corp.vn"
   ```
5. **Tạo AD Security Groups phân quyền vSphere**:
   - `GG_vSphere_Admins` (Toàn quyền quản trị vCenter)
   - `GG_vSphere_Operators` (Vận hành VM, Snapshot, Console — không sửa Host/SSO)
   - `GG_vSphere_ReadOnly` (Chỉ xem Audit/Monitoring)

---

## 3. Triển Khai Authentik trên Ubuntu 24.04 LTS Server

### 3.1. Cài đặt & Hardening OS Ubuntu 24.04 LTS
```bash
# 1. Cập nhật OS & cài công cụ bảo mật (vá CVE tương tự bài học NAS)
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y unattended-upgrades fail2ban ufw curl gnupg ca-certificates chrony

# 2. Đồng bộ thời gian NTP chính xác cho TOTP 30s
sudo systemctl enable --now chrony
chronyc tracking

# 3. Cấu hình UFW Firewall chặt chẽ
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.10.99.10/32 to any port 22 proto tcp comment 'SSH from Bastion only'
sudo ufw allow 443/tcp comment 'Authentik HTTPS SSO'
sudo ufw enable

# 4. Cài đặt Docker Engine & Compose Plugin
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### 3.2. Triển khai Authentik Container Stack
Tạo thư mục `/opt/authentik` và cấu hình file `.env` + `docker-compose.yml`:

```bash
sudo mkdir -p /opt/authentik/{certs,media,custom-templates}
cd /opt/authentik

# Sinh mật khẩu DB & Secret Key ngẫu nhiên chuẩn cryptographic
echo "PG_PASS=$(openssl rand -base64 36 | tr -d '\n')" | sudo tee .env
echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')" | sudo tee -a .env
echo "AUTHENTIK_ERROR_REPORTING__ENABLED=false" | sudo tee -a .env
sudo chmod 600 .env
```

File `/opt/authentik/docker-compose.yml`:
```yaml
services:
  postgresql:
    image: docker.io/library/postgres:16-alpine
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      interval: 30s
      timeout: 5s
      retries: 5
    volumes:
      - database:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${PG_PASS:?database password required}
      POSTGRES_USER: authentik
      POSTGRES_DB: authentik
    env_file:
      - .env

  redis:
    image: docker.io/library/redis:alpine
    command: --save 60 1 --loglevel warning
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
      interval: 30s
      timeout: 3s
      retries: 5
    volumes:
      - redis:/data

  server:
    image: ghcr.io/goauthentik/server:2024.10
    restart: unless-stopped
    command: server
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: authentik
      AUTHENTIK_POSTGRESQL__NAME: authentik
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
    volumes:
      - ./media:/media
      - ./custom-templates:/templates
      - ./certs:/certs:ro
    env_file:
      - .env
    ports:
      - "127.0.0.1:9000:9000"
      - "127.0.0.1:9443:9443"
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy

  worker:
    image: ghcr.io/goauthentik/server:2024.10
    restart: unless-stopped
    command: worker
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: authentik
      AUTHENTIK_POSTGRESQL__NAME: authentik
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
    volumes:
      - ./media:/media
      - ./certs:/certs:ro
      - ./custom-templates:/templates
    env_file:
      - .env
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy

volumes:
  database:
  redis:
```

```bash
# Khởi chạy Authentik
sudo docker compose up -d
```

### 3.3. Cấu hình Nginx Reverse Proxy TLS 1.3 trên Port 443
Cài đặt Nginx làm TLS termination với chứng chỉ `sso.corp.vn.crt` do Enterprise CA ký (hoặc cert hợp lệ nội bộ) và chặn TLS 1.0/1.1:
```nginx
server {
    listen 443 ssl http2;
    server_name sso.corp.vn;

    ssl_certificate /etc/ssl/authentik/sso.corp.vn.fullchain.crt;
    ssl_certificate_key /etc/ssl/authentik/sso.corp.vn.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:9000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Truy cập lần đầu: `https://sso.corp.vn/if/flow/initial-setup/` để khởi tạo tài khoản `akadmin` (đặt mật khẩu 24+ ký tự và bật ngay TOTP MFA cho `akadmin`).

---

## 4. Cấu Hình LDAPS 636 (AD1 & AD2), Mobile Phone & QR Code MFA

### 4.1. Import Enterprise Root CA vào Authentik
1. Vào **Admin Interface → System → Certificates → Create**.
2. Upload chứng chỉ `ad1-root-ca.crt` (đặt tên `AD1-corp.vn-RootCA`) và `ad2-root-ca.crt` (đặt tên `AD2-cloud.local-RootCA`).
3. Bỏ qua bước nhập Private Key (chỉ cần Certificate để xác thực LDAPS 636).

### 4.2. Tạo Custom Property Mapping cho Mobile Phone & Tránh Trùng Tên Miền
Vì hệ thống có **2 Forest/Domain độc lập** (`corp.vn` và `cloud.local`), cần đồng bộ thuộc tính `mobile` từ AD và giữ `userPrincipalName` làm định danh duy nhất.
Vào **Customization → Property Mappings → Create → LDAP Source Property Mapping**:
- **Name**: `AD Mobile & Domain Metadata Mapping`
- **Expression (Python)**:
```python
mobile = ldap.get("mobile")
if isinstance(mobile, list) and len(mobile) > 0:
    mobile = mobile[0]

upn = ldap.get("userPrincipalName")
if isinstance(upn, list) and len(upn) > 0:
    upn = upn[0]

domain = upn.split("@")[-1] if upn and "@" in upn else "unknown"

return {
    "username": upn or ldap.get("sAMAccountName"),
    "attributes": {
        "mobile": mobile or "",
        "ad_domain": domain,
        "ad_mobile_verified": bool(mobile and str(mobile).strip()),
        "samAccountName": ldap.get("sAMAccountName"),
    }
}
```

### 4.3. Kết nối LDAP Source cho AD1 (`corp.vn`) & AD2 (`cloud.local`)
Vào **Directory → Federation and Social login → Create → LDAP Source**:

| Thông số | AD1 (`corp.vn`) | AD2 (`cloud.local`) |
| :--- | :--- | :--- |
| **Name / Slug** | `AD1 Corp` / `ad1-corp-vn` | `AD2 Cloud` / `ad2-cloud-local` |
| **Server URI** | `ldaps://dc1.corp.vn:636` | `ldaps://dc2.cloud.local:636` |
| **Enable StartTLS** | `False` (dùng SSL/TLS thuần 636) | `False` |
| **TLS Verification Certificate** | `AD1-corp.vn-RootCA` | `AD2-cloud.local-RootCA` |
| **Bind CN** | `CN=svc_authentik_ldap,OU=ServiceAccounts,DC=corp,DC=vn` | `CN=svc_authentik_ldap,OU=ServiceAccounts,DC=cloud,DC=local` |
| **Base DN** | `DC=corp,DC=vn` | `DC=cloud,DC=local` |
| **User Object Filter** | `(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))` | `(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))` |
| **User Property Mappings** | Chọn mặc định + `AD Mobile & Domain Metadata Mapping` | Chọn mặc định + `AD Mobile & Domain Metadata Mapping` |
| **Sync Users / Groups** | Bật cả 2 (`Sync Users`, `Sync Groups`) | Bật cả 2 (`Sync Users`, `Sync Groups`) |

> Sau khi lưu, nhấn vào từng LDAP Source → kiểm tra **Connectivity** phải báo **Healthy** và chạy **Sync Now** → kiểm tra mục **Directory → Users**, xác nhận cột `Attributes` đã có `"mobile": "+84..."`.

### 4.4. Cấu Hình Luồng Kiểm Tra Mobile AD & Quét QR Code (MS / Google Authenticator)
1. **Tạo Expression Policy kiểm tra thuộc tính `mobile` từ AD**:
   Vào **Customization → Policies → Create → Expression Policy**:
   - **Name**: `Require AD Mobile Phone for MFA`
   - **Expression**:
   ```python
   mobile = str(request.user.attributes.get("mobile", "")).strip()
   if not mobile or len(mobile) < 9:
       ak_message("Tài khoản AD của bạn chưa được khai báo số điện thoại Mobile trên AD1/AD2. Vui lòng liên hệ IT Security để cập nhật trước khi đăng ký QR MFA.")
       return False
   return True
   ```
2. **Cấu hình TOTP Authenticator Setup Stage** (`Flows and Stages → Stages`):
   - Tạo hoặc chỉnh `default-authenticator-totp-setup`:
     - **Digits**: `6`
     - **Friendly name**: `vSphere Corp SSO` (tên hiển thị trong MS Authenticator / Google Authenticator).
   - Bind Policy `Require AD Mobile Phone for MFA` vào Stage Binding của `default-authenticator-totp-setup`.
3. **Cấu hình Authentication Flow (`default-authentication-flow`)**:
   - **Stage 10**: `default-authentication-identification` (cho phép nhập UPN `user@corp.vn` hoặc `user@cloud.local`; kích hoạt `Passwordless / Passkey` nếu cần).
   - **Stage 20**: `default-authentication-password` (Backends: chọn `LDAP Backend` để Authentik verify mật khẩu trực tiếp qua LDAPS 636 của AD1/AD2, **không lưu mật khẩu trong Authentik**).
   - **Stage 30**: `default-authentication-mfa-validation`:
     - **Device classes**: Chọn **TOTP Authenticators**.
     - **Not configured action**: **`Force user to configure`** → Chọn Configuration stages: `default-authenticator-totp-setup`.
   - **Stage 40**: `default-authentication-login`.

**Trải nghiệm người dùng thực tế khi kích hoạt**:
- User `admin@corp.vn` hoặc `ops@cloud.local` đăng nhập lần đầu vào vCenter 443 → được chuyển hướng sang `https://sso.corp.vn` → nhập username/password AD → Authentik kiểm tra `mobile` từ AD hợp lệ → hiển thị **QR Code**.
- User mở app **Microsoft Authenticator** hoặc **Google Authenticator** trên điện thoại → chọn **Scan a QR code** → nhập 6 số OTP xác nhận → hoàn tất đăng ký và tự động SSO vào vSphere 8.0 U3.

---

## 5. Cấu Hình vCenter 8.0 U3 (Port 443) với Authentik IdP & CA

### 5.1. Tạo OAuth2 / OpenID Connect Provider trên Authentik
1. Vào **Applications → Providers → Create → OAuth2/OpenID Provider**:
   - **Name**: `vCenter-8-OIDC`
   - **Authentication flow**: `default-authentication-flow` (đã ép MFA QR ở trên)
   - **Authorization flow**: `default-provider-authorization-implicit-consent`
   - **Client type**: `Confidential`
   - **Client ID** & **Client Secret**: Lưu lại để nhập vào vCenter.
   - **Redirect URIs / Origins**:
     - `https://vcenter.corp.vn/ui/oidc/callback`
     - `https://vcenter.corp.vn/ui/login/oauth2/code/*`
     - `https://vcenter.corp.vn/*` *(lưu ý thu hẹp theo đúng Callback URI mà wizard của vCenter 8.0 U3 hiển thị khi khởi tạo OIDC)*.
   - **Signing Key**: Chọn chứng chỉ ký RS256 của Authentik.
   - **Scopes**: `openid`, `profile`, `email`, `groups` (tạo Scope Mapping trả về danh sách AD Group).
2. Vào **Applications → Applications → Create**:
   - **Name**: `VMware vCenter 8.0 U3`
   - **Slug**: `vcenter-8`
   - **Provider**: `vCenter-8-OIDC`

### 5.2. Import CA của Authentik vào vCenter 8.0 U3
vCenter **bắt buộc** phải tin cậy chuỗi chứng chỉ TLS của `https://sso.corp.vn`, nếu không bước trao đổi OIDC Token sẽ báo lỗi SSL Handshake:
1. Đăng nhập vSphere Client bằng `Administrator@vsphere.local` (với quyền Admin tạm thời trước khi chuyển IdP).
2. Vào **Administration → Certificates → Certificate Management → Trusted Root Certificates → ADD**.
3. Import file **Root CA & Intermediate CA** đã ký cho `sso.corp.vn`.
4. *(Hoặc qua SSH/Console VCSA)*:
   ```bash
   /usr/lib/vmware-vmafd/bin/dir-cli trustedcert publish --cert /tmp/enterprise-root-ca.crt
   ```

### 5.3. Thiết lập Identity Provider (IdP) trên vCenter 8.0 U3
1. Vào **Administration → Single Sign-On → Configuration → Identity Provider**.
2. Nhấn **Change Identity Provider** → Chọn **OpenID Connect (Generic OIDC)** (hoặc ADFS/Okta nếu dùng tương thích; vSphere 8.0 U2/U3 hỗ trợ Generic OIDC / External IdP).
3. Copy **Redirect URI** mà vCenter cung cấp dán ngược lại vào Authentik Provider ở Bước 5.1.
4. Nhập thông số từ Authentik (`Applications → Providers → vCenter-8-OIDC`):
   - **OpenID Address (Discovery URL)**: `https://sso.corp.vn/application/o/vcenter-8/.well-known/openid-configuration`
   - **Client Identifier**: `<Client ID từ Authentik>`
   - **Shared Secret**: `<Client Secret từ Authentik>`
5. Hoàn tất Wizard. Từ lúc này, khi mở `https://vcenter.corp.vn/ui`, vCenter sẽ điều hướng qua Authentik + AD LDAPS 636 + TOTP QR MFA!
6. **Phân quyền Access trên vCenter**:
   - Vào **Administration → Access Control → Global Permissions → Add**.
   - Chọn Identity Source là IdP / AD Domain tương ứng → gán nhóm `GG_vSphere_Admins` quyền **Administrator** (propagate to children), `GG_vSphere_ReadOnly` quyền **Read-Only**.

---

## 6. Kích Hoạt MFA & Phân Quyền Theo Từng Loại Tài Khoản

### 6.1. Tài khoản AD Users (`corp.vn` & `cloud.local`)
- **MFA**: Bắt buộc 100% qua Authentik OIDC + TOTP (MS Authenticator / Google Authenticator) với điều kiện AD `mobile` đã khai báo.
- **Quản trị vòng đời**: Khi nhân sự nghỉ việc hoặc đổi bộ phận, chỉ cần **Disable User** trên AD1/AD2 → Authentik tự động vô hiệu hóa phiên đăng nhập trong chu kỳ đồng bộ (hoặc ngay lập tức khi bind LDAPS thất bại).

### 6.2. Tài khoản `Administrator@vsphere.local` (Break-Glass SSO)
- **Thực tế kỹ thuật**: Khi bật External IdP trên vCenter 8.0 U3, `Administrator@vsphere.local` vẫn tồn tại như tài khoản **Break-Glass cục bộ** (truy cập qua tùy chọn *"Sign in with local account"* hoặc URL `https://vcenter.corp.vn/ui?local=true` khi Authentik/AD sập toàn bộ). Kho SSO nội bộ `vsphere.local` **không** tự sinh QR Code TOTP độc lập.
- **Biện pháp bảo vệ tương đương MFA**:
  1. **Cấm sử dụng hàng ngày**: Tuyệt đối không dùng `Administrator@vsphere.local` cho tác vụ vận hành thường nhật.
  2. **Enterprise PAM Vault + MFA**: Lưu mật khẩu 24 ký tự vào Vault (Bitwarden Enterprise / CyberArk / HashiCorp Vault) có **bắt buộc MFA + Phê duyệt (Dual Control)** mới được xem mật khẩu.
  3. **Cô lập mạng + SIEM Alert**: Khóa quyền truy cập VCSA 443 với đường `?local=true` hoặc cấu hình Syslog gửi cảnh báo High/Critical ngay lập tức tới SOC/Telegram/Email bất cứ khi nào sự kiện `UserLoggedIn` với `Administrator@vsphere.local` phát sinh.

### 6.3. Tài khoản `root` (VCSA 8.0 U3 & ESXi 8.0 U3)
- **VCSA `root` (Photon OS & VAMI 5480)**:
  - Không dùng trên vSphere Client 443. Chỉ dùng cho VAMI (`https://vcenter.corp.vn:5480`) để backup/patch và cứu hộ OS.
  - Bật **VCSA Appliance Firewall** (trong VAMI hoặc vSphere Client) chỉ cho phép IP Bastion (`10.10.99.10`) kết nối port `5480` và `22`.
  - Khi cần SSH port 22: bắt buộc đi qua **Bastion (đã có PPK + MFA)** và kích hoạt `pam_google_authenticator` nếu mở SSH trực tiếp.
- **ESXi `root` (Hypervisor Hosts)**:
  - Bật **Lockdown Mode = Normal** trên toàn bộ ESXi Host (`Host → Configure → System → Security Profile → Lockdown Mode`). Khi bật Normal Lockdown, không ai có thể đăng nhập trực tiếp bằng `root` qua ESXi Host Client (443) hay SSH; mọi thao tác phải qua vCenter (đã bảo vệ bởi Authentik MFA).
  - Trong trường hợp vCenter chết hoàn toàn, chỉ dùng **DCUI (iDRAC/iLO console đã có MFA)** để tạm thời tắt Lockdown Mode hoặc dùng tài khoản trong danh sách **DCUI Exception Users**.

---

## 7. Chính Sách Mật Khẩu: Độ Dài, Độ Khó, Khóa Tài Khoản & Độ Tuổi

| Đối tượng tài khoản | Độ dài tối thiểu | Độ phức tạp (Complexity) | Khóa sai (Lockout) | Độ tuổi thay đổi (Max Age) | Lịch sử mật khẩu |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **AD Users (`corp.vn`, `cloud.local`)** | **14 – 16 ký tự** | 4 nhóm ký tự (Hoa, Thường, Số, Đặc biệt) + Chặn từ điển yếu (`corp`, `vsphere`) | Sai **5 lần** / 15 phút → Khóa **15 phút** | **90 ngày** (vSphere Admins) / **180 ngày** (User thường có MFA) | Không trùng **12** lần gần nhất |
| **`Administrator@vsphere.local`** | **20 – 24 ký tự** | Tối thiểu 2 Hoa, 2 Thường, 2 Số, 2 Ký tự đặc biệt; tối đa 2 ký tự giống nhau liền kề | Sai **3 lần** / 180 giây → Khóa **900 giây (15 phút)** | **90 ngày** (đồng bộ lịch xoay vòng trong PAM Vault) | Không trùng **10** lần gần nhất |
| **`root` VCSA 8.0 U3** | **20 – 24 ký tự** | `pam_pwquality`: `minlen=16`, `ucredit=-1`, `lcredit=-1`, `dcredit=-1`, `ocredit=-1`, `difok=8` | `pam_faillock`: `deny=3`, `unlock_time=900` | **90 ngày** *(Mặc định Photon OS hết hạn sau 90 ngày — kiểm tra bằng `chage -l root`)* | Không trùng **5** lần gần nhất |
| **`root` ESXi 8.0 U3** | **20 – 24 ký tự** | `Security.PasswordQualityControl`: `retry=3 min=disabled,disabled,disabled,disabled,16` | `AccountLockFailures = 3`, `AccountUnlockTime = 900` | **90 – 180 ngày** (Xoay vòng ngay khi có biến động nhân sự IT) | Không trùng **5** lần gần nhất |

### Lệnh cấu hình thực tế trên vCenter SSO & ESXi:
- **Trên vCenter SSO (Web UI)**: `Administration → Single Sign-On → Configuration → Local Accounts → Password Policy` và `Lockout Policy` → điền chính xác các tham số bảng trên.
- **Trên ESXi 8.0 U3 (qua PowerCLI hoặc vCenter Host Advanced System Settings)**:
  ```powershell
  Get-VMHost | Get-AdvancedSetting -Name "Security.AccountLockFailures" | Set-AdvancedSetting -Value 3 -Confirm:$false
  Get-VMHost | Get-AdvancedSetting -Name "Security.AccountUnlockTime" | Set-AdvancedSetting -Value 900 -Confirm:$false
  Get-VMHost | Get-AdvancedSetting -Name "Security.PasswordQualityControl" | Set-AdvancedSetting -Value "retry=3 min=disabled,disabled,disabled,disabled,16" -Confirm:$false
  Get-VMHost | Get-AdvancedSetting -Name "UserVars.ESXiShellTimeOut" | Set-AdvancedSetting -Value 900 -Confirm:$false
  Get-VMHost | Get-AdvancedSetting -Name "UserVars.ESXiShellInteractiveTimeOut" | Set-AdvancedSetting -Value 900 -Confirm:$false
  ```

---

## 8. Bảo Mật Cổng SSH Port 22: PPK Hay MFA?

### 8.1. Kết luận Kiến trúc: BẮT BUỘC CẢ HAI (PPK + TOTP MFA) + TẮT MẶC ĐỊNH
- **Nếu chỉ dùng PPK (`.ppk` / `id_ed25519`)**: Khi máy tính Admin bị nhiễm mã độc đánh cắp thông tin (Infostealer) hoặc bị chiếm quyền RDP, attacker copy file private key và truy cập thẳng port 22.
- **Nếu chỉ dùng Password + MFA**: Vẫn có nguy cơ brute-force hoặc khai thác lỗ hổng pre-auth trên daemon SSH nếu mở port rộng.
- **Mô hình chuẩn (Defense-in-Depth)**:
  1. **Trên Bastion Host (10.10.99.10)**: Mở Port 22 có chọn lọc và **cưỡng chế 2 lớp đồng thời**:
     - Lớp 1: **SSH Public Key (Ed25519 / RSA-4096 có đặt Passphrase cho file `.ppk`)**.
     - Lớp 2: **TOTP MFA (`libpam-google-authenticator` quét QR trên MS/Google Authenticator)**.
  2. **Trên VCSA 8.0 U3 & ESXi 8.0 U3**:
     - **Tắt dịch vụ SSH & Bash Shell mặc định** (trong VAMI 5480: `Access → Edit → SSH Login = Disabled`; trên ESXi: dừng `TSM-SSH`).
     - Chỉ bật tạm thời trong cửa sổ bảo trì (Auto-timeout 15 phút) và chỉ cho phép kết nối từ IP Bastion (`10.10.99.10`).

### 8.2. Cấu hình mẫu SSH PPK + TOTP MFA trên Bastion Ubuntu 24.04
```bash
# 1. Cài PAM Google Authenticator (tương thích 100% MS Authenticator & Google Authenticator)
sudo apt install -y libpam-google-authenticator

# 2. Mỗi Admin tự sinh QR Code (chạy dưới user của mình, KHÔNG chạy bằng root)
google-authenticator -t -d -f -r 3 -R 30 -w 3
# Quét QR Code hiển thị trên terminal bằng MS Authenticator / Google Authenticator
```

Cấu hình `/etc/pam.d/sshd` (bỏ comment hoặc thêm vào cuối file, **comment dòng `@include common-auth`** nếu muốn chỉ dùng Key + OTP mà không hỏi mật khẩu Linux):
```text
# @include common-auth
auth required pam_google_authenticator.so nullok
```
*(Sau khi tất cả admin đã quét QR, bỏ `nullok` để bắt buộc 100% user phải có TOTP).*

Cấu hình `/etc/ssh/sshd_config.d/99-hardening.conf`:
```text
Port 22
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication yes
ChallengeResponseAuthentication yes
UsePAM yes
AuthenticationMethods publickey,keyboard-interactive
MaxAuthTries 3
LoginGraceTime 30
AllowGroups ssh-vsphere-admins
```
```bash
sudo sshd -t && sudo systemctl restart ssh
```
*Khi kết nối bằng PuTTY / MobaXterm / OpenSSH: Client nạp file `.ppk` / `id_ed25519` + nhập Passphrase của key → Server hỏi tiếp `Verification code:` → nhập 6 số từ MS/Google Authenticator.*

> **Lưu ý đặc biệt cho VCSA 8.0 U3**: Không nên tùy biến sâu file `/etc/pam.d/sshd` trực tiếp trên VCSA vì mỗi lần **Update VCSA 8.0 U3** qua VAMI, bản vá có thể ghi đè cấu hình PAM/SSHD mặc định của Photon OS. Thay vào đó, hãy **tắt SSH VCSA mặc định**, cấu hình Firewall VCSA chỉ nhận SSH từ `10.10.99.10`, đặt `PermitRootLogin prohibit-password` (dùng SSH Key) và bắt buộc mọi kết nối SSH phải đi qua **Bastion (đã khóa chặt PPK + TOTP MFA)**.

---

## 9. Quy Trình Kiểm Thử & Vận Hành Sau Triển Khai

1. **Test SSO AD1 & AD2**: Đăng nhập `https://vcenter.corp.vn` bằng user thuộc `corp.vn` và user thuộc `cloud.local` → Xác nhận chuyển hướng sang `https://sso.corp.vn`, kiểm tra thuộc tính `mobile` AD, quét QR Code bằng MS Authenticator và Google Authenticator thành công.
2. **Test Chặn Thiếu Mobile**: Thử tạo 1 user test trên AD **không điền trường `mobile`** → xác nhận Authentik chặn không cho đăng ký QR MFA và hiển thị thông báo liên hệ IT Security.
3. **Test Break-Glass**: Tạm dừng container Authentik (`docker compose stop server`) → mở `https://vcenter.corp.vn/ui?local=true` → xác nhận vẫn đăng nhập cứu hộ bằng `Administrator@vsphere.local` từ Vault thành công, sau đó bật lại Authentik (`docker compose start server`).
4. **Test SSH Port 22**: Thử SSH trực tiếp vào VCSA/ESXi từ máy trạm thường → phải bị **Drop/Timeout**. Thử SSH vào Bastion thiếu PPK hoặc thiếu mã OTP → phải bị **Permission Denied**.
5. **Backup Định Kỳ**:
   - VCSA 8.0 U3: Bật lịch **File-Based Backup** hàng ngày trong VAMI 5480 đẩy sang Storage/SFTP hỗ trợ **Immutable / WORM** (chống ransomware xóa backup).
   - Authentik: Tự động `pg_dump` database PostgreSQL + backup file `.env` và thư mục `/opt/authentik/certs` hàng ngày ra kho lưu trữ tách biệt.
