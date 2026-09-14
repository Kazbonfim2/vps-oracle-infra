# Oracle VPS Base Infrastructure

Infraestrutura base para hospedar múltiplas aplicações com **Nginx Proxy Reverso**, **HTTPS automático (Certbot)** e **Docker Compose** em uma VPS Ubuntu na Oracle Cloud.

---

## 📋 Sumário das 5 Etapas

* [Etapa 1: Preparação da VPS](#-etapa-1-preparação-da-vps)
* [Etapa 2: Subir a Infraestrutura Base (Nginx)](#-etapa-2-subir-a-infraestrutura-base-nginx)
* [Etapa 3: Configurar Domínio e HTTPS (Let's Encrypt)](#-etapa-3-configurar-domínio-e-https-lets-encrypt)
* [Etapa 4: Conectar uma Aplicação de Exemplo](#-etapa-4-conectar-uma-aplicação-de-exemplo)
* [Etapa 5: Guia de Operação e Novas Aplicações](#-etapa-5-guia-de-operação-e-novas-aplicações)
* [💡 Dica: Usando o IP com `sslip.io` (Sem domínio próprio)](#-dica-usando-o-ip-com-sslipio-sem-domínio-próprio)

---

## 🚀 Etapa 1: Preparação da VPS

Execute no terminal SSH da sua VPS Ubuntu:

### 1.1 Atualizar o sistema operacional
```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2 Liberar portas no Firewall do Ubuntu (iptables / UFW)
```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```

> **Atenção (Oracle Cloud VCN):** No painel da Oracle Cloud (Networking > Virtual Cloud Networks > Security Lists > Default Security List), certifique-se de que existem regras de entrada (Ingress Rules) para:
> * **Porta 80 (HTTP):** Source CIDR `0.0.0.0/0`, IP Protocol `TCP`, Destination Port Range `80`.
> * **Porta 443 (HTTPS):** Source CIDR `0.0.0.0/0`, IP Protocol `TCP`, Destination Port Range `443`.

### 1.3 Instalar o Docker oficial e Compose V2
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

### 2.2 Verificar status dos containers
```bash
docker compose ps
```
*(O container `nginx-proxy` deve estar com status `Up` e portas `0.0.0.0:80->80/tcp`, `0.0.0.0:443->443/tcp`)*

### 2.3 Testar resposta local
```bash
curl -i http://localhost
```
*Deve retornar `200 OK` com a mensagem `Nginx Base Infrastructure is running.`*

---

## 🔒 Etapa 3: Configurar Domínio e HTTPS (Let's Encrypt)

### 3.1 Apontar o DNS
No seu gerenciador de domínio (Cloudflare, Registro.br, etc.), crie um registro **Tipo A**:
* **Nome / Host:** `app1` (ou seu subdomínio)
* **Tipo:** `A`
* **Valor / IP:** `<IP_PUBLICO_DA_SUA_VPS>`
* **Proxy Cloudflare:** Desativado (DNS Only / Cinza) durante a primeira emissão.

*(Se não tiver domínio, veja a seção [sslip.io](#-dica-usando-o-ip-com-sslipio-sem-domínio-próprio) abaixo).*

### 3.2 Emitir o Certificado SSL
Execute na pasta `~/oracle-vps-infra` da VPS:

```bash
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email seu-email@exemplo.com \
  --agree-tos \
  --no-eff-email \
  -d app1.meudominio.com
```

### 3.3 Ativar o Proxy Reverso com SSL no Nginx
1. Copie o template para o domínio desejado:
   ```bash
   cp nginx/conf.d/app.conf.example nginx/conf.d/app1.meudominio.com.conf
   ```
2. Edite o arquivo (`nano nginx/conf.d/app1.meudominio.com.conf`) ajustando `server_name`, caminhos do SSL e `proxy_pass`.
3. Recarregue o Nginx sem derrubar conexões:
   ```bash
   docker compose exec nginx nginx -s reload
   ```

### 3.4 Configurar Renovação Automática (Cron da VPS)
Abra o agendador de tarefas:
```bash
crontab -e
```
Adicione no final do arquivo:
```cron
0 3 * * * cd /home/ubuntu/oracle-vps-infra && docker compose run --rm certbot renew --webroot -w /var/www/certbot --quiet && docker compose exec -T nginx nginx -s reload
```

---

## 📦 Etapa 4: Conectar uma Aplicação de Exemplo

Para testar o fluxo completo de uma aplicação real:

### 4.1 Subir a aplicação de exemplo
```bash
# Na pasta da infraestrutura:
docker compose -f examples/demo-app/docker-compose.yml up -d
```
*(O container `demo-app` sobe conectado exclusivamente à rede `proxy-net`)*

### 4.2 Ativar a configuração no Nginx
```bash
cp examples/demo-app/demo.meudominio.com.conf nginx/conf.d/demo.meudominio.com.conf
docker compose exec nginx nginx -s reload
```

### 4.3 Testar a conexão
```bash
curl -H "Host: demo.meudominio.com" http://localhost
```

---

## ⚡ Etapa 5: Guia de Operação e Novas Aplicações

Para cada nova aplicação (Node, Laravel, Next.js, etc.) que você for adicionar:

1. **Criar a aplicação** em sua própria pasta (ex: `~/apps/minha-app`):
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
2. **Subir a aplicação:**
   ```bash
   cd ~/apps/minha-app && docker compose up -d
   ```
3. **Emitir o certificado SSL (na pasta `~/oracle-vps-infra`):**
   ```bash
   docker compose run --rm certbot certonly --webroot -w /var/www/certbot --email seu-email@exemplo.com --agree-tos --no-eff-email -d minhaapp.meudominio.com
   ```
4. **Configurar o Nginx:**
   ```bash
   cp nginx/conf.d/app.conf.example nginx/conf.d/minhaapp.meudominio.com.conf
   docker compose exec nginx nginx -s reload
   ```

---

## 💡 Dica: Usando o IP com `sslip.io` (Sem domínio próprio)

Se você não comprou um domínio ainda, use o serviço gratuito `sslip.io`:
* Qualquer subdomínio `*.137.131.172.167.sslip.io` resolve automaticamente para o seu IP.

### Exemplo: Proteger o Portainer com HTTPS agora mesmo

1. **Conecte o container do Portainer à rede `proxy-net`:**
   ```bash
   docker network connect proxy-net portainer
   ```

2. **Emita o certificado SSL real gratuito:**
   ```bash
   docker compose run --rm certbot certonly \
     --webroot \
     --webroot-path=/var/www/certbot \
     --email seu-email@exemplo.com \
     --agree-tos \
     --no-eff-email \
     -d portainer.137.131.172.167.sslip.io
   ```

3. **Ative a configuração do Portainer no Nginx:**
   ```bash
   cp nginx/conf.d/portainer.conf.example nginx/conf.d/portainer.137.131.172.167.sslip.io.conf
   docker compose exec nginx nginx -s reload
   ```

4. **Acesse no navegador com SSL ativo:**
   `https://portainer.137.131.172.167.sslip.io`
