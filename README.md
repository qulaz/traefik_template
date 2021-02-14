# Traefik Base Template

Репозиторий содержит два конфига: для `prod` среды и `dev` среды (без https, Lets Encrypt, с debug логированием и открытым дашбордом). Оформлены они в виде отдельных веток репозитория по названию среды.

Клонирование конфига для `dev` среды:

```bash
git clone --single-branch --branch dev https://github.com/qulaz/traefik_template.git
```

# Настройка

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

### Один сервис - один роут
```yaml
version: "3"

networks:
  traefik:
    external:
      name: traefik

services:
  grafana:
    container_name: grafana
    image: grafana/grafana:7.1.5
    user: "472"
    command: --config=/grafana/conf/custom.ini
    volumes:
      - ./config.ini:/grafana/conf/custom.ini
    networks:
      - traefik
    labels:
      traefik.enable: true
      traefik.http.routers.grafana.rule: Host(`localhost`) && PathPrefix(`/grafana`)
      traefik.http.services.grafana.loadbalancer.server.port: 3000
```

### Один сервис - несколько роутов с разными правилами
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
      traefik.http.routers.backend.rule: Host(`localhost`) && PathPrefix(`/`)
      traefik.http.routers.backend.service: PublicBackend
      # Порт, по которому доступно приложение
      traefik.http.services.PublicBackend.loadbalancer.server.port: 8000

      traefik.http.routers.admin.rule: Host(`localhost`) && PathPrefix(`/admin`)
      traefik.http.routers.admin.service: AdminBackend
      # Basic Auth только для роутов, начинающихся с /admin
      traefik.http.routers.admin.middlewares: BasicAuth@file
      traefik.http.services.AdminBackend.loadbalancer.server.port: 8000
```

### Динамическое имя хоста
Для динамического задания имени хоста/домена необходимо создать переменную среды с произвольным именем. Лучший для этого способ - создать рядом с `docker-compose.yml` запускаемого сервиса файл `.env` со следующим содержимым:
```
HOST=example.com
API_HOST=api.example.com
```
Затем в `docker-compose.yml` запускаемого сервиса
```yaml
version: "3"

networks:
  traefik:
    external:
      name: traefik

services:
  backend:
    build: .
    networks:
      - traefik
    labels:
      traefik.enable: true
      # Подставляем хост из созданного `.env` файла
      traefik.http.routers.grafana.rule: Host(`${API_HOST}`)
      traefik.http.services.grafana.loadbalancer.server.port: 8000

  frontend:
    build: .
    networks:
      - traefik
    labels:
      traefik.enable: true
      # Подставляем хост из созданного `.env` файла
      traefik.http.routers.grafana.rule: Host(`${HOST}`)
      traefik.http.services.grafana.loadbalancer.server.port: 3000
```

## Файловые конфиги

Добавить `yaml` файл в директорию [`configs`](./configs) по аналогии с файлом примера [`configs/NodeExporter.yaml.dist`](./configs/NodeExporter.yaml.dist)
