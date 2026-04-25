# Guia de Instalação e Configuração (Setup Guide)

Este documento descreve os passos operacionais para montar a infraestrutura de inferência do zero, desde a criação das contas Google até a injeção da URL pública gerada no Ouroboros.

## 1. Preparação do Google Drive (Conta Armazém)

O sistema exige uma conta central, denominada **Conta Armazém**, que atua como repositório persistente para checkpoints, logs e compartilhamento de pesos (caso deseje fazer cache de LFS) que as contas *Worker* vão acessar.

1. Crie uma conta no Google, ex: `armazem.colabinfinity@gmail.com`.
2. Acesse o Google Drive dessa conta e crie uma pasta chamada `ColabInfinity_Checkpoint`.
3. Clique com o botão direito na pasta -> Compartilhar. Defina a permissão geral como **Qualquer pessoa com o link pode editar** (ou adicione nominalmente os e-mails das contas *Worker* como "Editores"). Isso é fundamental para que scripts rodando nas contas Worker consigam salvar o log antes de morrer.

## 2. Configuração do Pool de Contas (Workers)

Você precisa de, no mínimo, duas contas adicionais (workers) que entrarão em um revezamento de banimentos/restrições de GPU no Colab.

1. Crie (ou selecione) as contas Worker (ex: `worker1.colab@...`, `worker2.colab@...`).
2. Nessas contas, acesse o link de compartilhamento da pasta `ColabInfinity_Checkpoint` e clique no ícone do Drive no topo da tela, ou "Adicionar atalho ao Google Drive" na raiz (My Drive). Isso fará a pasta da conta armazém aparecer ao montar o drive do Worker.

## 3. Preparação do Notebook Jupyter Base

Você subirá um arquivo `backend_inference.ipynb` no Drive de cada Worker ou manterá uma cópia no repositório GitHub e os Workers baixarão de lá. O notebook deve seguir estruturalmente:

**Célula 1: Montagem e Preparo (Shell + Python)**
```python
from google.colab import drive
drive.mount('/content/drive')

!pip install -q -U bitsandbytes transformers accelerate fastapi uvicorn pydantic pyngrok cloudflared
```

**Célula 2: Download/Carga do Modelo (Quantizado)**
```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig

model_id = "mistralai/Mistral-7B-Instruct-v0.2"

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_use_double_quant=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16
)

tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id, quantization_config=bnb_config, device_map="auto")
```

**Célula 3: Servidor FastAPI e Checkpoint Manager**
Definição dos endpoints REST simulando a OpenAI com FastAPI (vide API Spec). Incorporar uso de `threading` ou `asyncio` para salvar um arquivo texto ping/heartbeat em `/content/drive/MyDrive/ColabInfinity_Checkpoint/heartbeat.log`.

**Célula 4: Cloudflared (Abertura do Túnel)**
```bash
!wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64
!chmod +x cloudflared-linux-amd64
# Roda a API em background e depois sobe o túnel
!nohup uvicorn main:app --host 0.0.0.0 --port 8000 &
!./cloudflared-linux-amd64 tunnel --url http://127.0.0.0:8000
```
*(Nota: Extraia a URL que aparecerá no output do terminal e exporte ou logue para acesso do Orquestrador)*

## 4. Configuração do Ambiente Local (Orquestrador)

O orquestrador é a máquina onde o Ouroboros roda (seu PC, servidor bare metal local, etc.). Ele controla o rodízio.

### 4.1 Instalação
1. Certifique-se de ter `Python 3.10+`.
2. Instale dependências: `pip install playwright pyyaml requests`
3. Instale os navegadores: `playwright install chromium`

### 4.2 Configuração (config.yaml)
Crie o arquivo na raiz do repositório local com as credenciais do pool:
```yaml
pool:
  - email: "worker1.colab@gmail.com"
    password: "senha_do_worker_1"
  - email: "worker2.colab@gmail.com"
    password: "senha_do_worker_2"
tunnel_monitor_interval_sec: 30
notebook_url: "https://colab.research.google.com/drive/XYZ_MEU_NOTEBOOK"
```

## 5. Teste e Ativação

1. Inicie o Orquestrador rodando `python orchestrator.py`. Ele usará o Playwright para fazer login silenciosamente (headless) no Colab, abrir o notebook e executar todas as células.
2. Observe o log do console local. Quando o orquestrador ler no output a URL do Cloudflare (ex: `https://meu-tunel.trycloudflare.com`), ele imprimirá a URL base pronta.
3. Use o `curl` (visto no API Spec) contra essa URL para validar o funcionamento do Mistral-7B no Colab via Ouroboros.
