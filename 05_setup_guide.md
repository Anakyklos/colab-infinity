# Colab Infinity — Guia de Instalação e Configuração (Setup Guide)

**Versão:** 1.0.0
**Data:** 2025-07-14
**Status:** Aprovado
**Referências:** `03_sad.md`, `04_api_spec.md`, `09_integration_guide.md`

---

## Índice

1. [Visão Geral do Processo](#1-visão-geral-do-processo)
2. [Pré-requisitos](#2-pré-requisitos)
3. [Passo 1 — Criação das Contas Google](#3-passo-1--criação-das-contas-google)
4. [Passo 2 — Configuração da Conta Armazém (Google Drive)](#4-passo-2--configuração-da-conta-armazém-google-drive)
5. [Passo 3 — Configuração do ngrok](#5-passo-3--configuração-do-ngrok)
6. [Passo 4 — Preparação do Notebook Colab Base](#6-passo-4--preparação-do-notebook-colab-base)
7. [Passo 5 — Instalação Local das Dependências](#7-passo-5--instalação-local-das-dependências)
8. [Passo 6 — Configuração do Orquestrador](#8-passo-6--configuração-do-orquestrador)
9. [Passo 7 — Configuração do Ouroboros Runtime](#9-passo-7--configuração-do-ouroboros-runtime)
10. [Passo 8 — Configuração do Hermes Agent](#10-passo-8--configuração-do-hermes-agent)
11. [Passo 9 — Configuração do OpenClaw](#11-passo-9--configuração-do-openclaw)
12. [Passo 10 — Teste Manual do Pipeline Completo](#12-passo-10--teste-manual-do-pipeline-completo)
13. [Resolução de Problemas de Setup](#13-resolução-de-problemas-de-setup)

---

## 1. Visão Geral do Processo

O setup do Colab Infinity envolve três ambientes distintos que precisam ser configurados e
conectados entre si:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       AMBIENTES DE SETUP                                │
│                                                                         │
│  [A] Google (Contas + Drive + Colab)                                    │
│      ├── 1 conta armazém (Google Drive central)                         │
│      ├── 2–8 contas operacionais (pool rotativo do Colab)               │
│      └── 1 conta ngrok por conta operacional                            │
│                                                                         │
│  [B] Google Colab (Notebook)                                            │
│      └── colab_server.ipynb configurado e testado                       │
│                                                                         │
│  [C] Máquina Local (Orquestrador + Agentes)                             │
│      ├── orchestrator.py + proxy local :11434                           │
│      ├── Ouroboros Runtime (Bun/TypeScript)                             │
│      ├── Hermes Agent (Python)                                          │
│      └── OpenClaw                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

**Tempo estimado:** 3–5 horas para setup completo com 3 contas.

**Sequência obrigatória:**

```
[1] Criar contas Google
        │
        ▼
[2] Configurar Drive Armazém
        │
        ▼
[3] Configurar ngrok (1 token por conta)
        │
        ▼
[4] Preparar notebook Colab
        │
        ▼
[5] Instalar dependências locais
        │
        ▼
[6] Configurar orquestrador
        │
        ▼
[7–9] Configurar agentes consumidores
        │
        ▼
[10] Testar pipeline completo
```

---

## 2. Pré-requisitos

### 2.1 Hardware Local

| Recurso       | Mínimo                  | Recomendado              |
|---------------|-------------------------|--------------------------|
| Sistema Operacional | Linux (Ubuntu 20.04+) ou macOS 12+ | Ubuntu 22.04 LTS |
| CPU           | 2 núcleos               | 4+ núcleos               |
| RAM           | 4 GB                    | 8 GB                     |
| Armazenamento | 3 GB livres             | 10 GB livres             |
| Internet      | 10 Mbps, latência < 200 ms para Google APIs | 50+ Mbps |

> **Windows:** Use WSL2 com Ubuntu 22.04. A maioria dos scripts foi testada em Linux e macOS.
> Suporte nativo a Windows não é garantido.

### 2.2 Software Local (antes de começar)

| Software | Versão Mínima | Verificação                    | Instalação (Ubuntu)                     |
|----------|---------------|--------------------------------|-----------------------------------------|
| Python   | 3.10          | `python3 --version`            | `sudo apt install python3.10`           |
| pip      | 23.0          | `pip3 --version`               | `pip3 install --upgrade pip`            |
| git      | 2.30          | `git --version`                | `sudo apt install git`                  |
| curl     | 7.68          | `curl --version`               | `sudo apt install curl`                 |
| Bun      | 1.1.0         | `bun --version`                | `curl -fsSL https://bun.sh/install \| bash` |

### 2.3 Recursos de Terceiros Necessários

- **Mínimo 3 contas Google** válidas (com verificação por número de telefone)
- **1 conta ngrok** por conta Google operacional (plano free é suficiente)
- **1 conta Google dedicada como armazém** (pode ser uma das anteriores ou nova)
- Acesso ao [Google Colab](https://colab.research.google.com) em cada conta
- Acesso ao [Google Cloud Console](https://console.cloud.google.com) para ativar a Drive API

---

## 3. Passo 1 — Criação das Contas Google

### 3.1 Estratégia de Criação

Para maximizar a vida útil das contas e minimizar o risco de banimento:

| Regra                                     | Motivo                                                      |
|-------------------------------------------|-------------------------------------------------------------|
| Crie contas com intervalo de 24h entre elas | Evita padrão de criação em massa detectável pelo Google    |
| Use nomes não sequenciais                  | Evita correlação entre contas (`joao.silva`, não `conta1`) |
| Use telefones diferentes para cada conta  | Verificação única por número                                |
| Evite IPs comerciais de VPN na criação    | Maior risco de CAPTCHA e bloqueio preventivo                |
| Ative cada conta manualmente no Colab     | Confirma aceite dos ToS do Colab por conta                  |

### 3.2 Estrutura Mínima do Pool (3 Contas)

| Índice | Papel       | Descrição                                              |
|--------|-------------|--------------------------------------------------------|
| —      | Armazém     | Conta dedicada ao Drive; **não** usada para Colab      |
| 0      | Primária    | Primeira conta operacional; usada por padrão           |
| 1      | Secundária  | Substitui automaticamente a primária quando exaurir    |
| 2      | Terciária   | Terceira opção; cobre o cooldown das anteriores        |

### 3.3 Ativar o Google Colab em Cada Conta Operacional

Para **cada conta do pool** (não a armazém), execute:

1. Acesse [https://colab.research.google.com](https://colab.research.google.com)
2. Aceite os Termos de Serviço
3. Clique em **"Novo notebook"**
4. Execute a célula `print("hello world")` para confirmar que o runtime funciona
5. Vá em **Ambiente de execução > Alterar tipo de ambiente de execução**
6. Selecione **GPU** e salve
7. Execute uma célula simples para confirmar alocação de GPU:

```python
# Célula de verificação de GPU — execute no Colab de cada conta
import subprocess
result = subprocess.run(['nvidia-smi', '--query-gpu=name,memory.total', '--format=csv,noheader'],
                       capture_output=True, text=True)
if result.returncode == 0:
    print("GPU disponível:", result.stdout.strip())
else:
    print("AVISO: GPU não disponível nesta sessão")
```

Saída esperada: `Tesla T4, 15109 MiB` ou similar.

### 3.4 Registrar as Contas no Arquivo de Configuração Local

Crie o diretório e o arquivo de contas:

```bash
mkdir -p ~/.colab_infinity
chmod 700 ~/.colab_infinity

cat > ~/.colab_infinity/accounts.json << 'EOF'
{
  "warehouse": {
    "email": "meu.armazem@gmail.com",
    "drive_folder_name": "colab_infinity",
    "notes": "Conta dedicada ao armazenamento de estado"
  },
  "pool": [
    {
      "index": 0,
      "email": "conta.primaria@gmail.com",
      "ngrok_token": "SUBSTITUA_PELO_TOKEN_NGROK_DA_CONTA_0",
      "role": "primary",
      "status": "available",
      "notes": "Criada em 2025-07-14"
    },
    {
      "index": 1,
      "email": "conta.secundaria@gmail.com",
      "ngrok_token": "SUBSTITUA_PELO_TOKEN_NGROK_DA_CONTA_1",
      "role": "secondary",
      "status": "available",
      "notes": "Criada em 2025-07-15"
    },
    {
      "index": 2,
      "email": "conta.terciaria@gmail.com",
      "ngrok_token": "SUBSTITUA_PELO_TOKEN_NGROK_DA_CONTA_2",
      "role": "tertiary",
      "status": "available",
      "notes": "Criada em 2025-07-16"
    }
  ]
}
EOF

chmod 600 ~/.colab_infinity/accounts.json
```

> ⚠️ **SEGURANÇA:** Este arquivo contém tokens ngrok. Nunca versione-o no git.
> Adicione `~/.colab_infinity/` ao seu `.gitignore` global:
> ```bash
> echo "~/.colab_infinity/" >> ~/.gitignore_global
> git config --global core.excludesFile ~/.gitignore_global
> ```

---

## 4. Passo 2 — Configuração da Conta Armazém (Google Drive)

A conta armazém é o repositório central de estado do Colab Infinity. Ela é acessível tanto pelo
notebook Colab (via montagem do Drive) quanto pelo orquestrador local (via Drive API).

### 4.1 Criar a Estrutura de Pastas no Drive

Na conta armazém, acesse [https://drive.google.com](https://drive.google.com) e crie:

```
Meu Drive/
└── colab_infinity/                    ← Pasta raiz (anote o ID da URL)
    ├── checkpoints/                   ← Criada automaticamente pelo orquestrador
    ├── pool_state/                    ← Criada automaticamente pelo orquestrador
    ├── notebooks/                     ← Salve aqui o colab_server.ipynb
    ├── config/                        ← Configurações compartilhadas (sem credenciais)
    └── logs/                          ← Logs exportados (criado automaticamente)
```

**Obter o ID da pasta raiz:**
1. Abra a pasta `colab_infinity` no Google Drive
2. Copie o ID da URL: `https://drive.google.com/drive/folders/`**`1aBcDeFgHiJkLmNoPqRsTuVwXyZ`**
3. Salve este ID — será usado em `colab_infinity_config.yaml`

### 4.2 Configurar a Google Drive API

O orquestrador local usa OAuth2 para ler e escrever no Drive da conta armazém.

**4.2.1 Habilitar a API:**

1. Acesse [https://console.cloud.google.com](https://console.cloud.google.com) **com a conta armazém**
2. Crie um projeto: clique em **"Selecionar projeto"** → **"Novo Projeto"** → nome: `colab-infinity-storage`
3. No menu lateral: **"APIs e Serviços"** → **"Biblioteca"**
4. Busque **"Google Drive API"** → clique em **"Ativar"**

**4.2.2 Criar credenciais OAuth2:**

1. Vá em **"APIs e Serviços"** → **"Credenciais"**
2. Clique em **"Criar Credenciais"** → **"ID do cliente OAuth"**
3. Se solicitado, configure a tela de consentimento:
   - Tipo de usuário: **Externo**
   - Nome do app: `Colab Infinity Orchestrator`
   - E-mail de suporte: seu e-mail da conta armazém
   - Escopo: `https://www.googleapis.com/auth/drive.file` (acesso apenas a arquivos criados pelo app)
4. Tipo de aplicativo: **"App para computador"**
5. Nome: `colab-infinity-orchestrator`
6. Clique em **"Criar"**
7. Faça o **download do arquivo JSON** (botão de download no lado direito)
8. Salve como `~/.colab_infinity/drive_credentials.json`

```bash
chmod 600 ~/.colab_infinity/drive_credentials.json
```

**4.2.3 Primeira autorização (gera o token OAuth2):**

```bash
# Execute após instalar as dependências Python (Passo 5)
python3 -c "
from google_auth_oauthlib.flow import InstalledAppFlow
from google.auth.transport.requests import Request
import json, os, pickle

SCOPES = ['https://www.googleapis.com/auth/drive.file']
creds_path = os.path.expanduser('~/.colab_infinity/drive_credentials.json')
token_path = os.path.expanduser('~/.colab_infinity/drive_token.json')

flow = InstalledAppFlow.from_client_secrets_file(creds_path, SCOPES)
creds = flow.run_local_server(port=0)

import google.oauth2.credentials
token_data = {
    'token': creds.token,
    'refresh_token': creds.refresh_token,
    'token_uri': creds.token_uri,
    'client_id': creds.client_id,
    'client_secret': creds.client_secret,
    'scopes': creds.scopes
}
with open(token_path, 'w') as f:
    json.dump(token_data, f)
os.chmod(token_path, 0o600)
print('Token salvo em', token_path)
"
```

Um browser será aberto automaticamente. Faça login com a **conta armazém** e conceda permissão.

### 4.3 Verificar Acesso ao Drive

```bash
python3 -c "
import json, os
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

token_path = os.path.expanduser('~/.colab_infinity/drive_token.json')
with open(token_path) as f:
    data = json.load(f)

creds = Credentials(
    token=data['token'],
    refresh_token=data['refresh_token'],
    token_uri=data['token_uri'],
    client_id=data['client_id'],
    client_secret=data['client_secret'],
    scopes=data['scopes']
)

service = build('drive', 'v3', credentials=creds)
about = service.about().get(fields='user,storageQuota').execute()
print('Usuário:', about['user']['emailAddress'])
print('Armazenamento usado:', int(about['storageQuota']['usage']) // (1024**3), 'GB')
print('Limite:', int(about['storageQuota']['limit']) // (1024**3), 'GB')
print('Drive API: OK')
"
```

---

## 5. Passo 3 — Configuração do ngrok

### 5.1 Criar Contas ngrok

Para **cada conta Google do pool operacional** (não a armazém), crie uma conta ngrok:

1. Acesse [https://dashboard.ngrok.com/signup](https://dashboard.ngrok.com/signup)
2. Crie uma conta com um e-mail diferente (pode usar o mesmo e-mail da conta Google ou um dedicado)
3. Após o login, vá em [https://dashboard.ngrok.com/get-started/your-authtoken](https://dashboard.ngrok.com/get-started/your-authtoken)
4. Copie o token de autenticação (formato: `2xYzAbc123...xxxxxxx`)
5. Atualize o campo `ngrok_token` correspondente em `~/.colab_infinity/accounts.json`

### 5.2 Testar o Token ngrok Localmente (Opcional)

```bash
pip3 install pyngrok

python3 -c "
import pyngrok.ngrok as ngrok
import http.server, threading, time

# Servidor HTTP de teste
class Handler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'{\"status\": \"ok\"}')
    def log_message(self, *args): pass

server = http.server.HTTPServer(('', 9999), Handler)
t = threading.Thread(target=server.serve_forever, daemon=True)
t.start()

ngrok.set_auth_token('SEU_TOKEN_NGROK_AQUI')
tunnel = ngrok.connect(9999, 'http')
print('URL pública:', tunnel.public_url)
print('Testando...')

import urllib.request
resp = urllib.request.urlopen(tunnel.public_url)
print('Resposta:', resp.read().decode())
ngrok.disconnect(tunnel.public_url)
ngrok.kill()
print('Teste concluído com sucesso.')
"
```

---

## 6. Passo 4 — Preparação do Notebook Colab Base

O notebook `colab_server.ipynb` é o componente que executa o modelo LLM e expõe a API. Abaixo
estão as **células que devem compor o notebook**, com a explicação do propósito de cada uma.

### 6.1 Abrindo um Novo Notebook

1. Acesse o Colab com a **conta primária** (índice 0)
2. Vá em **Arquivo > Novo notebook**
3. Renomeie para `colab_server`
4. Configure o runtime: **Ambiente de execução > Alterar tipo > GPU**

### 6.2 Célula 1 — Verificação de GPU e Ambiente

**Tipo:** Code | **Objetivo:** Confirmar GPU disponível antes de prosseguir.

```python
# [CÉLULA 1] Verificação de GPU e ambiente
# Execute esta célula primeiro. Se falhar, altere o runtime para GPU e reconecte.
import subprocess, sys, os, time

# Verificar GPU
gpu_result = subprocess.run(
    ['nvidia-smi', '--query-gpu=name,memory.total,driver_version', '--format=csv,noheader'],
    capture_output=True, text=True
)
if gpu_result.returncode != 0:
    raise RuntimeError(
        "GPU não disponível. Vá em: Ambiente de execução > Alterar tipo > GPU T4 > Salvar"
    )

gpu_info = gpu_result.stdout.strip()
print(f"GPU detectada: {gpu_info}")

# Verificar CUDA
cuda_result = subprocess.run(['nvcc', '--version'], capture_output=True, text=True)
print(f"CUDA: {cuda_result.stdout.split('release')[-1].strip()[:20] if cuda_result.returncode == 0 else 'via PyTorch'}")

# Verificar RAM disponível
mem_result = subprocess.run(['free', '-h'], capture_output=True, text=True)
print(f"RAM:\n{mem_result.stdout}")
print("Verificação de GPU: OK")
```

### 6.3 Célula 2 — Instalação de Dependências

**Tipo:** Code | **Objetivo:** Instalar todos os pacotes Python necessários.

```python
# [CÉLULA 2] Instalação de dependências
# Esta célula leva 3-8 minutos na primeira execução.
# Pode ser mais rápida em sessões subsequentes (cache do pip).
import subprocess, sys

packages = [
    "fastapi==0.111.0",
    "uvicorn[standard]==0.30.1",
    "transformers==4.42.3",
    "accelerate==0.31.0",
    "bitsandbytes==0.43.1",
    "pyngrok==7.1.6",
    "pydantic==2.7.3",
    "torch>=2.3.0",
    "sentencepiece==0.2.0",
    "protobuf==4.25.3",
    "huggingface-hub==0.23.4",
    "google-api-python-client==2.134.0",
    "google-auth==2.30.0",
    "google-auth-oauthlib==1.2.0",
]

print(f"Instalando {len(packages)} pacotes...")
result = subprocess.run(
    [sys.executable, "-m", "pip", "install", "-q", "--no-warn-script-location"] + packages,
    capture_output=True, text=True
)
if result.returncode != 0:
    print("STDERR:", result.stderr[-2000:])
    raise RuntimeError("Falha na instalação de dependências")

print("Dependências instaladas com sucesso.")
```

### 6.4 Célula 3 — Parâmetros de Configuração

**Tipo:** Code | **Objetivo:** Centralizar todas as variáveis configuráveis.

```python
# [CÉLULA 3] Parâmetros de configuração
# EDITE ESTA CÉLULA conforme necessário antes de executar.
import os

# ─── Modelo LLM ───────────────────────────────────────────────────────────────
MODEL_ID     = os.environ.get("CI_MODEL_ID",     "mistralai/Mistral-7B-Instruct-v0.2")
QUANTIZATION = os.environ.get("CI_QUANTIZATION", "4bit")   # "4bit" | "8bit" | "none"
MAX_TOKENS   = int(os.environ.get("CI_MAX_TOKENS", "4096"))

# ─── ngrok ────────────────────────────────────────────────────────────────────
# IMPORTANTE: substitua pelo token da conta que está usando
NGROK_TOKEN  = os.environ.get("CI_NGROK_TOKEN", "COLE_SEU_TOKEN_NGROK_AQUI")

# ─── Servidor ─────────────────────────────────────────────────────────────────
SERVER_HOST  = "127.0.0.1"
SERVER_PORT  = 8000
API_KEY      = os.environ.get("CI_API_KEY", None)   # None = sem autenticação

# ─── Sessão ───────────────────────────────────────────────────────────────────
import time as _t
SESSION_ID      = f"ci_sess_{_t.strftime('%Y%m%d_%H%M%S')}"
ACCOUNT_INDEX   = int(os.environ.get("CI_ACCOUNT_INDEX", "0"))

# ─── Google Drive (Conta Armazém) ─────────────────────────────────────────────
WAREHOUSE_FOLDER_ID  = os.environ.get("CI_WAREHOUSE_FOLDER_ID", "COLE_O_ID_DA_PASTA_AQUI")
CHECKPOINT_INTERVAL  = int(os.environ.get("CI_CHECKPOINT_INTERVAL", "300"))  # segundos
QUOTA_WARNING_MIN    = int(os.environ.get("CI_QUOTA_WARNING_MIN",   "30"))   # minutos

print(f"Configuração carregada:")
print(f"  Modelo       : {MODEL_ID} ({QUANTIZATION})")
print(f"  Sessão       : {SESSION_ID}")
print(f"  Conta índice : {ACCOUNT_INDEX}")
print(f"  Autenticação : {'habilitada' if API_KEY else 'desabilitada'}")
print(f"  Checkpoint   : a cada {CHECKPOINT_INTERVAL}s")

if NGROK_TOKEN == "COLE_SEU_TOKEN_NGROK_AQUI":
    print("\n⚠️  AVISO: Token ngrok não configurado! Edite a variável NGROK_TOKEN.")
if WAREHOUSE_FOLDER_ID == "COLE_O_ID_DA_PASTA_AQUI":
    print("⚠️  AVISO: ID da pasta armazém não configurado! Edite WAREHOUSE_FOLDER_ID.")
```

### 6.5 Célula 4 — Carregamento do Modelo LLM

**Tipo:** Code | **Objetivo:** Carregar o modelo com quantização 4-bit na GPU.

```python
# [CÉLULA 4] Carregamento do modelo LLM com quantização 4-bit
# Esta célula leva 3-6 minutos (download do HuggingFace na 1a vez, ~14 GB).
# Nas sessões seguintes o modelo é baixado novamente (VM é efêmera).
# Alternativa: montar o Drive da conta armazém e copiar de lá.
import torch
from transformers import AutoTokenizer, AutoModelForCausalLM, BitsAndBytesConfig

print(f"Carregando modelo: {MODEL_ID} (quantização: {QUANTIZATION})...")
start_time = _t.time()

# Configuração de quantização
bnb_config = None
if QUANTIZATION == "4bit":
    bnb_config = BitsAndBytesConfig(
        load_in_4bit=True,
        bnb_4bit_quant_type="nf4",
        bnb_4bit_compute_dtype=torch.float16,
        bnb_4bit_use_double_quant=True,
    )
elif QUANTIZATION == "8bit":
    bnb_config = BitsAndBytesConfig(load_in_8bit=True)

# Carregamento do tokenizer
print("  Carregando tokenizer...")
tokenizer = AutoTokenizer.from_pretrained(MODEL_ID, use_fast=True)
if tokenizer.pad_token is None:
    tokenizer.pad_token = tokenizer.eos_token

# Carregamento do modelo
print("  Carregando modelo na GPU (pode levar vários minutos)...")
model = AutoModelForCausalLM.from_pretrained(
    MODEL_ID,
    quantization_config=bnb_config,
    device_map="auto",
    torch_dtype=torch.float16 if QUANTIZATION == "none" else None,
    trust_remote_code=False,
)
model.eval()

# Relatório de uso de VRAM
load_time = _t.time() - start_time
vram_used  = torch.cuda.memory_allocated() / 1024**2
vram_total = torch.cuda.get_device_properties(0).total_memory / 1024**2

print(f"\nModelo carregado em {load_time:.1f}s")
print(f"VRAM: {vram_used:.0f} MB / {vram_total:.0f} MB usados")
print(f"GPU:  {torch.cuda.get_device_name(0)}")

MODEL_LOAD_DURATION = load_time
```

### 6.6 Célula 5 — Definição da API FastAPI

**Tipo:** Code | **Objetivo:** Definir os endpoints da API (não inicia o servidor ainda).

```python
# [CÉLULA 5] Definição da API FastAPI
# Define os endpoints; o servidor é iniciado na Célula 6.
import asyncio, json, uuid, time as _time
from fastapi import FastAPI, HTTPException, Depends, Header, Request
from fastapi.responses import StreamingResponse, JSONResponse
from pydantic import BaseModel, Field
from typing import Optional, List, AsyncGenerator

app = FastAPI(title="Colab Infinity API", version="1.0.0")

# ─── Modelos Pydantic ─────────────────────────────────────────────────────────
class ChatMessage(BaseModel):
    role: str
    content: str

class ChatRequest(BaseModel):
    messages: List[ChatMessage]
    model: Optional[str] = None
    temperature: float = Field(default=0.7, ge=0.0, le=2.0)
    top_p: float = Field(default=0.95, gt=0.0, le=1.0)
    top_k: int = Field(default=50, ge=0)
    max_tokens: int = Field(default=512, ge=1, le=4096)
    stream: bool = False
    stop: Optional[List[str]] = None
    presence_penalty: float = Field(default=0.0, ge=-2.0, le=2.0)
    frequency_penalty: float = Field(default=0.0, ge=-2.0, le=2.0)
    n: int = Field(default=1, ge=1, le=1)
    user: Optional[str] = None

# ─── Estado global da sessão ──────────────────────────────────────────────────
_session_start = _time.time()
_requests_served = 0
_tokens_generated = 0

# ─── Endpoints ────────────────────────────────────────────────────────────────
@app.get("/health")
async def health():
    vram_used  = torch.cuda.memory_allocated() / 1024**2
    vram_total = torch.cuda.get_device_properties(0).total_memory / 1024**2
    uptime = int(_time.time() - _session_start)
    # Estimativa de quota: Colab Free ~720 min; descontar uptime
    quota_rem = max(0, 720 - uptime // 60)
    return {
        "status": "ok",
        "timestamp": _time.strftime("%Y-%m-%dT%H:%M:%SZ", _time.gmtime()),
        "uptime_seconds": uptime,
        "model": {
            "id": MODEL_ID, "loaded": True,
            "quantization": QUANTIZATION, "context_length": MAX_TOKENS
        },
        "runtime": {
            "device": "cuda",
            "gpu_name": torch.cuda.get_device_name(0),
            "vram_used_mb": int(vram_used),
            "vram_total_mb": int(vram_total),
            "vram_free_mb": int(vram_total - vram_used),
        },
        "session": {
            "id": SESSION_ID, "account_index": ACCOUNT_INDEX,
            "requests_served": _requests_served,
            "tokens_generated": _tokens_generated,
            "estimated_quota_remaining_minutes": quota_rem,
        },
    }

@app.post("/v1/chat/completions")
async def chat_completions(req: ChatRequest, request: Request):
    global _requests_served, _tokens_generated

    # Verificar autenticação
    if API_KEY:
        auth_header = request.headers.get("Authorization", "")
        if not auth_header.startswith("Bearer ") or auth_header[7:] != API_KEY:
            raise HTTPException(status_code=401, detail={"code": "UNAUTHORIZED",
                "message": "Token de API inválido ou ausente."})

    if req.n > 1:
        raise HTTPException(status_code=400, detail={"code": "UNSUPPORTED_N",
            "message": "Apenas n=1 é suportado."})

    _requests_served += 1
    req_start = _time.time()
    req_id = f"chatcmpl-{SESSION_ID}-{_requests_served:05d}"

    # Formatar mensagens com o chat template do modelo
    messages_dict = [{"role": m.role, "content": m.content} for m in req.messages]
    try:
        input_ids = tokenizer.apply_chat_template(
            messages_dict, return_tensors="pt", add_generation_prompt=True
        ).to(model.device)
    except Exception as e:
        raise HTTPException(status_code=400, detail={"code": "INVALID_MESSAGES_FORMAT",
            "message": f"Erro ao formatar mensagens: {str(e)}"})

    # Parâmetros de geração
    gen_kwargs = dict(
        max_new_tokens=req.max_tokens,
        temperature=max(req.temperature, 0.01),  # evitar temperatura zero
        top_p=req.top_p,
        do_sample=req.temperature > 0.01,
        pad_token_id=tokenizer.eos_token_id,
    )

    if not req.stream:
        # Modo batch
        with torch.no_grad():
            output = model.generate(input_ids, **gen_kwargs)

        generated = output[0][input_ids.shape[-1]:]
        content = tokenizer.decode(generated, skip_special_tokens=True)
        _tokens_generated += len(generated)

        inference_ms = int((_time.time() - req_start) * 1000)
        return {
            "id": req_id, "object": "chat.completion",
            "created": int(_time.time()), "model": MODEL_ID,
            "choices": [{"index": 0,
                "message": {"role": "assistant", "content": content},
                "finish_reason": "stop"}],
            "usage": {"prompt_tokens": input_ids.shape[-1],
                      "completion_tokens": len(generated),
                      "total_tokens": input_ids.shape[-1] + len(generated)},
            "x_colab_infinity": {"session_id": SESSION_ID,
                "account_index": ACCOUNT_INDEX, "inference_ms": inference_ms,
                "quota_remaining_minutes": max(0, 720 - int(_time.time() - _session_start) // 60)}
        }
    else:
        # Modo streaming (SSE)
        async def generate_stream() -> AsyncGenerator[str, None]:
            global _tokens_generated
            # Primeiro chunk com role
            first = {"id": req_id, "object": "chat.completion.chunk",
                     "created": int(_time.time()), "model": MODEL_ID,
                     "choices": [{"index": 0, "delta": {"role": "assistant",
                                  "content": ""}, "finish_reason": None}]}
            yield f"data: {json.dumps(first)}\n\n"

            # Gerar tokens em thread para não bloquear o loop async
            import concurrent.futures
            with concurrent.futures.ThreadPoolExecutor(max_workers=1) as pool:
                with torch.no_grad():
                    future = pool.submit(model.generate, input_ids, **gen_kwargs)
                    output = future.result()

            generated = output[0][input_ids.shape[-1]:]
            _tokens_generated += len(generated)

            # Enviar conteúdo em chunks de 5 tokens
            full_text = tokenizer.decode(generated, skip_special_tokens=True)
            words = full_text.split(" ")
            for i, word in enumerate(words):
                token_chunk = word + (" " if i < len(words) - 1 else "")
                chunk = {"id": req_id, "object": "chat.completion.chunk",
                         "created": int(_time.time()), "model": MODEL_ID,
                         "choices": [{"index": 0,
                             "delta": {"content": token_chunk}, "finish_reason": None}]}
                yield f"data: {json.dumps(chunk)}\n\n"
                await asyncio.sleep(0)  # yield para o event loop

            # Chunk final
            final = {"id": req_id, "object": "chat.completion.chunk",
                     "created": int(_time.time()), "model": MODEL_ID,
                     "choices": [{"index": 0, "delta": {}, "finish_reason": "stop"}]}
            yield f"data: {json.dumps(final)}\n\n"
            yield "data: [DONE]\n\n"

        return StreamingResponse(generate_stream(),
            media_type="text/event-stream",
            headers={"Cache-Control": "no-cache", "Connection": "keep-alive"})

print("API FastAPI definida. Execute a Célula 6 para iniciar o servidor.")
```

### 6.7 Célula 6 — Inicialização do Servidor, ngrok e Keepalive

**Tipo:** Code | **Objetivo:** Iniciar tudo e manter a sessão ativa indefinidamente.

```python
# [CÉLULA 6] Iniciar servidor FastAPI, túnel ngrok, checkpoint e keepalive
# ⚠️ Esta célula roda indefinidamente. Não interrompa manualmente.
# Para encerrar a sessão normalmente, execute a Célula 7 primeiro.
import threading, pyngrok.ngrok as ngrok_lib, uvicorn
from google.colab import drive

# 1. Montar Google Drive da conta armazém
print("1/5 Montando Google Drive...")
drive.mount('/content/drive', force_remount=True)
DRIVE_BASE = f"/content/drive/MyDrive/colab_infinity"
import os
os.makedirs(f"{DRIVE_BASE}/pool_state",  exist_ok=True)
os.makedirs(f"{DRIVE_BASE}/checkpoints", exist_ok=True)
os.makedirs(f"{DRIVE_BASE}/logs",        exist_ok=True)

# 2. Iniciar servidor FastAPI em thread daemon
print("2/5 Iniciando servidor FastAPI...")
def _run_server():
    uvicorn.run(app, host=SERVER_HOST, port=SERVER_PORT,
                log_level="warning", access_log=False)

server_thread = threading.Thread(target=_run_server, daemon=True)
server_thread.start()
_time.sleep(3)  # aguardar o servidor iniciar

# Verificar se está rodando
import urllib.request
try:
    resp = urllib.request.urlopen(f"http://{SERVER_HOST}:{SERVER_PORT}/health")
    print(f"   Servidor OK: {resp.read().decode()[:50]}...")
except Exception as e:
    raise RuntimeError(f"Servidor não iniciou: {e}")

# 3. Criar túnel ngrok
print("3/5 Criando túnel ngrok...")
ngrok_lib.set_auth_token(NGROK_TOKEN)
tunnel = ngrok_lib.connect(SERVER_PORT, "http")
PUBLIC_URL = tunnel.public_url
print(f"   URL pública: {PUBLIC_URL}")

# 4. Salvar URL no Drive para o orquestrador
print("4/5 Publicando URL no Drive...")
import json as _json
ngrok_state = {
    "url": PUBLIC_URL, "session_id": SESSION_ID,
    "account_index": ACCOUNT_INDEX,
    "published_at": _time.strftime("%Y-%m-%dT%H:%M:%SZ", _time.gmtime()),
    "model_id": MODEL_ID, "status": "active"
}
with open(f"{DRIVE_BASE}/pool_state/ngrok_url.json", "w") as f:
    _json.dump(ngrok_state, f, indent=2)
print(f"   ngrok_url.json salvo no Drive.")

# 5. Checkpoint periódico em background
print("5/5 Iniciando checkpoint periódico...")
def _checkpoint_worker():
    global _requests_served, _tokens_generated
    while True:
        _time.sleep(CHECKPOINT_INTERVAL)
        try:
            ckpt_time = _time.strftime("%Y%m%d_%H%M%S")
            checkpoint = {
                "schema_version": "1.1",
                "saved_at": _time.strftime("%Y-%m-%dT%H:%M:%SZ", _time.gmtime()),
                "save_reason": "periodic",
                "session": {"id": SESSION_ID, "account_index": ACCOUNT_INDEX,
                    "ngrok_url": PUBLIC_URL, "model_id": MODEL_ID,
                    "started_at": _time.strftime("%Y-%m-%dT%H:%M:%SZ",
                                                 _time.gmtime(int(_session_start))),
                    "requests_served": _requests_served,
                    "tokens_generated": _tokens_generated},
            }
            ckpt_path = f"{DRIVE_BASE}/checkpoints/ci_ckpt_{ckpt_time}.json"
            # Escrita atômica: temp file + rename
            tmp_path = ckpt_path + ".tmp"
            with open(tmp_path, "w") as f:
                _json.dump(checkpoint, f, indent=2)
            os.rename(tmp_path, ckpt_path)
            print(f"[{_time.strftime('%H:%M:%S')}] Checkpoint salvo: ci_ckpt_{ckpt_time}.json")
        except Exception as e:
            print(f"[WARN] Checkpoint falhou: {e}")

ckpt_thread = threading.Thread(target=_checkpoint_worker, daemon=True)
ckpt_thread.start()

# ─── Loop principal (keepalive + monitor) ────────────────────────────────────
print(f"\n{'='*60}")
print(f"Colab Infinity ativo!")
print(f"  URL pública : {PUBLIC_URL}")
print(f"  Sessão      : {SESSION_ID}")
print(f"  Modelo      : {MODEL_ID} ({QUANTIZATION})")
print(f"  Checkpoint  : a cada {CHECKPOINT_INTERVAL}s")
print(f"{'='*60}")
print("Aguardando requisições... (Ctrl+C para encerrar)\n")

_heartbeat = 0
while True:
    _time.sleep(60)
    _heartbeat += 1
    vram = torch.cuda.memory_allocated() / 1024**2
    print(f"[{_time.strftime('%H:%M:%S')}] ❤ reqs={_requests_served} "
          f"tokens={_tokens_generated} vram={vram:.0f}MB")

    # Reinjetar keepalive JavaScript no Colab a cada 25 minutos
    # (previne desconexão por inatividade do Colab)
    if _heartbeat % 25 == 0:
        from IPython.display import display, Javascript
        display(Javascript("""
            (() => {
                const btn = document.querySelector('colab-run-button') ||
                            document.querySelector('[aria-label="Run cell"]');
                if (btn) btn.click();
                console.log('[Colab Infinity] keepalive tick');
            })();
        """))
```

### 6.8 Célula 7 — Encerramento Gracioso (Opcional)

**Tipo:** Code | **Objetivo:** Salvar checkpoint final e encerrar túnel antes de fechar.

```python
# [CÉLULA 7] Encerramento gracioso
# Execute esta célula ANTES de fechar o notebook para garantir
# que o último checkpoint seja salvo e o túnel ngrok seja fechado.
print("Encerrando Colab Infinity...")

# Salvar checkpoint final
ckpt_time = _time.strftime("%Y%m%d_%H%M%S")
final_checkpoint = {
    "schema_version": "1.1",
    "saved_at": _time.strftime("%Y-%m-%dT%H:%M:%SZ", _time.gmtime()),
    "save_reason": "graceful_shutdown",
    "session": {"id": SESSION_ID, "account_index": ACCOUNT_INDEX,
        "ngrok_url": PUBLIC_URL, "model_id": MODEL_ID,
        "requests_served": _requests_served,
        "tokens_generated": _tokens_generated},
}
import json as _json
ckpt_path = f"{DRIVE_BASE}/checkpoints/ci_ckpt_{ckpt_time}_shutdown.json"
with open(ckpt_path, "w") as f:
    _json.dump(final_checkpoint, f, indent=2)
print(f"Checkpoint final salvo: {ckpt_path}")

# Marcar sessão como encerrada no Drive
ngrok_state["status"] = "shutdown"
ngrok_state["shutdown_at"] = _time.strftime("%Y-%m-%dT%H:%M:%SZ", _time.gmtime())
with open(f"{DRIVE_BASE}/pool_state/ngrok_url.json", "w") as f:
    _json.dump(ngrok_state, f, indent=2)

# Fechar túnel ngrok
ngrok_lib.disconnect(PUBLIC_URL)
ngrok_lib.kill()
print("Túnel ngrok encerrado.")
print("Sessão encerrada com sucesso.")
```

### 6.9 Salvar o Notebook no Drive da Conta Armazém

1. No Colab, vá em **Arquivo > Salvar uma cópia no Drive**
2. Salve na pasta `colab_infinity/notebooks/` da **conta armazém** como `colab_server.ipynb`
3. Anote o **ID do notebook** na URL:
   `https://colab.research.google.com/drive/`**`ID_DO_NOTEBOOK`**
4. Atualize `colab_notebook_id` no arquivo de configuração principal (próximo passo)

---

## 7. Passo 5 — Instalação Local das Dependências

### 7.1 Clonar o Repositório

```bash
git clone https://github.com/RenyEnnos/colab-infinity.git
cd colab-infinity
```

### 7.2 Criar Ambiente Virtual Python

```bash
python3 -m venv .venv
source .venv/bin/activate   # Linux/macOS
# .venv\Scripts\activate    # Windows (WSL2 recomendado)
pip install --upgrade pip
```

### 7.3 Instalar Dependências do Orquestrador

```bash
pip install -r requirements.txt
```

Conteúdo de referência do `requirements.txt`:

```
# Orquestrador e Proxy Local
requests==2.32.3
httpx==0.27.0
fastapi==0.111.0
uvicorn[standard]==0.30.1
pydantic==2.7.3
pyyaml==6.0.1
structlog==24.2.0
tenacity==8.3.0
schedule==1.2.1
click==8.1.7

# Google Drive API
google-auth==2.30.0
google-auth-oauthlib==1.2.0
google-api-python-client==2.134.0

# Utilitários
python-dateutil==2.9.0
```

### 7.4 Criar Estrutura de Configuração

```bash
mkdir -p ~/.colab_infinity/logs
cp config/colab_infinity_config.yaml.example ~/.colab_infinity/colab_infinity_config.yaml
```

---

## 8. Passo 6 — Configuração do Orquestrador

### 8.1 Editar o Arquivo de Configuração Principal

Abra `~/.colab_infinity/colab_infinity_config.yaml` e preencha todos os campos:

```yaml
# ~/.colab_infinity/colab_infinity_config.yaml
# Configuração principal do Colab Infinity

project:
  name: "colab-infinity"
  version: "1.0.0"
  environment: "production"

# Pool de contas Google
pool:
  accounts_file: "~/.colab_infinity/accounts.json"
  min_available_accounts: 2
  switch_threshold_hours: 10       # trocar conta após 10h de uso
  cooldown_hours: 24               # horas antes de reusar conta exaurida

# Servidor Colab
colab:
  notebook_id: "COLE_O_ID_DO_NOTEBOOK_AQUI"   # ID do colab_server.ipynb no Drive
  startup_timeout_seconds: 300     # aguardar até 5 minutos para o servidor subir
  health_check_interval_seconds: 30
  health_check_fail_threshold: 3   # trocas após N falhas consecutivas
  quota_warning_minutes: 30        # sinalizar troca quando restar 30 min de cota
  model_id: "mistralai/Mistral-7B-Instruct-v0.2"
  quantization: "4bit"

# Checkpoint
checkpoint:
  interval_seconds: 300            # salvar a cada 5 minutos
  max_files: 10                    # manter apenas os 10 mais recentes
  drive_folder: "colab_infinity/checkpoints"

# Proxy Local
proxy:
  host: "127.0.0.1"
  port: 11434                      # porta exposta para os agentes
  request_timeout_seconds: 120
  max_retries: 3
  retry_backoff_seconds: 5

# Segurança da API
server:
  api_key: null                    # null = sem autenticação (recomendado para uso local)
  require_auth: false

# Google Drive (Conta Armazém)
drive:
  credentials_file: "~/.colab_infinity/drive_credentials.json"
  token_file: "~/.colab_infinity/drive_token.json"
  warehouse_folder_id: "COLE_O_ID_DA_PASTA_ARMAZEM_AQUI"

# Logging
logging:
  level: "INFO"
  format: "json"
  file: "~/.colab_infinity/logs/orchestrator.log"
  max_size_mb: 100
  backup_count: 5

# Notificações (opcional)
notifications:
  enabled: false
  webhook_url: null
  events:
    - "account_switched"
    - "pool_exhausted"
    - "server_error"
```

### 8.2 Validar a Configuração

```bash
source .venv/bin/activate
python3 -m colab_infinity.cli validate-config \
  --config ~/.colab_infinity/colab_infinity_config.yaml

# Saída esperada:
# ✓ Arquivo de configuração: válido (schema v1.0)
# ✓ accounts.json: 3 contas encontradas (1 armazém + 2 pool)
# ✓ drive_credentials.json: OK
# ✓ drive_token.json: válido
# ✓ Pasta armazém no Drive: acessível
# ✓ Configuração validada com sucesso.
```

### 8.3 Inicializar o Estado do Pool no Drive

```bash
python3 -m colab_infinity.cli init-pool \
  --config ~/.colab_infinity/colab_infinity_config.yaml

# Saída esperada:
# Criando pool_state.json no Drive...
# ✓ Conta 0 (conta.primaria@gmail.com): disponível
# ✓ Conta 1 (conta.secundaria@gmail.com): disponível
# ✓ Conta 2 (conta.terciaria@gmail.com): disponível
# Pool inicializado com 3 contas. Estado salvo no Drive.
```

---

## 9. Passo 7 — Configuração do Ouroboros Runtime

O Ouroboros Runtime precisa ser configurado para usar o proxy local do Colab Infinity como
seu provider de LLM. Isso é feito via variáveis de ambiente no arquivo `.env` do projeto.

### 9.1 Localizar o Arquivo de Ambiente

Na raiz do repositório do Ouroboros Runtime:

```bash
cd /caminho/para/ouroboros-runtime
ls .env* 2>/dev/null || echo "Criar novo arquivo .env"
```

### 9.2 Adicionar as Variáveis do Colab Infinity

Adicione ao arquivo `.env` do Ouroboros Runtime:

```dotenv
# Colab Infinity — Provider de LLM
# Adicione estas variáveis ao .env do Ouroboros Runtime

LLM_PROVIDER=openai_compatible
LLM_BASE_URL=http://127.0.0.1:11434/v1
LLM_API_KEY=dummy
LLM_MODEL=mistralai/Mistral-7B-Instruct-v0.2
LLM_TIMEOUT_MS=120000
LLM_MAX_RETRIES=3
LLM_STREAM=true

# Configurações específicas por agente (opcional — sobrescrevem o padrão acima)
# VISION_AGENT_MODEL=mistralai/Mistral-7B-Instruct-v0.2
# ARCHITECT_AGENT_TEMPERATURE=0.4
# GUARDIAN_AGENT_MAX_TOKENS=2048
```

### 9.3 Verificar a Integração (após o servidor estar rodando)

```bash
# Com o orquestrador e o Colab ativos, execute no diretório do Ouroboros:
curl -s http://127.0.0.1:11434/health \
  | python3 -c "import sys,json; d=json.load(sys.stdin); \
    print('OK' if d['status']=='ok' else 'ERRO:', d.get('model', {}).get('id', '?'))"

# Saída esperada:
# OK: mistralai/Mistral-7B-Instruct-v0.2
```

---

## 10. Passo 8 — Configuração do Hermes Agent

### 10.1 Configuração Interativa (se disponível)

```bash
hermes-agent configure
# Responda:
# ? LLM Provider: openai_compatible
# ? Base URL: http://127.0.0.1:11434/v1
# ? API Key (deixe em branco): [Enter]
# ? Model: mistralai/Mistral-7B-Instruct-v0.2
# ? Enable streaming: Yes
# ? Timeout (seconds): 120
```

### 10.2 Configuração Manual

Edite `~/.config/hermes-agent/config.yaml` (ou o equivalente do Hermes Agent):

```yaml
llm:
  provider: openai_compatible
  base_url: "http://127.0.0.1:11434/v1"
  api_key: null
  model: "mistralai/Mistral-7B-Instruct-v0.2"
  timeout_seconds: 120
  max_retries: 3
  retry_backoff_seconds: 5
  stream: true
```

### 10.3 Teste de Conectividade

```bash
hermes-agent test-connection
# Saída esperada:
# Testando conexão com http://127.0.0.1:11434/v1...
# ✓ Health check: OK (modelo: mistralai/Mistral-7B-Instruct-v0.2)
# ✓ Chat completion: OK (8 tokens em 2.1s)
```

---

## 11. Passo 9 — Configuração do OpenClaw

O OpenClaw é compatível com a API OpenAI. A configuração é feita via variáveis de ambiente:

```bash
# Adicione ao .env ou ao arquivo de configuração do OpenClaw
export OPENAI_API_BASE="http://127.0.0.1:11434/v1"
export OPENAI_API_KEY="dummy"   # qualquer valor se require_auth: false
```

Ou no arquivo de configuração específico do OpenClaw (consultar documentação do projeto):

```yaml
# openclaw_config.yaml (exemplo ilustrativo)
llm:
  base_url: "http://127.0.0.1:11434/v1"
  api_key: "dummy"
  model: "mistralai/Mistral-7B-Instruct-v0.2"
  timeout: 120
```

---

## 12. Passo 10 — Teste Manual do Pipeline Completo

### 12.1 Sequência de Inicialização

**Terminal 1 — Orquestrador:**

```bash
cd ~/colab-infinity
source .venv/bin/activate
python3 -m colab_infinity.orchestrator \
  --config ~/.colab_infinity/colab_infinity_config.yaml
```

O orquestrador:
1. Lê `pool_state.json` do Drive
2. Detecta que nenhuma sessão está ativa
3. Imprime as instruções para abrir o notebook manualmente
4. Inicia o proxy local em `127.0.0.1:11434` em modo de espera
5. Faz polling em `ngrok_url.json` no Drive a cada 10 segundos

**No Google Colab (manualmente):**

1. Abra o notebook `colab_server.ipynb` com a conta primária
2. Execute **todas as células em ordem** (Células 1 a 6)
3. Aguarde a Célula 6 imprimir `"Colab Infinity ativo!"` com a URL pública

**De volta ao Terminal 1:**

O orquestrador detectará automaticamente a nova URL ngrok e iniciará o proxy:

```
2025-07-14T15:00:00Z INFO  orchestrator=ngrok_url_detected url=https://abc123.ngrok-free.app
2025-07-14T15:00:01Z INFO  orchestrator=health_check status=ok model=mistralai/Mistral-7B-Instruct-v0.2
2025-07-14T15:00:01Z INFO  orchestrator=proxy_started host=127.0.0.1 port=11434
2025-07-14T15:00:01Z INFO  orchestrator=state state=ACTIVE session=ci_sess_20250714_150000
```

### 12.2 Testes de Verificação

**Teste 1 — Health Check:**
```bash
curl -s http://127.0.0.1:11434/health | python3 -m json.tool
# Esperado: "status": "ok", "model.loaded": true
```

**Teste 2 — Inferência simples:**
```bash
curl -s http://127.0.0.1:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "Responda apenas a palavra: FUNCIONAL"}],
    "max_tokens": 10,
    "temperature": 0.1
  }' | python3 -c "
import sys,json
r=json.load(sys.stdin)
print('Resposta:', r['choices'][0]['message']['content'])
print('Tokens:', r['usage']['total_tokens'])
print('Latência:', r['x_colab_infinity']['inference_ms'], 'ms')
"
```

**Teste 3 — Streaming:**
```bash
curl -s http://127.0.0.1:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Conte até 5"}],"stream":true,"max_tokens":50}' \
  --no-buffer | grep "^data:" | head -10
```

**Teste 4 — Status da sessão:**
```bash
curl -s http://127.0.0.1:11434/v1/status | python3 -m json.tool
```

### 12.3 Critérios de Validação do Setup

| Verificação                              | Comando                                         | Resultado Esperado                    |
|------------------------------------------|-------------------------------------------------|---------------------------------------|
| Proxy local respondendo                  | `curl http://127.0.0.1:11434/health`           | `"status": "ok"`                      |
| Modelo carregado                         | Campo `model.loaded` no health                  | `true`                                |
| GPU alocada                              | Campo `runtime.device` no health                | `"cuda"`                              |
| Inferência funcionando                   | Teste 2 acima                                   | Texto gerado sem erro                 |
| Checkpoint salvo no Drive                | `ls` na pasta `colab_infinity/checkpoints/`     | Arquivo `ci_ckpt_*.json` presente     |
| ngrok_url.json atualizado                | Verificar no Drive                              | `"status": "active"`                  |
| Ouroboros Runtime conectado              | Curl ao health + configuração .env              | Sem erros na inicialização            |
| Hermes Agent conectado                   | `hermes-agent test-connection`                  | `✓ Chat completion: OK`               |

---

## 13. Resolução de Problemas de Setup

| Problema                                   | Causa Provável                             | Solução                                                           |
|--------------------------------------------|--------------------------------------------|-------------------------------------------------------------------|
| `GPU não disponível` no Colab              | Runtime em CPU                             | Alterar tipo para GPU T4 em: Ambiente de execução > Alterar tipo  |
| Timeout aguardando o servidor (5 min)      | Download lento do modelo (~14 GB)          | Aumentar `startup_timeout_seconds` para 600 no config             |
| `drive_token.json` expirado               | Token OAuth vencido (expira em ~1h sem uso)| Re-executar o script de autorização do Passo 4.2.3                |
| `ngrok_url.json` não aparece no Drive      | Célula 6 não executou completamente        | Verificar output da Célula 6; re-executar se necessário           |
| Proxy retorna `503 SESSION_SWITCHING`      | Orquestrador aguardando sessão ativa       | Comportamento normal; aguardar URL ngrok ser publicada            |
| `NGROK_TOKEN` inválido na Célula 3         | Token copiado errado ou expirado           | Verificar token em [dashboard.ngrok.com](https://dashboard.ngrok.com) |
| Modelo não carrega — `CUDA OOM`            | Modelo muito grande para T4 + quantização  | Usar `"8bit"` ou modelo menor (Phi-3 Mini)                        |
| `pip install` falha na Célula 2            | Versão incompatível ou timeout             | Tentar novamente; ou especificar versão diferente                 |
| Hermes Agent não encontra `config.yaml`    | Caminho incorreto ou config não criada     | Executar `hermes-agent configure` ou criar o arquivo manualmente  |
| Drive API retorna `403 Forbidden`          | Escopo OAuth incorreto ou sem permissão    | Re-executar autorização com escopo `drive.file`                   |
| `bitsandbytes` falha na instalação         | CUDA incompatível                          | Instalar versão específica: `pip install bitsandbytes==0.43.1`    |

---

*Documento gerado para o projeto Colab Infinity. Versão 1.0.0 — Julho 2025.*