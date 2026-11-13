# openpcbot v3

**ðŸ‡§ðŸ‡· [PortuguÃªs](README.md) Â· ðŸ‡ºðŸ‡¸ [English](README.en.md) Â· ðŸ‡ªðŸ‡¸ [EspaÃ±ol](README.es.md)**

Assistente pessoal multicanal (Telegram, CLI, HTTP) com fila durÃ¡vel em SQLite, gestor do Ollama com preflight de RAM, custo por chamada com orÃ§amento, cÃ©rebro com memÃ³ria PT-BR e consolidaÃ§Ã£o noturna. Sucessor do `openpcbotv2`, rodando **ao lado** dele (estrangulamento, nÃ£o corte).

Plano e arquitetura: [PLANO-MIGRACAO-V3.md](PLANO-MIGRACAO-V3.md). Estado por fase: [CHANGELOG.md](CHANGELOG.md). Falhas: [FALHAS.md](FALHAS.md).

Bot no Telegram: **@inemav3bot** (o v2 continua no @inemaclaudebot).

**Conectar e-mails, agendas e o Telegram ao cÃ©rebro:** [docs/CONECTORES.md](docs/CONECTORES.md).

## Jev para observar o roteamento

`/jev observar` ativa comparaÃ§Ã£o de rota, agente e skill pelo OpenRouter neste chat; `/jev` mostra o resultado e `/jev off` desliga. Desde a 3.4.4 cada comparaÃ§Ã£o fica guardada: `/jev historico` lista e `/jev relatorio dia|semana` resume (enviado sozinho no Telegram Ã s 8h05 e Ã s segundas 8h10). O roteador atual continua decidindo. Custos passam pelo gateway e aparecem em `/usage`. [ConfiguraÃ§Ã£o, limites e teste real](docs/JEV.md).

Os comandos Jev tambÃ©m estÃ£o na ajuda do bot: envie `/help` ou `/ajuda` para ver a lista e `/ajuda jev` para consultar a explicaÃ§Ã£o detalhada. A observaÃ§Ã£o envia ao provedor a mensagem atual e os critÃ©rios de classificaÃ§Ã£o, sem memÃ³ria ou histÃ³rico; as sugestÃµes nÃ£o alteram rotas nem executam skills.

## ðŸ“– Guia de uso

Guia completo (landing + passo a passo): **https://inematds.github.io/openpcbotv3/guia/**

---

## 1. Subir

```bash
npm install
cp .env.exemplo .env          # TELEGRAM_BOT_TOKEN_V3 = bot PRÃ“PRIO, do BotFather
npm run doctor                # checa env, Ollama, RAM, CLIs, v2, unit
npx tsx src/cli/importar.ts   # (uma vez) importa memÃ³rias do v2 por snapshot
bash scripts/instalar-servico.sh   # unit systemd --user, MemoryMax=2G, Restart=on-failure
```

Sem token do Telegram o serviÃ§o sobe mesmo assim com HTTP (`127.0.0.1:3142`) e CLI. O token do v2 Ã© **recusado** (dois `getUpdates` = 409 e bot surdo).

## 2. Operar

| O quÃª | Como |
|---|---|
| Health (o hub `wifi` consome) | `GET http://127.0.0.1:3142/health` |
| Painel de monitoramento (TV/celular) | `http://127.0.0.1:3142/` ou `/painel` Â· JSON em `/api/painel` (ver "O processo do painel" na seÃ§Ã£o 11) |
| Dashboard antigo | `http://127.0.0.1:3142/dashboard` |
| Testar o painel sem ligar o bot | `npx tsx src/cli/painel-teste.ts 3199` (sÃ³ HTTP do painel, banco sÃ³ leitura) |
| Falar com o bot sem Telegram | `npm run cli -- "mensagem"` ou `POST /mensagem {"texto":"..."}` |
| Logs | `journalctl --user -u openpcbotv3 -f -o cat` |
| Restart apÃ³s mudar `src/` ou `.env` | `bash scripts/instalar-servico.sh` |
| DiagnÃ³stico | `npm run doctor` (config) Â· `npm run doctor -- --deep` (probes reais: manda mensagem no Telegram e roda um job) |
| Parar tudo agora | `/parar tudo <motivo>` no chat; `/retomar` libera |
| Testes | `npm test` (301) |
| Senha do painel (porta 3143) | `npm run painel-senha` (ou `-- --gerar`) |

---

## 3. Como funciona (o caminho de uma mensagem)

