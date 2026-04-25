# Colab Infinity ♾️

![Status](https://img.shields.io/badge/Status-Beta-yellow)
![License](https://img.shields.io/badge/License-MIT-blue)
![Ouroboros](https://img.shields.io/badge/Ouroboros-Compatible-green)

Poder computacional contínuo, resiliente e 100% gratuito para o ecossistema de Agentes Autônomos (Ouroboros Runtime, Hermes, OpenClaw). Transforme o Google Colab em um endpoint perpétuo com API compatível com OpenAI.

## Visão Geral

LLMs são caros. Agentes autônomos locais, deixados trabalhando o dia todo, podem custar dezenas de dólares facilmente em inferência de API. O **Colab Infinity** resolve isso por meio de um Orquestrador Local (Python/Playwright) que realiza login em um *pool* de contas gratuitas do Google Colab. Quando a cota de uso de GPU da "Conta A" morre, ele instantaneamente salva checkpoints, loga na "Conta B" e reconstrói o servidor em minutos.

Seus agentes nem percebem a troca.

## Como funciona?
1. Uma **FastAPI** rodando num Jupyter Notebook no Colab hospeda (ex.) o `Mistral-7B-Instruct` em 4-bit quantizado.
2. A API é exposta globalmente através de **Cloudflare Tunnels**.
3. O Ouroboros (sua máquina local) direciona as queries como se fosse a API da OpenAI comum.
4. Ao esgotar o limite do Colab, o script local entra em ação, roda o *failover*, e altera o arquivo local de configuração do Ouroboros injetando a nova URL perfeitamente.

## Quick Start (Em 3 Passos)

**Passo 1:** No seu repositório local, crie seu pool de contas:
```yaml
# config.yaml
pool:
  - email: "conta_teste_1@gmail.com"
    password: "senha1"
  - email: "conta_teste_2@gmail.com"
    password: "senha2"
```

**Passo 2:** Rode o orquestrador na sua máquina base:
```bash
python orchestrator.py
# Aguarde ele abrir o Chromium invisível, upar as dependências no Colab e expor a API.
```

**Passo 3:** Ajuste seus agentes apontando para o console exibido:
```bash
# Copie do terminal do Orquestrador e cole no .env do Ouroboros Runtime!
OPENAI_BASE_URL=https://<seu-tunnel>.trycloudflare.com/v1
```

## Documentação Completa 📚

Para mergulhar fundo na arquitetura, instalação e limitações da infraestrutura, consulte nossa documentação segmentada:

1. [Visão e Escopo (Project Charter)](./01_VISAO_E_ESCOPO.md)
2. [Especificação de Requisitos (SRS)](./02_ESPECIFICACAO_REQUISITOS.md)
3. [Documento de Arquitetura](./03_DOCUMENTO_ARQUITETURA.md)
4. [Especificação da API](./04_ESPECIFICACAO_API.md)
5. [Guia de Instalação e Configuração](./05_GUIA_INSTALACAO_CONFIGURACAO.md)
6. [Manual de Operação](./06_MANUAL_OPERACAO.md)
7. [Plano de Testes](./07_PLANO_TESTES.md)
8. [Análise de Riscos](./08_ANALISE_RISCOS.md)
9. [Guia de Integração (Ouroboros)](./09_GUIA_INTEGRACAO_OUROBOROS.md)
10. [Checklist de Deploy](./11_CHECKLIST_DEPLOY.md)

## Aviso Legal

Este projeto foi construído para propósitos educacionais e de pesquisa com agentes autônomos pessoais. O uso massivo e automatizado de serviços em nuvem gratuitos como o Google Colab pode violar seus Termos de Serviço se feito em escala abusiva para mascarar servidores web de produção. Proceda por sua própria conta e risco e consulte o documento de [Riscos](./08_ANALISE_RISCOS.md).

## Licença
MIT.
