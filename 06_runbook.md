# Colab Infinity — Manual de Operação (Runbook)

**Versão:** 1.0.0
**Data:** 2025-07-14
**Status:** Aprovado
**Referências:** `03_sad.md`, `05_setup_guide.md`, `08_risk_analysis.md`

---

## Índice

1. [Visão Geral Operacional](#1-visão-geral-operacional)
2. [Rotinas Diárias](#2-rotinas-diárias)
3. [Monitoramento de Saúde do Sistema](#3-monitoramento-de-saúde-do-sistema)
4. [Gerenciamento do Pool de Contas](#4-gerenciamento-do-pool-de-contas)
5. [Procedimentos para Falhas Comuns](#5-procedimentos-para-falhas-comuns)
6. [Recuperação de Desastres](#6-recuperação-de-desastres)
7. [Comandos Úteis do CLI](#7-comandos-úteis-do-cli)
8. [Referência de Alertas e Significados](#8-referência-de-alertas-e-significados)

---

## 1. Visão Geral Operacional

### 1.1 Estados Normais do Sistema

```
IDLE         → Sistema parado; proxy local não está escutando
STARTING     → Aguardando sessão Colab subir (polling ngrok_url.json no Drive)
ACTIVE       → Operação normal; proxy ativo; health checks passando
FAILING      → N falhas consecutivas detectadas; investigando
SWITCHING    → Trocando de conta Google; proxy retorna 503 temporariamente
RECOVERING   → Nova sessão Colab ativa; aguardando confirmação de health
HALTED       → Pool exausto; nenhuma conta disponível; aguardando cooldown
```

### 1.2 Arquivos Operacionais Críticos

| Arquivo                                            | Localização                  | Propósito                                    |
|----------------------------------------------------|------------------------------|----------------------------------------------|
| `colab_infinity_config.yaml`                       | `~/.colab_infinity/`         | Configuração principal do orquestrador       |
| `accounts.json`                                    | `~/.colab_infinity/`         | Pool de contas com tokens ngrok              |
| `drive_token.json`                                 | `~/.colab_infinity/`         | Token OAuth2 da conta armazém                |
| `orchestrator.log`                                 | `~/.colab_infinity/logs/`    | Logs estruturados do orquestrador            |
| `ngrok_url.json`                                   | Drive: `colab_infinity/pool_state/` | URL ngrok da sessão ativa           |
| `pool_state.json`                                  | Drive: `colab_infinity/pool_state/` | Estado atual do pool de contas      |
| `ci_ckpt_YYYYMMDD_HHMMSS.json`                    | Drive: `colab_infinity/checkpoints/` | Checkpoints de estado            |
| `metrics.jsonl`                                    | Drive: `colab_infinity/logs/` | Métricas e eventos operacionais             |

### 1.3 Portas e Processos Locais

| Processo                  | Porta     | PID File                         | Bind              |
|---------------------------|-----------|----------------------------------|-------------------|
| Proxy Local (orquestrador)| `11434`   | `~/.colab_infinity/proxy.pid`    | `127.0.0.1` apenas|
| FastAPI no Colab          | `8000`    | N/A (dentro do Colab)            | `127.0.0.1` (Colab)|
| ngrok Agent (no Colab)    | `4040`    | N/A (dentro do Colab)            | `127.0.0.1` (Colab)|

---

## 2. Rotinas Diárias

### 2.1 Checklist de Início do Dia (≈ 5 minutos)

Execute os seguintes passos toda vez que iniciar o sistema ou retomar após uma pausa:

```
[ ] 1. Verificar se o orquestrador está rodando
        ps aux | grep orchestrator | grep -v grep

[ ] 2. Se não estiver rodando, iniciar:
        cd ~/colab-infinity
        source .venv/bin/activate
        python3 -m colab_infinity.orchestrator \
          --config ~/.colab_infinity/colab_infinity_config.yaml &

[ ] 3. Verificar health check do proxy local
        curl -s http://127.0.0.1:11434/health | python3 -c \
          "import sys,json; d=json.load(sys.stdin); \
           print(d['status'], '|', d['session']['id'][:30])"

[ ] 4. Se o health check falhar (servidor Colab não ativo):
        → Abrir o notebook colab_server.ipynb na conta ativa
        → Executar todas as células (Células 1 a 6)
        → Aguardar a URL ngrok ser publicada no Drive

[ ] 5. Verificar o estado do pool de contas
        python3 -m colab_infinity.cli pool list

[ ] 6. Verificar se há contas em cooldown prestes a expirar
        → Se sim: nenhuma ação necessária; expirará automaticamente

[ ] 7. Verificar espaço em disco no Drive da conta armazém
        python3 -m colab_infinity.cli drive info
```

### 2.2 Checklist de Fim do Dia (≈ 3 minutos)

```
[ ] 1. Forçar checkpoint antes de encerrar
        curl -s -X POST http://127.0.0.1:11434/v1/checkpoint \
          -H "Content-Type: application/json" \
          -d '{"reason": "end_of_day"}'

[ ] 2. Verificar que o checkpoint foi salvo com sucesso
        python3 -m colab_infinity.cli checkpoint list --limit 1

[ ] 3. Se for encerrar o Ouroboros Runtime:
        → Encerrar o Runtime normalmente (garante flush da memória SQLite)

[ ] 4. Se for manter rodando overnight:
        → Verificar que o keepalive JavaScript está ativo na Célula 6
        → Verificar que a sessão Colab tem tempo restante suficiente
          (verificar campo session.estimated_quota_remaining_minutes)

[ ] 5. Anotar a conta ativa e o session_id para referência:
        curl -s http://127.0.0.1:11434/v1/status | python3 -c \
          "import sys,json; s=json.load(sys.stdin); \
           print('Conta:', s['account']['index'], \
                 '| Sessão:', s['session']['id'])"
```

### 2.3 Rotina Semanal (≈ 20 minutos)

```
[ ] 1. Revisar logs da semana por erros críticos
        grep '"level":"error"\|"level":"critical"' \
          ~/.colab_infinity/logs/orchestrator.log | tail -50

[ ] 2. Verificar métricas de disponibilidade
        python3 -m colab_infinity.cli metrics --period 7d

[ ] 3. Limpar checkpoints antigos (mantém apenas os últimos 10 automaticamente,
        mas verificar manualmente se há acúmulo)
        python3 -m colab_infinity.cli checkpoint clean --keep 10

[ ] 4. Verificar logs do Drive por tamanho
        python3 -m colab_infinity.cli drive info

[ ] 5. Revisar status de cada conta do pool
        → Identificar contas com alto failure_count
        → Considerar substituir contas com failure_count > 5

[ ] 6. Atualizar tokens ngrok se necessário
        → Tokens ngrok não expiram, mas podem ser revogados
        → Verificar em dashboard.ngrok.com se os tokens estão ativos

[ ] 7. Testar cada conta do pool individualmente
        python3 -m colab_infinity.cli pool test --all
```

---

## 3. Monitoramento de Saúde do Sistema

### 3.1 Indicadores-Chave de Saúde

| Indicador                           | Saudável               | Atenção                   | Crítico                      |
|-------------------------------------|------------------------|---------------------------|------------------------------|
| Estado do orquestrador              | `ACTIVE`               | `RECOVERING`, `SWITCHING` | `HALTED`, processo morto     |
| Health check latência               | < 500ms                | 500ms–2s                  | > 2s ou timeout              |
| Falhas consecutivas de health check | 0                      | 1–2                       | ≥ 3 (dispara troca)          |
| Contas disponíveis no pool          | ≥ 2                    | 1                         | 0 (pool exausto)             |
| Quota restante da sessão ativa      | > 60 min               | 30–60 min                 | < 30 min (troca iminente)    |
| Uso de VRAM                         | < 80%                  | 80–90%                    | > 90%                        |
| Tamanho do Drive (conta armazém)    | < 60%                  | 60–80%                    | > 80%                        |
| Último checkpoint                   | < 10 min atrás         | 10–30 min atrás           | > 30 min (checkpoint falhou) |

### 3.2 Verificação Manual de Saúde (Smoke Test Completo)

```bash
#!/bin/bash
# Execução: bash smoke_test.sh
# Tempo estimado: 30 segundos

PROXY="http://127.0.0.1:11434"
PASS=0
FAIL=0

check() {
    local desc="$1"
    local cmd="$2"
    local expected="$3"
    result=$(eval "$cmd" 2>&1)
    if echo "$result" | grep -q "$expected"; then
        echo "  ✓ $desc"
        PASS=$((PASS + 1))
    else
        echo "  ✗ $desc → obtido: ${result:0:80}"
        FAIL=$((FAIL + 1))
    fi
}

echo "=== Colab Infinity Smoke Test ==="

check "Proxy local respondendo" \
  "curl -sf $PROXY/health" \
  "status"

check "Status OK (não loading)" \
  "curl -sf $PROXY/health | python3 -c \"import sys,json; d=json.load(sys.stdin); print(d['status'])\"" \
  "ok"

check "Modelo carregado" \
  "curl -sf $PROXY/health | python3 -c \"import sys,json; d=json.load(sys.stdin); print(d['model']['loaded'])\"" \
  "True"

check "GPU ativa (não CPU)" \
  "curl -sf $PROXY/health | python3 -c \"import sys,json; d=json.load(sys.stdin); print(d['runtime']['device'])\"" \
  "cuda"

check "Inferência simples funciona" \
  "curl -sf -X POST $PROXY/v1/chat/completions \
    -H 'Content-Type: application/json' \
    -d '{\"messages\":[{\"role\":\"user\",\"content\":\"ping\"}],\"max_tokens\":5}' \
  | python3 -c \"import sys,json; r=json.load(sys.stdin); print(r['choices'][0]['finish_reason'])\"" \
  "stop"

check "Pool tem contas disponíveis" \
  "curl -sf $PROXY/v1/status | python3 -c \"import sys,json; s=json.load(sys.stdin); print(s['pool']['available'])\"" \
  "[1-9]"

echo ""
echo "=== Resultado: $PASS passou, $FAIL falhou ==="
[ $FAIL -eq 0 ] && exit 0 || exit 1
```

### 3.3 Monitoramento Contínuo via Logs

Para acompanhar os logs em tempo real:

```bash
# Logs do orquestrador em tempo real (formato JSON Lines → pretty print)
tail -f ~/.colab_infinity/logs/orchestrator.log \
  | python3 -c "
import sys, json
for line in sys.stdin:
    try:
        d = json.loads(line)
        ts = d.get('timestamp', '')[:19]
        lvl = d.get('level', 'INFO').upper()
        evt = d.get('event', d.get('msg', ''))
        extra = {k:v for k,v in d.items()
                 if k not in ('timestamp','level','event','msg','logger')}
        extra_str = ' '.join(f'{k}={v}' for k,v in extra.items())[:80]
        print(f'{ts} [{lvl:8}] {evt} {extra_str}')
    except:
        print(line.rstrip())
"
```

Para filtrar apenas eventos críticos:

```bash
tail -f ~/.colab_infinity/logs/orchestrator.log \
  | grep -E '"level":"(error|critical)"' \
  | python3 -m json.tool
```

### 3.4 Alertas via Webhook (se configurado)

Se `notifications.enabled: true` e `webhook_url` estiver preenchido em
`colab_infinity_config.yaml`, os seguintes eventos enviam notificações automáticas:

| Evento                | Severidade | Descrição                                       |
|-----------------------|------------|------------------------------------------------|
| `account_switched`    | INFO       | Troca de conta bem-sucedida                     |
| `pool_exhausted`      | CRITICAL   | Todas as contas exauridas; sistema em HALTED    |
| `server_error`        | ERROR      | Erro interno durante inferência                 |
| `checkpoint_failed`   | ERROR      | Falha ao salvar checkpoint no Drive             |
| `drive_unreachable`   | ERROR      | Drive API inacessível por mais de 3 tentativas  |

Para testar o webhook manualmente:

```bash
python3 -m colab_infinity.cli notify-test \
  --event account_switched \
  --data '{"from_index": 0, "to_index": 1, "reason": "test"}'
```

---

## 4. Gerenciamento do Pool de Contas

### 4.1 Verificar Status Atual do Pool

```bash
python3 -m colab_infinity.cli pool list

# Saída esperada:
# ╔══════╦══════════════════════════════╦═══════════╦══════════════╦════════════╗
# ║ Idx  ║ Email (mascarado)            ║ Status    ║ Cooldown até ║ Uso (req)  ║
# ╠══════╬══════════════════════════════╬═══════════╬══════════════╬════════════╣
# ║  0   ║ conta***@gmail.com           ║ active    ║ —            ║ 247        ║
# ║  1   ║ colab***@gmail.com           ║ available ║ —            ║ 0          ║
# ║  2   ║ test***@gmail.com            ║ exhausted ║ 2025-07-15   ║ 312        ║
# ╚══════╩══════════════════════════════╩═══════════╩══════════════╩════════════╝
```

### 4.2 Adicionar Nova Conta ao Pool

**Passo 1:** Criar a conta Google e habilitá-la no Colab (ver Setup Guide, Passo 3).

**Passo 2:** Criar a conta ngrok correspondente e obter o token.

**Passo 3:** Adicionar ao arquivo de contas:

```bash
# Editar diretamente o arquivo de contas
nano ~/.colab_infinity/accounts.json

# Adicionar ao array "pool":
# {
#   "index": 3,
#   "email": "nova.conta@gmail.com",
#   "ngrok_token": "TOKEN_NGROK_DA_NOVA_CONTA",
#   "role": "reserve",
#   "status": "available",
#   "notes": "Adicionada em 2025-07-20"
# }
```

**Passo 4:** Recarregar o pool sem reiniciar o orquestrador:

```bash
python3 -m colab_infinity.cli pool reload
# Saída esperada:
# Pool recarregado. 4 contas encontradas.
# ✓ Nova conta: nova.conta@gmail.com (índice 3) — disponível
```

**Passo 5:** Testar a nova conta:

```bash
python3 -m colab_infinity.cli pool test --index 3
# Verifica se o token ngrok é válido e se o Colab é acessível
```

### 4.3 Remover Conta do Pool

Para remover permanentemente uma conta (ex.: conta banida):

```bash
# Marcar como disabled (não remove o registro, apenas desativa)
python3 -m colab_infinity.cli pool disable --index 2

# Ou editar diretamente o accounts.json:
# Mudar "status": "available" → "status": "disabled"
# E no pool_state.json do Drive atualizar também

# Para remover o registro completamente:
# Editar accounts.json e remover o objeto do array
# Reindexar os demais se necessário
```

> ⚠️ **Atenção:** Se a conta a ser removida estiver ativa no momento (`status: "active"`),
> force uma troca de conta antes de remover:
> ```bash
> python3 -m colab_infinity.cli switch-account --auto
> # Aguardar a troca completar, depois remover
> ```

### 4.4 Forçar Troca de Conta Manualmente

```bash
# Trocar para a próxima conta disponível automaticamente
python3 -m colab_infinity.cli switch-account --auto

# Trocar para uma conta específica
python3 -m colab_infinity.cli switch-account --target-index 2

# Saída esperada:
# Iniciando troca de conta: 0 → 2
# Salvando checkpoint pré-troca...
# ✓ Checkpoint salvo: ci_ckpt_20250714_160000_pre_switch.json
# Aguardando nova sessão Colab...
# ✓ Sessão ativa na conta 2 (ci_sess_20250714_160312)
# ✓ Proxy local atualizado para: https://xyz789.ngrok-free.app
# Troca concluída em 312 segundos.
```

### 4.5 Resetar Conta em Cooldown (Forçar Disponibilidade)

Use com cuidado — respeitar o cooldown reduz o risco de banimento de conta.

```bash
# Verificar quando o cooldown expira
python3 -m colab_infinity.cli pool list

# Se necessário (emergência), forçar disponibilidade:
python3 -m colab_infinity.cli pool reset-cooldown --index 2

# Saída:
# ⚠️  Cooldown resetado para conta 2. Status: available.
# AVISO: Usar conta antes do cooldown natural aumenta risco de limitação pelo Google.
```

---

## 5. Procedimentos para Falhas Comuns

### 5.1 FALHA: Túnel ngrok Expirou / URL Inválida

**Sintomas:**
- Health checks retornam `ConnectionError` ou `ConnectionTimeout`
- `curl https://<hash>.ngrok-free.app/health` retorna `ERR_TUNNEL_CONNECTION_FAILED`
- Log contém: `ngrok_tunnel_expired` ou `HTTP 402` na URL ngrok

**Causa provável:** O processo ngrok dentro do Colab morreu, ou a sessão Colab expirou.

**Diagnóstico:**

```bash
# 1. Verificar o status atual
curl -s http://127.0.0.1:11434/health 2>&1 | head -5

# 2. Verificar ngrok_url.json no Drive
python3 -m colab_infinity.cli drive cat \
  "colab_infinity/pool_state/ngrok_url.json"

# Se "status": "shutdown" → sessão encerrada graciosamente
# Se ausente ou vazio → sessão morreu abruptamente
```

**Procedimento de Recuperação:**

```
Opção A — Automática (recomendada):
  O orquestrador detectará a falha após 3 health checks (≈ 90 segundos)
  e iniciará a troca de conta automaticamente. Aguardar.

Opção B — Manual rápida:
  1. Abrir o notebook colab_server.ipynb na conta ativa (ou próxima disponível)
  2. Executar Células 1 a 6
  3. Aguardar a URL ngrok ser publicada no Drive
  4. O orquestrador detectará a nova URL automaticamente

Opção C — Forçar nova sessão via CLI:
  python3 -m colab_infinity.cli switch-account --auto
```

**Prevenção:**
- Verificar que o keepalive JavaScript da Célula 6 está rodando
- Monitorar `session.estimated_quota_remaining_minutes` no `/health`
- Configurar `quota_warning_minutes: 30` para troca proativa antes da expiração

---

### 5.2 FALHA: GPU Indisponível no Colab

**Sintomas:**
- Célula 1 do notebook imprime `"AVISO: GPU não disponível nesta sessão"`
- `torch.cuda.is_available()` retorna `False`
- O servidor inicia mas a inferência é extremamente lenta (CPU fallback)
- Log do orquestrador: `gpu_not_available` durante health check

**Causa provável:** Google não alocou GPU para esta conta no momento; cota de GPU temporariamente
esgotada para a conta; horário de pico de demanda pelo recurso.

**Diagnóstico:**

```bash
# Verificar se o servidor está em CPU
curl -s http://127.0.0.1:11434/health | python3 -c \
  "import sys,json; d=json.load(sys.stdin); \
   print('Device:', d['runtime']['device'])"
# Se retornar "cpu" → GPU não disponível
```

**Procedimento de Recuperação:**

```
1. No notebook Colab, executar:
   import torch
   print("GPU disponível:", torch.cuda.is_available())
   print("Device:", torch.cuda.get_device_name(0) if torch.cuda.is_available() else "CPU")

2. Se GPU indisponível:
   a. Ir em: Ambiente de execução > Desconectar e excluir ambiente de execução
   b. Aguardar 30 segundos
   c. Ir em: Ambiente de execução > Alterar tipo > GPU T4 > Salvar
   d. Reconectar e executar novamente as células

3. Se ainda sem GPU após reconexão:
   a. A conta atingiu o limite de GPU do período
   b. Trocar para próxima conta do pool:
      python3 -m colab_infinity.cli switch-account --auto
   c. Abrir o notebook na nova conta e verificar GPU lá

4. Se todas as contas sem GPU:
   a. Situação de alto tráfego no Colab; tentar em horários diferentes
   b. Madrugada UTC (2h–6h UTC) geralmente tem maior disponibilidade
   c. Considerar usar CPU temporariamente com modelo menor (Phi-3 Mini)
```

**Prevenção:**
- Distribuir o uso entre as contas do pool de forma equilibrada
- Evitar sessões muito longas em uma única conta (respeitará cooldown de 24h)
- Monitorar horários de alta disponibilidade de GPU para seu fuso horário

---

### 5.3 FALHA: Conta Google Banida ou Suspensa

**Sintomas:**
- Sessão Colab não consegue iniciar para uma conta específica
- Drive API retorna `403 Forbidden` para a conta
- Login na conta retorna mensagem de suspensão do Google
- Log: `account_auth_failed`, `google_account_suspended`

**Diagnóstico:**

```bash
# Verificar qual conta está ativa
python3 -m colab_infinity.cli pool list

# Tentar autenticar a conta suspeita
python3 -m colab_infinity.cli pool test --index 1
# Se retornar "AuthenticationError" → conta possivelmente suspensa
```

**Procedimento de Recuperação:**

```
1. IMEDIATO: Forçar troca para outra conta
   python3 -m colab_infinity.cli switch-account --auto

2. VERIFICAR a conta suspeita:
   a. Tentar login manual em accounts.google.com
   b. Se houver aviso de segurança: seguir o processo de recuperação do Google
   c. Se a conta foi banida por violação de ToS: não tentar recuperar; desativar

3. DESATIVAR a conta no pool:
   python3 -m colab_infinity.cli pool disable --index N

4. SUBSTITUIR com nova conta:
   Ver Passo 4.2 deste Runbook (Adicionar Nova Conta ao Pool)

5. DOCUMENTAR o ocorrido:
   Adicionar nota em accounts.json:
   "notes": "BANIDA em 2025-07-14. Não reusar."
```

**Prevenção:**
- Respeitar os cooldowns de 24h entre reusos de conta
- Não exceder o uso esperado para uma conta individual (> 720 min/sessão é risco)
- Criar contas com padrão de uso humanizado (não sequencial)
- Não usar as mesmas contas para outros fins de automação em paralelo

---

### 5.4 FALHA: Checkpoint Corrompido

**Sintomas:**
- Orquestrador falha ao iniciar com erro `CheckpointDecodeError` ou `KeyError`
- Log: `checkpoint_load_failed`, `schema_version_mismatch`
- `ci_ckpt_*.json` no Drive é inválido ou truncado (arquivo < 100 bytes)

**Diagnóstico:**

```bash
# Listar os checkpoints disponíveis, do mais recente ao mais antigo
python3 -m colab_infinity.cli checkpoint list

# Tentar carregar o mais recente manualmente
python3 -m colab_infinity.cli checkpoint inspect \
  --file ci_ckpt_20250714_160000.json

# Se falhar, tentar o anterior
python3 -m colab_infinity.cli checkpoint inspect \
  --file ci_ckpt_20250714_155500.json
```

**Procedimento de Recuperação:**

```bash
# Opção A: Usar o checkpoint anterior mais recente e válido
python3 -m colab_infinity.orchestrator \
  --config ~/.colab_infinity/colab_infinity_config.yaml \
  --restore-checkpoint ci_ckpt_20250714_155500.json

# Opção B: Ignorar checkpoint e iniciar do zero
# (perderá histórico de contadores, mas o serviço voltará a funcionar)
python3 -m colab_infinity.orchestrator \
  --config ~/.colab_infinity/colab_infinity_config.yaml \
  --ignore-checkpoint

# Opção C: Reconstruir estado mínimo manualmente
python3 -m colab_infinity.cli checkpoint create-minimal \
  --account-index 0 \
  --session-id "ci_sess_recovery_20250714"
```

**Prevenção:**
- O mecanismo de escrita atômica (`.tmp` + `rename`) protege contra truncamento
- Manter os 10 checkpoints mais recentes (`checkpoint.max_files: 10`)
- Validar integridade do JSON no Drive semanalmente:
  ```bash
  python3 -m colab_infinity.cli checkpoint validate-all
  ```

---

### 5.5 FALHA: Ouroboros Runtime Não Consegue Conectar

**Sintomas:**
- Ouroboros lança `ConnectionRefusedError` ou `ECONNREFUSED` ao tentar `http://127.0.0.1:11434`
- Waves falham com erro de LLM no início
- `curl http://127.0.0.1:11434/health` retorna `connection refused`

**Diagnóstico:**

```bash
# 1. Verificar se o proxy local está escutando
netstat -tlnp 2>/dev/null | grep 11434
# ou:
ss -tlnp | grep 11434

# 2. Verificar se o processo do orquestrador está ativo
ps aux | grep "colab_infinity.orchestrator" | grep -v grep

# 3. Verificar o log de erros recente
tail -20 ~/.colab_infinity/logs/orchestrator.log \
  | grep -E '"level":"(error|critical)"'
```

**Procedimento de Recuperação:**

```bash
# Caso 1: Proxy não iniciou (orquestrador em STARTING)
# Aguardar o servidor Colab subir; o proxy inicia automaticamente após ACTIVE

# Caso 2: Orquestrador caiu (processo não existe)
cd ~/colab-infinity
source .venv/bin/activate
python3 -m colab_infinity.orchestrator \
  --config ~/.colab_infinity/colab_infinity_config.yaml &
echo "PID: $!"

# Caso 3: Porta ocupada por outro processo
lsof -i :11434
# Identificar o processo e decidir se é legítimo ou encerrar:
# kill -9 <PID_DO_PROCESSO_BLOQUEADOR>

# Caso 4: Orquestrador em estado HALTED (pool exausto)
python3 -m colab_infinity.cli pool list
# → Ver se há contas em cooldown
# → Se cooldown expirou: orquestrador retomará automaticamente
# → Se todas banidas: adicionar nova conta (ver 4.2)
```

**Para verificar do lado do Ouroboros:**

```bash
# No diretório do Ouroboros Runtime:
cat .env | grep COLAB_INFINITY   # verificar variáveis de ambiente
bun run health-check             # se o Runtime tiver esse script
```

---

### 5.6 FALHA: Drive Armazém Inacessível

**Sintomas:**
- Checkpoints param de ser salvos
- Log: `drive_api_error`, `drive_unreachable`, `HTTP 403` ou `HTTP 500` no Drive API
- `drive_token.json` expirado

**Diagnóstico:**

```bash
# Testar acesso ao Drive
python3 -m colab_infinity.cli drive health

# Verificar validade do token
python3 -c "
import json, os
from google.oauth2.credentials import Credentials
from google.auth.transport.requests import Request

path = os.path.expanduser('~/.colab_infinity/drive_token.json')
with open(path) as f:
    data = json.load(f)
creds = Credentials(**{k: data[k] for k in
  ('token','refresh_token','token_uri','client_id','client_secret','scopes')})
if creds.expired:
    print('Token EXPIRADO. Renovando...')
    creds.refresh(Request())
    with open(path, 'w') as f:
        json.dump({'token': creds.token, 'refresh_token': creds.refresh_token,
                   'token_uri': creds.token_uri, 'client_id': creds.client_id,
                   'client_secret': creds.client_secret,
                   'scopes': list(creds.scopes)}, f)
    print('Token renovado com sucesso.')
else:
    print('Token VÁLIDO.')
"
```

**Procedimento de Recuperação:**

```bash
# Token expirado — renovar automaticamente:
python3 -m colab_infinity.cli drive reauth

# Se reauth falhar (credenciais OAuth revogadas):
# Re-executar o fluxo completo de autorização (ver Setup Guide Passo 4.2.3)
python3 -m colab_infinity.cli drive authorize

# Drive temporariamente indisponível (erro 500 do Google):
# O orquestrador faz retry automático com backoff exponencial
# Verificar status do Google Workspace: https://www.google.com/appsstatus

# Quota de API excedida:
# O orquestrador limita chamadas automaticamente
# Verificar: https://console.cloud.google.com → APIs → Google Drive API → Cotas
```

---

## 6. Recuperação de Desastres

### 6.1 Cenário: Orquestrador Local Completamente Perdido

**Situação:** Máquina local formatada, HD corrompido, ou migração para novo computador.
Todo o estado local foi perdido, mas o Drive armazém está intacto.

**Procedimento:**

```bash
# 1. Instalar dependências na nova máquina (ver Setup Guide, Passo 5)
git clone https://github.com/RenyEnnos/colab-infinity.git
cd colab-infinity
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 2. Recriar a estrutura de configuração
mkdir -p ~/.colab_infinity/logs

# 3. Recriar accounts.json com as contas originais
# (você precisa ter os tokens ngrok anotados separadamente)
nano ~/.colab_infinity/accounts.json

# 4. Recriar drive_credentials.json (baixar novamente do Google Cloud Console)
# Acessar: console.cloud.google.com → projeto → APIs → Credenciais

# 5. Re-autorizar acesso ao Drive
python3 -m colab_infinity.cli drive authorize

# 6. Copiar a configuração do Drive
python3 -m colab_infinity.cli drive fetch-config
# Baixa o colab_infinity_config.yaml do Drive para ~/.colab_infinity/

# 7. Restaurar a partir do último checkpoint
python3 -m colab_infinity.cli checkpoint restore --latest

# 8. Iniciar o orquestrador
python3 -m colab_infinity.orchestrator \
  --config ~/.colab_infinity/colab_infinity_config.yaml
```

### 6.2 Cenário: Todas as Contas do Pool Banidas

**Situação:** O Google suspendeu todas as contas do pool por uso excessivo ou violação de ToS.

**Procedimento:**

```
1. NÃO tentar recuperar as contas banidas imediatamente
   → Processo de apelação ao Google pode levar semanas sem garantia

2. CRIAR novas contas Google para o pool:
   → Seguir Setup Guide, Passo 3 (criação de contas)
   → Usar dispositivos/IPs diferentes dos que causaram o ban
   → Aguardar 48h entre criações de conta

3. CRIAR novas contas ngrok correspondentes

4. ATUALIZAR accounts.json:
   → Manter as contas banidas com status "banned" para referência
   → Adicionar novas contas com índices novos

5. RESTAURAR o estado do pool a partir do último checkpoint válido:
   python3 -m colab_infinity.cli checkpoint restore --latest

6. REVISAR as práticas de uso para evitar novo banimento:
   → Ver seção 5.3 deste Runbook (Prevenção)
   → Considerar aumentar o número de contas no pool (5–8 recomendado)
   → Reduzir switch_threshold_hours para 8h (mais conservador)
```

### 6.3 Cenário: Drive da Conta Armazém Inacessível Permanentemente

**Situação:** Conta armazém banida ou Drive corrompido; estado não acessível.

**Procedimento:**

```bash
# 1. Verificar se há checkpoints locais (orquestrador mantém cópia em memória)
ls ~/.colab_infinity/cache/ 2>/dev/null

# 2. Criar nova conta armazém
# → Seguir Setup Guide, Passo 4

# 3. Iniciar o orquestrador em modo sem checkpoint
python3 -m colab_infinity.orchestrator \
  --config ~/.colab_infinity/colab_infinity_config.yaml \
  --ignore-checkpoint \
  --new-warehouse-folder-id "ID_DA_NOVA_PASTA_ARMAZEM"

# 4. O sistema iniciará do zero com estado limpo
# → Pool state será reconstruído a partir de accounts.json
# → Métricas acumuladas serão perdidas (não-crítico)
```

---

## 7. Comandos Úteis do CLI

### 7.1 Tabela de Referência Rápida

| Comando                                              | Descrição                                      |
|------------------------------------------------------|------------------------------------------------|
| `colab-infinity status`                              | Estado atual do sistema (JSON formatado)       |
| `colab-infinity pool list`                           | Lista contas e status do pool                  |
| `colab-infinity pool test --index N`                 | Testa conectividade de uma conta               |
| `colab-infinity pool test --all`                     | Testa todas as contas do pool                  |
| `colab-infinity pool reload`                         | Recarrega pool sem reiniciar orquestrador      |
| `colab-infinity pool disable --index N`              | Desativa conta do pool                         |
| `colab-infinity pool reset-cooldown --index N`       | Força disponibilidade (uso emergencial)        |
| `colab-infinity switch-account --auto`               | Troca para próxima conta disponível            |
| `colab-infinity switch-account --target-index N`     | Troca para conta específica                    |
| `colab-infinity checkpoint list`                     | Lista checkpoints no Drive                     |
| `colab-infinity checkpoint list --limit 5`           | Lista os 5 mais recentes                       |
| `colab-infinity checkpoint inspect --latest`         | Mostra conteúdo do checkpoint mais recente     |
| `colab-infinity checkpoint restore --latest`         | Restaura estado do checkpoint mais recente     |
| `colab-infinity checkpoint clean --keep 10`          | Remove checkpoints antigos, mantém 10          |
| `colab-infinity checkpoint validate-all`             | Valida integridade de todos os checkpoints     |
| `colab-infinity drive health`                        | Testa acesso ao Drive armazém                  |
| `colab-infinity drive info`                          | Mostra uso de espaço no Drive                  |
| `colab-infinity drive reauth`                        | Renova token OAuth2 do Drive                   |
| `colab-infinity drive authorize`                     | Fluxo completo de autorização OAuth2           |
| `colab-infinity metrics --period 24h`                | Métricas das últimas 24 horas                  |
| `colab-infinity metrics --period 7d`                 | Métricas dos últimos 7 dias                    |
| `colab-infinity validate-config`                     | Valida arquivo de configuração YAML            |
| `colab-infinity init-pool`                           | Inicializa pool_state.json no Drive            |
| `colab-infinity notify-test --event <evt>`           | Testa envio de webhook                         |
| `colab-infinity version`                             | Exibe versão do Colab Infinity                 |

### 7.2 Exemplos de Uso Combinado

```bash
# Fluxo de diagnóstico completo
colab-infinity status && colab-infinity pool list && colab-infinity metrics --period 1h

# Adicionar conta e testar imediatamente
colab-infinity pool reload && colab-infinity pool test --index 3

# Backup manual completo antes de manutenção
curl -s -X POST http://127.0.0.1:11434/v1/checkpoint \
  -H "Content-Type: application/json" \
  -d '{"reason": "pre_maintenance"}' \
  && colab-infinity checkpoint list --limit 1

# Reinicialização limpa (reset completo do estado local)
colab-infinity switch-account --auto \
  && sleep 30 \
  && colab-infinity status
```

---

## 8. Referência de Alertas e Significados

### 8.1 Eventos de Log — Guia de Interpretação

| Evento no Log                    | Nível    | Significado                                             | Ação Necessária          |
|----------------------------------|----------|---------------------------------------------------------|--------------------------|
| `session_started`                | INFO     | Nova sessão Colab ativa com sucesso                     | Nenhuma                  |
| `health_check_ok`                | DEBUG    | Health check bem-sucedido                               | Nenhuma                  |
| `health_check_failed`            | WARNING  | Falha em 1 health check                                 | Monitorar                |
| `health_check_threshold_reached` | ERROR    | 3 falhas consecutivas; troca iniciada                   | Verificar se troca ocorre|
| `checkpoint_saved`               | INFO     | Checkpoint salvo no Drive com sucesso                   | Nenhuma                  |
| `checkpoint_save_failed`         | ERROR    | Falha ao salvar checkpoint                              | Verificar Drive API      |
| `account_switch_started`         | INFO     | Processo de troca de conta iniciado                     | Aguardar conclusão       |
| `account_switch_completed`       | INFO     | Troca de conta bem-sucedida                             | Nenhuma                  |
| `account_switch_failed`          | ERROR    | Troca de conta falhou; tentando próxima                 | Verificar contas         |
| `pool_exhausted`                 | CRITICAL | Todas as contas indisponíveis; HALTED                   | Adicionar contas urgente |
| `drive_api_error`                | ERROR    | Erro na Drive API (HTTP 4xx/5xx)                        | Ver seção 5.6            |
| `drive_unreachable`              | ERROR    | Drive API sem resposta por 3+ tentativas                | Ver seção 5.6            |
| `ngrok_tunnel_expired`           | ERROR    | Túnel ngrok inválido detectado                          | Ver seção 5.1            |
| `gpu_not_available`              | ERROR    | Servidor iniciou sem GPU                                | Ver seção 5.2            |
| `quota_warning`                  | WARNING  | Cota da conta ativa < QUOTA_WARNING_MINUTES             | Troca iminente esperada  |
| `inference_error`                | ERROR    | Exceção durante model.generate()                        | Verificar VRAM           |
| `inference_timeout`              | ERROR    | Inferência excedeu timeout                              | Reduzir max_tokens       |
| `rate_limit_triggered`           | WARNING  | Muitas requisições; rate limiting ativo                 | Agentes enviando rápido  |

### 8.2 Códigos de Saída do CLI

| Código | Significado                                              |
|--------|----------------------------------------------------------|
| `0`    | Sucesso                                                  |
| `1`    | Erro de configuração (YAML inválido, campo ausente)      |
| `2`    | Erro de autenticação (token expirado, credencial inválida)|
| `3`    | Pool exausto (todas as contas indisponíveis)             |
| `4`    | Drive inacessível (timeout, 403, 500)                    |
| `5`    | Checkpoint corrompido ou incompatível                    |
| `10`   | Erro interno inesperado (ver stack trace no log)         |

---

*Documento gerado para o projeto Colab Infinity. Versão 1.0.0 — Julho 2025.*