```
Telegram / CLI / HTTP
   â”‚  adaptador traduz para o BUS ("mensagem.recebida")
   â–¼
ORQUESTRADOR
   â”œâ”€ comeÃ§a com "/"?  â†’ comando (seÃ§Ã£o 4), responde na hora
   â”œâ”€ janela collect 2 s: mensagens seguidas viram uma sÃ³
   â”œâ”€ ROTEADOR: llama3.2 (local, grÃ¡tis) devolve JSON {rota, agente, tier, motivo}
   â”‚     "direto"  â†’ conversa, pergunta curta, resumo, traduÃ§Ã£o
   â”‚     "agente"  â†’ precisa de ferramenta: arquivo, shell, repo, web, skill, calendÃ¡rio
   â”œâ”€ CÃ‰REBRO monta o contexto de memÃ³ria (atÃ© 600 tokens, seÃ§Ã£o 6)
   â”‚
   â”œâ”€ direto â†’ GATEWAY LLM (tier local = qwen3.8:27b residente) â†’ resposta
   â”‚           grava turno + memÃ³ria; se for fato durÃ¡vel, propÃµe entrada no vault
   â”‚
   â””â”€ agente â†’ job na FILA (lane `agente`, 1 por vez)
               worker roda `claude -p --output-format json` com prompt em camadas
               (identidade + agente + memÃ³ria + skills + mensagem)
               resultado volta pelo bus ao chat de origem; custo real registrado
   â–¼
GUARDA DE EXFILTRAÃ‡ÃƒO: todo texto que sai Ã© varrido (tokens, keys, JWT, valores do .env) â†’ [REDIGIDO]
```

**SessÃ£o do agente e pausa (3.12.6).** O `claude` do chat Ã© retomado (`--resume`) sÃ³ enquanto o cache do provedor vale: atÃ© 55 min depois da Ãºltima resposta. Aos 40â€“52 min parada, o cron `resumo-sessoes` roda uma cÃ³pia da sessÃ£o (`--fork-session`, cache quente, sem ferramentas que escrevem) que escreve um resumo de retomada (pedido, o que foi feito com caminhos, o que falta, decisÃµes do Nei) e guarda em `sessions.resumo`. Depois de 55 min (atÃ© 24 h), o pedido seguinte abre sessÃ£o NOVA com esse resumo + as 6 Ãºltimas mensagens do chat no prompt. `/novo` descarta tudo isso. Hook que barrar o prompt (ex.: aviso de cache vencido do hook-longrun) faz o bot repetir uma vez em sessÃ£o nova; se barrar de novo, o job falha com o motivo (3.11.6).

Especialistas em paralelo (padrÃ£o grokky): quando o roteador indica `consultar`, atÃ© 2 agentes **sÃ³ leitura** (ex.: `research`) rodam antes e o `lead` recebe as saÃ­das deles no prompt. SÃ³ o lead escreve.

---

## 4. Comandos no chat

| Comando | Faz |
|---|---|
| `/status [id]` | fila por lane (rodando/pendente) ou detalhe de um job com resultado/erro |
| `/cancelar <id>` Â· `/prioridade <id> <n>` | opera a fila |
| `/usage` | custo hoje/semana/mÃªs, por tier e por agente, orÃ§amento |
| `/health` | Ollama (modelos carregados, latÃªncia), RAM/swap, RSS do processo, fila, heartbeat, canais |
| `/ollama status\|preflight <m>\|descarregar <m>` | gestor do Ollama (descarregar recusa modelo que nÃ£o Ã© do v3) |
| `/memoria lista\|buscar <t>\|salvar <t>\|esquecer <id>\|propostas\|aprovar <id>\|descartar <id>` | cÃ©rebro e vault |
| `/tarefa add [quando] <texto>` Â· `/tarefa lista` Â· `/tarefa feita <id>` | suas tarefas; `quando` = `em 2h`, `amanhÃ£ 9h`, `sex 10h`, `15/09 14:30` |
| `/daily` | resumo: tarefas, feitas nas Ãºltimas 24 h, custo do dia, fila, Ollama, MEMORY.md |
| `/cron lista\|on <nome>\|off <nome>` | tarefas agendadas |
| `/fontes` Â· `/fontes ingerir [gmail|agenda]` | chats observados e o que jÃ¡ virou memÃ³ria ([docs/CONECTORES.md](docs/CONECTORES.md)) |
| `/agentes` Â· `/skills` | o que estÃ¡ instalado |
| `@<agente> texto` | manda direto para um agente (ex.: `@mostratec`); palavras em `gatilhos:` do `agent.yaml` fazem o mesmo |
| `/novo` | limpa sessÃ£o do CLI e histÃ³rico do chat Â· `/compress` sÃ³ a sessÃ£o |
| `/consolidar` | roda a consolidaÃ§Ã£o de memÃ³ria agora |
| `/parar [tudo\|agentes\|<agente>] [motivo]` Â· `/retomar [alvo]` | interruptores persistentes; `/parar` sozinho lista |
| `/fila [collect\|followup\|steer\|interrupt]` | como tratar mensagem que chega com outra em curso (por chat) |
| `/personality [nome\|off]` | persona deste chat (`personalidades/*.md`, somada a `IDENTIDADE.md`) |
| `/context [detail] [texto]` | tokens por camada do prompt, direto e agente |
| `/jev [observar\|off]` | ativa/desativa a comparaÃ§Ã£o neste chat; sem argumento, mostra o estado e a Ãºltima sugestÃ£o com custo |
| `/jev historico [n]` | Ãºltimas n comparaÃ§Ãµes (padrÃ£o 10, mÃ¡x. 50) com concordÃ¢ncia acumulada |
| `/jev relatorio dia\|semana` | resumo das Ãºltimas 24 h ou 7 dias: concordÃ¢ncia, confianÃ§a, divergÃªncias, custo |
| `/ajuda jev` | explica a observaÃ§Ã£o Jev, os dados enviados e seus limites |
| `/chatid` Â· `/versao` Â· `/ajuda` Â· `/help` | identificaÃ§Ã£o do chat, versÃ£o e lista de comandos |

