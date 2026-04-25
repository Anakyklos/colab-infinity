# Colab Infinity — Análise de Riscos e Mitigações

**Versão:** 1.0.0
**Data:** 2025-07-14
**Status:** Aprovado
**Referências:** `01_project_charter.md`, `02_srs.md`, `03_sad.md`

---

## Índice

1. [Metodologia de Análise de Riscos](#1-metodologia)
2. [Riscos Técnicos](#2-riscos-técnicos)
3. [Riscos Operacionais](#3-riscos-operacionais)
4. [Riscos Legais e de Conformidade](#4-riscos-legais-e-de-conformidade)
5. [Riscos de Segurança](#5-riscos-de-segurança)
6. [Alternativas Pagas para Escalabilidade](#6-alternativas-pagas)
7. [Matriz de Riscos Consolidada](#7-matriz-consolidada)

---

## 1. Metodologia

### 1.1 Escala de Probabilidade

| Nível | Descrição | Frequência Estimada |
|-------|-----------|---------------------|
| Muito Alta | Quase certo de ocorrer | > 70% de chance em 30 dias |
| Alta | Provável de ocorrer | 40–70% de chance em 30 dias |
| Média | Possível de ocorrer | 15–40% de chance em 30 dias |
| Baixa | Improvável | 5–15% de chance em 30 dias |
| Muito Baixa | Raro | < 5% de chance em 30 dias |

### 1.2 Escala de Impacto

| Nível | Descrição | Efeito no Sistema |
|-------|-----------|-------------------|
| Crítico | Interrupção total do serviço sem recuperação automática | Downtime > 30 min; perda de dados |
| Alto | Degradação severa; recuperação manual necessária | Downtime 8–30 min; sem perda de dados |
| Médio | Degradação parcial; recuperação automática possível | Downtime 0–8 min; desempenho reduzido |
| Baixo | Impacto mínimo; sem interrupção perceptível | Sem downtime; pequena degradação |
| Negligível | Sem impacto prático | Sem efeito operacional |

### 1.3 Nível de Risco (Probabilidade × Impacto)

```
         IMPACTO
         Crítico  Alto    Médio   Baixo   Neglig.
P      ┌────────┬───────┬───────┬───────┬────────┐
R Mto  │  R1    │  R1   │  R2   │  R3   │  R4    │
O Alta │        │       │       │       │        │
B ────-├────────┼───────┼───────┼───────┼────────┤
A Alta │  R1    │  R2   │  R2   │  R3   │  R4    │
B      │        │       │       │       │        │
I ─────┼────────┼───────┼───────┼───────┼────────┤
L Méd  │  R2    │  R2   │  R3   │  R4   │  R5    │
I      │        │       │       │       │        │
D ─────┼────────┼───────┼───────┼───────┼────────┤
A Baix │  R2    │  R3   │  R3   │  R4   │  R5    │
D      │        │       │       │       │        │
E ─────┼────────┼───────┼───────┼───────┼────────┤
  MBaix│  R3    │  R3   │  R4   │  R5   │  R5    │
       └────────┴───────┴───────┴───────┴────────┘
       R1=Crítico R2=Alto R3=Médio R4=Baixo R5=Negligível
```

---

## 2. Riscos Técnicos

### RT-01 — Falha do Túnel ngrok

| Campo | Valor |
|-------|-------|
| **ID** | RT-01 |
| **Categoria** | Técnico |
| **Probabilidade** | Alta |
| **Impacto** | Alto |
| **Nível de Risco** | **R2 — Alto** |

**Descrição:** O túnel ngrok pode falhar por: expiração do processo ngrok na VM Colab, rate limit do plano free (20k req/mês), queda temporária dos servidores de relay do ngrok, ou remoção da sessão Colab pelo Google.

**Impacto Detalhado:** Quando o túnel cai, o proxy local não consegue encaminhar requisições para o servidor Colab. Os agentes consumidores recebem erros de conexão. O orquestrador detecta a falha via health checks e inicia troca de conta.

**Mitigações:**
1. Health check a cada 30s detecta falha em até 90s (3 × 30s threshold)
2. Troca automática de conta cria nova sessão com novo túnel
3. Keepalive JavaScript na Célula 6 previne desconexão por inatividade
4. Monitorar uso mensal de requisições ngrok (alerta a 80% do limite)
5. Configurar `switch_threshold_hours: 8` para trocar proativamente antes do limite de ~8h do ngrok

**Contingência:** Se ngrok descontinuar o plano free, migrar para Cloudflare Tunnel (gratuito com domínio próprio) com mudança mínima no notebook.

---

### RT-02 — GPU Indisponível no Google Colab

| Campo | Valor |
|-------|-------|
| **ID** | RT-02 |
| **Categoria** | Técnico |
| **Probabilidade** | Alta |
| **Impacto** | Alto |
| **Nível de Risco** | **R2 — Alto** |

**Descrição:** O Google Colab gratuito não garante alocação de GPU T4. Em horários de pico ou após uso intenso de uma conta, o Colab pode alocar apenas CPU, tornando a inferência de LLM 50–100× mais lenta ou impraticável.

**Impacto Detalhado:** Sem GPU, o modelo Mistral-7B levaria > 5 minutos por inferência vs. ~5 segundos com T4. O sistema degrada drasticamente mas não falha totalmente se o agente tiver timeout alto.

**Mitigações:**
1. Célula 1 verifica GPU na inicialização e falha com erro descritivo se ausente
2. Health check inclui `runtime.device` — orquestrador detecta `"cpu"` como condição de degradação
3. Pool de contas distribui o risco: se conta 0 não tiver GPU, tentar conta 1
4. Horários de madrugada UTC (2h–6h) têm maior disponibilidade de GPU
5. Manter modelos menores (Phi-3 Mini, ~3GB VRAM) como fallback para CPU

**Contingência:** Se nenhuma conta do pool tiver GPU disponível, o sistema entra em modo degradado ou HALTED. O operador é notificado e pode aguardar 1–2h para disponibilidade ser restaurada.

---

### RT-03 — Esgotamento de VRAM (OOM na GPU T4)

| Campo | Valor |
|-------|-------|
| **ID** | RT-03 |
| **Categoria** | Técnico |
| **Probabilidade** | Média |
| **Impacto** | Médio |
| **Nível de Risco** | **R3 — Médio** |

**Descrição:** A GPU T4 tem 15 GB de VRAM. Modelos quantizados em 4-bit de 7B params usam ~5 GB, mas picos de uso de KV-cache em contextos longos podem causar OOM.

**Impacto Detalhado:** `torch.cuda.OutOfMemoryError` durante `model.generate()` causa `500 MODEL_INFERENCE_ERROR`. A sessão Colab geralmente sobrevive ao OOM com `model.generate()`, mas pode ficar instável.

**Mitigações:**
1. Limitar `max_tokens` a 4096 (configurável via `MAX_TOKENS`)
2. Endpoint retorna `504 INFERENCE_TIMEOUT` em vez de travar se inference demorar > `timeout`
3. Usar quantização 4-bit NF4 como padrão (menor footprint VRAM)
4. Monitorar `vram_free_mb` no health check; alertar se < 10%
5. Implementar `torch.cuda.empty_cache()` entre inferências longas

---

### RT-04 — Checkpoint Corrompido

| Campo | Valor |
|-------|-------|
| **ID** | RT-04 |
| **Categoria** | Técnico |
| **Probabilidade** | Baixa |
| **Impacto** | Médio |
| **Nível de Risco** | **R3 — Médio** |

**Descrição:** O arquivo `ci_ckpt_*.json` no Drive pode ficar corrompido se a escrita for interrompida (queda de rede, encerramento do processo durante `files().update()`).

**Mitigações:**
1. Escrita atômica: arquivo `.tmp` + `os.rename()` (operação atômica no sistema de arquivos)
2. Manter os 10 checkpoints mais recentes; checkpoint corrompido não apaga o anterior
3. Validação de schema JSON ao carregar; `CheckpointCorruptError` tratado com fallback para inicialização limpa
4. Upload de checkpoint é assíncrono e não-bloqueante; falha no upload é não-fatal
5. Checkpoints salvos a cada 300s reduzem a perda máxima de estado a 5 minutos

---

### RT-05 — Mudança na Interface do Google Colab

| Campo | Valor |
|-------|-------|
| **ID** | RT-05 |
| **Categoria** | Técnico |
| **Probabilidade** | Média |
| **Impacto** | Crítico |
| **Nível de Risco** | **R2 — Alto** |

**Descrição:** O Google pode alterar a interface do Colab (HTML, endpoints internos, comportamento de montagem de Drive) sem aviso prévio, quebrando integrações que dependem de automação ou comportamento específico.

**Impacto Detalhado:** Alterações na API de montagem do Drive (`google.colab.drive.mount`) ou na forma como o ngrok é integrado podem exigir atualizações no notebook.

**Mitigações:**
1. Usar apenas APIs públicas e documentadas (`google.colab.drive`, `pyngrok`, `transformers`)
2. Evitar hacks de DOM ou APIs internas não documentadas do Colab
3. Monitorar o changelog do Google Colab e atualizações de pacotes
4. Testes de smoke test semanais detectam quebras rapidamente
5. A arquitetura de notebook com células independentes facilita a atualização de partes específicas

---

### RT-06 — Degradação de Desempenho da GPU T4

| Campo | Valor |
|-------|-------|
| **ID** | RT-06 |
| **Categoria** | Técnico |
| **Probabilidade** | Média |
| **Impacto** | Baixo |
| **Nível de Risco** | **R4 — Baixo** |

**Descrição:** A GPU T4 no Colab é compartilhada. Em períodos de alta demanda, o desempenho pode degradar (throttling de GPU), aumentando latência de inferência.

**Mitigações:**
1. Monitorar latência P95 via `metrics.jsonl`; alertar se > 20s para 50 tokens
2. Trocar de conta se degradação persistir (troca implica nova VM com GPU diferente)
3. Usar horários de menor demanda para tarefas de alta carga

---

## 3. Riscos Operacionais

### RO-01 — Limite de 12 Horas da Sessão Colab

| Campo | Valor |
|-------|-------|
| **ID** | RO-01 |
| **Categoria** | Operacional |
| **Probabilidade** | Muito Alta |
| **Impacto** | Médio |
| **Nível de Risco** | **R2 — Alto** |

**Descrição:** Toda sessão Colab gratuita expira após aproximadamente 12 horas de uso contínuo. Este é o evento de falha mais frequente e esperado do sistema.

**Impacto Detalhado:** Sistema fica indisponível por 4–8 minutos (MTTR) enquanto a troca de conta é processada. Com pool de 3 contas e cooldown de 24h por conta, o sistema pode operar ~36h antes de precisar aguardar reciclagem.

**Mitigações:**
1. **Esta é a falha principal para a qual o sistema foi projetado.** O mecanismo de rotação automática trata este caso como operação normal.
2. Checkpoint salvo antes da expiração (`quota_warning_minutes: 30`)
3. Pool com N contas cobre N × 12h de GPU antes do cooldown
4. Cooldown de 24h por conta garante recuperação natural
5. Recomendação: 5+ contas para cobertura confortável de 60h+ antes de reciclagem

---

### RO-02 — Pool de Contas Exausto

| Campo | Valor |
|-------|-------|
| **ID** | RO-02 |
| **Categoria** | Operacional |
| **Probabilidade** | Média |
| **Impacto** | Crítico |
| **Nível de Risco** | **R2 — Alto** |

**Descrição:** Se todas as contas do pool atingirem o limite diário de GPU ao mesmo tempo, o sistema entra em estado HALTED sem capacidade de retomar automaticamente até que alguma conta saia do cooldown (24h).

**Mitigações:**
1. Pool de 3+ contas com uso distribuído (rotação round-robin) reduz chance de esgotamento simultâneo
2. Sistema entra em HALTED graciosamente; proxy retorna `503 POOL_EXHAUSTED`
3. Retomada automática quando cooldown expira (sem intervenção humana)
4. Webhook envia alerta CRITICAL quando pool é exausto
5. Recomendação operacional: adicionar novas contas antes de esgotar o pool

---

### RO-03 — Tempo de Reciclagem Entre Contas

| Campo | Valor |
|-------|-------|
| **ID** | RO-03 |
| **Categoria** | Operacional |
| **Probabilidade** | Alta |
| **Impacto** | Médio |
| **Nível de Risco** | **R2 — Alto** |

**Descrição:** Contas exauridas ficam em cooldown de 24h. Com pool pequeno (3 contas), pode haver lacunas de disponibilidade se as contas forem exauridas em sequência rápida.

**Mitigações:**
1. Pool de 5+ contas distribui o uso e reduz o risco de lacunas
2. `switch_threshold_hours: 10` troca antes do limite de 12h, preservando margem
3. Operador pode resetar cooldown manualmente em emergências (com cuidado)
4. Sistema aguarda automaticamente sem intervenção humana

---

### RO-04 — Falha no Google Drive API

| Campo | Valor |
|-------|-------|
| **ID** | RO-04 |
| **Categoria** | Operacional |
| **Probabilidade** | Baixa |
| **Impacto** | Médio |
| **Nível de Risco** | **R3 — Médio** |

**Descrição:** A Drive API pode ficar temporariamente indisponível (manutenção do Google, rate limit, token OAuth expirado), impedindo leitura/escrita de checkpoints e `pool_state.json`.

**Mitigações:**
1. Retry com backoff exponencial (até 3 tentativas, backoff de 5s, 10s, 20s)
2. Estado mantido em memória no orquestrador; Drive é backup, não fonte primária de verdade em tempo real
3. Falha em salvar checkpoint é não-fatal; log de ERROR mas operação continua
4. Token OAuth com auto-refresh (google-auth renova automaticamente via refresh_token)
5. Alertar via webhook após 3 falhas consecutivas de escrita no Drive

---

## 4. Riscos Legais e de Conformidade

### RL-01 — Violação dos Termos de Serviço do Google Colab

| Campo | Valor |
|-------|-------|
| **ID** | RL-01 |
| **Categoria** | Legal |
| **Probabilidade** | Média |
| **Impacto** | Crítico |
| **Nível de Risco** | **R2 — Alto** |

**Descrição:** Os Termos de Serviço do Google Colab proíbem explicitamente:
- Uso comercial em larga escala sem autorização
- Automação que simule comportamento humano de forma abusiva
- Uso para mineração de criptomoedas ou processamento em lote não interativo
- Criar múltiplas contas para contornar limites de uso

**Análise do Risco:**
O Colab Infinity opera em uma "zona cinzenta":
- O uso para pesquisa e desenvolvimento pessoal é geralmente permitido
- O uso de múltiplas contas para contornar limites pode ser considerado violação
- A execução de um servidor HTTP contínuo não é o uso previsto pelo Colab

**Consequências Potenciais:**
- Suspensão das contas Google envolvidas
- Banimento temporário ou permanente do Colab
- Em casos extremos (uso comercial em escala), ação legal do Google

**Mitigações:**
1. Usar exclusivamente para desenvolvimento pessoal e pesquisa não-comercial
2. Documentar o propósito do uso como "desenvolvimento de agentes autônomos open-source"
3. Respeitar os cooldowns de 24h entre usos de uma mesma conta
4. Não criar contas exclusivamente para este propósito em escala industrial
5. Ter um plano de migração para APIs pagas quando o projeto ganhar escala comercial
6. **AVISO LEGAL no README:** informar explicitamente que o usuário é responsável pelo cumprimento dos ToS

**Nível de Responsabilidade:** O usuário/operador assume inteira responsabilidade pelo cumprimento dos ToS. O projeto não incentiva violação de ToS.

---

### RL-02 — Licenças dos Modelos LLM

| Campo | Valor |
|-------|-------|
| **ID** | RL-02 |
| **Categoria** | Legal |
| **Probabilidade** | Baixa |
| **Impacto** | Alto |
| **Nível de Risco** | **R3 — Médio** |

**Descrição:** Modelos LLM open-source têm licenças variadas. Uso em contexto comercial ou distribuição não autorizada pode violar as licenças dos modelos.

**Análise por Modelo:**

| Modelo | Licença | Uso Comercial | Uso Pessoal/Research |
|--------|---------|---------------|---------------------|
| Mistral-7B | Apache 2.0 | Permitido | Permitido |
| Llama 3 (Meta) | Meta Llama 3 Community License | Condicional* | Permitido |
| Phi-3 (Microsoft) | MIT | Permitido | Permitido |
| Gemma (Google) | Gemma ToU | Não | Permitido |

*Llama 3: uso comercial permitido para empresas com < 700M usuários mensais ativos.

**Mitigações:**
1. Usar Mistral-7B (Apache 2.0) como modelo padrão recomendado
2. Verificar a licença do modelo antes de usar em contexto comercial
3. O projeto não distribui pesos de modelos; apenas fornece a infraestrutura de serving
4. Documentar claramente que o usuário é responsável pela conformidade com licenças de modelos

---

### RL-03 — Privacidade de Dados

| Campo | Valor |
|-------|-------|
| **ID** | RL-03 |
| **Categoria** | Legal |
| **Probabilidade** | Média |
| **Impacto** | Alto |
| **Nível de Risco** | **R2 — Alto** |

**Descrição:** Prompts enviados ao servidor Colab transitam pela infraestrutura do ngrok (terceiro) e pela infraestrutura do Google (Colab). Dados sensíveis em prompts podem ser expostos ou retidos por esses serviços.

**Mitigações:**
1. **Não enviar dados pessoais, confidenciais ou sob regulação (LGPD, GDPR) através deste sistema**
2. Informar claramente no README que dados transitam por ngrok e Google
3. Para uso com dados sensíveis, substituir ngrok por solução self-hosted (Cloudflare Tunnel + domínio próprio)
4. Não armazenar histórico de conversas no Drive (checkpoints contêm apenas metadados operacionais, não conteúdo)

---

## 5. Riscos de Segurança

### RS-01 — Endpoint ngrok Público Exposto

| Campo | Valor |
|-------|-------|
| **ID** | RS-01 |
| **Categoria** | Segurança |
| **Probabilidade** | Alta |
| **Impacto** | Médio |
| **Nível de Risco** | **R2 — Alto** |

**Descrição:** A URL ngrok é pública e acessível por qualquer pessoa que a conheça. Um atacante poderia usar o servidor LLM para inferência não autorizada, esgotando a cota mais rapidamente.

**Mitigações:**
1. Habilitar autenticação via API Key (`require_auth: true`) se a URL puder ser descoberta
2. URLs ngrok aleatórias por sessão reduzem a exposição (não são previsíveis)
3. Rate limiting (`2 req/s`) limita o abuso mesmo sem autenticação
4. Monitorar `requests_served` por sessão; pico incomum pode indicar uso não autorizado
5. No plano ngrok pago, IP allowlist pode restringir o acesso

---

### RS-02 — Vazamento de Credenciais

| Campo | Valor |
|-------|-------|
| **ID** | RS-02 |
| **Categoria** | Segurança |
| **Probabilidade** | Baixa |
| **Impacto** | Crítico |
| **Nível de Risco** | **R2 — Alto** |

**Descrição:** Tokens ngrok ou tokens OAuth do Drive, se expostos (commit acidental no git, log com credencial em texto plano), permitem acesso não autorizado às contas Google.

**Mitigações:**
1. `~/.colab_infinity/accounts.json` e `drive_token.json` com permissão `chmod 600`
2. `.gitignore` inclui `~/.colab_infinity/`, `accounts.json`, `*.json` sensíveis
3. Logs filtrados (`SensitiveDataFilter`) para remover tokens antes de gravar
4. Nunca hardcodar tokens nas células do notebook Colab
5. Usar Colab Secrets ou variáveis de ambiente para tokens sensíveis no notebook
6. Rotação periódica dos tokens ngrok recomendada (mensal)
7. Pre-commit hook para detectar tokens acidentalmente commitados

---

### RS-03 — Acesso Não Autorizado ao Proxy Local

| Campo | Valor |
|-------|-------|
| **ID** | RS-03 |
| **Categoria** | Segurança |
| **Probabilidade** | Muito Baixa |
| **Impacto** | Médio |
| **Nível de Risco** | **R4 — Baixo** |

**Descrição:** Se o proxy local (`127.0.0.1:11434`) for acessível por outros processos na mesma máquina, usuários locais não autorizados poderiam usar o LLM.

**Mitigações:**
1. Proxy escuta exclusivamente em `127.0.0.1` (loopback), nunca em `0.0.0.0`
2. Validação de bind no início do orquestrador; falha se tentar bind em `0.0.0.0`
3. Autenticação opcional via API Key para ambientes multi-usuário
4. Não expor a porta `11434` em firewall externo

---

## 6. Alternativas Pagas para Escalabilidade

Quando o projeto atingir escala que justifique custo, as seguintes alternativas devem ser avaliadas:

### 6.1 Google Colab Pro / Pro+

| Plano | Custo | Benefício |
|-------|-------|-----------|
| Colab Pro | ~USD 10/mês | Sessões mais longas (~24h), mais VRAM (A100 disponível) |
| Colab Pro+ | ~USD 50/mês | Prioridade na alocação de GPU, sessões em background |

**Quando migrar:** Quando o projeto exigir sessões > 12h sem rotação ou quando a disponibilidade de GPU for crítica.

### 6.2 APIs de Inferência Gerenciadas

| Provider | Modelos | Custo Estimado | Latência |
|----------|---------|----------------|----------|
| OpenAI | GPT-4o, GPT-4o-mini | USD 0.15–2.50/1M tokens | 0.5–2s |
| Anthropic | Claude 3.5 Sonnet | USD 3.00/1M tokens input | 1–3s |
| Groq | Llama 3, Mixtral | USD 0.05–0.27/1M tokens | 0.1–0.5s (muito rápido) |
| Together AI | Llama 3, Mistral | USD 0.20/1M tokens | 0.5–2s |

**Quando migrar:** Quando a carga de tokens superar 1M tokens/dia ou quando SLA de latência < 2s for necessário.

**Como migrar:** A compatibilidade com a API OpenAI permite migração com apenas 2 linhas de configuração:

```dotenv
# De:
LLM_BASE_URL=http://127.0.0.1:11434/v1
LLM_API_KEY=dummy

# Para (Groq, por exemplo):
LLM_BASE_URL=https://api.groq.com/openai/v1
LLM_API_KEY=gsk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 6.3 Infraestrutura Cloud Própria

| Opção | Custo/hora | GPU | Quando Usar |
|-------|-----------|-----|-------------|
| GCP Cloud Run (GPU) | ~USD 2.50/h | T4 | Baixa latência + SLA |
| AWS SageMaker | ~USD 0.75–5/h | T4/A10G | Integração AWS |
| Modal.com | ~USD 0.10/h (T4) | T4 | Serverless, paga por uso |
| RunPod | ~USD 0.17/h | RTX 3090 | Custo-benefício |

**Quando migrar:** Quando o Colab Infinity se tornar um gargalo operacional ou quando dados sensíveis exigirem isolamento total.

---

## 7. Matriz Consolidada

| ID | Risco | Categoria | Probabilidade | Impacto | Nível | Proprietário | Status |
|----|-------|-----------|---------------|---------|-------|--------------|--------|
| RT-01 | Falha do túnel ngrok | Técnico | Alta | Alto | **R2** | Operador | Mitigado |
| RT-02 | GPU indisponível | Técnico | Alta | Alto | **R2** | Colab/Google | Parcialmente mitigado |
| RT-03 | OOM na GPU T4 | Técnico | Média | Médio | **R3** | Desenvolvedor | Mitigado |
| RT-04 | Checkpoint corrompido | Técnico | Baixa | Médio | **R3** | Desenvolvedor | Mitigado |
| RT-05 | Mudança na interface Colab | Técnico | Média | Crítico | **R2** | Google | Aceito |
| RT-06 | Degradação de desempenho GPU | Técnico | Média | Baixo | **R4** | Google | Aceito |
| RO-01 | Limite de 12h por sessão | Operacional | Muito Alta | Médio | **R2** | Google (design) | Mitigado |
| RO-02 | Pool de contas exausto | Operacional | Média | Crítico | **R2** | Operador | Mitigado |
| RO-03 | Tempo de reciclagem de contas | Operacional | Alta | Médio | **R2** | Design | Aceito |
| RO-04 | Falha na Drive API | Operacional | Baixa | Médio | **R3** | Google | Mitigado |
| RL-01 | Violação de ToS do Colab | Legal | Média | Crítico | **R2** | Operador | Aceito (risco do operador) |
| RL-02 | Licença dos modelos LLM | Legal | Baixa | Alto | **R3** | Operador | Mitigado |
| RL-03 | Privacidade de dados | Legal | Média | Alto | **R2** | Operador | Mitigado (instrução) |
| RS-01 | Endpoint ngrok público | Segurança | Alta | Médio | **R2** | Operador | Mitigado |
| RS-02 | Vazamento de credenciais | Segurança | Baixa | Crítico | **R2** | Operador | Mitigado |
| RS-03 | Acesso ao proxy local | Segurança | Muito Baixa | Médio | **R4** | Desenvolvedor | Mitigado |

### 7.1 Riscos Residuais Aceitos

Os seguintes riscos foram identificados como **aceitos** pelo projeto — as mitigações disponíveis são insuficientes para eliminá-los, mas o benefício do projeto supera os riscos:

1. **RT-05 (Mudança na interface Colab):** Risco inerente ao uso de serviço de terceiro. Mitigação: atualização rápida do notebook quando necessário.
2. **RL-01 (Violação de ToS):** Risco transferido ao operador. Mitigação: documentação clara e uso moderado.
3. **RO-03 (Tempo de reciclagem):** Risco inerente ao design com recursos gratuitos limitados. Mitigação: pool maior.

---

*Documento gerado para o projeto Colab Infinity. Versão 1.0.0 — Julho 2025.*
