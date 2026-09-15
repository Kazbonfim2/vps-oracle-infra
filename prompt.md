# Instrução para IA: Padrão Docker para Deploy na VPS (Oracle Cloud Infra)

Você é um engenheiro DevOps especialista. Sua tarefa é preparar este projeto para rodar na infraestrutura Docker da VPS, seguindo rigorosamente o padrão arquitetural estabelecido abaixo.

---

## 🏛️ Arquitetura e Regras Obrigatórias

1. **Proxy Reverso Central:** A VPS possui um Nginx central rodando na rede Docker externa `proxy-net`. Ele recebe todo o tráfego externo nas portas `80` e `443`.
2. **Proibido Mapear Portas no Host:** **NUNCA** use `ports: "3000:3000"` no `docker-compose.yml`. As portas dos containers NUNCA são expostas no host da VPS. A comunicação é 100% interna via rede Docker `proxy-net`.
3. **Rede Compartilhada:** Todo container web/api DEVE se conectar à rede externa `proxy-net`.
4. **Nomes Fixos:** Todo container web/api DEVE possuir `container_name` explícito (ex: `meu-app-web`, `meu-app-api`).
5. **Banco de Dados Externo:** Nenhum banco relacional pesado deve rodar na VPS (utilizar Turso, Supabase, Neon, etc.).
6. **Otimização de Recursos:** Use imagens Alpine/Slim e Dockerfile multi-stage build para manter o consumo de RAM mínimo (< 100MB quando possível).

---

## 📋 Entregáveis que Você (IA) Deve Gerar para o Projeto

### 1. `Dockerfile` (Multi-stage e Otimizado)
Gere um Dockerfile de produção leve (Alpine/Slim), expondo apenas a porta interna. Use estes templates como referência:

#### Node.js / Next.js / TypeScript
```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --production

# Production stage
FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

#### PHP / Laravel
```dockerfile
FROM php:8.3-cli-alpine
WORKDIR /var/www/html
RUN apk add --no-cache libpq-dev linux-headers && docker-php-ext-install pcntl bcmath
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
COPY . .
RUN composer install --no-dev --optimize-autoloader --no-interaction
EXPOSE 8000
CMD ["php", "artisan", "serve", "--host=0.0.0.0", "--port=8000"]
```

#### Python / FastAPI
```dockerfile
FROM python:3.12-slim
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
USER 1000
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 2. `docker-compose.yml` (Na raiz do projeto)
Estrutura padrão obrigatória:

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: <NOME_DO_PROJETO>
    restart: unless-stopped
    environment:
      - NODE_ENV=production
      - PORT=<PORTA_INTERNA>
    env_file:
      - .env
    networks:
      - proxy-net

networks:
  proxy-net:
    external: true
```

### 3. Configuração Nginx: `<dominio>.conf`
Gere o arquivo de configuração para ser colocado em `oracle-vps-infra/nginx/conf.d/<dominio>.conf`:

```nginx
# 1. Redirecionamento HTTP -> HTTPS + Validação Certbot
server {
    listen 80;
    listen [::]:80;
    server_name <DOMINIO>;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# 2. Servidor HTTPS Seguro
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name <DOMINIO>;

    ssl_certificate /etc/letsencrypt/live/<DOMINIO>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<DOMINIO>/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    client_max_body_size 50M;

    location / {
        proxy_pass http://<CONTAINER_NAME>:<PORTA_INTERNA>;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 90;
    }
}
```

### 4. Guia Passo a Passo de Deploy (Comandos Exatos)
Apresente os comandos para o usuário executar na VPS:

```bash
# 1. No diretório do projeto (~/apps/<NOME_DO_PROJETO>):
docker compose up -d --build

# 2. No diretório da infra (~/oracle-vps-infra), gerar certificado SSL:
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email seu-email@exemplo.com \
  --agree-tos \
  --no-eff-email \
  -d <DOMINIO>

# 3. Criar a config do Nginx e recarregar:
# (Salvar o conteúdo em ~/oracle-vps-infra/nginx/conf.d/<DOMINIO>.conf)
docker compose exec nginx nginx -s reload
```