---

## 5. Fila (`src/fila/`, portada do inemaccbot)

SQLite com claim atÃ´mico (`UPDATE ... RETURNING`), lease com heartbeat a cada 30 s, backoff exponencial, idempotÃªncia por `idem_key`, cancelamento sem corrida, drain gracioso no `SIGTERM` (renova lease enquanto espera, aborta apÃ³s 110 s). Jobs nunca sÃ£o apagados: sÃ£o o histÃ³rico.

Lanes e concorrÃªncia (`src/fila/filas.ts`):

| lane | conc. | uso |
|---|---|---|
| `chat` | 2 | resposta direta que chegou durante outra em curso |
| `agente` | 1 | `claude -p` (cada um ~500 MB de RAM) |
| `ollama` | 1 | consolidaÃ§Ã£o, embeddings |
| `cron` | 1 | tarefas agendadas |
| `io` | 4 | rede/arquivo, junÃ§Ã£o de especialistas |

Todo job com `chat_id` avisa o chat ao terminar (sucesso ou falha). Se o envio falhar, uma varredura a cada 20 s reentrega.

---

## 6. CÃ©rebro (`src/cerebro/`)

**MemÃ³rias** (tabela `memories`, igual Ã  do v2): cada mensagem sua com mais de 20 caracteres vira uma memÃ³ria, classificada por regex PT-BR + inglÃªs em `semantic` (durÃ¡vel: "meu", "prefiro", "sempre", "nunca", "lembra", "moro em", "meu nome"...) ou `episodic`. Pergunta nunca vira fato. Dedupe por hash da frase normalizada.

**Salience**: comeÃ§a em 1.0, +0.1 a cada uso; decai por dia sem uso (semÃ¢ntica 0,5 %, episÃ³dica 2 %); abaixo de 0.05 e sem acesso hÃ¡ 30 dias, Ã© apagada.

**Busca**: FTS5 (palavra-chave) + vetor `bge-m3` (BLOB de 1024 floats, cosseno em JS). Vetores sÃ£o indexados pelo cron `indexar-memoria` a cada 15 min.

**Contexto no prompt** (3 camadas, teto 600 tokens): 3 por palavra-chave, 3 por vetor, 3 mais importantes, 3 mais recentes, sem repetir, mais 3 insights.

**ConsolidaÃ§Ã£o noturna** (cron 4h, no Ollama, custo zero): funde duplicatas (fica a mais recente), detecta contradiÃ§Ãµes por data (a antiga recebe `superseded_by`, nunca Ã© apagada) e gera atÃ© 3 insights por chat.

**Vault curado** (`~/vault/MEMORY.md` e `USER.md`): o bot propÃµe, vocÃª aprova (`/memoria aprovar <id>`). Nada Ã© escrito sem aprovaÃ§Ã£o. `USER.md` entra no prompt de toda conversa.

