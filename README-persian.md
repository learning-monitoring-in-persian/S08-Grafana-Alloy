[فارسی](README-persian.md) | [English](README.md)

# راه‌اندازی Grafana Alloy

گرافانا اَلوی (Grafana Alloy) یک سیستم جمع‌آوری داده‌های تله‌متری (تلِمتری کالکتر) متن‌باز است. این ابزار طراحی شده تا همزمان لاگ‌ها، متریک‌ها و تریس‌ها رو جمع‌آوری کنه و به دیتابیس‌های مختلف (مثل Loki، Prometheus یا Tempo) ارسال کنه. اگر قبلاً از ابزارهایی مثل **Promtail**، **Grafana Agent** یا **Telegraf** استفاده می‌کردید، حالا می‌تونید Alloy رو به عنوان یک جایگزین مدرن و یکپارچه برای تمام اون‌ها در نظر بگیرید!

> ### نکته
> ما توی ریپازیتوری‌های قبلی در مورد **Promtail**، **Grafana Agent** و **Telegraf** صحبتی نکردیم. در آینده در مورد ابزار **Telegraf** توی یک ریپازیتوری جداگانه حرف می‌زنیم :)

---

## نصب Alloy به‌صورت سرویس systemd (باینری)

با وجود اینکه پیشنهاد میشه Alloy رو از طریق کانتینر داکر اجرا کنید، اما در صورت نیاز می‌تونید اون رو به صورت مستقل (سرویس systemd) هم مستقیماً روی سرور راه‌اندازی کنید تا راحت‌تر بتونید لاگ فایل‌های لوکال رو بخونید.

### ۱. دانلود و نصب فایل باینری

برای دانلود و استخراج نسخه‌ی باینری Alloy مخصوص لینوکس، دستورات زیر را اجرا کنید:

```bash
# نکته‌ مهم
# اگر معماری سیستمی که Alloy قرار است روی آن نصب شود amd64 نیست دستورات زیر به درستی برای شما کار نخواهند کرد.
# برای مثال اگر معماری شما arm64 است باید تمامی  `amd64` ها را با `arm64` در کامند‌های زیر جایگزین کنید:

VERSION=$(curl -s https://api.github.com/repos/grafana/alloy/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
wget -O alloy.zip https://github.com/grafana/alloy/releases/download/${VERSION}/alloy-linux-amd64.zip
unzip alloy.zip
```
*(نکته: اگر `unzip` را ندارید، با `sudo apt install unzip` آن را نصب کنید)*

فایل استخراج شده را به مسیر اجرایی سیستم منتقل کنید:
```bash
sudo mv alloy-linux-amd64 /usr/local/bin/alloy
sudo chmod +x /usr/local/bin/alloy
rm -f alloy.zip
```

برای امنیت بیشتر، یک یوزر سیستمی اختصاصی و دایرکتوری‌های لازم رو بسازید:
```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin alloy
sudo mkdir -p /etc/alloy /var/lib/alloy
```

### ۲. ساخت فایل کانفیگ

