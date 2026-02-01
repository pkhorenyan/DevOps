
### 1. Bridge (default) - Без DNS имён

Самый базовый вариант, контейнеры общаются по IP из docker network:
```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"  # Хост:8080 → контейнер:80
    volumes:
      - ./html:/usr/share/nginx/html
    depends_on:
      - db
    # networks не указано - использует default bridge
  
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

В пользовательских сетях Docker (а любая сеть, созданная через Compose, считается пользовательской, даже если она «дефолтная») **всегда работает встроенный DNS-резолвер**.

**Когда DNS НЕ работает?**

DNS-имена не работают только в одном случае: если вы запускаете контейнеры вручную через `docker run` без создания своей сети. В таком случае они попадают в стандартную сеть с названием `bridge` (по умолчанию), где поиск по именам отключен и разрешен только по IP.

### 2. Bridge с DNS именами (рекомендуемый)

Docker Compose автоматически создает DNS для имен сервисов:

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - db
    networks:
      - app_network
    # Может обращаться к БД по имени "db"
    # В конфиге: proxy_pass http://api:3000;
  
  api:
    build: ./api  # Ваше приложение на Node/Python/etc
    environment:
      DATABASE_URL: "postgresql://user:password@db:5432/myapp"
    depends_on:
      - db
    networks:
      - app_network
  
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app_network
    # healthcheck для правильного depends_on
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:alpine
    networks:
      - app_network

networks:
  app_network:
    driver: bridge
    # internal: true  # если не нужен выход в интернет

volumes:
  postgres_data:
```


### 3. Host network (все на одном хосте)

Все контейнеры используют сеть хоста напрямую:

```yaml

# docker-compose.yml
version: '3.8'

services:
  web:
    image: nginx:alpine
    network_mode: "host"  # ← Ключевая строка!
    volumes:
      - ./html:/usr/share/nginx/html
      - ./nginx-host.conf:/etc/nginx/nginx.conf
    # ports: не нужны! Контейнер слушает напрямую на хосте
    # Вместо 8080:80 просто слушает на 80 порту хоста
    depends_on:
      - api
  
  api:
    build: ./api
    network_mode: "host"  # ← Тоже host
    environment:
      DATABASE_URL: "postgresql://user:password@localhost:5432/myapp"
      # Обратите внимание: localhost, а не db!
    depends_on:
      - db
  
  db:
    image: postgres:15-alpine
    network_mode: "host"  # ← Тоже host
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    # Postgres будет слушать на 5432 порту ВСЕГО ХОСТА
    # Будьте осторожны - БД доступна извне!

volumes:
  postgres_data:

# networks: не нужны при network_mode: host
```

**Особенности host mode:**
1. **Все контейнеры на одном localhost**
2. **API подключается к БД как** `localhost:5432` (или `127.0.0.1:5432`)
3. **Порты конфликтуют** - нельзя два контейнера на одном порту
4. **Безопасность** - все службы видны в сети
5. **Производительность** - нет NAT, выше скорость

*nginx-host.conf* для host mode:

```nginx
server {
    listen 80;  # Слушаем на всех интерфейсах хоста
    server_name localhost;
    
    location /api/ {
        # API тоже на localhost, но другой порт (например 3000)
        proxy_pass http://127.0.0.1:3000/;
    }
    
    location / {
        root /usr/share/nginx/html;
    }
}
```