**ImportaÃ§Ã£o do v2**: `VACUUM INTO` faz um snapshot consistente do banco do v2 (que continua em uso) e insere com `origem='v2'`. Idempotente.

**Loop de aprendizado de skills**: rascunhos vÃ£o para `skills/_rascunhos/`; nunca sÃ£o promovidos sozinhos.

---

## 7. Gestor do Ollama (`src/ollama/`)

SÃ³ HTTP com o serviÃ§o systemd (nunca `ollama serve`). `config/ollama.yaml`:

| papel | modelo | por quÃª |
|---|---|---|
| `roteador` | `llama3.2` | classificaÃ§Ã£o em JSON, 2 s, 4 GB |
| `geral` | `qwen3.8:27b`, residente | **mesma tag do v2**: um Ãºnico modelo serve os dois bots |
| `embed` | `bge-m3` | vetores da memÃ³ria, 0,7 GB |
| `pesado` | `llama3.1:70b`, **desligado** | 42 GB; sÃ³ depois do corte |

**Preflight** antes de qualquer chamada local: modelo jÃ¡ residente passa; pequeno precisa de tamanho + 4 GB livres; grande nÃ£o residente precisa de `PISO_RAM_GB` (40) e de **nenhum outro grande carregado**. Se falhar, cai para o tier barato e alerta. O v3 **nunca descarrega** modelo que nÃ£o carregou (pode ser do v2).

**Probe** a cada 60 s: `/api/ps`, latÃªncia de um "ok" no roteador, RAM. Alerta se Ollama fora, latÃªncia > 5 s, RAM < 10 GB, dois modelos grandes residentes.

---

## 8. Custo (`src/custo/`)

`gateway.ts` Ã© o **Ãºnico** ponto por onde uma chamada de LLM passa (inclusive embeddings). Ordem: orÃ§amento â†’ escolhe provedor pelo tier â†’ preflight de RAM (se local) â†’ chama â†’ mede latÃªncia â†’ calcula custo â†’ grava em `chamadas_llm`.

| tier | provedor | modelo | quando |
|---|---|---|---|
| `local` | Ollama | qwen3.8:27b | padrÃ£o; custo 0 |
| `barato` | OpenRouter | claude-haiku-4.5 | raciocÃ­nio longo em resposta direta, ou Ollama sem RAM |
| `premium` | `claude -p` (CLI) | sonnet/opus por agente | rota agente; custo real vem do prÃ³prio CLI |

OrÃ§amento (`config/orcamento.yaml`, `ORCAMENTO_MENSAL_USD`): aviso a 70 %, **trava a 100 %** (sÃ³ Ollama passa). Subir de tier registra o motivo.

---

## 9. Tarefas, cron e heartbeat (`src/tarefas/`)

- **Cron** vira job na fila com `idem_key = nome@ocorrÃªncia` (nunca dispara duas vezes). SessÃ£o `isolada` (contexto limpo) ou `principal`. PadrÃ£o: `lembretes` (1 min), `resumo-sessoes` (1 min, seÃ§Ã£o 3), `indexar-memoria` (15 min), `decaimento` (3h30), `consolidacao-noturna` (4h), `backup-noturno` (4h15), `daily-8h`.
- **Heartbeat** a cada 30 min: recupera leases vencidos, cancela zumbi (rodando > 2Ã— o timeout de 20 min), alerta 3 falhas da mesma tarefa em 1 h.
- **Suas tarefas** (`tarefas_usuario`): lembrete por data, resumo no `/daily`.

---

## 9b. Controle da conversa (fase 9: interruptores, modos de fila, persona, contexto)

Tudo desta seÃ§Ã£o fica na tabela `prefs` (KV por chat; `'*'` Ã© global) e sobrevive a restart.

### Interruptores (`/parar`, `/retomar`)

| Alvo | O que bloqueia | O que continua |
|---|---|---|
| `tudo` | qualquer resposta (Telegram, CLI, HTTP, cronâ†’responder) e qualquer agente | comandos de barra; manutenÃ§Ã£o na lane `ollama` (consolidaÃ§Ã£o, ingestÃ£o, indexaÃ§Ã£o, backup) |
| `agentes` | despacho de `claude -p` (lead e especialistas) | resposta direta no Ollama |
| `agente:<id>` | sÃ³ aquele agente (`/parar ops quebrou`) | o resto |

- `/parar` sozinho lista o que estÃ¡ ligado, com motivo. `/retomar` sem alvo desliga todos.
- Ligar cancela o que estÃ¡ em voo nas lanes `chat`, `agente` e `io` que bate com o alvo. Um `claude -p` em execuÃ§Ã£o morre na batida seguinte do worker (atÃ© 30 s).
- Job enfileirado **antes** do `/parar` falha ao ser pego, com o motivo, sem gastar token.

