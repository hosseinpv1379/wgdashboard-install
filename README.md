# WGDashboard — نصب

فایل‌های نصب پنل WGDashboard و ایجنت نود (`wg-node`). سورس کد پنل خصوصی است؛
نصب فقط از طریق Docker و ایمیج‌های از پیش ساخته‌شده (`ghcr.io/hosseinpv1379/*`)
انجام می‌شود.

## باز کردن پورت‌های لازم روی فایروال

```bash
# روی سرور پنل
ufw allow 10086/tcp   # پنل وب
ufw allow 51820/udp   # وایرگارد (نود محلی پنل)

# روی سرور هر نود
ufw allow 51820/udp   # وایرگارد
```

اگه از پروفایل HTTPS استفاده می‌کنی، `80/tcp` و `443/tcp` رو هم روی سرور پنل باز کن.

## نصب پنل

### دانلود فایل‌ها

```bash
mkdir -p /opt/WGDashboard/docker && cd /opt/WGDashboard/docker
curl -fsSL https://raw.githubusercontent.com/hosseinpv1379/wgdashboard-install/main/docker/compose.yaml -o compose.yaml
curl -fsSL https://raw.githubusercontent.com/hosseinpv1379/wgdashboard-install/main/docker/.env.example -o .env
```

### کانفیگ `.env`

```bash
nano .env
```

حداقل این مقادیر رو ست کن:

| متغیر | توضیح |
| --- | --- |
| `WGD_ADMIN_PASSWORD` | پسورد ادمین پنل |
| `POSTGRES_PASSWORD` | پسورد دیتابیس |
| `PUBLIC_IP` | آی‌پی عمومی سرور |
| `WGD_DOMAIN` | (اختیاری) دومین برای HTTPS خودکار — با ست شدنش `COMPOSE_PROFILES=https` رو هم بزن |

### اجرا

```bash
docker compose --env-file .env pull
docker compose --env-file .env up -d
```

پنل بالا میاد روی `http://PUBLIC_IP:10086` (یا `https://WGD_DOMAIN` اگه HTTPS رو فعال کردی).

### توقف

```bash
docker compose --env-file .env down
```

### آپدیت

```bash
docker compose --env-file .env pull
docker compose --env-file .env up -d --remove-orphans
```

### لاگ‌ها

```bash
docker compose --env-file .env logs -f wgdashboard
```

## نصب نود

هر نود روی سرور جدا نصب می‌شه. اول از **پنل → Nodes** یک نود بساز و **Node ID** و **Node Token** رو کپی کن.

### دانلود فایل‌ها

```bash
mkdir -p /opt/wg-node && cd /opt/wg-node
curl -fsSL https://raw.githubusercontent.com/hosseinpv1379/wgdashboard-install/main/wg-node/compose.example.yaml -o compose.yaml
curl -fsSL https://raw.githubusercontent.com/hosseinpv1379/wgdashboard-install/main/wg-node/.env.example -o .env
```

### کانفیگ `.env`

```bash
nano .env
```

حداقل این مقادیر رو ست کن:

| متغیر | توضیح |
| --- | --- |
| `WG_PANEL_URL` | آدرس پنل، مثل `https://panel.example.com` |
| `WG_NODE_ID` / `WG_NODE_TOKEN` | از صفحه‌ی Nodes پنل |
| `WG_NODE_PUBLIC_ENDPOINT` | آی‌پی این سرور نود، مثل `1.2.3.4:51820` |
| `WG_NODE_NAME` / `WG_NODE_REGION` | نام و لوکیشن نود |

### اجرا

```bash
docker compose --env-file .env pull
docker compose --env-file .env up -d
```

### توقف

```bash
docker compose --env-file .env down
```

### آپدیت

```bash
docker compose --env-file .env pull
docker compose --env-file .env up -d
```

### لاگ‌ها

```bash
docker compose --env-file .env logs -f
```
