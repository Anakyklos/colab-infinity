# Documento de Visão e Escopo (Project Charter)

**Projeto:** Colab Infinity
**Versão:** 1.0.0
**Status:** Aprovado
**Referências:** `02_srs.md`, `03_architecture.md`, `09_integration_guide.md`

---

## 1. Resumo Executivo

O **Colab Infinity** é um módulo de infraestrutura de MLOps projetado para fornecer poder computacional contínuo e gratuito para inferência de Large Language Models (LLMs), transformando o Google Colab em um servidor de LLM resiliente. Em vez de depender exclusivamente de APIs pagas (como OpenAI e Anthropic) durante o desenvolvimento e a operação contínua, o sistema orquestra um *pool* de múltiplas contas Google, rotacionando-as automaticamente para evitar interrupções de cota, com persistência de estado através do Google Drive e túneis públicos via **Ngrok**.

O sistema é focado em alimentar o ecossistema de agentes autônomos locais (como Ouroboros Runtime, Hermes Agent e OpenClaw), garantindo 100% de compatibilidade com a especificação da API da OpenAI.

## 2. Problema e Motivação

O desenvolvimento, testes e execução prolongada de agentes autônomos geram altos custos operacionais quando apoiados em endpoints de inferência pagos. Alternativas gratuitas, como as instâncias T4 do Google Colab, oferecem excelente capacidade computacional, mas impõem severas restrições:
- Interrupções abruptas por expiração de cota de tempo ou inatividade.
- Desconexões de sessão que quebram o fluxo de raciocínio contínuo dos agentes.
- IPs e endpoints dinâmicos que exigem reconfiguração manual constante por parte dos desenvolvedores.

**Motivação:** Criar uma camada de abstração (orquestrador local) que esconda a volatilidade do Colab, fornecendo um endpoint local estável (`127.0.0.1:11434`) com altíssima disponibilidade (via túneis Ngrok), custo financeiro zero e resiliência de dados, democratizando o acesso a LLMs poderosos.

## 3. Objetivos SMART

1. **Redução de Custos Operacionais (Econômico):**
   - Viabilizar 100% de inferência LLM ininterrupta com custo zero de API (OpenAI/Anthropic).
2. **Alta Disponibilidade e Rotação (Resiliência):**
   - Manter *uptime* operacional superior a 95% orquestrando a rotação automática entre 3 a 10 contas Google.
   - O Mean Time To Recovery (MTTR) durante a transição de conta deve ser inferior a 8 minutos.
3. **Persistência de Estado (Confiabilidade):**
   - Preservar o estado do *pool* e a URL ativa via um mecanismo de *checkpoint* assíncrono em uma conta central no Google Drive (Conta Armazém), com atraso de sincronização inferior a 10 segundos.
4. **Proxy Transparente com Ngrok (Acessibilidade):**
   - Garantir que o orquestrador local roteie perfeitamente as chamadas da API da OpenAI para a URL dinâmica do Ngrok gerada pelo notebook Colab em execução, de forma totalmente transparente para o cliente.

## 4. Público-Alvo e Stakeholders

- **Desenvolvedores de Agentes Autônomos:** Criadores de sistemas cognitivos e orquestradores multi-agente que necessitam de recursos computacionais abundantes e ilimitados.
- **Equipe Ouroboros Runtime:** O ecossistema de agentes que usará diretamente o Colab Infinity como infraestrutura *backend* de inferência primária.
- **Engenheiros de MLOps:** Profissionais que gerenciam, implantam e mantêm a infraestrutura de *machine learning* escalável.

## 5. Escopo do Projeto

### 5.1 No Escopo (In-Scope)
- ✅ Carga otimizada de modelos quantizados em 4-bits (ex: Mistral-7B-Instruct-v0.2 usando `bitsandbytes`).
- ✅ Exposição de API local totalmente compatível com os *endpoints* e *payloads* de `POST /v1/chat/completions` da OpenAI.
- ✅ Automação de instanciamento e *keep-alive* de sessões Colab por meio de um Orquestrador Local (Python + Playwright/Selenium).
- ✅ Configuração de rede e *tunneling* seguros através da tecnologia **Ngrok Free** (gerenciamento via `pyngrok`).
- ✅ Sistema de *checkpoint* descentralizado, gravando o estado vital no Google Drive para partilha inter-contas.

### 5.2 Fora do Escopo (Out-of-Scope)
- ❌ Criação de contas Google (processo deve ser feito manualmente pelo usuário, lidando com verificações por SMS).
- ❌ Suporte a múltiplos túneis Ngrok na mesma conta simultaneamente (restrição do *tier free*).
- ❌ Escalonamento horizontal real (múltiplas instâncias Colab executando requisições em paralelo visando aumento de *throughput*).
- ❌ *Finetuning* de LLMs (o sistema é focado puramente em inferência otimizada).

## 6. Métricas de Sucesso

- **Custo:** R$ 0,00 gastos em chamadas de API durante ciclos de desenvolvimento de agentes.
- **Uptime Contínuo:** Sistema ativo por ciclos de mais de 72 horas sem intervenção manual severa, operando sobre o limite diário padrão das contas individuais (~12h cada).
- **Tempo de Rotação (MTTR):** Novo túnel Ngrok estabelecido e acessível pelo Orquestrador em menos de 8 minutos após a queda de uma sessão Colab.
- **Integração:** "Zero alterações de código" (*Drop-in Replacement*) para integrar o Ouroboros Runtime à nova URL da API local.

## 7. Premissas e Restrições

**Premissas:**
- O usuário fornecerá ao menos 3 contas Google previamente criadas e em estado "saudável" (sem bloqueios severos no Colab).
- O usuário possui *tokens* ativos do **Ngrok**, um associado a cada conta Google do *pool*.
- A máquina hospedeira do Orquestrador possui conectividade estável com a internet para manter o *proxy* local funcional.

**Restrições:**
- **Termos de Serviço do Google:** O Colab Infinity visa um modelo de uso justo, porém operações de natureza comercial massiva ou de infraestrutura 24/7 ininterrupta podem, sob os TOS do Google, sofrer mitigação temporal (shadowbans). O sistema assume isso via rotação.
- **Ngrok Free Tier:** O plano gratuito restringe o volume de conexões por minuto (rate limits) e impede túneis múltiplos simultâneos por *token*.

## 8. Riscos de Alto Nível

- **Suspensão de Contas (Shadowban):** Risco de banimento em massa do Colab.
  - *Mitigação:* Distanciamento semântico nas contas, simulação de comportamento humano via Playwright e rotação estrita de IPs/cookies.
- **Rate Limiting do Ngrok:** Interceptação ou queda de conexões por ultrapassar o limite gratuito.
  - *Mitigação:* Rotação associada do token Ngrok por conta e implementação de *backoff* exponencial no *proxy* local.
- **Alterações na Interface do Google Colab:** Quebra dos *scripts* de automação web.
  - *Mitigação:* Uso robusto de seletores XPath/CSS dinâmicos e testes end-to-end contínuos no CI.

---