### Modos de fila (`/fila`)

O que fazer com uma mensagem que chega enquanto outra estÃ¡ em curso. Por chat; padrÃ£o `collect`.

| Modo | Comportamento | DiferenÃ§a para o openclaw |
|---|---|---|
| `collect` | junta mensagens seguidas por 2 s e responde uma vez | igual |
| `followup` | sem janela; se ocupado, entra na fila e responde depois | igual |
| `steer` | se ocupado, **substitui** o que estava na fila deste chat pela nova, prioridade alta | o openclaw injeta no agente em execuÃ§Ã£o; com `claude -p` em subprocesso nÃ£o dÃ¡, entÃ£o a mensagem sÃ³ entra quando o agente atual terminar |
| `interrupt` | cancela fila e agentes deste chat (por `flow_ref = canal:chat`) e responde Ã  nova agora | resposta direta jÃ¡ em voo no Ollama nÃ£o Ã© abortÃ¡vel: ela termina e Ã© descartada (contador de geraÃ§Ã£o por chat) |

### Persona (`/personality`, `SOUL.md`)

TrÃªs camadas, sempre somadas, nesta ordem no prompt:

1. `IDENTIDADE.md` (raiz): a base, nunca sai.
2. `agents/<id>/SOUL.md`: persona fixa daquele agente (opcional; precisa de restart para o registry reler).
3. `personalidades/<nome>.md`: escolhida por chat com `/personality <nome>`; `/personality off` volta ao padrÃ£o. Vale para resposta direta e agentes.

Trocar a personalidade apaga as sessÃµes `--resume` do chat, senÃ£o o agente retomaria com a voz antiga por atÃ© 55 min. Exemplos que jÃ¡ vÃªm: `curto`, `professor`. Criar uma = um `.md` novo na pasta.

### `/context [detail] [texto]`

Mede, sem chamar modelo de chat, quanto do prompt Ã© cada camada (identidade, persona, `USER.md`, memÃ³ria e insights, histÃ³rico, skills, regras, mensagem), para a resposta direta e para o agente. `detail` lista os itens de memÃ³ria e histÃ³rico. Sem `texto`, usa a Ãºltima mensagem sua do chat como consulta. O retrieval roda em modo somente leitura: `/context` nÃ£o infla saliÃªncia.

### `doctor --deep`

`npm run doctor -- --deep` acrescenta quatro probes que **exercitam** o sistema, marcados como probe: `GET /health` do serviÃ§o; `sendMessage` no Telegram para o `ALLOWED_CHAT_ID` (nunca `getUpdates`); uma chamada ao modelo `roteador` pelo gateway de custo em banco de memÃ³ria (nÃ£o entra em `chamadas_llm`); e um job `doctor-probe` na lane `io` que o worker do serviÃ§o precisa pegar e concluir em 30 s (falha com "serviÃ§o antigo" se o binÃ¡rio no ar nÃ£o tiver a tarefa: reinstale).

### MCP por agente (`mcp_config`)

No `agents/<id>/agent.yaml`, `mcp_config: mcp.json` (caminho relativo Ã  pasta) faz o `claude -p` subir com `--mcp-config <arquivo> --strict-mcp-config`: o agente vÃª **sÃ³** esses servidores. Sem a chave, herda os MCP do `~/.claude` como sempre. Modelo em `agents/_template/mcp.json.example`. Um cliente MCP nativo no caminho Ollama estÃ¡ anotado como fase 10 em [docs/INCORPORAR-V3.md](docs/INCORPORAR-V3.md), sem ordem de inÃ­cio.

---

## 10. Canais (`src/canais/`)

- **Telegram** (grammy): sÃ³ o `ALLOWED_CHAT_ID`; markdown simples vira HTML; mensagens longas fatiadas; 409 vira log, nÃ£o loop.
- **CLI**: stdin/stdout quando hÃ¡ TTY ou `--cli`.
- **HTTP**: `127.0.0.1` por padrÃ£o (`POST /mensagem` dispara agente com permissÃµes). Expor na LAN sÃ³ com `HTTP_BIND_V3=0.0.0.0` **e** `DASHBOARD_TOKEN_V3`. Rotas do painel em `src/painel/http-painel.ts` (`/painel`, `/api/painel[/<grupo>]`, `/api/painel/falha`, `POST /api/vagas`).
- **Slack**: Web API via fetch, probe de OAuth no doctor; desligado atÃ© o corte (token Ã© do v2).
- **WhatsApp**: stub que recusa enquanto o v2 estÃ¡ ativo (sessÃ£o Ãºnica do nÃºmero; whatsapp-web.js sobe um Chromium). ImplementaÃ§Ã£o real na fase 8, em daemon separado.