> ### نکته
> **اَلوی فقط برای لاگ نیست!** کانفیگی که در ادامه می‌نویسیم صرفاً یک مثال برای خوندن لاگ و ارسالش به لوکی هست. ابزار Alloy می‌تونه متریک‌های پرومتئوس رو هم اسکریپ (Scrape) کنه، تریس‌های OpenTelemetry رو جمع‌آوری کنه و به مقصدهای بسیار متنوعی بفرسته. Alloy از زبان کانفیگ مخصوصی به نام "River" استفاده می‌کنه. برای دیدن لیست کامل کامپوننت‌هایی که پشتیبانی می‌کنه به [مستندات کامپوننت‌های Alloy](https://grafana.com/docs/alloy/latest/reference/components/) سر بزنید.

ما قصد داریم یک کانفیگ بنویسیم که همزمان لاگ‌های داکر و لاگ‌های سیستمی (مسیر `/var/log`) رو بخونه و به Loki بفرسته.

فایل `/etc/alloy/config.alloy` را با کانفیگ زیر ایجاد کنید:
```alloy
// ۱. کجا باید لاگ‌ها رو بفرستیم؟ (آدرس لوکی)
loki.write "local_loki" {
  endpoint {
    url = "http://localhost:3100/loki/api/v1/push"
  }
  // قابلیت WAL لاگ‌ها رو روی دیسک بافر می‌کنه. اگه ارتباط با لوکی قطع بشه،
  // لاگ‌ها لوکال ذخیره میشن و بعداً ارسال میشن تا دیتایی از دست نره!
  wal {
    enabled = true
  }
}

// ۲. خوندن لاگ کانتینرهای داکر
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

loki.source.docker "docker_logs" {
  host       = "unix:///var/run/docker.sock"
  targets    = discovery.docker.containers.targets
  forward_to = [loki.write.local_loki.receiver]
}

// ۳. خوندن (Tail کردن) لاگ فایل‌های سیستمی
// شما می‌تونید مسیر هر لاگ فایلی که توسط هر سرویسی تولید میشه رو اینجا بدید تا تیل بشه
local.file_match "system_logs" {
  path_targets = [{"__path__" = "/var/log/syslog"}]
}

loki.source.file "tail_syslog" {
  targets    = local.file_match.system_logs.targets
  forward_to = [loki.process.extract_labels.receiver]
}

// ۴. استخراج لیبل مستقیما از متن لاگ!
// اگر لاگ شما شامل کلماتی مثل "ERROR" یا "INFO" باشه، این بخش اون کلمه رو به عنوان یک لیبل در نظر می‌گیره.
// اگر هم متن لاگ با فرمت رجکس نخونه، لاگ بدون مشکل ارسال میشه اما لیبل اضافه نمی‌خوره.
loki.process "extract_labels" {
  forward_to = [loki.write.local_loki.receiver]

  stage.regex {
    expression = "(?P<level>(ERROR|INFO|WARN|DEBUG)) (?P<message>.*)"
  }

  stage.labels {
    values = {
      level = "", // در صورت مچ شدن رجکس، لیبل level به لاگ اضافه میشه
    }
  }
}
```

> ### نکته (مرتب‌سازی کانفیگ)
> زبان River یک ابزار داخلی برای فرمت کردن کدهاتون داره (درست مثل `go fmt`). هر موقع خواستید می‌تونید دستور `alloy fmt -w /etc/alloy/config.alloy` رو اجرا کنید تا فایل کانفیگتون استاندارد، مرتب و خوشگل بشه!

حالا باید دسترسی‌ها را تنظیم کنید. **خیلی مهم:** از اونجایی که Alloy باید لاگ‌های داکر و فایل‌های `/var/log` رو بخونه، یوزر `alloy` باید دسترسی‌های لازم رو داشته باشه (اضافه شدن به گروه‌های `docker` و `adm`).

```bash
sudo chown -R alloy:alloy /etc/alloy /var/lib/alloy
sudo usermod -aG docker alloy
sudo usermod -aG adm alloy
```

### ۳. ساخت فایل سرویس systemd

فایل `/etc/systemd/system/alloy.service` را با محتوای زیر بسازید:

```ini
[Unit]
Description=Grafana Alloy
Wants=network-online.target
After=network-online.target

[Service]
User=alloy
Group=alloy
Type=simple
ExecStart=/usr/local/bin/alloy run /etc/alloy/config.alloy \
  --storage.path=/var/lib/alloy

Restart=always

[Install]
WantedBy=multi-user.target
```

> ### نکته (Positions)
> **چرا نوشتیم `--storage.path`؟**  
> وقتی اَلوی فایل‌های سیستم رو می‌خونه، باید یادش بمونه که تا کدوم خط از فایل رو خونده (به این میگن Position). این اطلاعات رو در مسیر `--storage.path` ذخیره می‌کنه. اینطوری اگه سرور ری‌استارت بشه، اَلوی دقیقاً از همون خطی که مونده بود ادامه میده و لاگ تکراری برای لوکی نمی‌فرسته!

در نهایت سرویس را فعال و اجرا کنید:
```bash
sudo systemctl daemon-reload
sudo systemctl enable alloy
sudo systemctl start alloy
```

---

## راه‌اندازی Alloy با Docker Compose (پیشنهادی)

اگر ترجیح می‌دهید از Docker استفاده کنید، راه‌اندازی Alloy بسیار ساده است.

یک فایل `docker-compose.yml` به شکل زیر بسازید:

```yaml
services:
  alloy:
    image: grafana/alloy:latest
    container_name: alloy
    restart: unless-stopped
    ports:
      - "12345:12345" # پنل وب اَلوی
    volumes:
      - ./config.alloy:/etc/alloy/config.alloy:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro # برای خوندن لاگ کانتینرها
      - /var/log:/var/log:ro # برای خوندن لاگ‌های سیستمی
    command: run --server.http.listen-addr=0.0.0.0:12345 /etc/alloy/config.alloy
```

مطمئن شوید فایل `config.alloy` شما کنار داکر کمپوز قرار داره و کانتینر را اجرا کنید:
```bash
docker compose up -d
```

---

## دسترسی به پنل وب Alloy

> ### نکته
> **اَلوی یک پنل وب داخلی فوق‌العاده داره!**
> به صورت پیش‌فرض اَلوی پورت `12345` رو باز می‌کنه. اگه آدرس `http://{IP_ADDRESS}:12345` رو توی مرورگر باز کنید، می‌تونید گراف کامپوننت‌ها (Component Graph) رو ببینید. این گراف به صورت بصری نشون میده که دیتای شما دقیقاً از کدوم کامپوننت به کدوم کامپوننت در حال حرکته (مثلاً از `discovery.docker` رفته به `loki.source` و بعد به `loki.write`). این ویژگی برای دیباگ کردن بی‌نظیره!

> ### نکته مهم
>
> برای اینکه بتونید از بیرون سرور به این پنل وب دسترسی داشته باشید، پورت `12345/tcp` باید باز باشه. اگر فایروال فعال دارید، یادتون نره که این پورت رو با **ufw** یا **iptables** یا **nftables** باز کنید.

> ### نکته (Clustering)
> برای محیط‌های خیلی بزرگ، اَلوی می‌تونه به صورت [کلاستر (Clustered Mode)](https://grafana.com/docs/alloy/latest/configure/clustering/) اجرا بشه. توی این حالت چندین سرور اَلوی با هم صحبت می‌کنن و بار پردازشی (مثلاً اسکریپ کردن هزاران تارگت پرومتئوس) رو بین خودشون تقسیم (Shard) می‌کنن که برای مقیاس‌پذیری عالیه!
