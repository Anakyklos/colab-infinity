# Plano de Testes (Test Plan)

**Projeto:** Colab Infinity
**Versão:** 1.0.0
**Status:** Aprovado

Este documento especifica as rotinas de verificação de qualidade (QA) necessárias para atestar que o Colab Infinity orquestra com sucesso as máquinas efêmeras e provê um serviço compatível para inferência via Ngrok.

---

## 1. Casos de Teste End-to-End (E2E)

### T-E2E-01: Automação Completa de Login
- **Objetivo:** Verificar se o Playwright consegue transpor as telas de verificação iniciais do Google.
- **Ação:** O Orquestrador é iniciado com uma sessão zerada. O sistema deve abrir a aba, digitar e-mail, senha, contornar verificações básicas de inatividade e clicar em "Executar tudo" no Colab.
- **Resultado Esperado:** As células rodam, e o indicador visual do Playwright mostra a GPU sendo conectada sem intervenção manual.

### T-E2E-02: Rotação de Conta (Failover)
- **Objetivo:** Garantir a transição suave de URL em caso de limite de cota.
- **Ação:** Com o sistema rodando na conta `worker1`, interromper propositalmente o *Runtime* no Colab manualmente (simulando corte). Enviar requisição pelo cliente local.
- **Resultado Esperado:**
  1. O Orquestrador enfileira a requisição (Retorna `503/Retry-After`).
  2. O log indica falha, mata a sessão antiga e lança o Playwright para iniciar `worker2`.
  3. Ao subir a nova Ngrok URL no Drive, o Orquestrador destrava e atende a requisição em fila na nova conta.
  4. Agente recebe a resposta sem *Crash* (embora com atraso de ~8 min).

### T-E2E-03: Sincronização do Drive Central
- **Objetivo:** Testar a integridade da troca de metadados via Drive.
- **Ação:** O notebook gera uma nova sessão Ngrok.
- **Resultado Esperado:** O arquivo `hermes_infinito/pool_state/ngrok_url.json` deve conter o link e atualizar o campo `timestamp` corretamente. O proxy local reconhece a mudança em menos de 10s.

---

## 2. Testes de API Compatível (Integração)

Os agentes locais que assumem a infraestrutura se comunicam exclusivamente via protocolo OpenAI na porta exposta do Proxy local (`11434`).

### T-API-01: Inferência Básica (Chat Completions)
- **Ação:** Realizar uma chamada HTTP POST de verificação via Python (cliente oficial OpenAI).
```python
from openai import OpenAI
client = OpenAI(base_url="http://127.0.0.1:11434/v1", api_key="dummy")
resp = client.chat.completions.create(
    model="mistralai/Mistral-7B-Instruct-v0.2",
    messages=[{"role": "user", "content": "Diga oi"}]
)
print(resp.choices[0].message.content)
```
- **Resultado Esperado:** Conexão trafegada pela rede Ngrok transparente. Retorna uma mensagem válida de texto contendo "Oi" ou derivado.

### T-API-02: Suporte a Streaming
- **Ação:** Enviar chamada com atributo `stream=True`.
- **Resultado Esperado:** Os pacotes HTTP devem chegar de forma assíncrona (SSE) em blocos de *tokens* delimitados por `data: `, sem que o proxy engasgue as chamadas até o final para devolver tudo.

### T-API-03: Resistência a Contextos Grandes
- **Ação:** Enviar um *prompt* contendo 6.000 *tokens* de texto, solicitando um resumo.
- **Resultado Esperado:** A VRAM aloca adequadamente (não estoura OOM), o Ngrok mantém a conexão ativa apesar dos minutos processando a resposta sem retornar dados imediatamente (Timeouts não são acionados indevidamente no Orquestrador).

---

## 3. Simulação de Testes Operacionais (Caos)

1. **Test-Caos-Ngrok-Limit:** Fornecer dois workers com o *mesmo token* do Ngrok. Ao acionar o segundo, o *notebook* irá falhar. O Orquestrador deve relatar o erro de conflito nos logs e tentar avançar para a próxima conta de forma elegante, e não travar no *loop*.
2. **Test-Caos-Network-Loss:** Desligar a internet da máquina que hospeda o orquestrador durante um instanciamento de Playwright. O *script* deve falhar graciosamente com um *TimeoutException* claro, salvando uma *screenshot* da falha em `~/.colab_infinity/logs/screenshots/`.