---

## 11. Telemetria e alertas

Logs pino JSON no journal. `GET /health` devolve `ok` (Ollama online, RAM â‰¥ 10 GB, heartbeat < 90 min) mais fila, modelos, RAM, orÃ§amento. Alertas com dedupe de 30 min por chave, entregues no chat permitido do Telegram: Ollama fora, RAM baixa, lease morto, zumbi, 3 falhas, orÃ§amento 70 %/100 %.

Dashboard antigo (`/dashboard`): health, Ollama/RAM, fila por lane, custo 14 dias (sparkline) e por tier/agente, memÃ³ria, jobs recentes. Atualiza a cada 10 s.

### Painel de monitoramento (3.13.6, `src/painel/`)

Tela para ficar aberta numa TV (1920Ã—1080 sem rolagem) ou no celular (uma coluna). Nove grupos: Esperando vocÃª Â· MÃ¡quina Â· Bots Â· ProduÃ§Ã£o Â· PublicaÃ§Ã£o Â· CrÃ©ditos e logins Â· Sites Â· Vagas Â· Terminais. Clique abre o detalhe. ProduÃ§Ã£o, PublicaÃ§Ã£o e Terminais tÃªm trÃªs modos (Lista, Ãcones, Imagens), guardados no navegador. Faixa de alerta no topo; "Voz" (pt-BR, desligada por padrÃ£o) fala sÃ³ alerta novo.

NÃ£o gasta token nem modelo: sÃ³ lÃª `/proc`, `statfs`, o banco do v3, o `inemaccbot.db` (somente leitura), o fim dos JSONL, `journalctl`/`systemctl --user` e os comandos `fila-vaga`/`terminais` do hub, tudo com cache (painel 5 s, timers 60 s, GPU 5 min, boots 10 min). Medido: ~10 ms e ~20 ms de CPU por leitura quente; a primeira leitura fria ~250 ms.

**O processo do painel, do inÃ­cio ao fim**

1. âš™ï¸ O navegador pede `GET /api/painel` a cada 15 s (troca sÃ³ o cartÃ£o que mudou).
2. âš™ï¸ `agregador.ts` coleta cada grupo isolado; fonte ausente vira "sem dados" no cartÃ£o, nunca erro 500.
3. âš™ï¸ `alertas.ts` aplica as regras (lista Ãºnica): disco > 90 %, memÃ³ria < 10 GB, GPU sem memÃ³ria na Ãºltima hora, reinÃ­cio < 6 h, job/rodada falhou (24 h), rodada presa > 90 min, fila com erro recente (48 h), site fora, limite > 80 %, serviÃ§o parado, terminal interrompido, aviso manual a â‰¤ 2 dias.
4. âš™ï¸ Sites: um GET por site a cada 5 min, sÃ³ enquanto alguÃ©m olha o painel (`PAINEL_SITES=0` desliga). Lista em `~/.config/inema/painel-sites.json`.
5. ðŸ§  Avisos manuais (ex.: "OAuth YouTube: conferir") em `~/.config/inema/painel-avisos.json` (`[{titulo, ate, nivel}]`).
6. ðŸ§  Vagas: o Nei muda no detalhe de Vagas (âˆ’/+) ou no terminal com `vagas <classe> <n>`. O painel chama o mesmo comando (`execFile`, sem shell); exige `DASHBOARD_TOKEN_V3` e o token no cabeÃ§alho Bearer. Sem token, a tela fica sÃ³ leitura.
7. âš™ï¸ Alerta no Telegram: pronto e DESLIGADO (`PAINEL_ALERTA_TELEGRAM=1` liga; nunca repete a mesma chave, estado em `store/painel-avisados.json`).

**VersÃ£o JARVIS (3.14.7):** `/jarvis` mostra os mesmos dados no visual HUD do kit jarvis-painel (nÃºcleo com anÃ©is, bom dia falado, teclas 1â€“4). Mesmo login; botÃ£o "VersÃ£o Jarvis" no painel e "Painel clÃ¡ssico" para voltar. SÃ³ lÃª.

