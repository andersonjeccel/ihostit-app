# ihostit.app

Discover and explore self-hosted software curated from the awesome-selfhosted list. The backend syncs and stores data in SQLite; the React frontend consumes a simple HTTP API.

Highlights:
- Server-side SQLite (better-sqlite3): fast, reliable, single-file DB.
- Scheduled sync on the server via cron to keep data fresh.
- Client fetches pre-parsed categories and apps from `/api` endpoints.

## Quick Start (Dev)

- Requirements: Node.js 18+

Commands:
- `npm install`
- `npm run build:server` — compile the Node API
- `npm run start:server` — start the Node API (defaults to http://localhost:8787)
- `npm run dev` — start Vite dev server (proxies `/api` to 8787)
- `npm run build` — production build of the client
- `npm run preview` — preview built assets
- [Mautic](https://www.mautic.org) - Open-source, self-hostable marketing automation platform for campaigns, segmentation, and customer journeys.

## Data Flow

- Server:
  - `server/index.ts` exposes `/api` routes.
  - `server/sync.ts` runs the sync, writing to SQLite via `src/db/database.ts`.
  - `src/db/database.ts` is the Node-only DB service (better-sqlite3).
  - A cron job runs daily at 02:00 UTC to refresh data.
- Client:
  - `src/services/github.ts` contains the parser used by the server.
  - `src/hooks/useAwesomeSelfHosted.ts` fetches `{ categories, apps }` from `/api/data`.
  - UI components render cards and links.

Note: A browser-only localStorage DB was replaced in favor of server-side SQLite.

## Production Deployment (Nginx + systemd + TLS)

These are the exact steps used to deploy ihostit.app on Ubuntu with Nginx, a systemd service for the API, and Let’s Encrypt TLS.

Requirements:
- Ubuntu with `nginx` and `certbot` installed
- Node.js 18+ and npm

1) Clone and build
- `cd /var/www`
- `git clone https://github.com/Gaion-chat/ihostit-app.git`
- `cd ihostit-app`
- `npm ci`
- `npm run build:server` (compiles API to `server-dist/`)
- `npm run build` (builds client to `dist/`)

Note: `package.json` includes `postbuild:server` to copy `src/db/schema.sql` into `server-dist` ensuring the API can initialize the SQLite schema.

2) Create systemd service for the API
Create `/etc/systemd/system/ihostit-api.service`:

```
[Unit]
Description=ihostit.app API service
After=network.target

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/var/www/ihostit-app
Environment=NODE_ENV=production
Environment=PORT=8787
ExecStart=/usr/bin/node /var/www/ihostit-app/server-dist/server/index.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Then:
- `sudo chown -R www-data:www-data /var/www/ihostit-app`
- `sudo systemctl daemon-reload`
- `sudo systemctl enable --now ihostit-api.service`
- Verify: `curl http://127.0.0.1:8787/api/status`

3) Nginx configuration (serve client and proxy API)
Create `/etc/nginx/sites-available/ihostit.app`:

```
server {
    listen 80;
    listen [::]:80;
    server_name ihostit.app;

    root /var/www/ihostit-app/dist;
    index index.html;

    location /assets/ {
        try_files $uri =404;
        access_log off;
        expires 30d;
        add_header Cache-Control "public";
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8787;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_read_timeout 60s;
    }

    location / {
        try_files $uri /index.html;
    }
}
```

Enable and reload:
- `sudo ln -s /etc/nginx/sites-available/ihostit.app /etc/nginx/sites-enabled/ihostit.app`
- `sudo nginx -t && sudo systemctl reload nginx`

4) HTTPS with Let’s Encrypt
- `sudo certbot --nginx -d ihostit.app --agree-tos --register-unsafely-without-email --redirect`
- Optional: include `-d www.ihostit.app` if you add the www host in DNS and Nginx.

5) Initialize data (one-time)
- `curl -X POST http://127.0.0.1:8787/api/sync`

6) Operations
- Health: `curl https://ihostit.app/api/status`
- Logs: `sudo journalctl -u ihostit-api.service -f`
- Manual sync: `curl -X POST http://127.0.0.1:8787/api/sync`

7) Updating deployment
- `cd /var/www/ihostit-app && git pull`
- `npm ci`
- `npm run build:server && npm run build`
- `sudo systemctl restart ihostit-api.service && sudo systemctl reload nginx`

## Development Notes

- This project uses path alias `@` mapped to `src/` (see `vite.config.ts`).
- The UI guards demo/website/source links to avoid confusing duplicate repo links and only shows a Play button for real demos.

## License

This project is licensed under the Creative Commons Attribution-ShareAlike 3.0 Unported license (CC BY-SA 3.0).

- You must provide attribution and indicate changes.
- Derivatives must be licensed under the same terms.

See `LICENSE` or https://creativecommons.org/licenses/by-sa/3.0/ for details.

Attribution: Portions of the data are derived from the awesome-selfhosted project.
