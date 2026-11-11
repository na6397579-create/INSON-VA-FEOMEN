# INSON VA FENOMEN — sayt, so'rovnoma va admin panel (Django)

- `/` — sayt va so'rovnoma
- `/api/survey/` — so'rovnoma javoblarini qabul qiladi (bazaga yozadi)
- `/admin/` — admin panel: har bir savol va javob bo'yicha statistika, kurs bo'yicha filtr, CSV
- `/django-admin/` — Django'ning standart admin'i (javoblarni qo'lda ko'rish/o'chirish, superuser kerak)

Admin panel paroli `.env` dagi `ADMIN_PANEL_PASSWORD` (standart: `AZIZ123`).
Google Sheets va Apps Script endi kerak emas — javoblar Django bazasida saqlanadi.

## Tuzilma

```
config/            settings.py, urls.py, wsgi.py
survey/
  questions.py     ← so'rovnoma savollari (yagona manba: sayt ham, admin ham shu yerdan oladi)
  models.py        SurveyResponse (javoblar JSON ko'rinishida)
  views.py         sayt, API, admin login/dashboard/CSV
  stats.py         statistika hisoblash
  templates/survey/index.html          sayt
  templates/survey/admin/*.html        admin panel
  static/survey/logo.png
```

## 1. Lokal ishga tushirish (Windows / Linux)

```bash
python -m venv venv
# Windows:  venv\Scripts\activate
# Linux:    source venv/bin/activate
pip install -r requirements.txt

cp .env.example .env          # Windows: copy .env.example .env
```

`.env` ni oching va lokal test uchun quyidagicha qiling:

```
DEBUG=True
SECURE_COOKIES=False
ALLOWED_HOSTS=127.0.0.1,localhost
```

```bash
python manage.py migrate
python manage.py createcachetable
python manage.py runserver
```

- Sayt: http://127.0.0.1:8000/
- Admin panel: http://127.0.0.1:8000/admin/ → parol `AZIZ123`

## 2. Serverga joylash (Ubuntu + Nginx + Gunicorn)

Misol uchun domen `jurnal.uz`, loyiha papkasi `/var/www/jurnal`.

**2.1. Kodni joylash va o'rnatish**

```bash
sudo apt update && sudo apt install -y python3-venv nginx
sudo mkdir -p /var/www/jurnal && sudo chown $USER:$USER /var/www/jurnal
# loyiha fayllarini /var/www/jurnal ga ko'chiring (scp, git clone ...)
cd /var/www/jurnal
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

**2.2. `.env` ni to'ldirish**

```bash
python -c "import secrets; print(secrets.token_urlsafe(50))"   # chiqqan qiymatni SECRET_KEY ga qo'ying
nano .env
```

```
SECRET_KEY=<yuqoridagi kalit>
DEBUG=False
ALLOWED_HOSTS=jurnal.uz,www.jurnal.uz
CSRF_TRUSTED_ORIGINS=https://jurnal.uz,https://www.jurnal.uz
ADMIN_PANEL_PASSWORD=AZIZ123
SECURE_COOKIES=True
BEHIND_HTTPS_PROXY=True
TRUST_X_FORWARDED_FOR=True
```

**2.3. Baza va static fayllar**

```bash
python manage.py migrate
python manage.py createcachetable
python manage.py collectstatic --noinput
python manage.py createsuperuser        # ixtiyoriy, /django-admin/ uchun
```

**2.4. Gunicorn xizmati** — `sudo nano /etc/systemd/system/jurnal.service`

```ini
[Unit]
Description=INSON VA FENOMEN (gunicorn)
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/jurnal
ExecStart=/var/www/jurnal/venv/bin/gunicorn config.wsgi:application --bind 127.0.0.1:8001 --workers 3
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo chown -R www-data:www-data /var/www/jurnal
sudo systemctl daemon-reload
sudo systemctl enable --now jurnal
sudo systemctl status jurnal
```

**2.5. Nginx** — `sudo nano /etc/nginx/sites-available/jurnal`

```nginx
server {
    listen 80;
    server_name jurnal.uz www.jurnal.uz;
    client_max_body_size 1m;

    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/jurnal /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d jurnal.uz -d www.jurnal.uz      # HTTPS
```

**2.6. Kod yangilanganda**

```bash
cd /var/www/jurnal && source venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py collectstatic --noinput
sudo systemctl restart jurnal
```

## 3. Agar sayt GitHub Pages'da qolsa

Django faqat API va admin panel uchun ishlatiladi:

1. `.env` ga qo'shing: `SURVEY_CORS_ORIGINS=https://na6397579-create.github.io` va `sudo systemctl restart jurnal`.
2. GitHub'dagi `index.html` da `SURVEY_ENDPOINT` ni Django manziliga o'zgartiring:
   `const SURVEY_ENDPOINT = 'https://jurnal.uz/api/survey/';`
   va `submitSurvey` ni shu loyihadagi `templates/survey/index.html` dagidek qiling (`mode: 'no-cors'` olib tashlanadi).
3. Admin panel: `https://jurnal.uz/admin/`.

## Savollarni o'zgartirish

Faqat `survey/questions.py` ni tahrirlang — sayt va admin panel avtomatik yangilanadi.
Variant matnini o'zgartirsangiz, eski javoblar statistikada yangi variantga sanalmaydi.

## Xavfsizlik

- Parol HTML kodida yo'q — server tomonida tekshiriladi.
- 5 ta noto'g'ri urinishdan keyin IP 15 daqiqaga bloklanadi (`ADMIN_MAX_FAILED_LOGINS`, `ADMIN_LOCKOUT_SECONDS`).
- Bitta IP dan soatiga 20 tadan ortiq so'rovnoma qabul qilinmaydi (`SURVEY_MAX_PER_HOUR`).
- Faqat savollardagi variantlar qabul qilinadi, matn 2000 belgigacha.
- `AZIZ123` kuchsiz parol — `.env` da kuchlirog'iga almashtirish tavsiya etiladi.
