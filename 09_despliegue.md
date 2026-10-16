# 09 — Despliegue en VPS, respaldos y migraciones

Guía para instalar SCI en un servidor propio (VPS) y mantenerlo. Supone **Ubuntu 24.04**, un dominio que apunta al servidor (en los ejemplos, `sci.ejemplo.bo`) y acceso por SSH con `sudo`.

## 1. Arquitectura

```
Internet ──HTTPS──▶ Caddy (443) ──┬─▶ /api/*, /socket.io/*  ─▶ backend Node (127.0.0.1:3000)
                                  └─▶ todo lo demás          ─▶ front compilado (/var/www/sci)
                                                    backend ─▶ PostgreSQL 16 (127.0.0.1:5432)
                                                    backend ─▶ archivos (/var/lib/sci/data)
```

- **Un solo dominio** para el front y la API: no hay CORS cruzado y el WebSocket va por el mismo HTTPS.
- **Caddy** obtiene y renueva solo el certificado HTTPS (Let's Encrypt) y deja pasar el WebSocket sin configuración extra.
- **PostgreSQL y Node no se exponen** a Internet: solo escuchan en `127.0.0.1`.

## 2. Preparar el servidor

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl ufw postgresql-16 debian-keyring debian-archive-keyring apt-transport-https

# Node.js 24 (LTS). Con 24.15 o superior se podrá pasar el front a Angular 22.
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs

# Caddy (repositorio oficial)
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install -y caddy

# Firewall: solo SSH y web
sudo ufw allow OpenSSH && sudo ufw allow 80 && sudo ufw allow 443 && sudo ufw enable

# Usuario de servicio y carpetas
sudo useradd --system --create-home --home-dir /opt/sci --shell /usr/sbin/nologin sci
sudo mkdir -p /var/lib/sci/data /var/backups/sci /var/www/sci
sudo chown -R sci:sci /var/lib/sci /var/backups/sci
sudo chmod 700 /var/backups/sci
```

**Base de datos** (contraseña larga y aleatoria; guárdala solo en el `.env`):

```bash
sudo -u postgres psql -c "CREATE USER sci WITH PASSWORD 'CAMBIAR_POR_UNA_CLAVE_LARGA';"
sudo -u postgres psql -c "CREATE DATABASE sci OWNER sci;"
sudo -u postgres psql -d sci -c 'CREATE EXTENSION IF NOT EXISTS "uuid-ossp";'
```

## 3. Backend

```bash
sudo -u sci git clone https://github.com/DAVV422/back-sci.git /opt/sci/back-sci
cd /opt/sci/back-sci
sudo -u sci cp .env.example .env && sudo chmod 600 .env && sudo -u sci nano .env
```

Valores de producción del `.env` (el resto, como en `.env.example`):

| Variable | Valor |
|:---|:---|
| `APP_PROD` | `true` |
| `PORT` | `3000` |
| `APP_URL`, `FRONTEND_URL`, `CORS_ORIGINS` | `https://sci.ejemplo.bo` |
| `DB_HOST` / `DB_PORT` / `DB_DATABASE` / `DB_USERNAME` / `DB_PASSWORD` | `127.0.0.1` / `5432` / `sci` / `sci` / la clave del paso anterior |
| `DB_SSL` | `false` (Postgres en el mismo servidor) |
| `JWT_AUTH`, `JWT_REFRESH_SECRET` | dos valores distintos de `openssl rand -base64 48` |
| `JWT_EXPIRATION` | `1h` (la web renueva sola) |
| `ADMIN_EMAIL`, `ADMIN_PASSWORD` | el primer superadministrador; la clave, de 12 caracteres o más |
| `TRUST_PROXY` | `1` (Caddy delante) |
| `STORAGE_LOCAL_ROOT` | `/var/lib/sci/data` |
| `APP_TIMEZONE` | `America/La_Paz` |
| `SMTP_*` | el servidor de correo de la institución (sin esto, los correos se simulan) |
| `SWAGGER_ENABLED` | `false` |

> No pongas `TZ` ni `DB_SYNCHRONIZE`. El proceso trabaja en UTC por diseño (la zona de la institución es `APP_TIMEZONE`) y el esquema lo crean las migraciones.

Con `APP_PROD=true` la API **se niega a arrancar** si `JWT_AUTH` tiene menos de 32 caracteres o es el de ejemplo, si `ADMIN_PASSWORD` es débil o si CORS queda abierto. El mensaje de error dice qué corregir.

```bash
sudo -u sci npm ci
sudo -u sci npm run build
```

Servicio `systemd` en `/etc/systemd/system/sci-backend.service`:

```ini
[Unit]
Description=SCI backend (NestJS)
After=network.target postgresql.service

[Service]
User=sci
WorkingDirectory=/opt/sci/back-sci
ExecStart=/usr/bin/node dist/main
Restart=on-failure
RestartSec=5
Environment=NODE_ENV=production
NoNewPrivileges=true
ProtectSystem=full
ReadWritePaths=/var/lib/sci/data

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now sci-backend
journalctl -u sci-backend -f      # al primer arranque: migraciones, catálogo de cargos y superadministrador
curl -s http://127.0.0.1:3000/api/config/public   # {"timezone":"America/La_Paz",...}
```

## 4. Front

El front se compila con la URL de la API y la clave de Google Maps. Como se sirve en el mismo dominio, las URLs son relativas. En `src/environments/environment.ts` **del servidor** (no se sube al repositorio):

```ts
export const environment = {
  production: true,
  apiUrl: '/api',
  socketUrl: '/notifications',
  googleMapsApiKey: 'CLAVE_RESTRINGIDA_AL_DOMINIO',
  userPasswordRequired: true,
};
```

La clave de Google Maps siempre queda visible en el navegador: debe estar **restringida por dominio** (referentes HTTP `https://sci.ejemplo.bo/*`) y solo a la *Maps JavaScript API*, desde la consola de Google Cloud.

```bash
sudo -u sci git clone https://github.com/DAVV422/front-sci-emergyai.git /opt/sci/front-sci
cd /opt/sci/front-sci
sudo -u sci nano src/environments/environment.ts
sudo -u sci corepack enable && sudo -u sci npx -y pnpm@10 install --frozen-lockfile
sudo -u sci npx -y pnpm@10 build
sudo rsync -a --delete dist/front-sci/browser/ /var/www/sci/
```

## 5. Caddy (HTTPS)

`/etc/caddy/Caddyfile`:

```caddy
sci.ejemplo.bo {
	encode zstd gzip

	# API y tiempo real (Caddy deja pasar el WebSocket sin configuración extra)
	handle /api/* {
		request_body {
			max_size 30MB   # notas de voz hasta 25 MB
		}
		reverse_proxy 127.0.0.1:3000
	}
	handle /socket.io/* {
		reverse_proxy 127.0.0.1:3000
	}

	# Front (aplicación de una sola página)
	handle {
		root * /var/www/sci
		try_files {path} /index.html
		file_server
		header /index.html Cache-Control "no-cache"
	}

	header {
		Strict-Transport-Security "max-age=31536000; includeSubDomains"
		X-Content-Type-Options "nosniff"
		Referrer-Policy "strict-origin-when-cross-origin"
		Permissions-Policy "camera=(), geolocation=(self), microphone=(self)"
	}
}
```

```bash
sudo systemctl reload caddy
```

## 6. Respaldos

`back-sci/scripts/backup.sh` respalda la base (`pg_dump`, formato custom) y los archivos (fotos, croquis y notas de voz). Deja por corrida `db-<fecha>.dump`, `files-<fecha>.tar.gz` y `SHA256SUMS-<fecha>`, y borra lo que tenga más de `BACKUP_RETENTION_DAYS` días (14 por defecto).

Cron diario del usuario `sci` (`sudo -u sci crontab -e`):

```cron
15 3 * * *  cd /opt/sci/back-sci && BACKUP_DIR=/var/backups/sci scripts/backup.sh .env >> /var/backups/sci/backup.log 2>&1
```

- **Copia fuera del servidor.** Un respaldo en el mismo VPS no sirve si se pierde el VPS. Copia `/var/backups/sci` a otro lugar (otro servidor con `rsync` por SSH, o un bucket con `rclone`), al menos una vez al día.
- **Prueba de restauración, una vez al mes.** Un respaldo que nunca se restauró no está probado:

  ```bash
  cd /opt/sci/back-sci
  sudo -u postgres createdb -O sci sci_restore_test      # el usuario sci no crea bases
  sudo -u sci scripts/restore.sh .env /var/backups/sci/db-AAAAMMDDTHHMMSSZ.dump --target sci_restore_test --yes
  sudo -u postgres psql -d sci_restore_test -c 'SELECT count(*) FROM emergency;'
  sudo -u postgres psql -c 'DROP DATABASE sci_restore_test;'
  ```

- **Restaurar de verdad** (pérdida de datos): `sudo systemctl stop sci-backend`, luego `sudo -u sci scripts/restore.sh .env <db.dump> <files.tar.gz> --target sci --yes`, y `sudo systemctl start sci-backend`.
- **Versión de las herramientas:** `pg_dump` y `pg_restore` deben ser de la misma versión mayor que el servidor. Un `pg_restore` 17 contra un servidor 16 falla, y el script lo avisa antes de tocar nada. En Windows se elige con `PG_BIN`.

## 7. Actualizar a una versión nueva

```bash
cd /opt/sci/back-sci
sudo -u sci env BACKUP_DIR=/var/backups/sci scripts/backup.sh .env   # 1. respaldo previo, siempre
sudo -u sci git pull && sudo -u sci npm ci && sudo -u sci npm run build
sudo systemctl restart sci-backend                                  # 2. aplica las migraciones al arrancar
journalctl -u sci-backend -n 50                                     # 3. revisar "has been executed successfully"

cd /opt/sci/front-sci
sudo -u sci git pull && sudo -u sci npx -y pnpm@10 install --frozen-lockfile && sudo -u sci npx -y pnpm@10 build
sudo rsync -a --delete dist/front-sci/browser/ /var/www/sci/
```

**Volver atrás:** `git checkout <versión anterior>`, compilar y restaurar el respaldo del paso 1. `npm run migration:revert` deshace la última migración, pero solo si su `down` no pierde datos; restaurar el respaldo es lo seguro.

## 8. Migraciones del esquema

El esquema de la base se versiona en `back-sci/src/migrations` y **las migraciones corren solas al arrancar** el backend (`DB_MIGRATIONS_RUN`, activo por defecto). `synchronize` quedó apagado: alteraba las tablas según las entidades y podía borrar columnas con datos.

Al cambiar una entidad, en desarrollo:

```bash
npm run migration:generate -- src/migrations/NombreDelCambio   # compara entidades vs. tu base local
# revisar el archivo generado ANTES de commitearlo
npm run migration:run
```

- **Revisa siempre lo generado.** Ante un cambio de tipo de columna, TypeORM genera `DROP COLUMN` + `ADD COLUMN`, que borra los datos: reemplázalo por `ALTER COLUMN … TYPE … USING …`, como en `TimestampsWithTimeZone`.
- `npm run migration:show` lista las aplicadas y `npm run migration:revert` deshace la última.
- **Bases creadas antes de las migraciones** (con `synchronize`): la migración inicial detecta que las tablas ya existen y solo se registra. Si `npm run migration:generate -- src/migrations/X --dryrun` muestra diferencias, arranca **una vez** con `DB_SYNCHRONIZE=true` (solo fuera de producción) para alinearla y luego quítalo.

**Fechas:** todos los instantes son `timestamptz`. Emergencias, asignaciones, acciones y registros de triage guardan además `occurred_at`, que un trigger calcula a partir de `date` + `hour` (en UTC). Es la columna para filtrar y ordenar por rango en los reportes.

## 9. Monitoreo mínimo

- **Estado:** `curl -fsS https://sci.ejemplo.bo/api/config/public` (sin sesión). Úsalo en un monitor externo, por ejemplo UptimeRobot, cada 5 minutos.
- **Registros:** `journalctl -u sci-backend` (backend) y `journalctl -u caddy` (HTTPS y accesos).
- **Disco:** las notas de voz crecen con el uso. Aviso diario si el disco supera el 80 %:

  ```cron
  0 8 * * *  df -P /var/lib/sci | awk 'NR==2 && $5+0 > 80 {print "SCI: disco al " $5}' | logger -t sci-disk
  ```

## 10. Lista final antes de abrir el sistema

- [ ] HTTPS activo y `http://` redirige a `https://` (Caddy lo hace solo).
- [ ] `.env` con `APP_PROD=true`, secretos propios y permisos `600`.
- [ ] Primer ingreso con `ADMIN_EMAIL` y cambio de contraseña desde el perfil.
- [ ] Clave de Google Maps nueva, restringida al dominio. La anterior está en el historial público del repositorio: revócala.
- [ ] Respaldo diario funcionando, copia fuera del servidor y una restauración de prueba hecha.
- [ ] Monitor de estado y aviso de disco configurados.
- [ ] Puertos 3000 y 5432 cerrados desde fuera: `nc -zv sci.ejemplo.bo 3000` debe fallar.
