# Migrating from Laravel Echo Server to Laravel Reverb

This guide explains how to replace the legacy Node‑based `laravel-echo-server` deployment with [Laravel Reverb](https://reverb.laravel.com/).

## Prerequisites

- UNIT3D updated with Reverb installed via Composer and configured in `config/reverb.php`
- `.env` variables prepared for Reverb (use the latest `env.example` as a reference)

## Remove Laravel Echo Server

> [!NOTE]  
> Run only the commands that match your current process manager (Supervisor or systemd).

- Stop and disable the Echo process:
  ```bash
  sudo systemctl disable --now laravel-echo-server.service
  sudo supervisorctl stop laravel-echo-server
  ```
- Remove process config and artifacts:
  ```bash
  sudo rm -f /etc/systemd/system/laravel-echo-server.service
  rm -f laravel-echo-server.json laravel-echo-server.lock
  ```
- Edit `/etc/supervisor.d/unit3d.conf` and delete the `laravel-echo-server` blocks (for example, `[program:unit3d-chat-server]`), then reload Supervisor:
  ```bash
  sudo supervisorctl reread
  sudo supervisorctl update
  ```
- Drop client env values used by Echo:
  ```dotenv
  # delete
  VITE_ECHO_ADDRESS=
  ```
- If the Node server was installed locally, uninstall it:
  ```bash
  bun remove laravel-echo-server
  ```

## Add Reverb environment

Start from the `REVERB_*` and `VITE_REVERB_*` keys in `env.example` to avoid missing variables, then set values for your deployment.

> [!IMPORTANT]  
> `VITE_REVERB_*` must reflect the public hostname, port, and scheme reachable by browsers (typically your TLS vhost).  
> `REVERB_HOST` and `REVERB_PORT` should remain bound to `127.0.0.1` behind Nginx.

### Generate Reverb credentials

Run the following to generate a random key and secret:

```bash
php -r "echo bin2hex(random_bytes(16)) . PHP_EOL;"
````
then place the outputs into `REVERB_APP_KEY` and `REVERB_APP_SECRET`:


```dotenv
BROADCAST_CONNECTION=reverb

REVERB_APP_ID=100001
REVERB_APP_KEY=
REVERB_APP_SECRET=

REVERB_HOST=127.0.0.1
REVERB_PORT=1967
REVERB_SCHEME=http
REVERB_SERVER_HOST=127.0.0.1
REVERB_SERVER_PORT=1967
REVERB_APP_MAX_MESSAGE_SIZE=1000000
REVERB_MAX_REQUEST_SIZE=1000000

VITE_REVERB_APP_KEY="${REVERB_APP_KEY}"
VITE_REVERB_HOST=chat.example.test
VITE_REVERB_PORT=443
VITE_REVERB_SCHEME=https
```

After updating `.env`, run:
```bash
php artisan set:all_cache
```

## Nginx: switch from `/socket.io` to `/app/`

Remove the old Echo proxy (for deployments that previously terminated Echo with Nginx):
```nginx
# remove
location /socket.io/ {
    proxy_pass http://127.0.0.1:8443;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

Add a dedicated vhost for Reverb traffic (adjust domain and paths as needed):

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name chat.example.test;

    ssl_certificate /etc/letsencrypt/live/chat.example.test/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chat.example.test/privkey.pem;

    location /app/ {
        proxy_pass http://127.0.0.1:1967/app/;
        proxy_http_version 1.1;
        proxy_read_timeout 600s;
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_set_header Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location = /up {
        proxy_pass http://127.0.0.1:1967/up;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Test and reload Nginx:
```bash
sudo nginx -t && sudo systemctl reload nginx
```

## TLS certificates

Issue a certificate for the Reverb subdomain using Certbot:
```bash
sudo certbot --nginx -d chat.example.test
```

For a wildcard certificate (manual DNS challenge):
```bash
sudo certbot certonly --manual --preferred-challenges dns -d '*.example.test' -d example.test
```

## Supervisord service (default)

Edit `/etc/supervisor.d/unit3d.conf`:

```ini
[program:reverb]
directory = /var/www/html
command = /usr/bin/php artisan reverb:start --no-interaction
autostart = true
autorestart = true
user = www-data
stdout_logfile = /var/www/html/storage/logs/reverb.log
redirect_stderr = true
stopsignal = INT
```

Apply and start:

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart reverb
```

## Systemd service (advanced)

If you prefer systemd, create a dedicated unit so Reverb survives deploys and reboots.

Create `/etc/systemd/system/reverb.service` (adjust user and paths):

```ini
[Unit]
Description = Laravel Reverb
After = network.target
Wants = network-online.target

[Service]
Type = simple
User = www-data
Group = www-data
WorkingDirectory = /var/www/html
EnvironmentFile = /var/www/html/.env
ExecStart = /usr/bin/php artisan reverb:start --no-interaction
ExecReload = /usr/bin/php artisan reverb:restart
KillSignal = SIGINT
TimeoutStopSec = 30s
Restart = always
RestartSec = 5s
LimitNOFILE = 65536

[Install]
WantedBy = multi-user.target
```

Enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now reverb.service
```