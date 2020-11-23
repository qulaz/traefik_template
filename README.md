# Traefik Base Template

Репозиторий содержит два конфига: для `prod` среды и `dev` среды (без https, Lets Encrypt, с debug логированием и открытым дашбордом). Оформлены они в виде отдельных веток репозитория по названию среды.

Клонирование конфига для `prod` среды:

```bash
git clone --single-branch --branch prod https://github.com/qulaz/traefik_template.git
```

# Что есть

- https с Lets Encrypt + автопродление сертификатов
- Редирект с http на https
- Middleware для Basic Auth
- Docker и File providers

# Настройка

- Перименовать `.env.dist` в `.env` и наполнить данными

```bash
mv .env.dist .env
```

- Для использования `Basic Auth` необходимо переименовать `.htpasswd.dist` в `.htpasswd` и наполнить данными

```bash
mv .htpasswd.dist .htpasswd
sudo apt install apache2-utils
htpasswd -nb user password >> .htpasswd
```

или для генерации паролей можно использовать `openssl`:

```bash
openssl passwd -apr1
```

- Переименовать `acme.json.dict` в `acme.json` и выставить ему разрешения 600

```bash
mv acme.json.dict acme.json
chmod 600 acme.json
```

- Переименовать `configs/middlewares.yaml.dist` в `configs/middlewares.yaml`

```bash
mv configs/middlewares.yaml.dist configs/middlewares.yaml
```

- Запустить `traefik`:

```bash
docker-compose up -d
```

# Использование

## В Docker Compose

```yaml
version: "3.7"

networks:
  internal:
    driver: bridge
  traefik:
    external:
      name: traefik

services:
  backend:
    build: .
    networks:
      - traefik
      - internal
    labels:
      traefik.enable: true
      # Правило хоста применяется по-умолчнаию
      traefik.http.routers.backend.rule: PathPrefix(`/`)
      traefik.http.routers.backend.service: PublicBackend
      # Порт, по которому доступно приложение
      traefik.http.services.PublicBackend.loadbalancer.server.port: 8000

      traefik.http.routers.admin.rule: PathPrefix(`/admin`)
      traefik.http.routers.admin.service: AdminBackend
      # Basic Auth только для роутов, начинающихся с /admin
      traefik.http.routers.admin.middlewares: BasicAuth@file
      traefik.http.services.AdminBackend.loadbalancer.server.port: 8000
```

## Файловые конфиги

Добавить файл в директорию [`configs`](./configs) по аналогии с файлом примера [`configs/NodeExporter.yaml.dist`](./configs/NodeExporter.yaml.dist)
