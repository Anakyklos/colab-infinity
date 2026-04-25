# Guia de Instalação e Configuração

**Projeto:** Colab Infinity
**Versão:** 1.0.0
**Status:** Aprovado
**Referências:** `03_architecture.md`, `README.md`

Este documento é voltado para Engenheiros de DevOps, MLOps e Desenvolvedores implementando o Orquestrador do Colab Infinity localmente. Siga os passos cautelosamente, especialmente as instruções relacionadas ao Ngrok e contorno de permissões do Google.

---

## 1. Pré-Requisitos

### 1.1 Conta Google Armazém (Master State)
2. Na raiz do Drive desta conta, crie a pasta: `colab_infinity/pool_state`.
2. Na raiz do Drive desta conta, crie a pasta: `hermes_infinito/pool_state`.
3. Essa é a conta onde os metadados (como o `ngrok_url.json`) serão persistidos. Compartilhe acesso de Edição desta pasta com todas as contas de *Workers*.

### 1.2 Contas Workers (Nós de Processamento)
1. Crie entre 3 a 10 contas Google genéricas (requer verificação por telefone).
2. Não utilize essas contas para fins pessoais, elas atuarão como nós efêmeros.
3. Para cada conta, acesse `colab.research.google.com` pelo menos uma vez e aceite os termos.

### 1.3 Ngrok Tokens (Rede)
1. Crie uma conta no portal [Ngrok](https://dashboard.ngrok.com/) para cada Conta Worker. (Isso impede limites de *Free Tier* de sobreposição).
2. Guarde todos os *Auth Tokens* (ex: `2a9Z8xW...`).

### 1.4 Ambiente Local
- Python 3.10 ou superior.
- Node.js 18+ (caso use algum utilitário JS do Playwright).
- O navegador Chromium gerenciado pelo Playwright.

---

## 2. Instalação do Orquestrador

```bash
# 1. Clone o repositório
git clone https://github.com/RenyEnnos/colab-infinity.git
cd colab-infinity

# 2. Configure o ambiente Virtual do Python
python3 -m venv .venv
source .venv/bin/activate

# 3. Instale as dependências
pip install -r requirements.txt

# 4. Instale os navegadores do Playwright (Essencial para o bypass do Login)
playwright install
```

---

## 3. Configurações de Segurança e Mapeamento

O Orquestrador necessita de dois arquivos cruciais de segurança: o mapeamento de contas (`accounts.json`) e a base de configuração YAML.

### 3.1 accounts.json
Crie e edite o arquivo `~/.colab_infinity/accounts.json` com o seu editor favorito, contendo o seguinte formato:

```json
[
  {
    "index": 1,
    "email": "worker1@gmail.com",
    "password": "senhaSegura1!",
    "ngrok_token": "token_ngrok_worker_1",
    "role": "worker"
  },
  {
    "index": 2,
    "email": "worker2@gmail.com",
    "password": "senhaSegura2!",
    "ngrok_token": "token_ngrok_worker_2",
    "role": "worker"
  }
]
```

### 3.2 Segurança OBRIGATÓRIA dos Arquivos
Garanta que os arquivos de conta não fiquem expostos na máquina host.

```bash
mkdir -p ~/.colab_infinity
chmod 600 ~/.colab_infinity/accounts.json
```

---

## 4. Subindo o Colab Inicial (Notebook)

O repositório contém um arquivo na pasta de notebooks: `notebooks/colab_server.ipynb`.

1. Faça o *upload* manual deste `.ipynb` para o Drive de cada conta *Worker* (ou gerencie via Github e carregue dinamicamente).
2. O *notebook* já está parametrizado para ler variáveis de ambiente como `$NGROK_TOKEN` e `$HUGGINGFACE_TOKEN`. Estas não devem ser digitadas no *notebook*. O Orquestrador se encarrega de injetá-las através do console dinamicamente via Playwright.

---

## 5. Inicialização Contínua do Serviço

Após a instalação e o *setup* da configuração, basta iniciar o ciclo de vida e o Proxy via:

```bash
python3 -m colab_infinity.orchestrator --config ~/.colab_infinity/colab_infinity_config.yaml
```

**O que vai ocorrer nesse momento:**
1. O servidor local web sobe em `http://127.0.0.1:11434`.
2. Uma aba imperceptível (*Headless*) do navegador abrirá.
3. Fará login no Gmail `worker1`.
4. Abrirá o Colab de `worker1`, injetará o token Ngrok de `worker1`.
5. Acionará a T4, baixará os modelos e abrirá o túnel.
6. A URL surgirá via Drive na pasta `hermes_infinito`.
7. O servidor se destrava e os agentes podem começar a postar *prompts*.
