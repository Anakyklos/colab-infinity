# Manual de Operação (Runbook)

Este documento orienta os operadores e desenvolvedores do Ouroboros sobre como manter a infraestrutura de LLM saudável no dia a dia, identificar falhas e realizar recuperações.

## 1. Rotinas Diárias

Para garantir que os agentes do Ouroboros Runtime não percam o cérebro (LLM) durante execuções de longo prazo:

1. **Início da Operação:** Abra o terminal na máquina local dedicada e inicie o orquestrador `python orchestrator.py`. Ele irá verificar o GDrive da Conta Armazém, identificar qual Worker está disponível e inicializar a API.
2. **Saúde do Túnel:** Certifique-se de que a URL exposta do Cloudflare (`trycloudflare.com`) foi injetada corretamente no arquivo `.env` do Ouroboros. Se você usa atualização automatizada via arquivo, confirme se as permissões de leitura no seu PC estão certas.
3. **Monitoramento do Heartbeat:** Acompanhe o arquivo de heartbeat (geralmente salvo em `/content/drive/MyDrive/ColabInfinity_Checkpoint/`) ou os logs do console do Orquestrador. Um timeout consecutivo de 3 pings sinaliza o início iminente do failover (troca de contas).

## 2. Gestão do Pool de Contas

### Adicionar uma nova Conta:
- Crie o e-mail do Google (ou compre/reutilize um antigo, o que reduz chances de banimento de spam).
- Acesse o GDrive, conecte a pasta da Conta Armazém.
- Inclua credenciais em `config.yaml` sob a chave `pool`. O Orquestrador carrega esse arquivo a quente durante o failover, sem precisar reiniciar o Python local.

### Remoção / Pausa de Conta:
- Se uma conta for sinalizada como "Cota Esgotada para GPUs" (costuma durar 12-24h), o script do Orquestrador fará isso automaticamente removendo-a temporariamente da fila e colocando-a no fim de um cooldown.

## 3. Procedimentos para Falhas Comuns (Troubleshooting)

### Falha 1: "Cannot Connect" (Erro 502 no Cloudflare Tunnel)
**Causa:** O processo FastAPI morreu dentro da VM do Colab, o Cloudflared parou, ou a Google derrubou a sessão inativa.
**Ação:**
1. Verifique o terminal do Orquestrador. Ele deveria detectar a queda em < 30 segundos.
2. Aguarde 3-5 minutos para que a rotina automatizada do Playwright deslogue do Colab, realize o swap da conta Worker e repasse a nova URL do túnel.
3. Não reinicie o daemon do Ouroboros bruscamente; a lógica do agente deve conter retries configurados com "backoff exponencial" em falhas de API até que a nova URL seja reconhecida.

### Falha 2: "No Supported GPU Available"
**Causa:** Durante horários de pico, o Colab gratuito pode designar você para uma VM somente-CPU ou matar sua GPU T4.
**Ação:** O notebook deve possuir uma célula de validação de hardware (ex: `assert torch.cuda.is_available()`). Se não tiver, o Playwright lerá o log de erro vermelho na tela. O Orquestrador pulará para o próximo Worker imediatamente.

### Falha 3: "Checkpoint Corrompido ou Drive Não Montado"
**Causa:** Permissões do GDrive expiraram, a conta Armazém parou de compartilhar, ou limite diário de leitura.
**Ação:**
1. Logue manualmente via navegador na Conta Armazém.
2. Certifique-se de que a pasta `ColabInfinity_Checkpoint` está íntegra e sem bloqueios.
3. Se os JSONs (ex. histórico de rotação) corromperem, apague o arquivo `state.json` (recomeçando o registro).

## 4. Recuperação de Desastres

O cenário de "pior caso" é a falha massiva das contas Google (banimento do pool inteiro) em um meio do processo de raciocínio de longo prazo (Wave) do Ouroboros.

- **Mitigação Local:** Todos os dados do Ouroboros residem no banco SQLite local. O *estado do agente não depende do Colab*. Apenas pause a interface do Daemon (ou deixe o limite de Retry infinito atuando).
- **Recuperação Manual:**
  1. Forneça uma chave de API paga real (OpenAI ou Anthropic) temporária no `.env` do Ouroboros para desbloquear o daemon enquanto reabastece o Pool de Contas do Colab.
  2. Substitua o `config.yaml` com as novas contas, volte a URL do Orquestrador no `.env`, e os agentes voltarão à infraestrutura gratuita perfeitamente de onde pararam.
