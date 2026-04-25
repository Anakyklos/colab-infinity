# Manual de Operação e Runbook

**Projeto:** Colab Infinity
**Versão:** 1.0.0
**Status:** Aprovado

Este Runbook provê diretrizes para a manutenção do ciclo de vida, Troubleshooting contínuo de conectividade e contorno rápido de anomalias na infraestrutura do Colab Infinity baseada em **Ngrok**.

---

## 1. Monitoramento do Orquestrador

O principal indicador de saúde da inferência local é provido pelos *Logs* nativos do Daemon Python.

**Tail dos logs principais:**
```bash
tail ~/.colab_infinity/logs/orchestrator.log
```

**Verificação Rápida de Saúde (Health Check via Proxy Local):**
```bash
curl -s http://127.0.0.1:11434/v1/status | python3 -m json.tool
```

---

## 2. Comportamentos Normais de Rotação (Não requerem ação)

Eventos que são parte do ciclo de vida esperado do Colab e são resolvidos autonomamente pelo Orquestrador:
1. **Desconexão do Ngrok 502/503:** Após ~12 horas de sessão contínua na mesma conta, o Ngrok corta a conexão ou o Colab atinge o limite do *Free Tier*.
   - *O que o sistema faz:* Bloqueia requisições novas (retornando `Retry-After: 60`), inicia Playwright com a `worker2` e reconecta o *tunnel*. O processo dura em média de 6 a 8 minutos.
2. **"Are you still there?" do Colab:** Modal do Google questionando se o usuário está ativo.
   - *O que o sistema faz:* O *script* Playwright injeta cliques dinâmicos na interface Web via *DOM selectors* a cada 10-15 minutos para manter a sessão quente.

---

## 3. Incidentes, Sintomas e Recuperação Rápida (Troubleshooting)

### Incidente 3.1: "Too Many Sessions" no Ngrok
**Sintoma:** Log do orquestrador emite `pyngrok.exception.PyngrokNgrokError: The authtoken you specified does not look like a valid...` ou recusa conexão afirmando `account may not run more than 1 tunnels...`.
**Causa:** Duas contas Colab (ex: `worker1` e `worker2`) tentaram usar o *mesmo* token do Ngrok no `accounts.json`, ou uma sessão antiga do Colab ficou travada em modo *Zombie*.
**Ação Imediata:**
1. Abra o painel [Ngrok Dashboard](https://dashboard.ngrok.com/) e force o encerramento manual da sessão ativa do token conflituoso.
2. Verifique o `~/.colab_infinity/accounts.json` e garanta que cada `index` possua um `ngrok_token` **único**.

### Incidente 3.2: Shadowban e Bloqueio Captcha (Google)
**Sintoma:** O Playwright atinge estado de *TimeoutException* ao tentar logar. Células do Colab não iniciam. `curl` na saúde da rede acusa "Offline por mais de 15 minutos".
**Causa:** O Google marcou o IP do seu host ou os padrões de interação do Bot como suspeitos, ativando tela de verificação de SMS/Captcha no login ou impedindo a ativação da T4.
**Ação Imediata:**
1. Desligue o Orquestrador (`kill $(pgrep -f colab_infinity.orchestrator)`).
2. Abra um navegador anônimo real no seu computador, acesse `colab.research.google.com` usando a conta afetada. Resolva manualmente qualquer Captcha, Tela de SMS ou Alerta Crítico.
3. Altere o IP do seu servidor host local se possível (resetando o modem, por ex), para obter nova fama com a Cloud do Google.
4. Reinicie o Orquestrador.

### Incidente 3.3: Falha na Sincronização do Drive
**Sintoma:** O proxy do orquestrador aponta continuamente para uma URL de Ngrok antiga que já foi desligada, gerando `502 Bad Gateway` constante, mesmo que a conta tenha rotacionado perfeitamente no Google Colab.
**Causa:** A conta Mestre (Armazém) no Google Drive está com atraso massivo na sincronização ou o token de API do Drive expirou para o nó de escrita.
**Ação Imediata:**
1. Verifique manualmente o arquivo em `hermes_infinito/pool_state/ngrok_url.json` via Web UI do Drive.
2. Remova o arquivo se estiver corrompido, force a re-execução da célula final no *notebook* Colab ativo para gerar o novo link de imediato.

---

## 4. Atualização e Manutenção de Rotina

Uma vez a cada ciclo bimestral (60 dias):
- **Verificar Sessões Abertas no Drive:** Apague o cache ou arquivos lixo dentro de `hermes_infinito`.
- **Rotacionar Tokens do Ngrok:** Cancele tokens velhos por segurança no painel web, emita novos e atualize editando o `~/.colab_infinity/accounts.json`.
