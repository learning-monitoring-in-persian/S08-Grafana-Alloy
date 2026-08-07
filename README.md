[English](README.md) | [فارسی](README-persian.md)

# Set up Grafana Alloy

Grafana Alloy is an open-source telemetry collector. It is designed to collect logs, metrics, and traces, and forward them to various databases (like Loki, Prometheus, or Tempo). If you have previously used **Promtail**, **Grafana Agent**, or **Telegraf**, you can think of Alloy as the modern, unified replacement for all of them!

> [!NOTE]
> We haven't talked about **Promtail**, **Grafana Agent**, and **Telegraf** in the previous repositories. We will cover **Telegraf** in another repo later :)

---

## Install Grafana Alloy as a System Service (Binary)

While running Alloy in a Docker container is highly recommended, you can also run it as a standalone binary (systemd service) directly on your host to easily read local log files.

### 1. Download and Install the Binary

First, download the pre-compiled binary for Linux:

```bash
# Important Note
# If your system architecture is not amd64 the command below will not work for you.
# For example if it is arm64, replace all `amd64` with `arm64` in the commands below:

VERSION=$(curl -s https://api.github.com/repos/grafana/alloy/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
wget -O alloy.zip https://github.com/grafana/alloy/releases/download/${VERSION}/alloy-linux-amd64.zip
unzip alloy.zip
```
*(Note: You may need to install `unzip` if you don't have it: `sudo apt install unzip`)*

Move the extracted binary to the system's bin path:
```bash
sudo mv alloy-linux-amd64 /usr/local/bin/alloy
sudo chmod +x /usr/local/bin/alloy
rm -f alloy.zip
```

For security, create a dedicated system user and directories:
```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin alloy
sudo mkdir -p /etc/alloy /var/lib/alloy
```

### 2. Create a configuration file

> [!NOTE]
> **Alloy is not just for logs!** The configuration below is merely an example of reading logs and sending them to Loki. Alloy can also scrape Prometheus metrics, collect OpenTelemetry traces, and forward them to many different databases. Grafana Alloy uses a configuration language called "River". You can find the complete list of components it supports in the official [Grafana Alloy Components Reference](https://grafana.com/docs/alloy/latest/reference/components/).

We will create a configuration that reads system logs from `/var/log` and Docker container logs, then forwards them to Loki.

Create `/etc/alloy/config.alloy`:
```alloy
// 1. Where should we send the logs? (Loki endpoint)
loki.write "local_loki" {
  endpoint {
    url = "http://localhost:3100/loki/api/v1/push"
  }
  // WAL (Write-Ahead Log) buffers logs to disk. If Loki is down or network drops, 
  // logs are saved locally and sent later, preventing data loss!
  wal {
    enabled = true
  }
}

// 2. Read Docker Container Logs
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

loki.source.docker "docker_logs" {
  host       = "unix:///var/run/docker.sock"
  targets    = discovery.docker.containers.targets
  forward_to = [loki.write.local_loki.receiver]
}

// 3. Tail a custom System Log file
// You can tail any log file created by any service on your server.
local.file_match "system_logs" {
  path_targets = [{"__path__" = "/var/log/syslog"}]
}

loki.source.file "tail_syslog" {
  targets    = local.file_match.system_logs.targets
  forward_to = [loki.process.extract_labels.receiver]
}

// 4. Extract labels directly from the log text!
// If your log contains lines like: "ERROR Connection failed", this will extract "ERROR" as a label.
// If the regex doesn't match, the log is still sent but without the extra label.
loki.process "extract_labels" {
  forward_to = [loki.write.local_loki.receiver]

  stage.regex {
    expression = "(?P<level>(ERROR|INFO|WARN|DEBUG)) (?P<message>.*)"
  }

  stage.labels {
    values = {
      level = "", // Creates a label named "level" if the regex matched it
    }
  }
}
```

> [!TIP]
> **Formatting:** Alloy has a built-in formatter just like Go! You can always run `alloy fmt -w /etc/alloy/config.alloy` to make your configuration file look neat and standard.

Set the correct permissions. **Important:** Since Alloy needs to read Docker logs and `/var/log`, we need to make sure the `alloy` user has the right permissions (adding it to the `docker` and `adm` groups).

```bash
sudo chown -R alloy:alloy /etc/alloy /var/lib/alloy
sudo usermod -aG docker alloy
sudo usermod -aG adm alloy
```

### 3. Create a systemd service

Create a file named `/etc/systemd/system/alloy.service`:
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

> [!NOTE]
> **Why `--storage.path`? (Positions)**  
> When Alloy reads files like `/var/log`, it needs to remember exactly which line it read last. It saves this "position" in the directory specified by `--storage.path`. If your server restarts, Alloy looks at this file and resumes reading exactly from where it left off, preventing duplicate logs from being sent to Loki!

Enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable alloy
sudo systemctl start alloy
```

---

## Set up Grafana Alloy with Docker Compose (Recommended)

If you prefer using Docker, deploying Alloy is very simple.

Create a `docker-compose.yml`:

```yaml
services:
  alloy:
    image: grafana/alloy:latest
    container_name: alloy
    restart: unless-stopped
    ports:
      - "12345:12345" # Alloy UI
    volumes:
      - ./config.alloy:/etc/alloy/config.alloy:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro # Needed to read Docker logs
      - /var/log:/var/log:ro # Needed to read system logs
    command: run --server.http.listen-addr=0.0.0.0:12345 /etc/alloy/config.alloy
```

Make sure your `config.alloy` file is in the same directory, then run:
```bash
docker compose up -d
```

---

## Access the Alloy UI

> [!NOTE]
> **Alloy has an amazing built-in UI!** 
> By default, Alloy exposes a web interface on port `12345`. If you open `http://{IP_ADDRESS}:12345` in your browser, you can see the **Component Graph**. This graph visually shows how your data flows from one component to another (e.g., from `discovery.docker` to `loki.source` and finally to `loki.write`). It is an incredibly powerful tool for debugging!

> [!IMPORTANT]
> To access the UI remotely, the `12345/tcp` port must be accessible. If you have an active firewall on your server, ensure you allow this port using **ufw**, **iptables**, or **nftables**.

> [!NOTE]
> **Clustering:** For large environments, Alloy can run in a [Clustered Mode](https://grafana.com/docs/alloy/latest/configure/clustering/). Multiple Alloy nodes can talk to each other and automatically shard the workload (e.g., splitting thousands of Prometheus targets among themselves). It's amazing for horizontal scaling!
