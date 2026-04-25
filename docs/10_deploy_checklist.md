# Checklist de Deploy e Validação

**Projeto:** Colab Infinity
**Versão:** 1.0.0
**Status:** Aprovado

Utilize este checklist antes de iniciar um ciclo longo de execução do Orquestrador (ex: antes de colocar agentes em execução autônoma prolongada). Ele garante que todo o ambiente e os túneis do Ngrok não venham a sofrer colapso precocemente.

---

## 1. Verificações de Infraestrutura de Nuvem

- [ ] **Limpeza do Drive (Armazém):** Arquivos residuais em `colab_infinity/pool_state/ngrok_url.json` foram limpos, evitando leituras acidentais de URLs mortas.
- [ ] **Contas Ngrok (Painel):** Acessou `dashboard.ngrok.com` de cada conta *worker* e confirmou que "Tunnels Online = 0" (Garantindo que não há sessões zombies consumindo o *Free Tier*).
- [ ] **Google Colab (Limpeza):** Garantiu que não há instâncias de GPU ativas travadas nas contas associadas antes de acionar o Orquestrador.

## 2. Verificações do Orquestrador Local

- [ ] **Arquivo accounts.json:** O arquivo possui configuração válida. Nenhum índice (`index`) e nenhum `ngrok_token` estão duplicados. O caminho de leitura está restrito a `chmod 600`.
- [ ] **Virtual Environment:** Ambiente Python devidamente ativado (`source .venv/bin/activate`).
- [ ] **Playwright:** Browsers do Playwright instalados sem restrições (`playwright install`).
- [ ] **Portas de Rede Livres:** Confirmação de que a porta `11434` (ou a configurada como proxy) está livre (`lsof -i :11434` não deve retornar lixo de processos).

## 3. Etapa de Aquecimento (Warm-Up)

- [ ] **Iniciar o Daemon:** Executar o script do orquestrador via terminal mantendo acesso visual aos logs.
- [ ] **Validação do Proxy:** Rodar `curl http://127.0.0.1:11434/v1/status` logo após os logs afirmarem "Tunnel Established". O JSON deve indicar "healthy".
- [ ] **Requisição Zero (Ping de Inferência):** Realizar um envio de um *prompt* minúsculo ("Diga Ok") para atestar que os *tensors* do modelo terminaram de carregar na VRAM do Colab através da rede Ngrok.
- [ ] **Liberação para Consumo:** Caso o Ping responda "Ok", inicializar a rede de agentes clientes.
