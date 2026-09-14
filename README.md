# Oracle VPS Base Infrastructure

Infraestrutura base para hospedar múltiplas aplicações com **Nginx Proxy Reverso**, **HTTPS automático (Let's Encrypt / Certbot)** e **Docker** em uma VPS Ubuntu (Oracle Cloud Free Tier ou similar).

Projetada para ser **leve** (consumo < 50MB de RAM), **modular** (cada app tem seu próprio ciclo de vida) e **segura** (apenas as portas 22, 80 e 443 expostas no host).

---

## 🏗️ Topologia da Arquitetura

```text
                                Internet
                                   │
                           (Portas 80 / 443)
                                   │
                        [Oracle VCN Security List]
                                   │
                         [Ubuntu UFW Firewall]
                                   │
                                   ▼
┌─────────────────────── Nginx Reverse Proxy ────────────────────────┐
│  Container: nginx-proxy                                            │
│  Portas no Host: 80, 443                                          │
│  Certificados: Let's Encrypt (/var/www/certbot webroot)            │
│  Acesso via IP: Proxy direto para o Portainer                      │
└──────────────────────────────────┬─────────────────────────────────┘
                                   │
                     Rede Docker: `proxy-net` (bridge)
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         ▼                         ▼                         ▼
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  Portainer (CE)  │      │  App 1 (Node)    │      │ App 2 (Laravel)  │
│  container:      │      │  container: app1 │      │ container: app2  │
│    portainer:9000│      │  porta: 3000     │      │ porta: 8000      │
│  (Sem porta host)│      │  (Sem porta host)│      │ (Sem porta host) │
└──────────────────┘      └──────────────────┘      └──────────────────┘
```

---

## 📋 Sumário

1. [Etapa 1: Preparação da VPS](#-etapa-1-preparação-da-vps)
2. [Etapa 2: Subir a Infraestrutura Base (Nginx)](#-etapa-2-subir-a-infraestrutura-base-nginx)
3. [Etapa 3: Gerenciador de Containers (Portainer)](#-etapa-3-gerenciador-de-containers-portainer)
4. [Etapa 4: Configurar Domínio e HTTPS (Let's Encrypt)](#-etapa-4-configurar-domínio-e-https-lets-encrypt)
5. [Etapa 5: Padrão para Deploy de Novas Aplicações](#-etapa-5-padrão-para-deploy-de-novas-aplicaçães)
6. [💡 Dica: Usando sslip.io (SSL sem domínio próprio)](#-dica-usando-sslipio-ssl-sem-domínio-próprio)
7. [🤖 Validação e CI](#-validação-e-ci)

---

## 🚀 Etapa 1: Preparação da VPS

Acesse a VPS via SSH e execute os comandos abaixo:

### 1.1 Atualizar o sistema operacional
```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2 Configurar o Firewall do Ubuntu (UFW)
```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```

> **Atenção (Oracle Cloud VCN):** No painel da Oracle Cloud (*Networking > Virtual Cloud Networks > Security Lists > Default Security List*), certifique-se de que existem Ingress Rules para:
> * **Porta 80 (HTTP):** Source CIDR `0.0.0.0/0`, TCP, Port `80`.
> * **Porta 443 (HTTPS):** Source CIDR `0.0.0.0/0`, TCP, Port `443`.

### 1.3 Instalar o Docker e Docker Compose
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
rm get-docker.sh
```

### 1.4 Configurar permissões do usuário
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### 1.5 Criar a rede Docker compartilhada
```bash
docker network create proxy-net
```

---

## 🌐 Etapa 2: Subir a Infraestrutura Base (Nginx)

No diretório deste repositório na VPS (`~/oracle-vps-infra`):

### 2.1 Subir o container do Nginx
```bash
docker compose up -d
```

### 2.2 Verificar status
```bash
docker compose ps
```
O container `nginx-proxy` estará em execução nas portas `80` e `443`.

---

## 🐳 Etapa 3: Gerenciador de Containers (Portainer)

O Portainer pode ser executado sem expor nenhuma porta no host — todo o tráfego passa pelo Nginx através da rede `proxy-net`.

### 3.1 Subir o Portainer
```bash
docker volume create portainer_data

docker run -d \
  --name portainer \
  --restart always \
  --network proxy-net \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

### 3.2 Acessar o Portainer
Acesse no navegador pelo IP público da sua VPS:
```text
http://<IP_PUBLICO_DA_SUA_VPS>
```
*O Nginx (`nginx/conf.d/default.conf`) já está configurado como proxy reverso padrão para `http://portainer:9000` com suporte a WebSockets (terminal e logs).*

---

## 🔒 Etapa 4: Configurar Domínio e HTTPS (Let's Encrypt)

Quando você tiver um domínio registrado:

### 4.1 Criar apontamento DNS (Tipo A)
No seu gerenciador DNS (Cloudflare, Registro.br, etc.):
* **Tipo:** `A`
* **Nome / Host:** `app1` (ou `@` para a raiz)
* **Valor / Destino:** `<IP_PUBLICO_DA_SUA_VPS>`
* **Cloudflare Proxy:** Desativado (*DNS Only*) na primeira emissão.

### 4.2 Emitir o Certificado SSL
Na pasta `~/oracle-vps-infra`:
```bash
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email seu-email@exemplo.com \
  --agree-tos \
  --no-eff-email \
  -d app1.meudominio.com
```

### 4.3 Criar a configuração do site no Nginx
1. Copie o template de exemplo:
   ```bash
   cp nginx/conf.d/app.conf.example nginx/conf.d/app1.meudominio.com.conf
   ```
2. Edite o arquivo (`nano nginx/conf.d/app1.meudominio.com.conf`) ajustando:
   * `server_name` com seu domínio;
   * Caminho dos certificados (`ssl_certificate` e `ssl_certificate_key`);
   * Endereço interno do backend em `proxy_pass` (ex: `http://minha-app:3000`).
3. Recarregue o Nginx sem reiniciar o container:
   ```bash
   docker compose exec nginx nginx -s reload
   ```

### 4.4 Renovação Automática (Crontab da VPS)
Abra o cron do servidor:
```bash
crontab -e
```
Adicione a linha para renovar diariamente às 03:00 da manhã:
```cron
0 3 * * * cd /home/ubuntu/oracle-vps-infra && docker compose run --rm certbot renew --webroot -w /var/www/certbot --quiet && docker compose exec -T nginx nginx -s reload
```

---

## 📦 Etapa 5: Padrão para Deploy de Novas Aplicações

Para adicionar qualquer nova aplicação (Node, Next.js, Laravel, Go, etc.) no futuro:

### 1. Crie a pasta do projeto (fora da pasta da infraestrutura)
Exemplo em `~/apps/minha-app/`:
```yaml
# ~/apps/minha-app/docker-compose.yml
services:
  web:
    build: .
    container_name: minha-app
    restart: unless-stopped
    networks:
      - proxy-net

networks:
  proxy-net:
    external: true
```
> **Regra de Ouro:** Não use a seção `ports:` no Compose da sua aplicação. O container só precisa do `container_name` e da rede `proxy-net`.

### 2. Inicie a aplicação
```bash
cd ~/apps/minha-app
docker compose up -d --build
```

### 3. Emita o certificado SSL
```bash
cd ~/oracle-vps-infra
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email seu-email@exemplo.com \
  --agree-tos \
  --no-eff-email \
  -d minhaapp.meudominio.com
```

### 4. Ative a rota no Nginx
```bash
cp nginx/conf.d/app.conf.example nginx/conf.d/minhaapp.meudominio.com.conf
# Ajuste o server_name e o proxy_pass http://minha-app:<porta_interna>
docker compose exec nginx nginx -s reload
```

---

## 💡 Dica: Usando `sslip.io` (SSL sem domínio próprio)

Se você precisa de HTTPS ou subdomínios antes de adquirir um domínio personalizado, use o serviço gratuito [sslip.io](https://sslip.io):
* Qualquer subdomínio `*.<SEU_IP>.sslip.io` aponta automaticamente para `<SEU_IP>`.

### Exemplo: HTTPS no Portainer via `sslip.io`
1. Emita o certificado:
   ```bash
   docker compose run --rm certbot certonly \
     --webroot \
     --webroot-path=/var/www/certbot \
     --email seu-email@exemplo.com \
     --agree-tos \
     --no-eff-email \
     -d portainer.<SEU_IP>.sslip.io
   ```
2. Ative a configuração dedicada:
   ```bash
   cp nginx/conf.d/portainer.conf.example nginx/conf.d/portainer.<SEU_IP>.sslip.io.conf
   # Substitua SEU_IP pelo seu endereço no arquivo copiado
   docker compose exec nginx nginx -s reload
   ```
3. Acesse `https://portainer.<SEU_IP>.sslip.io`.

---

## 🤖 Validação e CI

O repositório inclui automação via GitHub Actions (`.github/workflows/ci.yml`) que testa:
* Sintaxe do arquivo `docker-compose.yml`.
* Sintaxe das configurações do Nginx (`nginx -t`).

Para validar manualmente na sua máquina:
```bash
# Validar compose
docker compose config --quiet

# Validar Nginx localmente
docker run --rm --add-host portainer:127.0.0.1 \
  -v $(pwd)/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v $(pwd)/nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf:ro \
  nginx:1.27-alpine nginx -t
```
