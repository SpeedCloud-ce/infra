# speedcloud / infra

Infraestrutura SpeedCloud — imagens 1-clique, Caddy templates, Docker Compose
stacks, runbooks. Brasil-primeiro: timezone padrão `America/Sao_Paulo`,
escolhas de region e CDN priorizam latência local.

## Layout

```
.
├── caddy/Caddyfile        # Reverse proxy template (Auto-HTTPS, single domain)
├── compose/n8n.yml        # Stack n8n (serviço único, healthcheck, volume persistente)
├── .env.example           # Variáveis documentadas (sem segredos reais)
└── README.md
```

## Stack n8n — subir local

Pré-requisitos: Docker 24+ e Docker Compose v2.

```bash
# 1. Subir (o compose carrega variáveis de .env.example direto — basta clonar)
docker compose -f compose/n8n.yml up -d

# 2. Acompanhar até ficar healthy (≤ 60s em hardware razoável)
docker compose -f compose/n8n.yml ps

# 3. Logs
docker compose -f compose/n8n.yml logs -f n8n

# 4. UI (binding em loopback no compose padrão)
#    http://127.0.0.1:5678

# 5. Teardown limpo (apaga volume!)
docker compose -f compose/n8n.yml down -v
```

Para deployments reais, copie o exemplo, edite valores e troque o `env_file`:

```bash
cp .env.example .env
# edite .env (em especial N8N_ENCRYPTION_KEY, N8N_DOMAIN, WEBHOOK_URL)
# depois aponte o compose pra ele:
#   sed -i 's|../.env.example|../.env|' compose/n8n.yml
# ou rode com: docker compose --env-file ../.env -f compose/n8n.yml up -d
```

`.env` está no `.gitignore` — nunca commite valores reais.

O healthcheck usa `wget --spider http://127.0.0.1:5678/healthz` dentro do
container. `docker ps` mostra `(healthy)` quando n8n termina o boot.

## Caddy — validar template

```bash
# Sintaxe
caddy validate --config caddy/Caddyfile --adapter caddyfile

# Render (ver JSON final que o Caddy executa)
caddy adapt --config caddy/Caddyfile --pretty
```

Para deploy real, exporte `N8N_DOMAIN` e `CADDY_ACME_EMAIL` no ambiente onde
o Caddy roda; o template substitui esses placeholders em runtime.

## O que NÃO está incluso (ainda)

Backlog explícito — ficará em issues separadas:

- TLS de produção end-to-end (Caddy + n8n no mesmo network, sem `ports:` exposto).
- Backup automático para S3 (volume `speedcloud_n8n_data` + `.env`/encryption key).
- Stack de monitoramento (logs estruturados + alerts úteis).
- Secrets management (Vault / SOPS / equivalente).
- CI: lint de Caddyfile e `compose config` em PRs.

## Convenções

- Scripts bash: `set -euo pipefail`, idempotentes, com log do que está acontecendo.
- Toda mudança em produção entra via PR; merge é humano. Sem auto-merge, sem auto-deploy.
- Backup que nunca foi restaurado é teatro: documente o tempo de restore antes de declarar "temos backup".
- Least privilege em IAM, sudo e network rules.

## Licença

[MIT](./LICENSE).
