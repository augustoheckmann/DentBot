# WhatsApp Chatbot — Infraestrutura

## Serviços

| Serviço | URL | Credenciais |
|---|---|---|
| n8n (editor de fluxos) | http://localhost:5678 | ver `.env` |
| Evolution API (WhatsApp) | http://localhost:8080 | API key no `.env` |
| PostgreSQL | localhost:5432 | ver `.env` |
| Redis | localhost:6379 | sem senha |

## Setup Inicial (Primeira Vez)

### 1. Copiar e preencher o `.env`
```bash
cp .env.example .env
# Edite o .env e troque todos os valores CHANGE_ME
# Gere uma chave de criptografia segura:
openssl rand -hex 16
# Cole o resultado em N8N_ENCRYPTION_KEY no .env
```

### 2. Subir todos os serviços
```bash
docker compose up -d
```

### 3. Verificar que está tudo rodando
```bash
docker compose ps
# Todos os 4 containers devem estar "Up" e com health "healthy"
```

### 4. Conectar o WhatsApp (fazer uma vez)
1. Abra http://localhost:8080
2. Clique em **"Create Instance"** e dê um nome (ex: `chatbot`)
3. Clique na instância criada e depois em **"Connect"**
4. Escaneie o QR Code com o WhatsApp que será o bot
5. O status fica verde — sessão salva no volume Docker (sobrevive a reinicializações)

### 5. Abrir o n8n
1. Abra http://localhost:5678
2. Login com `N8N_BASIC_AUTH_USER` / `N8N_BASIC_AUTH_PASSWORD` do `.env`
3. Crie um novo Workflow e comece a construir a lógica do chatbot

---

## Como construir o chatbot no n8n (para os colegas)

1. Abra http://localhost:5678 e faça login
2. Clique em **"+ New Workflow"**
3. Adicione um nó **Webhook**:
   - Method: `POST`
   - Path: `whatsapp`
   - Clique em **"Listen for Test Event"** e envie uma mensagem pelo WhatsApp para capturar um exemplo real
4. Adicione um nó **IF** para verificar o texto da mensagem:
   - Valor: `{{ $json.body.data.message.conversation }}`
5. Adicione um nó **HTTP Request** para enviar resposta:
   - Method: `POST`
   - URL: `http://evolution-api:8080/message/sendText/chatbot`
   - Header: `apikey: <valor de EVOLUTION_API_KEY no .env>`
   - Body (JSON):
     ```json
     {
       "number": "{{ $json.body.data.key.remoteJid }}",
       "text": "Olá! Recebi sua mensagem."
     }
     ```
6. Clique em **"Save"** e depois em **"Activate"** o workflow

---

## Comandos Úteis

```bash
# Subir tudo em segundo plano
docker compose up -d

# Parar tudo (dados são preservados)
docker compose down

# Ver logs em tempo real de um serviço
docker compose logs -f n8n
docker compose logs -f evolution-api
docker compose logs -f postgres

# Verificar status dos containers
docker compose ps

# Reiniciar um serviço específico
docker compose restart n8n

# Baixar versões mais recentes das imagens
docker compose pull

# CUIDADO: Para e apaga todos os dados
docker compose down -v
```

## Acesso pela Rede Local (para colegas na mesma rede)

Descubra seu IP local:
```bash
ip a | grep "inet " | grep -v 127
```

Altere no `.env`:
```
WEBHOOK_URL=http://<seu_ip>:5678
```

Reinicie o n8n:
```bash
docker compose restart n8n
```

Colegas acessam `http://<seu_ip>:5678` e `http://<seu_ip>:8080`.

---

## Solução de Problemas

**Porta 5432 em uso (outro PostgreSQL rodando):**
Mude `POSTGRES_PORT=5433` no `.env` e reinicie.

**n8n mostra erro de banco na inicialização:**
```bash
docker compose restart n8n
```
Ocorre quando o n8n sobe antes do Postgres terminar de inicializar.

**Sessão do WhatsApp expirou (QR Code necessário novamente):**
Acesse http://localhost:8080, abra a instância e escaneie novamente. Sessões duram semanas/meses normalmente.

**Evolution API não consegue chamar o n8n:**
Dentro da rede Docker, o endereço correto é `http://n8n:5678` (nome do serviço, não `localhost`). Já está configurado corretamente no `docker-compose.yml`.
