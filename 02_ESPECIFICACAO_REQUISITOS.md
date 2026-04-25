# Especificação de Requisitos (SRS)

Este documento detalha os Requisitos Funcionais, Requisitos Não Funcionais e as restrições tecnológicas para o sistema Colab Infinity, garantindo o suporte robusto às necessidades do Ouroboros Runtime e agentes similares.

## 1. Requisitos Funcionais

- **RF01: Carga de Modelo Quantizado**
  O sistema deve carregar um modelo de linguagem pré-treinado no Google Colab, com suporte explícito a quantização 4-bits (`bitsandbytes`), priorizando modelos focados em instrução (ex: Mistral-7B-Instruct-v0.2).

- **RF02: Endpoint Compatível com OpenAI**
  O sistema deve expor um endpoint na rota `POST /v1/chat/completions` capaz de receber e interpretar o payload JSON no padrão da OpenAI (suportando atributos como `model`, `messages`, `temperature`, `max_tokens`, etc.) e retornar a resposta formatada correspondentemente.

- **RF03: Exposição via Cloudflare Tunnels**
  O serviço local do Colab deve ser acessível externamente utilizando um túnel público criado através do `cloudflared` (Cloudflare Tunnels), criando uma URL pública estável e segura para o Ouroboros Runtime.

- **RF04: Monitoramento e Checkpointing**
  O sistema deve detectar o consumo de recursos e, proativamente, salvar checkpoints regulares (ex.: logs de requisição, métricas de cota da GPU) na pasta da **Conta Armazém** no Google Drive.

- **RF05: Rotação Automática de Contas**
  Um script de orquestração local (desenvolvido em Python com automação de navegador via Playwright/Selenium) deve alternar automaticamente a sessão ativa do Google Colab para uma conta de fallback no instante em que for detectado o esgotamento da cota de GPU ou interrupção forçada.

- **RF06: Configuração de Pool de Contas**
  O sistema deve permitir a parametrização de um pool de contas do Google (ex.: `config.yaml` ou variáveis de ambiente local no orquestrador) listando credenciais de, no mínimo, uma conta "armazém" e duas ou mais contas "operacionais".

- **RF07: Integração Simples**
  A integração da API gerada com o Ouroboros Runtime, Hermes Agent e OpenClaw deve requerer, no máximo, a alteração da URL Base da API e (opcionalmente) de uma chave de autorização nas configurações do agente local.

## 2. Requisitos Não Funcionais

- **RNF01: Desempenho (Latência)**
  O sistema deve apresentar um *Time To First Token* (TTFT) inferior a 5 segundos na maioria das requisições geradas pelo Ouroboros. A latência geral da API não deve obstruir as filas assíncronas do daemon do Ouroboros.

- **RNF02: Confiabilidade e Tolerância a Falhas**
  O failover entre contas Google não deve exceder 5 minutos de inatividade visível para a API do agente. O checkpointing do estado deve ocorrer a cada intervalo predefinido ou ao detectar alertas de uso da GPU.

- **RNF03: Segurança**
  O endpoint do túnel pode ser protegido opcionalmente por um "bearer token" fixo simulando uma API Key OpenAI. Nenhuma chave do mundo real (OpenAI/Anthropic) será exposta. A automação local de login via Playwright/Selenium deve rodar sem vazamento de senhas das contas Google em tela ou log puro.

- **RNF04: Escalabilidade Lógica**
  O pool do orquestrador deve suportar uma matriz com `N` contas Google disponíveis, sem limite sistêmico rígido imposto pelo código de automação.

- **RNF05: Portabilidade do Core**
  Toda a lógica de backend rodando no Google Colab deve residir em um único Notebook Jupyter (arquivo `.ipynb`), cujas células contenham os scripts Python, shell e dependências para simplificar as migrações pelo orquestrador.

## 3. Requisitos de Hardware e Software

### 3.1 Infraestrutura (Google Colab)
- Acelerador de Hardware: GPU T4, L4 ou A100 (quando disponível nas contas gratuitas/Pro).
- Armazenamento: Google Drive de 15GB gratuito por conta (foco em Drive de Armazém compartilhado).
- Software no Colab: Python 3.8+, Transformers (Hugging Face), FastAPI, Uvicorn, Bitsandbytes, Cloudflared.

### 3.2 Ambiente Local (Orquestrador)
- Sistema Operacional: Linux, macOS ou WSL2 no Windows.
- Runtime/Linguagens: Python 3.10+ (para o Orquestrador com Playwright).
- Bibliotecas: Playwright/Selenium, PyYAML, Requests.
- Bun (para desenvolvimento nativo se envolver o daemon do Ouroboros localmente).