**Terminais (3.15.7):** cada sessÃ£o mostra quem abriu (ðŸ‘¤ vocÃª no terminal Ã— ðŸ¤– agente: timer, bot, outra sessÃ£o), modelo, contexto, custo e Ãºltimo pedido, lidos do retrato que a statusline grava em `~/.local/share/inema/terminais/estado/` e do transcript. A conversa abre sÃ³ para leitura (`/api/painel/terminal`, segredos mascarados).

**Painel pela rede local / tÃºnel, com senha (3.13.7)**

A porta **3142 nunca sai do loopback**: ela tem o `POST /mensagem`, que chega a um agente com permissÃ£o total. Para a LAN ou para o tÃºnel (ccc.inema.pro) existe a **porta prÃ³pria do painel, 3143**, que sÃ³ serve login, painel e vagas (qualquer outra rota â†’ 404).

1. ðŸ§  Definir a senha (uma vez): `npm run painel-senha` (pede duas vezes, sem mostrar) ou `npm run painel-senha -- --gerar` (cria uma forte e a escreve UMA vez em `~/.config/inema/painel-senha-inicial.txt`; leia, guarde e apague). SÃ³ o hash scrypt fica em `~/.config/inema/painel-senha.json` (600). Trocar a senha derruba todas as sessÃµes.
2. âš™ï¸ Para a rede local, no `.env` do v3: `PAINEL_BIND=0.0.0.0` (porta `PAINEL_PORT=3143`). Sem senha definida o painel recusa sair do loopback. Depois: `bash scripts/instalar-servico.sh`.
3. ðŸ§  Abrir `http://<IP-da-mÃ¡quina>:3143/` â†’ tela de login. O aparelho fica lembrado por 365 dias, renovados a cada uso; o navegador pode guardar a senha. "Sair deste aparelho" e "Sair de todos" no topo do painel.
4. ðŸ§  Com sessÃ£o, as vagas mudam pelo detalhe de Vagas (sem token Bearer; CSRF por `X-Painel` + Origin).
5. âš™ï¸ TÃºnel (depois): cloudflared aponta `ccc.inema.pro` para `http://127.0.0.1:3143`. O cookie ganha `Secure` com `X-Forwarded-Proto: https`; o limite de tentativas usa `CF-Connecting-IP`.
6. ðŸ§  Google (depois): criar a aplicaÃ§Ã£o no Cloudflare Access e definir `PAINEL_CF_ACCESS_AUD`, `PAINEL_CF_ACCESS_TEAM` e `PAINEL_EMAILS` (lista de e-mails aceitos). Sem essas trÃªs, o caminho Google fica desligado.

| variÃ¡vel | padrÃ£o | |
|---|---|---|
| `PAINEL_PORT` | 3143 | `0` desliga o servidor do painel |
| `PAINEL_BIND` | 127.0.0.1 | `0.0.0.0` para a LAN (exige senha) |
| `PAINEL_SENHA_ARQ` | `~/.config/inema/painel-senha.json` | |
| `PAINEL_CF_ACCESS_AUD` / `PAINEL_CF_ACCESS_TEAM` / `PAINEL_EMAILS` | | login Google via Cloudflare Access |

Backup noturno: `VACUUM INTO` + `age` (se `AGE_RECIPIENT`) ou gzip, retenÃ§Ã£o 14, em `store/backups/`.

---

## 12. Garantias de memÃ³ria na coexistÃªncia com o v2

- **Mesmas tags do v2**: um modelo residente para os dois bots. Mudar a tag do `geral` sem mudar no v2 = dois modelÃµes = OOM.
- **Preflight de RAM** e polÃ­tica de 1 grande residente; nunca descarrega modelo alheio; 70b desligado.
- **Unit com `MemoryHigh=1.5G` / `MemoryMax=2G`** (cgroup v2 delegado, verificado) e `--max-old-space-size=1024`. Pico observado: ~1 GB com um `claude -p`.
- Lane `agente` com concorrÃªncia 1. WhatsApp/Slack desligados.
- O `keep_alive: -1` do modelo geral fixa 17 GB na RAM (a mÃ¡quina foi de 57 para 35 GB livres). Ã‰ o desenho do plano; para soltar, troque para `10m` em `config/ollama.yaml`.

---

## 13. ConfiguraÃ§Ã£o

`.env` do v3 (sÃ³ o especÃ­fico; keys compartilhadas vÃªm em runtime do `.env` do v2):

