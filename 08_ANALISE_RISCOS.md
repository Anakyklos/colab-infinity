# Análise de Riscos e Mitigações

Ao ancorar uma arquitetura corporativa e de agentes em instâncias gratuitas efêmeras, inerentes fragilidades são introduzidas. Abaixo avaliamos a probabilidade e o plano de ação.

## Riscos Operacionais e Técnicos

### 1. Limites de Alocação de GPU Dinâmica
**Risco (Alto):** Durante horários de pico (horário comercial global), o Google Colab pode simplesmente recusar a entrega da GPU T4, resultando na execução em CPU que é inviável para LLMs (latência de >60s/token).
**Mitigação:**
- O código do Notebook faz `assert torch.cuda.is_available()`. Se falhar no início, o processo desliga, sinaliza ao Orquestrador e o Playwright tenta rotacionar até o final do pool.
- Se nenhuma Worker tiver GPU, o orquestrador bloqueia por 30 minutos e entra em *Sleep Loop* até recurso disponível.

### 2. Duração Limitada (Cota de 12 horas)
**Risco (Médio):** Uma conta Worker atinge a cota dura diária de 12 horas seguidas (ou cota invisível de uso computacional) e cai subitamente.
**Mitigação:** Como já projetado, a Arquitetura com falha de 5 minutos, Playwright e Cloudflare lida transparentemente com isso, exigindo apenas que os clientes (Ouroboros) rodem retries infinitos configurados em requisições de rede.

### 3. Variação de Endereço URL (Cloudflare)
**Risco (Baixo):** Mudança de sessão Colab sempre altera o subdomínio da URL (`xyz-cloudflare.trycloudflare.com`).
**Mitigação:** O Ouroboros Runtime tem suporte nativo para leitura de variáveis ambientais "hot-reloaded". O Orquestrador escreve um `.env` local recém-atualizado assim que o URL surge. O Daemon e os agentes re-lêem o `.env` ao sofrer timeout da velha porta antes do retry.

## Riscos Legais / Viabilidade

### Violação dos Termos de Serviço do Colab
**Risco (Crítico):** Os ToS do Google Colab Gratuito afirmam explicitamente restrições sobre a "hospedagem contínua de servidores de inferência Web (APIs)" e automação excessiva burlando limites de uso. Se flagrado, a Google pode banir as contas Google envolvidas do serviço ou fechar permanentemente as credenciais do Drive (Conta Armazém incluída).
**Mitigações:**
1. Manter a **Conta Armazém** desvinculada de rotinas pesadas de automação. Usar contas "Worker" secundárias e descartáveis puramente feitas para rodar notebooks, para não perder dados reais (checkpoint) se banidas.
2. Usar o ambiente apenas para pesquisa pessoal ou uso não escalável.
3. Não exceder os limits de ping do Ngrok/Cloudflare (não fazer polling abusivo a cada milissegundo), preferindo heartbeats a cada 30-60s.

## Alternativas Futuras (Escalabilidade Paga)

Quando o projeto Multiagente do Ouroboros necessitar de operação `24/7` livre das dores da rotação e confiável além de provas-de-conceito:
1. **Google Colab Pro:** A assinatura básica elimina a interrupção súbita se você rodar scripts moderados de inferência por ~10$.
2. **RunPod / Vast.ai:** Fornece GPUs RTX 3090 / 4090 na faixa de US$ 0.20-0.30 a hora (sem interrupções randômicas). A API do FastAPI feita no notebook Colab é diretamente portátil para um Docker num serviço de GPU Cloud comum.
