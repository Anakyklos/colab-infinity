# Plano de Testes (Test Plan)

A estratégia de testes assegura que as peças interdependentes — Orquestrador Local, API Colab/Cloudflare e o Ouroboros Runtime — possam lidar com a natureza instável do Colab sem perder contextos críticos.

## Estratégia
O projeto fará uso de testes de integração na ponta (Orquestrador) e testes End-to-End simulando cargas de trabalho de um agente autônomo. A natureza do Colab inviabiliza testes unitários no sentido clássico de CI/CD para o ambiente remoto, priorizando asserções vivas (health-checks).

## Casos de Teste

### CT01: Inicialização do Modelo e Conectividade
- **Objetivo:** Garantir que o Colab carrega com sucesso um LLM 4-bits e abre a comunicação.
- **Passos:** Iniciar o Orquestrador com 1 Worker. Aguardar a geração da URL do Cloudflare. Rodar curl para `/v1/chat/completions` passando mensagem simples ("Oi").
- **Critério de Aceite:** Resposta HTTP 200 contendo uma frase coesa de boas vindas do Mistral-7B em formato OpenAI; Tempo < 30s.

### CT02: Rotação Simulada (Failover de Worker)
- **Objetivo:** Verificar se o Orquestrador detecta falha total e retoma com outro Worker sem estragar o ambiente do agente.
- **Passos:** Iniciar o notebook no Worker 1 e disparar pings na URL a cada 5s. Parar a célula local do Colab via botão "Stop" na UI (simulando corte de GPU). Observar os logs do Orquestrador.
- **Critério de Aceite:** Orquestrador percebe queda no ping (Erro 502/Timeout). Playwright abre o Colab do Worker 2, roda o Notebook novamente. Ping começa a retornar na nova URL do Cloudflare em < 5 minutos.

### CT03: Compartilhamento da Conta Armazém (Estado)
- **Objetivo:** Comprovar leitura e escrita de checkpoints persistidos pelo Colab.
- **Passos:** Worker 1 envia 10 requisições que atualizam `requests_count.json` no drive da Conta Armazém (montado no /content/drive). Falhar Worker 1 e iniciar Worker 2. O Notebook no Worker 2 lê `requests_count.json` na inicialização.
- **Critério de Aceite:** Worker 2 inicia no mesmo contador do Worker 1, provando o compartilhamento GDrive funcional.

### CT04: Estresse e OOM (Out Of Memory)
- **Objetivo:** Validar contenção e limites de max_tokens para a quantização T4 no Colab.
- **Passos:** Utilizar Ouroboros Runtime, disparando 5 requisições concorrentes massivas (ex. histórico de 6000 tokens e pedindo resposta de 1500 tokens).
- **Critério de Aceite:** O servidor não deve reiniciar devido a CUDA Out-of-Memory. Deve retornar Erro 500 ou processar graciosamente usando fila estrita e bloqueio em `threading`.

### CT05: Compatibilidade End-to-End (Ouroboros)
- **Objetivo:** Validar se a API gerada substitui completamente um proxy OpenAI no Ouroboros.
- **Passos:** Injetar a URL no `.env` do Ouroboros Daemon (Bun). Lançar a TUI. Ativar a persona `Ouroboros (Core)` pedindo um "Wave simples: listar arquivos do diretório atual e retornar um JSON com o plano."
- **Critério de Aceite:** O Ouroboros interpreta a resposta como válida (formato JSON Anti-Vibe respeitado sem lixo textual do modelo quantizado), criando a Wave com sucesso na tela da TUI.
