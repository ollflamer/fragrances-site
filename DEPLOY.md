# Публикация сайта

Главное правило: **база `data.db` должна лежать на постоянном диске.** Если её положить внутрь контейнера, при каждом перезапуске или обновлении кода вы потеряете и каталог, и все заявки. За это отвечает переменная `DB_DIR`.

---

## Вариант A. VPS — рекомендую для боевого сайта

Подходит любой сервер за $4–6/мес (Hetzner, DigitalOcean, Timeweb, ahost.uz). Работает без сна, база лежит на диске, можно настроить бэкапы.

```bash
# на сервере, Ubuntu
sudo apt update && sudo apt install -y python3-pip python3-venv nginx
sudo mkdir -p /var/www/fragrances && cd /var/www/fragrances
# сюда загрузите файлы проекта (scp, git clone или через панель)

python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env && nano .env      # впишите пароль, SECRET_KEY, GOOGLE_SHEET_ID
```

Автозапуск через systemd — создайте `/etc/systemd/system/fragrances.service`:

```ini
[Unit]
Description=Fragrances & Co
After=network.target

[Service]
User=www-data
WorkingDirectory=/var/www/fragrances
EnvironmentFile=/var/www/fragrances/.env
ExecStart=/var/www/fragrances/venv/bin/gunicorn -w 2 -b 127.0.0.1:8000 app:app
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now fragrances
```

nginx — `/etc/nginx/sites-available/fragrances`:

```nginx
server {
    server_name fragrances.uz www.fragrances.uz;
    client_max_body_size 5m;
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/fragrances /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d fragrances.uz -d www.fragrances.uz   # бесплатный HTTPS
```

Ежедневный бэкап базы (`crontab -e`):

```
0 3 * * * cp /var/www/fragrances/data.db /var/backups/data-$(date +\%F).db
```

---

## Вариант B. Render — быстрее, но есть нюанс

1. Загрузите проект в репозиторий на GitHub.
2. На render.com: **New → Web Service** → выберите репозиторий. Файл `render.yaml` уже готов.
3. В разделе **Environment** задайте `ADMIN_PASSWORD` и `GOOGLE_SHEET_ID`.
4. Получите адрес вида `fragrances-co.onrender.com`, потом подключите свой домен.

**Про бесплатный тариф.** Он не подойдёт для рабочего сайта по двум причинам: <cite index="16-2">бесплатные сервисы засыпают после 15 минут без запросов, и пробуждение занимает 30–60 секунд</cite> — клиент увидит белый экран почти минуту. И главное: на бесплатном тарифе нет постоянного диска, то есть база будет стираться. Нужен тариф **Starter ($7/мес) + диск 1 ГБ** — в `render.yaml` это уже прописано.

Файл `credentials.json` от Google на Render не выкладывают в репозиторий — содержимое ключа кладут в секретный файл через настройки сервиса (Secret Files).

---

## Перед публикацией

- [ ] Сменить `ADMIN_PASSWORD` и `SECRET_KEY` в `.env`
- [ ] Проверить, что `.env`, `credentials.json` и `data.db` не попали в Git (они уже в `.gitignore`)
- [ ] Проверить `DB_DIR`, если база лежит на отдельном диске
- [ ] Настроить бэкап `data.db`
- [ ] Отправить тестовую заявку и убедиться, что строка появилась в Google Sheets
