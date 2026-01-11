# Deploying Laravel Reverb

[Laravel Reverb](https://reverb.laravel.com/) provides the WebSocket backend required for real‑time chat functionality. This guide uses **Supervisord** by default (the standard Laravel approach), with an optional **systemd** setup for advanced operators who prefer it.

> [!TIP]
> If you are migrating from the legacy Node‑based Echo server, see  
> [Migrating from Laravel Echo Server to Laravel Reverb](migrating_to_reverb.md).

---

## Environment variables

Configure Reverb as the broadcasting driver and define both the **server‑side** and **frontend** connection settings.

Start with the `REVERB_*` and `VITE_REVERB_*` entries in `env.example` to ensure no required keys are missing. Use the **public** host, port, and scheme that browsers connect to, while keeping the Reverb process itself bound to `localhost` on an internal port.

> [!IMPORTANT]  
> `VITE_REVERB_*` must reflect the **public** hostname, port, and scheme reachable by browsers (typically your TLS vhost).  
> `REVERB_HOST` and `REVERB_PORT` should remain bound to `127.0.0.1` behind Nginx.

```dotenv
BROADCAST_CONNECTION=reverb

# Reverb application credentials
REVERB_APP_ID=100001
REVERB_APP_KEY=example_reverb_key
REVERB_APP_SECRET=example_reverb_secret

# Reverb server process (binds locally)
REVERB_HOST=127.0.0.1
REVERB_PORT=1967
REVERB_SCHEME=http
REVERB_SERVER_HOST=127.0.0.1
REVERB_SERVER_PORT=1967
REVERB_APP_MAX_MESSAGE_SIZE=1000000
REVERB_MAX_REQUEST_SIZE=1000000

# Frontend connection settings (used by browsers)
VITE_REVERB_APP_KEY="${REVERB_APP_KEY}"
VITE_REVERB_HOST=chat.example.test
VITE_REVERB_PORT=443
VITE_REVERB_SCHEME=https
```

---

## Supervisord service (default)

For most Laravel deployments on Ubuntu, run Reverb under **Supervisord**. This aligns with existing UNIT3D server management conventions.

Create or update `/etc/supervisor.d/unit3d.conf`:

```ini
[program:reverb]
directory=/var/www/html
command=/usr/bin/php artisan reverb:start --no-interaction
autostart=true
autorestart=true
user=www-data
stdout_logfile=/var/www/html/storage/logs/reverb.log
redirect_stderr=true
stopsignal=INT
```

Apply the configuration and start the service:

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart reverb
```

---

## Systemd service (advanced)

If you prefer **systemd**, create a dedicated unit so Reverb survives deploys and reboots.

```ini
[Unit]
Description=Laravel Reverb
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/var/www/html
EnvironmentFile=/var/www/html/.env
ExecStart=/usr/bin/php artisan reverb:start --no-interaction
ExecReload=/usr/bin/php artisan reverb:restart
KillSignal=SIGINT
TimeoutStopSec=30s
Restart=always
RestartSec=5s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

Reload systemd and enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now reverb.service
```

---

## Nginx vhost

Proxy WebSocket traffic (`/app/`) and the health check endpoint (`/up`) to the local Reverb process. Adjust `server_name` and certificate paths for your environment.

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

---

## TLS certificates

Issue a certificate for the Reverb subdomain using Certbot:

```bash
sudo certbot --nginx -d chat.example.test
```

For a wildcard certificate (manual DNS challenge):

```bash
sudo certbot certonly --manual --preferred-challenges dns -d '*.example.test' -d example.test
```

Update `server_name` and the SSL certificate paths in the Nginx vhost as needed, then reload Nginx:

```bash
sudo nginx -t && sudo systemctl reload nginx
```