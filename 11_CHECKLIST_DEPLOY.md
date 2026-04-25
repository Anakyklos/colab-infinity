# Checklist de Deploy

Antes de ligar o daemon persistente do Ouroboros Runtime acoplado ao Colab Infinity, valide os seguintes itens para garantir estabilidade operacional. Se qualquer item estiver não-conforme, a falha do agente é iminente no médio prazo.

### 1. Infraestrutura Google (Contas e Armazém)
- [ ] A Conta Google "Armazém" foi criada e não está no pool de alternância?
- [ ] O Drive da Conta Armazém contém a pasta principal (`ColabInfinity_Checkpoint`)?
- [ ] As permissões de compartilhamento da pasta estão como "Editor" para todos no Pool?
- [ ] As contas Worker (Mínimo 2) acessaram o link e "Adicionaram atalho ao Drive"?

### 2. Notebook e Orquestrador
- [ ] O script de automação (`orchestrator.py`) foi preenchido com e-mail/senha das Worker accounts no `config.yaml`?
- [ ] O `Playwright` e o navegador Chromium base foram instalados na máquina do Orquestrador?
- [ ] O arquivo `backend_inference.ipynb` contém a linha explícita do `BitsAndBytesConfig` definindo 4-bits?
- [ ] A célula de instalação do Cloudflare (`cloudflared-linux-amd64`) está baixando da release oficial?

### 3. Integração na Camada API
- [ ] Foi realizado um `curl` manual batendo no `/v1/chat/completions` simulado com resultado "200 OK"?
- [ ] O arquivo de variáveis `.env` (ou outro método) do Ouroboros Runtime está apontando para a URL dinâmica do Cloudflare?
- [ ] O `OPENAI_API_KEY` do Ouroboros possui a chave sintética de Bearer Token que o Colab espera (ou foi desligada de ambos os lados)?

### 4. Camada Agente / Ouroboros
- [ ] A lógica de RPC e chamadas HTTP de inferência do Ouroboros tem uma função de `Timeout` permissiva (> 60s/120s) e retries em caso de erro 502/Down?
- [ ] O "System Prompt" do Agente (Persona Core, Hermes, etc.) restringe severamente a resposta a saídas programáticas (JSON), mitigando "alucinações" causadas pela quantização extrema do modelo?
- [ ] A configuração Anti-Vibe (gates de qualidade) no Ouroboros está relaxada para não punir loops em caso de latência variável da GPU gratuita?

**Pronto para Iniciar.**
Se os itens acima foram concluídos com êxito, execute:
1. `python orchestrator.py` (Deixe rodando em terminal ou nohup)
2. `bun run dev` (Inicie a TUI do Ouroboros)