| variÃ¡vel | default | |
|---|---|---|
| `TELEGRAM_BOT_TOKEN_V3` | | token prÃ³prio; o do v2 Ã© recusado |
| `PORT_V3` | 3142 | |
| `HTTP_BIND_V3` | 127.0.0.1 | |
| `DASHBOARD_TOKEN_V3` | | protege `/` e `/api/*`; sem ele o `POST /api/vagas` responde 403 |
| `PAINEL_SITES` | 1 | `0` desliga a checagem dos sites |
| `PAINEL_ALERTA_TELEGRAM` | 0 | `1` liga o aviso de alerta novo no Telegram (aguarda OK do Nei) |
| `VAGAS_ARQ` | `~/.config/inema/vagas.json` | mesmo override do comando `vagas` |
| `PISO_RAM_GB` | 40 | GB livres para carregar modelo grande |
| `ORCAMENTO_MENSAL_USD` | 50 | |
| `SLACK_ENABLED` / `WHATSAPP_ENABLED` | 0 | |
| `AGE_RECIPIENT` | | backup cifrado |
| (tabela `prefs`) | | interruptores, modo de fila e personalidade por chat; `/parar`, `/fila`, `/personality` gravam aqui |
| `CLAUDE_BIN` / `CODEX_BIN` | claude / codex | |

YAML em `config/`: `ollama.yaml` (papÃ©is, piso, probe), `precos.yaml` (USD/1M tokens, tiers), `orcamento.yaml`.

---

## 14. Layout

```
src/
  bus/           barramento de eventos tipado
  canais/        telegram Â· cli Â· http(+dashboard) Â· slack Â· whatsapp(stub) Â· guarda de exfiltraÃ§Ã£o
  painel/        painel de monitoramento: fontes/ (mÃ¡quina, bots, produÃ§Ã£o, publicaÃ§Ã£o, crÃ©ditos, sites, vagas, terminais) Â· alertas Â· tela/
  fila/          store/worker/runner portados do inemaccbot (lanes: chat, agente, ollama, cron, io)
  orquestrador/  orquestrador Â· roteador Â· comandos Â· agente-cli Â· skills Â· agentes Â· tarefas-fila
  ollama/        gestor (registry, preflight, probe) Â· cliente HTTP Â· ram
  custo/         gateway (Ãºnico ponto de LLM) Â· registro Â· orÃ§amento Â· preÃ§os
  provedores/    ollama Â· openrouter Â· anthropic
  cerebro/       memoria (FTS5+vetor) Â· contexto (3 camadas) Â· consolidacao Â· vault Â· importar-v2
  tarefas/       cron Â· heartbeat Â· usuario Â· padrao
  telemetria/    logger pino Â· alertas com dedupe
  dashboard/     html
  cli/           doctor Â· importar Â· chat
  db/            abrir Â· migrations (com checksum) Â· backup
config/          ollama.yaml Â· precos.yaml Â· orcamento.yaml
systemd/         openpcbotv3.service
skills/, agents/ copiados do v2 como estÃ£o (skills/_rascunhos = loop de aprendizado)
IDENTIDADE.md    persona e limites do bot (entra em todo prompt)
```

Regras: nenhum mÃ³dulo acima de 500 linhas; toda chamada de LLM passa por `custo/gateway.ts`; Ollama sÃ³ por HTTP; falha real vira linha em `FALHAS.md`.

## 15. O que ainda nÃ£o estÃ¡

- **Fase 8 (corte)**: trocar o token de produÃ§Ã£o, v2 sÃ³ leitura por 30 dias, arquivar. SÃ³ com ordem explÃ­cita.
- **Fase 10 (anotada)**: cliente MCP nativo no caminho Ollama, servidor MCP do inemavox, voz no Telegram ([docs/VOZ-TELEGRAM.md](docs/VOZ-TELEGRAM.md) â€” hoje sÃ³ o v2 tem), `/retry` `/undo`, rodapÃ© de uso, saÃºde de skills ([docs/INCORPORAR-V3.md](docs/INCORPORAR-V3.md)).
- WhatsApp real (daemon separado), Slack ligado.
- `steer` nÃ£o injeta no agente em execuÃ§Ã£o (ver 9b).
- RetenÃ§Ã£o da tabela `jobs` (hoje nunca purga; ~1,5 k linhas/dia com o cron de lembretes).
- MemÃ³ria guarda a frase inteira, nÃ£o um fato extraÃ­do.

---

Projeto adaptado da comunidade INEMA TDS · Marca **Arckhé**

Feito pelo **[ARCKHÉ.AI](https://linnoff88-dev.github.io/arckhe-ai)**.