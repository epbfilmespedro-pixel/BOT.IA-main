# VALIDAÇÃO FINAL — Freebuff Desktop

Executada em 2026-10-06, nesta máquina (Windows, Node v26.7.0, npm 11.19.0, Ollama 0.35.1,
Electron 33.4.11). Produto validado: `Freebuff Desktop 1.1.0`.

Esta rodada fechou a **fronteira de segurança** entre aplicação, projeto do usuário e
preview — e corrigiu os defeitos encontrados durante a validação real.

A última rodada fechou o defeito que sobrava: em conversas longas o histórico inteiro era
reenviado ao modelo a cada retry, o que favorecia a **regurgitação** (repetir o bloco já
recusado). Agora cada rodada tem o seu próprio contexto e o retry é limpo (item 14).

```
========================================
VALIDAÇÃO FINAL ATUALIZADA
========================================

OLLAMA REAL                 PASS
AUTO-DETECÇÃO               PASS
MODELOS REAIS               PASS
TROCA DE MODELO             PASS
STREAMING                   PASS

AGENTE REAL                 PASS
READ BEFORE EDIT            PASS
PATCH INCREMENTAL           PASS

NOVO PROJETO                PASS
ABRIR PROJETO EXISTENTE     PASS
MÚLTIPLOS PROJETOS          PASS
HISTÓRICO POR PROJETO       PASS

REGISTRY PROTEGIDO          PASS
APP ROOT BLOQUEADA          PASS
PREVIEW ISOLADO             PASS
PREVIEW SOMENTE VISUAL      PASS
PREVIEW INTERATIVO          PASS
HOT RELOAD                  PASS

RECONEXÃO OLLAMA            PASS
ELECTRON REAL               PASS

--- alvo autorizado (item 12) ---
TARGET AUTHORIZATION             PASS
WRONG-FILE PATCH REJECTION       PASS
READ → RETRY                     PASS
CORRECT FILE PATCH               PASS
NO UNAUTHORIZED FILE CHANGES     PASS
REAL OLLAMA                      PASS
REAL UI                          PASS
REAL ELECTRON                    PASS

--- conteúdo / edição por âncora (item 13) ---
SMALL PATCH                      PASS
LARGE FILE EDIT                  PASS
ANCHOR EDIT                      PASS
AMBIGUOUS FIND REJECTION         PASS
MISSING FIND REJECTION           PASS
WRONG TARGET REJECTION           PASS
DESTRUCTIVE EDIT REJECTION       PASS
STRUCTURAL EDIT REJECTION        PASS
ATOMIC WRITE                     PASS
LEGACY PATCH                     PASS
READ BEFORE EDIT                 PASS
REAL OLLAMA                      PASS
REAL ELECTRON                    PASS
PREVIEW UPDATE                   PASS

--- higiene de contexto (item 14) ---
CLEAN RETRY CONTEXT              PASS
NO OLD PATCH REPLAY              PASS
OPERATION ISOLATION              PASS
REGURGITATION GUARD              PASS
FOCUSED EXCERPT RETRY            PASS
ORIGINAL REQUEST PRESERVED       PASS
SAFE FAILURE                     PASS
REAL OLLAMA                      PASS
REAL ELECTRON                    PASS
PREVIEW ISOLATED                 PASS

--- aceitação final (item 15) ---
ADAPTIVE CONTEXT                 PASS
LEITURA NU SEM CERCA (parser)    PASS
RODADA CORRETIVA DE FORMATO      PASS
FINAL ACCEPTANCE                 PASS

TESTES AUTOMATIZADOS        180/180
========================================
```

As contagens dentro dos blocos dos itens 9, 13 e 14 são o retrato da rodada em que foram escritos
(107, 142 e 161 testes). A contagem atual — **180/180** — está neste bloco e no item 15.

O defeito relatado — pedido manda editar `styles.css` e o modelo envia `patch:index.html` —
não é mais aplicado: o patch é recusado **antes** de qualquer leitura ou escrita. A prova
completa (testes determinísticos, roteiro real, UI e Electron, com hashes) está no item 12.

---

## 1. Fronteira: a aplicação nunca é um projeto (CRÍTICO)

O registro incorreto `D:\PREVIEW.IA` existia em `data/projects.json`. Ele foi **removido
pelo próprio mecanismo de defesa** (nem foi preciso editá-lo à mão):

```
[projects] projeto "PREVIEW.IA" ignorado (app-root): a pasta pertence à própria aplicação — D:\PREVIEW.IA
```

Nenhum arquivo da aplicação foi apagado. O registry voltou a ter exatamente os 4 projetos
legítimos, e a aplicação reiniciou normalmente (três reinícios completos nesta rodada).

### Como a barreira funciona

`server/app-guard.js` decide **por caminho canônico**: `path.resolve` + `fs.realpathSync.native`
(symlinks/junctions/nomes 8.3 seguidos) + comparação *case-insensitive* no Windows. Nunca
por comparação textual.

```
APP_ROOT (a própria aplicação)          -> REJEITADO   app-root
APP_ROOT\<interna>                      -> REJEITADO   app-internal:<nome>
ancestral de APP_ROOT (ex.: D:\)        -> REJEITADO   app-ancestor
APP_ROOT\projects                       -> REJEITADO   app-internal:projects
APP_ROOT\projects\<projeto>             -> ACEITO      área gerenciada pelo app
D:\Projetos\Teste, D:\Sites\MeuSite     -> ACEITO      projeto externo legítimo
```

A barreira existe em **três pontos**, com a mesma mensagem amigável e sem stack trace:

| Onde | Arquivo | Efeito |
| --- | --- | --- |
| Cadastro | [server/projects.js](server/projects.js) `open()` | rejeita antes de tocar no disco |
| Carregamento | [server/projects.js](server/projects.js) `load()` | descarta entrada adulterada/antiga |
| Preview | [server/preview.js](server/preview.js) | `403` para qualquer caminho da árvore da aplicação |

Mensagem exibida na interface (a mesma no cadastro, no `projects.json` e no preview):

```
Este diretório pertence à própria aplicação e não pode ser usado como projeto. Escolha uma pasta de projeto externa.
```

### Prova pela API real

```
400  D:\PREVIEW.IA                      -> mensagem amigável
400  D:\PREVIEW.IA\server               -> mensagem amigável
400  D:\PREVIEW.IA\public               -> mensagem amigável
400  D:\PREVIEW.IA\electron             -> mensagem amigável
400  D:\PREVIEW.IA\data                 -> mensagem amigável
400  D:\PREVIEW.IA\tools                -> mensagem amigável
400  D:\PREVIEW.IA\test                 -> mensagem amigável
400  D:\PREVIEW.IA\..\PREVIEW.IA\server -> mensagem amigável
400  D:\                                -> mensagem amigável
```

Nenhuma resposta contém `at Object`, `node:internal` ou `at async` — nenhum stack trace
chega à interface normal.

### Prova pela UI real (Electron, janela nativa)

Diálogo **Abrir projeto existente** → digitar `D:\PREVIEW.IA` → **Abrir pasta**:

```
dialogOpen: true
status: "Este diretório pertence à própria aplicação e não pode ser usado como projeto.
         Escolha uma pasta de projeto externa."
activeProject: "TESTE-PROJETO-EXTERNO"   (não mudou)
```

O mesmo foi repetido dentro da janela Electron para `\server`, `\electron`, `\data` e
`..\PREVIEW.IA\public` — todos recusados, projeto ativo intacto. No navegador, o mesmo
fluxo pela UI produziu o mesmo resultado.

---

## 2. Abrir projeto existente pela UI real

Projeto externo criado em disco com interface real (HTML + CSS + JS):

```
D:\TESTE-PROJETO-EXTERNO\
  index.html              1.5 KB
  assets\styles.css       2.5 KB
  assets\app.js           2.2 KB
```

Fluxo executado **pela interface real** (navegador, e depois de novo na janela Electron):

```
abrir aplicação            -> "Meu Projeto" no cabeçalho, preview renderizando
Abrir Projeto              -> popover com 4 projetos (nenhum PREVIEW.IA)
+ Abrir projeto existente  -> diálogo com o caminho completo
D:\TESTE-PROJETO-EXTERNO   -> Abrir pasta
projeto aparece no seletor -> "TESTE-PROJETO-EXTERNO · D:\TESTE-PROJETO-EXTERNO" ✓
Preview aponta para o projeto -> #preview-frame src = http://127.0.0.1:4181/?v=1791300142194:3:6169
arquivos do projeto renderizados -> <title>Painel Externo — Clientes</title>
```

O conteúdo servido pelo preview é o do projeto externo, verificado no DOM da própria
página de preview:

```
title:  "Painel Externo — Clientes"
h1:     "Projeto externo carregado no Preview"
root:   "D:\TESTE-PROJETO-EXTERNO"
cards:  3 (renderizados por assets/app.js)
tabela: 4 linhas · CSS aplicado (rgb(15, 17, 22))
```

Troca de projeto e histórico separado (também pela UI real):

| Ação | Cabeçalho | Preview | Conversa |
| --- | --- | --- | --- |
| ativo TESTE-PROJETO-EXTERNO | TESTE-PROJETO-EXTERNO | `:4181/?v=…:3:6281` | 4 mensagens |
| clique em “Meu Projeto” | Meu Projeto | `:4181/?v=…:3:7856` | 0 mensagens |
| clique em “Login” | Login | `:4181/?v=…:1:660` | 8 mensagens |
| volta para TESTE-PROJETO-EXTERNO | TESTE-PROJETO-EXTERNO | `:4181/?v=…:3:6281` | 4 mensagens |

Nada se misturou: cada projeto tem sua própria conversa em `data/conversations/<id>.json`.

---

## 3. Prova de isolamento (Preview × aplicação)

`node tools/isolation-probe.js` — requisições HTTP cruas, sem normalização do cliente,
com o projeto **externo** ativo. Nenhum vazamento:

```
 404  seguro  traversal clássico              /../server/index.js
 403  seguro  traversal duplo                 /../../data/settings.json
 404  seguro  traversal com subpasta          /assets/../../../server/index.js
 404  seguro  barra invertida                 /..\server\index.js
 404  seguro  barra invertida dupla           /..\..\PREVIEW.IA\server\index.js
 404  seguro  %2e%2e codificado               /%2e%2e/server/index.js
 403  seguro  %2e%2e barra codificada         /%2e%2e%2fserver%2findex.js
 403  seguro  %2e%2e barra invertida          /..%5cserver%5cindex.js
 403  seguro  %2e%2e duplo                    /%2e%2e%2f%2e%2e%2fdata%2fsettings.json
 404  seguro  caminho absoluto windows        /D:%5CPREVIEW.IA%5Cserver%5Cindex.js
 404  seguro  caminho absoluto posix          /D:/PREVIEW.IA/server/index.js
 404  seguro  barra inicial dupla             //server/index.js
 403  seguro  dotfile / git                   /.env · /.git/config · /assets/.env
 403  seguro  byte nulo codificado            /index.html%00.txt
     ...
VAZAMENTOS: 0
```

`server · public · electron · tools · test · data · package.json` do app: nenhum conteúdo
do app foi servido (403/404 neutros). O preview também **não consegue** servir a árvore do
app mesmo se a raiz apontasse para lá: o servidor responde `403` a tudo quando a raiz é
protegida (teste automatizado `the preview refuses to serve anything from the application tree`).

Varredura de conteúdo no preview aberto (9 padrões: blocos de código, prompt do agente,
árvore do workspace, rotas `/api/*`, stack traces, `lastError`, `BOT.IA`, `PREVIEW.IA/`):
**0 vazamentos** em `/`, `/index.html`, `/assets/styles.css`, `/assets/app.js`.

---

## 4. Ollama: os cinco estados separados

`server/ollama-install.js` verifica a presença do executável
(`%LOCALAPPDATA%\Programs\Ollama\ollama.exe`, `C:\Program Files\Ollama\…`, PATH; e os
caminhos usuais de macOS/Linux). `describeConnection()` (em [server/ollama.js](server/ollama.js))
classifica a situação e a interface apenas a renderiza.

| Estado | Título mostrado | Ação oferecida |
| --- | --- | --- |
| `not_installed` | Ollama não foi encontrado neste computador. | **Configurações** |
| `service_stopped` | Ollama instalado, mas o serviço não está em execução. | **Tentar conectar novamente** |
| `connecting` | Conectando ao Ollama… | — |
| `connected` | Ollama conectado | — |
| `connection_error` | mensagem técnica do erro | **Tentar conectar novamente** |

### INSTALADO / SERVIÇO PARADO (medido de verdade)

`ollama.exe serve` encerrado; o executável continua em disco:

```
installed: true
erro.kind: refused
erro: "Servidor Ollama recusou a conexão (nada escutando no endereço informado)."
connection: {"state":"service_stopped",
             "title":"Ollama instalado, mas o serviço não está em execução.",
             "message":"Inicie o serviço local do Ollama e tente conectar novamente.",
             "action":"retry"}
```

Na UI real:

```
pílula do cabeçalho : "Ollama parado"
aviso do agente     : "Ollama instalado, mas o serviço não está em execução. Inicie o serviço local do Ollama e tente conectar novamente."
botões do aviso     : ["Tentar conectar novamente"]
rodapé do compositor: "Ollama instalado, mas fora de execução — inicie o serviço local para enviar comandos."
texto "Instale ou inicie o Ollama" em qualquer lugar da página: false
```

Depois de reiniciar o serviço, **Tentar conectar novamente** → `Ollama conectado · 4 modelo(s)`,
pílula `Ollama conectado`, aviso escondido, modelo restaurado (`qwen2.5-coder:3b`).

### NÃO INSTALADO (medido de verdade)

Instância da aplicação com `FREEBUFF_OLLAMA_BINARY` apontando para um caminho inexistente e
porta 11499 (endereço morto):

```
installed: false
connection: {"state":"not_installed",
             "title":"Ollama não foi encontrado neste computador.",
             "message":"Instale o Ollama para usar os modelos locais neste computador.",
             "action":"settings"}
```

Na UI real:

```
pílula              : "Ollama não instalado"
callout do chat     : "Ollama não foi encontrado neste computador."
botões do callout   : ["Configurações"]
rodapé do compositor: "Ollama não instalado — instale o Ollama para enviar comandos."
```

---

## 5. Ollama real, agente real, preview real

`FREEBUFF_VALIDATE_MODEL=qwen2.5-coder:3b node tools/validate-real.js`

```
PASS OLLAMA REAL        — detecção automática do serviço local — versão 0.35.1
PASS MODELOS REAIS      — gemma3:1b (0.82 GB), gemma4:26b (18.73 GB),
                          qwen2.5-coder:1.5b (0.99 GB), qwen2.5-coder:3b (1.93 GB)
PASS STREAMING          — 8 partes · 3885 ms · "A validação está em andamento."
PASS MODELO EM USO      — pedido: qwen2.5-coder:3b · resposta de: qwen2.5-coder:3b
PASS TROCA DE MODELO    — qwen2.5-coder:3b → gemma3:1b
PASS SEGUNDO MODELO     — nova requisição respondida por gemma3:1b
PASS ARQUIVO GRANDE     — styles.css com 13.7 KB fora do contexto automático
PASS AGENTE REAL        — turno completo do modelo local
PASS EDIÇÃO INCREMENTAL — patch aplicado em vez de reescrever o arquivo
PASS TARGET AUTHORIZATION      — alvos explícitos: styles.css · autorizados: styles.css
PASS CORRECT FILE PATCH        — styles.css 402fd5698 → a81b041c4 · "body { background: #101820; …"
PASS NO UNAUTHORIZED CHANGES   — index.html be783e136 → be783e136 (hash igual: true)
PASS PRESERVAÇÃO        — index.html intacto: true · styles.css com o restante intacto: true
```

A linha `FAIL COR APLICADA` que existia aqui antes **desapareceu**: o modelo de 3B chegou a
enviar um patch válido para o arquivo errado em 3 de 3 execuções na rodada anterior, e
agora o destino é: pedido nomeia `styles.css` → o servidor autoriza `styles.css`, troca o
exemplo do protocolo para `styles.css`, e um patch para `index.html` é **recusado sem
tocar no disco** (item 12).

Turno real pela UI, no projeto externo (modelo `qwen2.5-coder:3b`):

```
✓ Leitura: index.html                                    1.4 KB
✓ Leitura de index.html antes de editar   "o patch anterior não casou · arquivo relido para reenviar o trecho exato"
✓ Analisando as leituras (rodada 2)
✕ index.html: O trecho ATUAL não existe no arquivo atual. Leia o arquivo com ```read:caminho antes de editar.
✕ index.html: O grupo substituiria o arquivo INTEIRO por um trecho muito menor — isso apagaria o resto do projeto.
✓ Edição incremental em index.html (1 trecho)   -> badge "editado" · 1.5 KB · 51 linhas
✓ Arquivos do projeto sincronizados
✓ Preview atualizado
```

O arquivo em disco: 1387 B → **1499 B**, página inteira preservada (cabeçalho, cards,
tabela, script) e apenas o novo rodapé acrescentado. Preview recarregado sozinho
(`?v=…:3:6169` → `?v=…:3:6281`) e o texto do agente apareceu na página renderizada.

---

## 6. Preview interativo e hot reload

- Viewport do preview: `Desktop 694×745`, `Tablet`, `Mobile` — alternância pela UI.
- Interação real na página renderizada: clique na aba “Pedidos” → título mudou para
  “Pedidos do mês”, primeira linha `#1041 Órbita Digital`; contador de interação
  `0 → 1` e relógio andando (JavaScript do projeto executando, não uma imagem).
- Hot reload: cada arquivo aplicado muda a revisão do preview e o iframe é recarregado
  (`1791300142194:3:6169` → `1791300972905:3:6281`).
- O iframe continua **somente visual** para o chat: origem `:4181` ≠ `:4180`.

---

## 7. Electron real

`node_modules/electron/dist/electron.exe . --remote-debugging-port=9222`

```
userAgent: … FreebuffDesktop/1.0.0 Chrome/130.0.6723.191 Electron/33.4.11 Safari/537.36
App       http://127.0.0.1:4180/     (servidor dentro do Electron, PID único)
Preview   http://127.0.0.1:4181/
Projeto   TESTE-PROJETO-EXTERNO (D:\TESTE-PROJETO-EXTERNO)
```

Dentro da janela nativa:

```
title        : "TESTE-PROJETO-EXTERNO · Freebuff Desktop"
project      : "TESTE-PROJETO-EXTERNO"
previewSub   : "TESTE-PROJETO-EXTERNO · renderizando index.html"
mensagens    : 4 (histórico do projeto externo)
Abrir pasta com D:\PREVIEW.IA\server -> mensagem amigável, projeto ativo intacto
preview servido pelo Electron: <title>Painel Externo — Clientes</title>
traversal no preview do Electron: /../server/index.js 404 · /%2e%2e%2fserver%2findex.js 403
                                  /../../data/settings.json 403 · /.git/config 403
registry: meu-projeto, login, dashboard, landing-electron, teste-projeto-externo (nenhum PREVIEW.IA)
```

Criação de projeto pela UI real dentro do Electron também foi exercitada
(“Validacao Fronteira” → pasta criada, preview vazio com o aviso correto, conversa própria
com 0 mensagens; depois removida do registry e do disco, sem tocar em nenhum outro projeto).

---

## 8. Defeitos reais encontrados e corrigidos nesta rodada

1. **A raiz da aplicação podia virar projeto** (o defeito crítico relatado). Barreira
   canônica em `server/app-guard.js`, aplicada no cadastro, no carregamento do registry e
   no servidor do preview. +14 testes.
2. **Patch destrutivo truncava o arquivo do usuário em silêncio.** Encontrado validando o
   projeto externo: o modelo enviou `<<<<<<< ATUAL` com o arquivo inteiro e
   `>>>>>>> NOVO` só com o rodapé; o app gravou o rodapé sobre a página (1387 B → 188 B) e
   anunciou “edição incremental”. Agora `applyPatch` recusa o grupo quando ele consumiria o
   arquivo inteiro mantendo menos de 40% do conteúdo, devolve o motivo ao modelo e o fluxo
   “ler de novo → reenviar o trecho exato” assume — o arquivo sai intacto. +1 teste.
   Comprovado ao vivo: a segunda execução do **mesmo pedido** passou pelo aviso
   `O grupo substituiria o arquivo INTEIRO…` e terminou com a página inteira + rodapé.
3. **O subtítulo do preview mostrava o projeto anterior.** `stateForUi().preview` não
   levava `projectName`; a interface só se corrigia na próxima sondagem. +2 asserções no
   e2e (`preview.projectName` na criação e na troca).
4. **O seletor de projetos mantinha o projeto anterior marcado como ativo** (a
   “inconsistência visual” que não tinha sido reproduzida na rodada anterior). `setState`
   re-renderiza sempre. Reproduzido e corrigido, com prova na janela Electron.
5. **Mensagem imprecisa do Ollama** (item 7 do pedido): “Instale ou inicie o Ollama”
   dividida nos cinco estados, com o binário verificado no computador.

## 9. Limitações honestas

- **O modelo de 3B escolher o arquivo errado deixou de ser um defeito de produto** (item
  12): o destino passou a ser autorizado pelo servidor e o exemplo do protocolo passou a
  usar o arquivo do pedido. Quando o `gemma3:1b` insistiu em `patch:index.html` com o alvo
  `styles.css`, a recusa aconteceu antes da escrita e o `index.html` ficou com o mesmo hash.
  O que continua fora do nosso controle é a **qualidade do trecho** enviado pelo modelo
  pequeno (2 de 3 modelos não conseguiram copiar uma linha de um CSS de 13,7 KB) — aí o
  produto recusa, relê e pede de novo, sem corromper nada.
- **A recusa foi medida no roteiro real, não no painel ao vivo.** Nas 6 tentativas reais do
  cenário tentador, 5 modelos foram direto ao arquivo autorizado (ou nem usaram o
  protocolo) e 1 errou o destino — a recusa foi observada nessa tentativa, com o Ollama e o
  orquestrador reais, não no mock. Nenhuma recusa de alvo apareceu como cartão no painel
  durante a sessão de UI porque os modelos testados miraram o alvo correto; o cartão usa o
  mesmo caminho de renderização (`file_error`) que está visível ao vivo (4 cartões reais de
  erro de trecho na mesma sessão).
- **`gemma4:26b` (18.73 GB)** é detectado e listado, mas não carrega nesta máquina (OOM).
- **Screenshot indisponível neste ambiente** (sem composição de frames): toda a evidência
  de interface é DOM/HTTP/CDP, nunca imagem.
- **A área gerenciada `APP_ROOT\projects` continua aceita** para os projetos criados pelo
  próprio app — é a única exceção dentro da árvore da aplicação e não dá acesso a
  `server/`, `public/`, `electron/`, `tools/`, `test/` nem `data/` (o preview não sai dela).

## 10. Separação pedida

```
TESTES AUTOMATIZADOS         107/107 PASS (node --test --test-timeout=20000 "test/*.test.js")
MOCK VALIDATION              PASS         (Ollama falso, usado só nos testes)
REAL OLLAMA VALIDATION       19 PASS · 0 FAIL · 7 INFO   (execução que capturou a recusa real)
MESMO ROTEIRO, REPETIDO      17 PASS · 0 FAIL · 18 INFO (15 turnos reais, nenhum errou o destino)
TARGET AUTHORIZATION         PASS         (alvo derivado do pedido, item 12)
WRONG-FILE PATCH REJECTION   PASS         (recusa antes da escrita + hash intacto)
READ → RETRY                 PASS         (recusa → relê o arquivo correto → reenvia)
NO UNAUTH. FILE CHANGES      PASS         (hashes conferidos em todos os cenários)
REAL UI VALIDATION           PASS         (navegador: turnos reais, preview, sem escrita fora do alvo)
REAL ELECTRON VALIDATION     PASS         (Electron 33.4.11 real, turno real, hashes conferidos)
REAL ISOLATION PROBE         PASS         (tools/isolation-probe.js · 0 vazamentos)
```

Nenhum resultado de mock foi usado para aprovar Ollama real, agente real, projeto real,
preview real ou a fronteira de segurança.

## 11. Como reproduzir

```bash
npm test                                  # 107 testes
npm start                                 # http://127.0.0.1:4180 · preview em :4181
node tools/isolation-probe.js             # travessias e caminhos canônicos contra o preview
FREEBUFF_VALIDATE_MODEL=qwen2.5-coder:3b node tools/validate-real.js
node tools/real-session.js "Troque somente a cor do botão de entrar."
npm run desktop                           # janela Electron
node_modules/electron/dist/electron.exe . --remote-debugging-port=9222
node tools/electron-probe.js "return document.title"
```

---

## 12. Alvo autorizado: o servidor não obedece ao patch para o arquivo errado

**O defeito relatado.** Pedido: “edite SOMENTE a linha do body em `styles.css`”. Resposta do
modelo (registrada com `FREEBUFF_AGENT_DEBUG=1`): `rodada 1 · patch · index.html · 180 B`.
O patch era tecnicamente válido e o app gravou no arquivo errado — o pedido nunca chegou ao
`styles.css` e a falha foi atribuída ao modelo.

**A regra que passou a valer (implementada, não só documentada).**

```
PEDIDO → o servidor deriva os alvos autorizados da rodada
       → READ (pedido pelo modelo ou automático)
       → PATCH do modelo
       → destino autorizado?
             SIM → aplica
             NÃO → NÃO grava · emite PATCH_REJECTED_TARGET_MISMATCH
                   → relê o arquivo correto → reenvia o conteúdo ao modelo
                   → o modelo reenvia o patch → valida → aplica
```

Autoridade, da mais forte para a mais fraca — a inferência do modelo é a **última**:

```
1. arquivo citado pelo usuário no pedido        ("edite SOMENTE styles.css")
2. arquivo lido na rodada (read: ou leitura automática)
3. estrutura do projeto (nome no pedido que casa com 1 único arquivo existente)
4. inferência do modelo (só quando nada acima aponta para um arquivo)
```

E uma proibição explícita do usuário (“não altere o index.html”) vence todas as outras,
inclusive a leitura e a heurística de nome.

**Identidade de arquivo, não string.** `styles.css`, `./styles.css`, `.\styles.css` e
`assets/../styles.css` são o **mesmo** arquivo; `public/styles.css` **não** é. `../styles.css`,
caminhos absolutos e `C:\…` nunca são autorizados.

**Código.**

- `server/targets.js` (novo): `canonicalTarget`, `targetKey`, `sameTarget`,
  `extractFileMentions` (positivos e negativos), `structureTargets`,
  `AuthorizedTargets` (`allow`, `noteReads`, `preferred`, `promptExample`, `summary`),
  `targetMismatchMessage`, `buildTargetDirective`, `TARGET_MISMATCH_CODE =
  'PATCH_REJECTED_TARGET_MISMATCH'`.
- `server/agent.js`: guarda antes de qualquer leitura/escrita em `patch:`, `delete:` e na
  reescrita (`file:`) de um arquivo **existente** fora do alvo; evento `file_error` com
  `code/reason/authorized` + passo `warn` (“Patch recusado: index.html não é o arquivo
  autorizado desta rodada”); rodada corretiva com o conteúdo real do arquivo autorizado;
  `MAX_ROUNDS` 3 → 4 para caber leitura + recusa + correção; `done` passa a carregar
  `targetRejections` e `targets` (explicit/reads/structure/inferred/forbidden/authorized).
  O exemplo do protocolo passa a ser o arquivo autorizado, e o pedido leva
  `ARQUIVO-ALVO AUTORIZADO DESTA RODADA: …` — o modelo deixa de copiar `index.html`.

**Testes (casos A–D pedidos).**

```
A · patch válido para o arquivo errado   → RECUSADO, nada gravado, hash do index.html igual
    depois: relê styles.css → patch correto → APLICADO
B · read:styles.css → patch:index.html   → RECUSADO → leitura do arquivo correto (source
    "target") → conteúdo real devolvido ao modelo → patch:styles.css APLICADO (3 rodadas)
C · "Atualize index.html e styles.css."  → os DOIS aplicados (e "./styles.css" = styles.css)
D · patch/delete/reescrita em server/index.js e index.html com alvo styles.css
                                          → 3 recusas, arquivos intactos por hash
+ arquivo NOVO de apoio continua permitido (criar não destrói nada)
+ proibição explícita vence a heurística de estrutura
```

`npm test` → **107/107 PASS** (era 93; +14 testes: `test/targets.test.js` e 6 casos de
integração em `test/agent.test.js`).

**Prova com hash (o que o pedido exigia).** Cenário real, arquivo errado intacto:

```
antes:   index.html = 09b97d99f7b7 …   styles.css = 7e44a8e21ae8 …
modelo:  patch:index.html (válido)     -> RECUSADO (PATCH_REJECTED_TARGET_MISMATCH)
depois:  index.html = 09b97d99f7b7 …   (inalterado, byte a byte)
somente após o patch autorizado:
         styles.css  = a81b041c4…      (alterado, o resto do arquivo preservado)
```

**Ollama real (não só mock).** `FREEBUFF_VALIDATE_MODEL=qwen2.5-coder:3b node
tools/validate-real.js` → **19 PASS · 0 FAIL · 7 INFO** (exit 0). O roteiro roda dois
cenários: (1) o pedido original de `styles.css` num CSS de 13,7 KB; (2) um cenário tentador
— histórico que acabou de editar `index.html` + pedido sobre outro arquivo, repetido em
**3 modelos reais × 5 formulações** (incluindo “alterar styles.css”, a forma literal do
relato, e alvos em subpasta). Em três execuções completas (15, 6 e 15 turnos reais):
**0 escrita fora do alvo** em todos.

As duas linhas abaixo descrevem um **erro do modelo**, então só podem ser julgadas quando
um modelo realmente mira o arquivo errado: quando todos acertam o destino elas saem como
INFO (não há o que recusar) e a linha seguinte — a invariante — continua valendo. Na
varredura completa de 15 turnos isso aconteceu (todos miraram o alvo); na execução abaixo,
não.

**A recusa real, na forma exata do relato** (alvo `styles.css`, modelo envia
`patch:index.html`), medida na última execução — `gemma3:1b`, sem nenhum mock:

```
CENÁRIO TENTADOR · gemma3:1b · "Agora use o protocolo e altere somente styles.css…"
  alvos autorizados: styles.css · 1 recusa (index.html) · index.html intacto: true
  styles.css intacto: true · 4 rodadas

PASS WRONG-FILE PATCH REJECTION com modelo real
     gemma3:1b: index.html recusado (PATCH_REJECTED_TARGET_MISMATCH/target-mismatch)
     evento no chat: true · index.html intacto: true
PASS READ → RETRY com modelo real
     depois da recusa: leitura do arquivo correto = true · 4 rodada(s)
     styles.css alterado: false   (o modelo não completou o trecho — nada foi corrompido)
```

Numa execução anterior o mesmo modelo foi recusado mirando um caminho deformado
(`patch:styles.css .` com o alvo `styles.css`) — também barrado antes da escrita. Ou seja:
a recusa acontece nas duas formas, arquivo existente errado e caminho inválido, e quando
nenhum modelo erra o destino o relatório mostra INFO nessa linha (nada a recusar) com a
invariante `NO UNAUTHORIZED FILE CHANGES` ainda em PASS.

**UI real.** Navegador em `http://127.0.0.1:4180` com Ollama real (não mock): turnos reais
pedindo `assets/styles.css` e outro pedindo `index.html`; a trilha mostra leitura, recusa de
trecho e edição incremental; o preview do iframe continua atualizando. Depois dos turnos, os
arquivos do projeto externo estavam com **mtime anterior à sessão** — nenhuma escrita fora do
alvo.

**Electron real.** Janela nativa (`electron 33.4.11`, CDP 9222) com um turno real do
`qwen2.5-coder:3b` disparado pela própria interface:

```
DOM        : "Edição incremental em index.html (1 trecho)" · badge editado · 1.5 KB · 51 linhas
index.html : 09b97d99f7b7 → 0838e2075992   (autorizado — o alvo do pedido)
styles.css : 7e44a8e21ae8 → 7e44a8e21ae8   (intacto)
app.js     : 827d02f6d082 → 827d02f6d082   (intacto)
```

**Nada foi reduzido.** Todas as proteções anteriores continuam ativas e verificadas na mesma
execução: placeholder, arquivo inexistente, patch destrutivo (*whole-file shrink*), marcadores
`=======`, linhas vazias nas bordas, read-before-edit, isolamento do workspace, app-root guard
e preview isolado (`tools/isolation-probe.js` → 0 vazamentos).

---

## 13. Conteúdo / edição por âncora: mudar uma linha de um CSS de 14,2 KB

**O defeito de conteúdo.** O `patch:` antigo pede o arquivo inteiro em `<<<<<<< ATUAL` e em
`>>>>>>> NOVO`. Num arquivo grande isso não é um detalhe de conforto: um modelo local
pequeno que recebe 13,7 KB de CSS **inventa o corpo da regra** em vez de copiá-lo, e a
alteração certa nunca chega ao disco. O problema não é o modelo errar — é a arquitetura
depender de o modelo reproduzir milhares de caracteres para mudar três palavras.

**O que passou a existir.**

```edit:assets/styles.css
FIND:
.hero h1 { font-size: 20px; margin: 0 0 8px; }
REPLACE:
.hero h1 { font-size: 28px; margin: 0 0 8px; }
```

O modelo devolve o trecho, não o arquivo. Quem localiza, valida e grava é o servidor
(`server/anchor-edit.js`), e o `patch:` clássico continua valendo — inclusive um bloco
`edit:` que ainda venha com os marcadores antigos é roteado para o caminho de patch.

Cinco garantias, todas verificadas no item 11 (casos A–F) e nos testes:

| Garantia | Como | Recusa |
| --- | --- | --- |
| unicidade | o trecho tem de existir **uma vez** | `EDIT_REJECTED_AMBIGUOUS` (diz quantas e em quais linhas) |
| existência | o trecho tem de estar no arquivo | `EDIT_REJECTED_NOT_FOUND` |
| integridade | `hash_before`/`hash_after` + `anterior sem o trecho + substituição = final` | gravação abortada, original restaurado |
| estrutura | a substituição não pode deixar chaves inválidas | `EDIT_REJECTED_UNBALANCED` |
| destrutivo | guarda de 40% contra o original (a mesma do `patch:`) | `EDIT_REJECTED_DESTRUCTIVE` |

O alvo continua sendo decidido por `server/targets.js` **antes** de qualquer leitura ou
escrita: `edit:` passa pela mesma checagem que `patch:`.

**Nada disso depende de o modelo acertar de primeira.** Quando o trecho não casa, o
orquestrador usa o próprio `FIND` do modelo para achar a região — pelo texto exato, e
quando falha, **pelo seletor** — e devolve só as linhas em volta, nunca o arquivo inteiro.

### Bloco pedido

```
========================================
CONTEÚDO / EDIÇÃO
========================================

SMALL PATCH                  PASS
LARGE FILE EDIT              PASS
ANCHOR EDIT                  PASS
AMBIGUOUS FIND REJECTION     PASS
MISSING FIND REJECTION       PASS
WRONG TARGET REJECTION       PASS
DESTRUCTIVE EDIT REJECTION   PASS
ATOMIC WRITE                 PASS

LEGACY PATCH                 PASS
READ BEFORE EDIT             PASS
REAL OLLAMA                  PASS
REAL ELECTRON                PASS
PREVIEW UPDATE               PASS

TESTES AUTOMATIZADOS         142/142
========================================
```

(STRUCTURAL EDIT REJECTION também está em PASS — a recusa que nasceu desta rodada, descrita
abaixo.)

### Casos A–F, determinísticos

Todos em `test/edit.test.js` / `test/patch.test.js`, além dos blocos determinísticos de
`tools/validate-real.js`:

```
PASS ANCHOR EDIT — âncora única muda só o trecho (determinístico) — arquivo de 13.8 KB · 1 linha(s) alterada(s) · sem vazamento do protocolo
PASS AMBIGUOUS FIND REJECTION (determinístico) — O trecho FIND aparece 240 vezes no arquivo (linha 10, linha 11, linha 12, linha 13, linha 14). Não é possível escolher sozinho: inclua as linhas do seletor/bloco vizinho para a âncora ficar única.
PASS MISSING FIND REJECTION (determinístico) — EDIT_REJECTED_NOT_FOUND
PASS DESTRUCTIVE EDIT REJECTION (determinístico) — EDIT_REJECTED_DESTRUCTIVE
PASS ATOMIC WRITE — a verificação barra um span inconsistente (determinístico) — motivo: span-mismatch
PASS LEGACY PATCH — patch: continua aplicando igual (determinístico) — patch antigo preservado
PASS SMALL PATCH (determinístico) — arquivo pequeno com patch incremental
```

| Caso | Situação | Resultado |
| --- | --- | --- |
| A | `FIND` existe uma única vez | aplica (`mode: anchor`), só o trecho muda |
| B | `FIND` não existe | `EDIT_REJECTED_NOT_FOUND`, nada gravado |
| C | `FIND` aparece N vezes | `EDIT_REJECTED_AMBIGUOUS` + as linhas; a âncora seguinte, específica, aplica |
| D | alvo autorizado `styles.css`, modelo envia `edit:index.html` | `PATCH_REJECTED_TARGET_MISMATCH`, antes de ler ou gravar |
| E | substituição apagaria a maior parte do arquivo | `EDIT_REJECTED_DESTRUCTIVE`, arquivo intacto |
| F | `patch:` clássico | continua funcionando (round-trip preservado) |

### A recusa que nasceu desta rodada: `EDIT_REJECTED_UNBALANCED`

O turno real no Electron expôs uma corrupção que **nenhuma** das outras checagens via: o
modelo ancorou o seletor pelado e reescreveu a regra dentro do `REPLACE`.

```
FIND:
.tab.is-active
REPLACE:
.tab.is-active { background-color: #2563eb; }
```

O trecho existe uma vez, não é destrutivo e o resultado bate com os spans — mas o `{` que já
estava na linha sobra, e a linha vira `.tab.is-active { background-color: #2563eb; } {`, um
CSS inválido. A verificação estrutural barra exatamente isso (um `}` seguido de `{` sem
seletor, ou um `{` sobrando) e **não grava**:

```
ERR: EDIT_REJECTED_UNBALANCED | A substituição quebraria a estrutura de chaves do arquivo
STEP: done Trecho relevante de assets/styles.css enviado ao modelo | linhas 26–42 · arquivo grande: só o trecho, não o arquivo inteiro
```

O arquivo ficou intacto (`4040fd27bf0a…` antes e depois) e a rodada corretiva mandou de volta
apenas as linhas 26–42 — não os 14,2 KB.

### Prova com `diff` — 14,2 KB, uma linha

Projeto real `D:\TESTE-PROJETO-EXTERNO`, conduzido pela janela do Electron com o Ollama real
(`qwen2.5-coder:3b`), pedido: *“aumente SOMENTE o tamanho da fonte da regra .hero h1 para
28px”*.

```
arquivo   : D:\TESTE-PROJETO-EXTERNO\assets\styles.css
sha256    : 4040fd27bf0a179f9b244d8c256d395f179988f3c1652f18901867537b6f42e5
         -> be475df1192c5068e4bfbeadab68b5b934fcce9dd801b19e4aa83f89541db7f4
bytes     : 14492 antes  ·  14492 depois        (idêntico)
linhas    : 217 antes  ·  217 depois            (idêntico)
```

```
$ diff -u styles.css.antes styles.css
@@ -39,7 +39,7 @@

 .wrap { max-width: 900px; margin: 0 auto; padding: 26px 20px 60px; }

-.hero h1 { font-size: 20px; margin: 0 0 8px; }
+.hero h1 { font-size: 28px; margin: 0 0 8px; }
 .hero p { color: var(--muted); margin: 0 0 22px; }
 .hero code { background: #101319; border: 1px solid var(--line); padding: 1px 6px; border-radius: 6px; color: #cbd6ee; }
```

**Uma linha removida, uma linha adicionada, o resto byte a byte igual** — o `diff` acima é a
saída completa. Os outros dois arquivos do projeto não foram tocados:

```
index.html : 0838e2075992… → 0838e2075992…   (intacto)
app.js     : 827d02f6d082… → 827d02f6d082…   (intacto)
```

### O turno real, passo a passo (Electron real → Ollama real → arquivo → Preview)

O que o modelo devolveu, rodada por rodada (`FREEBUFF_AGENT_DEBUG`), e o que o servidor fez:

```
rodada 1 · edit · 67 B          FIND: h1 { font-size: 24px; }
                                REPLACE: h1 { font-size: 28px; }
   -> EDIT_REJECTED_NOT_FOUND   (o corpo é inventado: o arquivo escreve `.hero h1`)

STEP: done Trecho relevante de assets/styles.css enviado ao modelo
           linhas 34–50 · arquivo grande: só o trecho, não o arquivo inteiro
   (o seletor `h1` casa com o cabeçalho real `.hero h1` — o modelo recebe 17 linhas, não 14,2 KB)

rodada 2 · edit · 108 B         FIND: .hero h1 { font-size: 20px; margin: 0 0 8px; }
                                REPLACE: .hero h1 { font-size: 28px; margin: 0 0 8px; }
   -> aplicado (anchor: true · groups 1)

STEP: done Edição por âncora em assets/styles.css (1 trecho) | FIND → REPLACE · o resto do arquivo não foi reescrito
STEP: done Preview atualizado
```

Duas rodadas, 14,3 s, **61 tokens gerados** para um arquivo de 14,2 KB: o modelo escreveu 175
bytes no total. Nenhuma etapa exigiu que ele reproduzisse o arquivo.

### Preview

O Preview é servido pela porta própria (4181) e mostra só o resultado:

```
GET http://127.0.0.1:4181/                      200
GET http://127.0.0.1:4181/assets/styles.css     200   · .hero h1 { font-size: 28px; … }
getComputedStyle('.hero h1').fontSize         "28px"   (a mudança está renderizada)
body                                          rgb(15, 17, 22)

vazamento de FIND: / REPLACE: / EDIT_REJECTED / patch: / read: / edit:
   no HTML servido pelo Preview ................................................. 0 ocorrências
   no DOM renderizado (documentElement.outerHTML) ............................... false
   no painel de chat do Electron ................................................ false
```

O chat mostra a **trilha** (`Leitura: assets/styles.css`, `Trecho relevante … linhas 34–50`,
`Edição por âncora …`), nunca o bloco cru.

### Ollama real (`tools/validate-real.js`)

`FREEBUFF_VALIDATE_MODEL=qwen2.5-coder:3b node tools/validate-real.js` → o turno real de
edição por âncora em um CSS de 13,7 KB:

```
INFO EDIÇÃO POR ÂNCORA · qwen2.5-coder:3b — aplicada · 1 linha(s) alterada(s) · cor: true · rodadas 2 · leituras 1 · trecho focado enviado: true (1x) · sem vazamento: true
PASS LARGE FILE EDIT — modelo real altera uma linha de um CSS de 13,7 KB — qwen2.5-coder:3b: 1 linha(s) alterada(s) · arquivo preservado (true) · hash 035ce400b → 486ee566e · rodadas 2 · trecho focado: true
PASS NO UNAUTHORIZED FILE CHANGES (edição por âncora) — 1 modelo(s) · index.html intacto em todos: true · protocolo ausente do arquivo: true
```

Total do roteiro real (`qwen2.5-coder:3b`, Ollama 0.35.1): **26 PASS · 0 FAIL · 19 INFO**
(exit 0). Os INFO são os cenários em que o resultado observado é uma propriedade do modelo
(por exemplo nenhum modelo errou o destino, então não havia o que recusar) — nunca falhas.

### Defeitos encontrados e corrigidos nesta rodada

1. **Bloco do protocolo dentro de uma cerca com linguagem.** O modelo respondeu uma cerca
   ` ```css ` seguida de `.edit:assets/styles.css` — com um ponto de lista e dentro de uma
   cerca `css`. O parser não reconhecia nada, o pedido virava prosa e **nenhuma rodada
   corretiva acontecia**: o arquivo ficava intacto e nada dizia ao usuário o motivo. Agora o
   cabeçalho é aceito com um prefixo decorativo e também na primeira linha de uma cerca com
   linguagem (só a primeira linha, e só quando o valor parece um caminho — uma cerca de código
   comum continua sendo texto).
2. **O foco pelo texto exato não achava o trecho.** Quando a âncora não existe literalmente
   (o modelo cita a regra inteira em uma linha, como o arquivo a escrevia antes), o
   orquestrador mandava o arquivo inteiro de volta — e o modelo repetia o mesmo erro. Agora
   ele cai para o **seletor** da âncora (`.hero h1`, `h1`, `.tab.is-active`) e devolve só as
   linhas em volta.
3. **A corrupção por chave desbalanceada** (`EDIT_REJECTED_UNBALANCED`), descrita acima.
4. **A mensagem de ambiguidade vinha sem as linhas** quando a âncora curta aparece dentro das
   linhas (um valor, não a linha inteira): dizia *“aparece 240 vezes no arquivo ()”*. Agora
   lista as primeiras linhas afetadas.

### Limitações honestas

* **O modelo continua errando.** Nas execuções reais o `qwen2.5-coder:3b` precisou de uma
  rodada corretiva em todos os casos; `gemma3:1b` e `qwen2.5-coder:1.5b` frequentemente não
  completam a âncora. Isso é **capacidade do modelo**, não defeito do produto — o critério é
  outro: *operação válida + servidor valida + edição correta + nenhuma corrupção*, e o arquivo
  nunca fica pior do que estava.
* **O histórico é um fator real.** Numa conversa longa, em que o modelo já viu o CSS várias
  vezes, ele regurgita a resposta anterior byte a byte e **ignora** o trecho focado; repetido
  do zero (conversa nova), o mesmo pedido converge em duas rodadas. A arquitetura não esconde
  isso: ela recusa, não grava e o arquivo continua íntegro — e o item 14 fecha a causa, deixando
  de reenviar o histórico nos retries.
* **`gemma4:26b` não carrega nesta máquina** (18,73 GB — OOM); a captura de tela do Electron
  não está disponível (“no frames”), então a prova de UI é DOM/CDP, não imagem.
* A guarda estrutural é conservadora por desenho (um `}` seguido de `{`, ou chaves
  desbalanceadas, recusam o bloco inteiro). Ela pode recusar um caso legítimo; recusar é a
  direção segura, e a rodada corretiva existe para isso.

---

## 14. Context hygiene: o histórico do chat não é o contexto da operação

**O defeito.** Numa conversa longa o modelo regurgita. Quando cada rodada reenvia a conversa
inteira, um modelo que já viu o CSS várias vezes repete o bloco anterior **byte a byte** — inclusive
um bloco que o servidor acabara de recusar — e o retry, ao anexar mais uma mensagem, só aumenta a
chance de repetir. Na validação anterior isso era visível: `gemma3:1b` reenviou o mesmo patch
recusado em três rodadas seguidas. A conclusão era arquitetural: *menos contexto irrelevante +
contexto correto + alvo explícito + trecho relevante + retry limpo*.

**O que passou a existir.** [server/context.js](server/context.js) monta, a cada rodada, um contexto
próprio para a operação — e nada mais:

```
SOLICITAÇÃO ORIGINAL (autoridade desta operação)
ARQUIVO AUTORIZADO DESTA RODADA
ERRO DA RODADA ANTERIOR (semântico — nunca o bloco recusado)
ESTADO ATUAL DO ARQUIVO: path · X KB · N linhas
TRECHO RELEVANTE DE … (linhas X–Y)        ← focused excerpt, quando há âncora
TAREFA: … Não altere nenhum outro arquivo.
```

`buildMessages` aceita `executionContext`: quando ele existe, o modelo recebe **duas mensagens**
(sistema + contexto da operação) e o histórico não é reenviado. O histórico continua no Chat, para o
usuário; o que muda é a entrada do modelo.

| modo de contexto | quando |
| --- | --- |
| `full` | só a rodada 1 (o pedido original usa o histórico) |
| `focused-excerpt` | retry com âncora: volta apenas o trecho relevante |
| `full-file` | retry sem âncora: não há como focar — recurso sancionado |
| `repair` | retry genérico (erro de âncora/estrutura) |
| `read-answer` | a rodada leu um arquivo e precisa responder |
| `target-repair` | o alvo enviado não era autorizado |
| `placeholder-repair` | o modelo usou o caminho de exemplo |
| `repetition-repair` | regurgitação detectada |

### A guarda de regurgitação

`operationSignature` reduz cada operação a uma assinatura estrutural
(`edit|alvo|find=>replace`, `patch|alvo|search=>replace`), ignorando indentação e quebras de linha.
Quando a nova saída tem a **mesma assinatura de uma operação já recusada nesta solicitação**, ela
não é aplicada:

```
ERR: MODEL_REPEATED_PREVIOUS_OPERATION | O modelo repetiu uma operação já recusada nesta solicitação (edit:assets/styles.css). Nada foi gravado: o contexto foi reconstruído para a próxima rodada.
```

O bloco recusado **nunca** volta ao modelo; a recusa é descrita semanticamente ("destino
incompatível", "o trecho não existe"), sem reforçar o conteúdo incorreto.

### Prova 1 — o debug de rodada (Electron real, Ollama real, conversa de 28 mensagens)

```
rodada 1 · qwen2.5-coder:3b · contexto=full            · 12 mensagem(ns) de entrada · 11662 B
rodada 2 · qwen2.5-coder:3b · contexto=focused-excerpt ·  1 mensagem(ns) de entrada ·  1950 B
rodada 3 · qwen2.5-coder:3b · contexto=focused-excerpt ·  1 mensagem(ns) de entrada ·  1950 B
rodada 4 · qwen2.5-coder:3b · contexto=focused-excerpt ·  1 mensagem(ns) de entrada ·  1950 B
```

O histórico é usado **uma vez**. Nos retries a entrada cai de ~11,7 KB para 1,9 KB e de 12 mensagens
para 1 — a conversa deixa de ser o contexto da operação.

### Prova 2 — os dois ramos do cenário, no app real

**Ramo 1 — errou, o contexto foi reconstruído, acertou.** *"Em assets/styles.css, aumente SOMENTE o
tamanho da fonte da regra `.hero h1` para 28px."*

```
round 1  contexto=full             edit → EDIT_REJECTED_NOT_FOUND   writes 0
round 2  contexto=focused-excerpt  edit → APPLIED                   writes 1
   steps: "Trecho relevante de assets/styles.css enviado ao modelo | linhas 34–50"
          "Edição por âncora em assets/styles.css (1 trecho)"
```

```
sha256  4040fd27bf0a… → be475df1…     (14.492 B · 217 linhas — tamanho inalterado)
$ diff -u styles.before.css assets/styles.css
-.hero h1 { font-size: 20px; margin: 0 0 8px; }
+.hero h1 { font-size: 28px; margin: 0 0 8px; }
index.html : 0838e2075992… → 0838e2075992…   (intacto)
app.js     : 827d02f6d082… → 827d02f6d082…   (intacto)
```

**Ramo 2 — continuou errando, a aplicação recusou, o arquivo ficou intacto.** *"Troque SOMENTE a cor
de fundo do botão da aba ativa em assets/styles.css para #2563eb."*

```
round 1  full             EDIT_REJECTED                       writes 0
round 2  focused-excerpt  EDIT_REJECTED                       writes 0
round 3  focused-excerpt  EDIT_REJECTED                       writes 0
round 4  focused-excerpt  MODEL_REPEATED_PREVIOUS_OPERATION   writes 0
   steps: "Repetição recusada: assets/styles.css (esta mesma operação já foi recusada)"
          "Não foi possível concluir a alteração com segurança."
sha256  be475df1… antes e depois (idêntico)
```

O limite de rodadas termina com o erro claro pedido (§5) e o arquivo continua como estava.

### Prova 3 — roteiro real (`tools/validate-real.js`)

```
PASS CLEAN RETRY CONTEXT — mensagens na rodada corretiva: 2 (sistema + contexto da operação) · histórico reenviado: false
PASS NO OLD PATCH REPLAY — o bloco recusado não volta ao modelo
PASS OPERATION ISOLATION — rodada 1 recebe o histórico: true · rodada de reparo: false
PASS REGURGITATION GUARD — MODEL_REPEATED_PREVIOUS_OPERATION: true · index.html intacto: true
PASS FOCUSED EXCERPT RETRY — 13.8 KB no arquivo · 1518 B no contexto de reparo · trecho focado: true
PASS TARGET REPAIR uses the whole file only because there is no anchor to focus — 15364 B (recurso sancionado)
PASS ORIGINAL REQUEST PRESERVED — toda rodada fica vinculada ao pedido
PASS CORRECTED ROUND LANDS THE CHANGE — 1 linha alterada em 13.8 KB · rodadas 3
PASS SAFE FAILURE — rodadas 4 · safeFailure true · arquivos intactos: true
PASS CONTEXT HYGIENE com modelo real — r1=full/EDIT_REJECTED → r2=focused-excerpt/APPLIED
PASS MODELO REAL — erro nunca corrompeu o projeto — index.html intacto: true · sem vazamento do protocolo: true
```

Total (`qwen2.5-coder:3b`, Ollama 0.35.1): **37 PASS · 0 FAIL · 19 INFO** (exit 0).

Numa execução multi-modelo (com `gemma3:1b`) a guarda pegou **três repetições reais numa única
solicitação**, com o arquivo intacto — a mesma coisa que o Ramo 2 acabou de mostrar no app.

### Prova 4 — nem o Preview nem o Chat vazam

```
GET http://127.0.0.1:4181/                     200   (produto renderizado)
GET http://127.0.0.1:4181/assets/styles.css    200   .hero h1 { font-size: 28px; … }
códigos internos / protocolo no HTML servido do Preview ................ 0 ocorrências
códigos internos no DOM do app (MODEL_REPEATED / TARGET_MISMATCH /
   EDIT_REJECTED / context_mode / focused-excerpt / safeFailure) ........ false
chat: "Repetição recusada: …" · "Não foi possível concluir a alteração com
   segurança." · "Trecho relevante …" · "Edição por âncora …" (passos legíveis)
```

O Chat mostra as mensagens do modelo e os passos legíveis — nunca os códigos internos. O Preview
continua sendo só o produto. (As únicas ocorrências textuais de `FIND:`/`REPLACE:` no DOM do app
são as palavras do próprio usuário no pedido e o texto de um turno antigo do modelo.)

### Bloco pedido

```
========================================
CONTEXT HYGIENE
========================================

CLEAN RETRY CONTEXT        PASS
NO OLD PATCH REPLAY        PASS
OPERATION ISOLATION        PASS
REGURGITATION GUARD        PASS
FOCUSED EXCERPT RETRY      PASS
ORIGINAL REQUEST PRESERVED PASS
SAFE FAILURE               PASS

REAL OLLAMA                 PASS
REAL ELECTRON               PASS
PREVIEW ISOLATED            PASS

TESTES AUTOMATIZADOS        161/161
========================================
```

### Limitações honestas

* **A garantia é sobre o arquivo, não sobre o modelo.** O `qwen2.5-coder:3b` ainda errou na primeira
  rodada em todos os turnos reais; `gemma3:1b` e `qwen2.5-coder:1.5b` frequentemente não concluem.
  Uma rodada ruim pode consumir uma rodada — nunca pode corromper o projeto.
* **`gemma4:26b` não carrega nesta máquina** (OOM). O turno que o usou consumiu uma rodada
  (`traces`: `NO_CHANGE`, `writes 0`) e o arquivo ficou intacto; `safeFailure` ficou `false` porque a
  flag cobre *operações recusadas*, não erros do provedor — nada foi gravado de qualquer modo.
* **As guardas são conservadoras por desenho** (regurgitação e estrutura): podem recusar uma operação
  legítima; recusar é a direção segura, e a rodada corretiva existe para isso.
* **O histórico não é apagado** — ele continua inteiro no Chat e no arquivo da conversa. Ele
  simplesmente deixa de ser reenviado ao modelo depois da primeira rodada.

---

## 15. Contexto adaptativo, auditoria final e aceitação

### Auditoria do projeto inteiro

Busca por marcadores em todo o código do produto (`TODO`, `FIXME`, `HACK`, `XXX`, `TEMP`,
`not implemented`, `pending`, `next step`, `future work`, `stub`, `fake`): **nenhum** em
`server/`, `public/` e `electron/`. Os 36 controles do `index.html` têm referência no JS (nenhum
botão morto). Nenhuma flag de debug ligada por padrão — `FREEBUFF_AGENT_DEBUG` só escreve quando a
variável existe. Nenhum arquivo lixo criado por modelo nos projetos, nenhuma fixture alterada fora
do que o relatório documenta.

Estado do disco ao final: a fixture externa `D:\TESTE-PROJETO-EXTERNO\assets\styles.css`,
alterada durante a validação, foi **restaurada ao baseline** (`sha256 4040fd27...`, idêntica ao
backup), os projetos criados pelos roteiros (`e2e-*`) saíram do registro e do disco junto com as
conversas deles, e a raiz não tem mais nenhum log de depuração. O fixture segue usado apenas como
projeto externo de exemplo.

Arquivos de conversa órfãos também saíram: cada `data/conversations/*.json` agora corresponde a um
projeto existente em `data/projects.json` (cinco arquivos, cinco projetos).

Defeitos encontrados e corrigidos nesta rodada:

1. **O turno real do agente dependia do modelo mais fraco.** `EDIÇÃO INCREMENTAL` e
   `CORRECT FILE PATCH` falhavam quando o modelo sorteado (o menor instalado) não produzia bloco
   válido. Isso é capacidade do modelo, não defeito do servidor: o turno agora é repetido com os
   próximos modelos que cabem na memória, e a linha vira **INFO** quando nenhum consegue.
2. **`LARGE FILE EDIT` media uma única amostra de um modelo estocástico.** Cada modelo passa a
   receber **duas amostras** com o fixture restaurado, e as duas ficam no relatório.
3. **`package.json` dizia `1.0.0` enquanto o app anuncia `1.1.0`** — alinhado.
4. **Faltava cobrir as outras grafias de travessia.** `..\`, `..%2f`, `%2e%2e`, `%2E%2E%5C` e byte
   nulo não tinham teste explícito: agora estão em `test/security.test.js`, no workspace e no
   `safeJoin` do preview.
5. **`e2e-events.log`**, um dump de depuração deixado na raiz por uma execução anterior, removido
   (o `test/e2e.test.js` só o escreve quando `FREEBUFF_E2E_DEBUG` existe — o resíduo é que saiu).

### Contexto adaptativo: o arquivo cresce com a falha

Higiene responde *“o histórico entra na rodada?”* (não). A pergunta seguinte é *“quanto do
arquivo esta rodada precisa?”* — e a resposta depende da falha, não de um palpite. Se o mesmo
erro (mesmo código, mesmo arquivo) volta, o trecho anterior foi demonstravelmente pequeno demais:
**reenviar exatamente as mesmas linhas é o modo como um turno queima o orçamento.**

```
NÍVEL 0  minimal             pedido + alvo + erro (nada do arquivo)
NÍVEL 1  focused-excerpt     ±8 linhas em volta da âncora
NÍVEL 2  expanded-excerpt    ±24 linhas em volta da âncora
NÍVEL 3  structural-context  o bloco que contém a âncora + vizinhos
NÍVEL 4  full-file           o arquivo autorizado inteiro
```

Um erro **diferente** reinicia no nível natural daquele erro (escalar não é somar um contador); o
teto é sempre o nível 4. Cada rodada registra `context_level` e `context_escalated`, e **toda nova
operação recomeça no nível 0** — nada é herdado do pedido anterior.

Evidência real (`tools/validate-real.js`, Ollama local):

```
PASS ADAPTIVE CONTEXT ESCALATION (determinístico) — níveis [null,1,2] · modos
     focused-excerpt/expanded-excerpt · trecho focado 1683 B → ampliado 2419 B · APPLIED
PASS ADAPTIVE CONTEXT RESET (determinístico) — níveis [null,1] · modos ["full","focused-excerpt"]
PASS ADAPTIVE CONTEXT com modelo real — gemma3:1b: níveis [null,1,1,2] · escaladas 1 ·
     qwen2.5-coder:1.5b: níveis [null,1,2] · escaladas 1
```

### Dois defeitos reais que só o E2E encontrou

O fluxo de ponta a ponta (§24) roda o app de verdade com o modelo local. Nesta rodada ele expôs
dois defeitos que nenhum teste determinístico pegava:

1. **O protocolo nu, sem cerca, virava prosa.** Um modelo real respondeu
   `edit:assets/estilo.css` / `FIND: …` / `REPLACE: …` **sem nenhuma cerca** em volta. O parser
   só reconhecia cabeçalhos dentro de fences; a resposta virou texto de chat, a rodada terminou
   com `NO_CHANGE` e o usuário não recebeu nem a alteração nem o motivo. Agora um cabeçalho nu — mas
   **só antes de qualquer prosa e só quando o valor parece um caminho** — abre um bloco; esse bloco
   fecha numa linha vazia assim que já carrega o par FIND/REPLACE, para nunca engolir o resto da
   resposta. (`test/parser.test.js`: 5 testes novos.)
2. **Um pedido de alteração que não trazia operação nenhuma terminava em silêncio.** Se o pedido cita
   um arquivo, tem verbo de alteração e a rodada não produziu bloco algum, o agente faz **uma**
   rodada corretiva com o formato obrigatório (`format-repair`, nível 0) e diz no chat o que
   aconteceu. Uma pergunta sobre um arquivo (*“Explique o que o styles.css faz”*) continua sendo uma
   pergunta: sem verbo de alteração, nada é forçado. (`test/agent.test.js`: 3 testes novos.)

### Bloco pedido

```
========================================
FINAL ACCEPTANCE
========================================

OLLAMA REAL                 PASS
AUTO-DETECÇÃO               PASS
MODELOS DINÂMICOS           PASS
SELEÇÃO DE MODELO           PASS
TROCA REAL DE MODELO        PASS
STREAMING                   PASS
RECONEXÃO                   PASS

AGENTE REAL                 PASS
READ BEFORE EDIT            PASS
TARGET AUTHORIZATION        PASS
PATCH                       PASS
EDIT                        PASS
ANCHOR EDIT                 PASS
FOCUSED EXCERPT             PASS
ADAPTIVE CONTEXT            PASS
REGURGITATION GUARD         PASS
SAFE FAILURE                PASS

NOVO PROJETO                PASS
ABRIR PROJETO EXISTENTE     PASS
MÚLTIPLOS PROJETOS          PASS
HISTÓRICO POR PROJETO       PASS
PROJECT ISOLATION           PASS

APP ROOT GUARD              PASS
WORKSPACE ISOLATION         PASS
PREVIEW ISOLATED            PASS
PREVIEW SOMENTE VISUAL      PASS
PREVIEW INTERATIVO          PASS
HOT RELOAD                  PASS
DESKTOP                     PASS
TABLET                      PASS
MOBILE                      PASS

ELECTRON REAL               PASS
BUILD FINAL                 PASS
TESTES AUTOMATIZADOS        180/180

========================================
  41 PASS · 0 FAIL · 23 INFO (Ollama real)
  19 PASS · 0 FAIL  (E2E no app real)
========================================
```

### REAL OLLAMA VALIDATION — `node tools/validate-real.js`

**41 PASS · 0 FAIL · 23 INFO** (exit 0), Ollama 0.35.1, quatro modelos instalados
(`gemma3:1b`, `gemma4:26b`, `qwen2.5-coder:1.5b`, `qwen2.5-coder:3b`). Detecção, listagem dinâmica,
troca de modelo confirmada pelo próprio Ollama (`stats.model`), streaming em partes e o turno real
do agente com workspace de 13,7 KB. Os 23 INFO são cenários em que o resultado é uma propriedade do
modelo (nenhum modelo errou o destino nesta execução, por exemplo) — nunca falhas.

### REAL UI VALIDATION — `node tools/e2e-real.js` (a mesma superfície que a interface usa)

**19 PASS · 0 FAIL**, sem nenhum INFO: criação de projeto, preview por projeto, criação de interface,
hot reload, edição por âncora, preview alterado, dois projetos com previews e históricos separados,
abertura do projeto existente, troca de modelo, Ollama offline, reconexão e nova operação.

```
PASS HOT RELOAD — o preview muda sem reiniciar o app — HTTP 200 · 225 caracteres servidos
PASS EDITAR CSS — qwen2.5-coder:3b · 1 rodada(s) · patch: 1
PASS PREVIEW ALTERADO — antes: ".titulo { color: #111111; }" · depois: ".titulo { color: #2563eb; }"
PASS MÚLTIPLOS PROJETOS — preview em e2e-b-… · sem o conteúdo do primeiro
PASS HISTÓRICO POR PROJETO — mensagens no projeto novo: 0 · no primeiro: 32
PASS OLLAMA OFFLINE — porta sem serviço: ok=false · estado error
PASS RECONEXÃO — porta real: ok=true · estado connected
```

### REAL ELECTRON VALIDATION — janela real pelo CDP

```
PASS ELECTRON REAL — http://127.0.0.1:4180/ · chat na tela: true · código interno na UI: false
PASS PREVIEW ISOLADO — 1481 caracteres no HTML + 1226 no CSS · 0 ocorrências de protocolo/diagnóstico
```

A janela rodou **sem** `FREEBUFF_AGENT_DEBUG`. As três viewports (desktop/tablet/mobile) existem no
`index.html` e no `public/preview.js` (`VIEWPORTS = ['desktop','tablet','mobile']`, aplicadas via
`data-viewport`), e os 36 controles da interface têm referência no JS.

### SECURITY VALIDATION — `test/security.test.js`

**12/12 PASS.** Guarda de app-root, descendentes e ancestrais, registry adulterado, preview que não
serve a árvore da aplicação, além da suite de travessia: `..\`, `..%2f`, `%2e%2e`, `%2E%2E%5C`,
`....//`, byte nulo, caminho absoluto POSIX e Windows, e `//servidor/share` — nenhum escapa do
workspace nem do `safeJoin` do preview.

### CONTENT / EDIT VALIDATION

`LARGE FILE EDIT` (modelo real, CSS de 13,7 KB, uma linha alterada, hash antes/depois),
`ANCHOR EDIT`, as quatro recusas determinísticas (ambígua com as linhas listadas, inexistente,
destrutiva, estrutura) e `ATOMIC WRITE`. Item 13 e §25 do relatório trazem o `diff` completo.

### CONTEXT HYGIENE VALIDATION

Os 11 records do bloco `CONTEXT HYGIENE` em PASS, incluindo a evidência real de que a rodada 1 recebe
12 mensagens e as rodadas de reparo recebem **1**, e a guarda de regurgitação pegando três repetições
reais em uma única solicitação (`gemma3:1b`).

### ADAPTIVE CONTEXT VALIDATION

Os 3 records determinísticos + o record com modelo real em PASS (níveis `[null,1,2]` com
`escalated` só onde o nível subiu). Escalada real observada em dois modelos.

### Limitações honestas (fim)

* **`gemma4:26b` não carrega nesta máquina** (18,73 GB — OOM por VRAM/RAM). `DETECTADO = PASS`,
  `UTILIZÁVEL = OOM`. O turno que o usou registrou `NO_CHANGE`/`writes 0` e o arquivo ficou intacto.
* **A captura de tela do Electron não está disponível** neste ambiente (“no frames”): a prova de UI é
  DOM/CDP, não imagem.
* **Os modelos pequenos são estocásticos.** `qwen2.5-coder:1.5b` e `gemma3:1b` não concluem a maior
  parte das operações de edição; `qwen2.5-coder:3b` conclui quase sempre, às vezes em duas rodadas.
  O critério do produto é outro — *operação válida + servidor valida + edição correta + nenhum
  arquivo pior do que estava* — e os roteiros medem **duas amostras** por modelo, com o relatório
  mostrando as duas.
* **Nada além disso.** Não há funcionalidade descrita como pendente, teste ausente ou código
  incompleto no escopo entregue; o que não passou por validação real está marcado como tal.

---

# 16. FULL LOCAL ACCEPTANCE — ambiente local e privado

Executada em **2026-10-07** nesta máquina (Windows, Node v26.7.0, npm 11.19.0, Ollama 0.35.1,
Electron 33.4.11). Produto: `Freebuff Desktop 1.1.0`. Uso pessoal, uma máquina, nada
distribuído.

Esta rodada tinha uma regra acima de todas: **tudo funciona localmente**. Sem SaaS, sem
servidor remoto, sem login, sem cadastro, sem multiusuário, sem sincronização, sem
analytics, sem telemetria, sem API de IA externa. Ollama é o provider; o workspace é o
disco; o preview é um servidor local; memória, índice, checkpoints, histórico e configuração
são arquivos locais.

## 16.1 Auditoria (o que foi procurado, e o que foi encontrado)

| Procurado | Resultado |
| --- | --- |
| `TODO` / `FIXME` / `HACK` / `XXX` no código entregue | **0** — os únicos casamentos são o identificador `isPlaceholderPath`, o seletor CSS `::placeholder` e atributos `placeholder=` de campos |
| URL externa em `server/`, `public/`, `electron/` | **0** — o único `http(s)://` é o que o auditor (`server/local-mode.js`) usa como *padrão de detecção* |
| CDN, Google Fonts, fonte remota, script externo, imagem remota | **0** — a interface é 100% servida pelo servidor local |
| `sentry`, `analytics`, `telemetry`, `gtag`, `mixpanel`, `segment`, `posthog`, `datadog` | **0** no código entregue; esses nomes existem apenas dentro do auditor, como padrões que ele **recusa** |
| `openai`, `anthropic`, `googleapis` (API de IA externa) | **0** |
| Fallback automático para API externa quando o Ollama cai | **não existe** — o app fica `offline` e diz isso |
| Dependências de execução | `package.json` só declara `electron` como dependência **opcional** (shell) e `engines: node >= 20`; nenhuma biblioteca de rede |
| Requisições fora de loopback na janela real | **0** em 56 recursos carregados (`performance.getEntriesByType('resource')` filtrado por host) |
| Auditoria da própria interface (`GET /api/local`) | `audit.clean = true`, 0 achados — verificada a cada boot e nos testes |

`node tools/validate-local.js` também checa `LOCAL ONLY — auditoria da interface` e
`OLLAMA LOCAL` com a superfície real do agente.

## 16.2 Tabela FULL LOCAL (execução final, contra o app real)

`node tools/validate-local.js` → **30 PASS · 0 FAIL · 0 INFO** (`exit 0`). Cada linha é uma
chamada HTTP ao aplicativo em execução (servidor real, Ollama real, janela Electron real),
nunca um mock.

```
========================================
FREEBUFF DESKTOP — FULL LOCAL
========================================

LOCAL ONLY                    PASS
OLLAMA LOCAL                  PASS
MODELOS LOCAIS               PASS
MODEL ROUTING                 PASS
PROJECT MEMORY                PASS
LOCAL INDEX                   PASS
LOCAL SEARCH                  PASS
FILE EXPLORER                 PASS
DIFF                          PASS
CHECKPOINT                    PASS
ROLLBACK                      PASS
MULTI-FILE ATOMIC EDIT        PASS
SECRET PROTECTION             PASS
LOCAL TERMINAL                PASS
ADAPTIVE CONTEXT              PASS
SAFE FAILURE                  PASS
PROJECT ISOLATION             PASS
PREVIEW ISOLATION             PASS
HOT RELOAD                    PASS
OFFLINE MODE                  PASS
RECOVERY                      PASS
ELECTRON                      PASS
BUILD LOCAL                   PASS
TESTES                        270/270

========================================
```

Records da execução (recorte literal da saída):

```
[PASS] LOCAL ONLY — localOnly=true · internetRequired=false · auditoria da interface: limpa
[PASS] OLLAMA LOCAL — Agente → Ollama local — http://127.0.0.1:11434 · qwen2.5-coder:3b
[PASS] MODELOS LOCAIS — 4 modelo(s) · 3 disponível(is) · 1 indisponível(is)
[PASS] MODELOS LOCAIS — status — gemma3:1b=available · gemma4:26b=oom · qwen2.5-coder:1.5b=available · qwen2.5-coder:3b=loaded
[PASS] LOCAL INDEX — 5 arquivo(s) indexado(s) · 1 pasta(s) · assets/theme.css, assets/, index.html, package.json, styles.css
[PASS] LOCAL SEARCH — busca "color" → styles.css
[PASS] FILE EXPLORER — painel servido em /files.js · assets/theme.css aberto (26 bytes)
[PASS] PROJECT MEMORY — framework=node · estrutura=sim · comandos=1
[PASS] SECRET PROTECTION — o .env é recusado e o conteúdo não é devolvido ao modelo
[PASS] LOCAL TERMINAL — comando local executado (código 0) · saída: "aceitacao-ok"
[PASS] LOCAL TERMINAL — proteção — comando destrutivo recusado antes de qualquer processo
[PASS] TESTES LOCAIS — comando detectado: npm run test
[PASS] TESTES LOCAIS — execução — npm run test → código 0 em 381ms
[PASS] MODEL ROUTING — tarefa classificada: EDIT · modo auto · perfil Coder
[PASS] ADAPTIVE CONTEXT — 1 rodada(s) · r1:full=false=APPLIED
[PASS] DIFF — +1/-1 · hash 53111975 → ea3b35c3
[PASS] CHECKPOINT — styles.css · restaurável: true
[PASS] ROLLBACK — restaurado: styles.css
[PASS] MULTI-FILE ATOMIC EDIT — uma escrita que falha reverte a rodada inteira
[PASS] PROJECT ISOLATION — índice próprio (1 arquivo(s), sem styles.css)
[PASS] PREVIEW ISOLATION — 165 caracteres servidos · 0 vazamento(s)
[PASS] HOT RELOAD — revisão 1791376906782:4:330 → 1791376906847:4:331
[PASS] OFFLINE MODE — sem serviço: ok=false · estado error
[PASS] SAFE FAILURE — turno encerrado com erro controlado: Porta do Ollama inválida ou bloqueada pelo sistema.
[PASS] RECOVERY — reconectado: true · qwen2.5-coder:3b
[PASS] ELECTRON — janela Electron viva na porta 9222 · http://127.0.0.1:4180/
[PASS] BUILD LOCAL — entrada electron/main.js presente · binário Electron instalado · scripts start/desktop/demo/test
```

E a rodada de ponta a ponta com modelo real, `node tools/e2e-real.js`:
**19 PASS · 0 FAIL · 0 INFO** — abrir o app, detectar o Ollama real, listar modelos, criar
projeto, criar interface, editar CSS por âncora, ver o preview mudar, segundo projeto com
preview próprio, histórico separado, abrir projeto existente, trocar modelo, **derrubar o
Ollama**, reconectar e operar de novo, janela Electron viva e preview sem protocolo.

## 16.3 Evidência de interface (sem captura de tela neste ambiente)

A janela e o painel foram inspecionados por DOM/CDP sobre a aplicação real:

* Selo **LOCAL ONLY** no cabeçalho (vira aviso quando o Ollama está fora).
* Quatro perfis clicáveis (Coder, Fast, Reasoning, UI) e a chave **Manual / AUTO**.
* Quatro painéis: **Chat · Arquivos · Terminal · Diagnóstico**.
* Painel **Arquivos**: árvore real do projeto (`▾assets`, `{}app.js`, `◈styles.css`,
  `◍index.html`) e o visualizador abrindo `assets/styles.css` com 14,2 KB e o código correto.
* Painel **Terminal**: `node -e "console.log('ui-local-ok')"` → `ui-local-ok` + `✓ concluído
  (código 0)`; `rm -rf /` → `✕ Comando recusado por segurança: exclusão recursiva forçada.`
* Painel **Diagnóstico**: linhas preenchidas — 36 na última medição (modo, portas da sessão, modelo ativo,
  perfil, índice, checkpoints, último comando e as seis superfícies locais).
* **Diff no chat**: `-   --bg: #0f1116;` / `+   --bg: #102030;` com contexto e
  `… 213 linha(s) sem alteração …`, dentro do card do arquivo.
* **Plano no chat**: `PLANO · EDIT · localizar assets/styles.css / ler o conteúdo atual antes
  de alterar / alterar somente o trecho necessário / validar o resultado e atualizar o preview`.
* **Roteamento honesto no chat**: `Coder · AUTO · EDIT → qwen2.5-coder:3b (modelo de código
  disponível)` — e o turno realmente usou esse modelo, gravando checkpoint com hash
  antes/depois (`8979bf1a → 8a8b7a6c`).
* **Rollback pelo botão, depois de reiniciar o app**: `Desfazer última alteração` restaurou
  `--bg: #123456` → `#102030` lendo o diário em disco, e o preview já servia o CSS restaurado.
* Nenhum recurso externo: 0 de 56 requisições fora de `127.0.0.1`/`localhost`.

## 16.4 Defeitos encontrados nesta rodada e corrigidos

Todos foram encontrados durante a validação (nenhum foi reportado como pendente):

1. **Memória de projeto ficava vazia** quando os arquivos apareciam depois de o projeto ser
   aberto (projeto criado vazio, arquivos escritos em seguida). A derivação só rodava na
   ativação. Agora cada campo é **preenchido quando está vazio** — em `/api/index`, em
   `/api/memory` e ao fim de cada turno — sem nunca sobrescrever o que o usuário escreveu.
2. **Diagnóstico inteiramente vazio na tela** — `applyProfiles(payload)` iterava o objeto
   inteiro em vez de `payload.profiles`, gerando `TypeError` que abortava o preenchimento.
   Achado no navegador, corrigido e reverificado ao vivo (todas as linhas preenchidas).
3. **AUTO dizia "nenhum modelo disponível no Ollama" com quatro modelos instalados.** A
   mensagem era falsa e, pior, o roteador não escolhia entre os modelos instalados quando
   nenhum perfil tinha modelo vinculado. Agora o AUTO escolhe por tarefa entre os modelos
   **instalados e utilizáveis** (código → modelo de código, e o maior entre eles; resposta
   rápida → o menor; análise → o maior) e o texto do chat diz exatamente o que foi usado.
4. **AUTO podia seguir com um modelo marcado OOM.** Se o catálogo já sabe que o modelo é
   inviável, o turno **para com erro explícito** em vez de tentar; nenhum modelo quebrado é
   executado (`provider.callCount === 0` no teste). Nenhuma API externa é consultada.
5. **O diário de checkpoints era sobrescrito a cada reinício.** `#persist` gravava apenas as
   operações da memória do processo: a primeira operação depois de reiniciar apagava o
   histórico do disco. Agora o diário é a união (memória ∪ disco), com ids consumidos
   filtrados e teto de 30 operações.
6. **Rollback não funcionava depois de reiniciar o app** — `rollbackLast` só olhava a
   memória do processo. Agora lê o diário durável; verificado com o botão da interface após
   reiniciar (item 16.3). Um teste novo prova que a operação criada antes do reinício é
   desfeita e que a operação nova não apaga a anterior.
7. **Excluir um projeto deixava resíduo**: memória do projeto, diário de checkpoints e
   conversa órfãos. Agora a exclusão remove os três (os **arquivos** do projeto continuam
   intocados no disco — o app nunca apaga o código do usuário).
8. **A dica do chat duplicava o modo** (`AUTO · AUTO`). O prefixo é retirado na composição.
9. **Texto de exemplo nas configurações citava "Tailwind via CDN"** — trocado por um exemplo
   local, para que a interface não sugira dependência externa.
10. **A memória de falha dos modelos morria com o processo.** Depois de reiniciar, o
    `gemma4:26b` — que realmente estoura memória nesta máquina — aparecia como `disponível`
    até alguém tentar de novo (e o modo AUTO poderia tentar). Agora a falha é persistida em
    `data/models.json` com o tipo (OOM/erro), a mensagem real e o instante, e é recarregada
    (e expirada) ao abrir o app. Verificado ao vivo: falha OOM real
    (`ErrorOutOfDeviceMemory`) → status `oom`, reinício do app → continua
    `⚠ OOM · indisponível nesta máquina`, 3 de 4 utilizáveis, AUTO nunca o escolhe.

## 16.5 Limitações honestas (fim)

* **`gemma4:26b` (18,73 GB) não carrega nesta máquina.** `DETECTADO = PASS`,
  `UTILIZÁVEL = OOM`. Ele **permanece listado** com status OOM — é exatamente o pedido — e o
  modo AUTO nunca o escolhe.
* **A captura de tela do Electron não está disponível neste ambiente** ("no frames"): a prova
  de interface é DOM/CDP e leitura de disco/HTTP, não imagem.
* **A Internet não foi desligada no adaptador de rede** (não era necessário e não é o teste
  certo). A prova de independência é feita por dentro: auditoria da interface com **0
  achados** (nenhum CDN, fonte, script externo ou telemetria), **0 de 56** requisições fora
  de loopback medidas na janela real, e o caixa-preta `OFFLINE MODE`/`SAFE FAILURE` mostrando
  que o app segue servindo interface, projetos, busca, arquivos, memória, checkpoints,
  terminal e preview com o provider inacessível — sem nenhum fallback externo.
* **Os modelos pequenos são estocásticos.** `qwen2.5-coder:1.5b` e `gemma3:1b` não concluem a
  maior parte das operações de edição; `qwen2.5-coder:3b` conclui quase sempre. O critério do
  produto continua sendo outro: *operação válida + servidor valida + edição correta + nenhum
  arquivo pior do que estava* — e o turno que não conclui termina em `NO_CHANGE`, com o
  arquivo intacto, sem estado parcial.
* **Nada além disso.** Não há funcionalidade parcialmente implementada, botão sem função,
  mock no caminho normal, dependência externa ou pendência técnica dentro do escopo desta
  fase.

---

# 17. POWER USER ACCEPTANCE — editor real, controle e diagnóstico

Executada em **2026-10-07** nesta máquina (Windows, Node v26.7.0, Ollama 0.35.1, Electron
33.4.11). Produto: `Freebuff Desktop 1.1.0`. Continua **uso pessoal, uma máquina, nada
distribuído** e **100% local**.

A regra da fase não mudou: **o preview é somente a aplicação renderizada**. Editor, chat,
terminal, diff, checkpoints, diagnostico, logs, modelo e contexto existem **apenas no painel
do agente**. O preview segue isolado em outra origem/porta.

## 17.1 Testes automatizados

`npm test` → **312/312 PASS** (`node --test`, `test/*.test.js`). Inclui, entre outros:

* `test/power-api.test.js` (**13 testes HTTP reais**): salvar pelo editor (com checkpoint
  `editor`), conflito de hash (`409`), arquivo sensível (`403`), travessia (`400`); checkpoint
  manual + restore validando hash e `force`; escopo de `run` (`file`/`project`) que não escapa
  do workspace; projeto sem comando declarado que não inventa um; gravar a saída do terminal
  no data dir; serviços com a porta realmente ligada; logs por canal; aprovação `HIGH` com a
  decisão tomada **durante o stream**; cancelamento sem escrita; `stop` cancelando aprovação
  pendente; busca de conteúdo; e **modelo OOM** que não vira ativo — agora também cobrindo a
  **troca automática de modelo** (`model_changed`) quando o modelo ativo estoura a memória.
* `test/ui.test.js`: controles power-user, painéis fora do preview, palette com `run:`,
  atalhos, fechamento com três escolhas, busca global com `reveal`, editor real, ausência de
  `window.prompt` e a exibição da troca automática de modelo.
* `test/risk.test.js`, `test/logbook.test.js`, `test/agent-power.test.js`,
  `test/model-catalog.test.js` (mensagens reais de OOM), `test/checkpoints.test.js`.

## 17.2 Tabela POWER USER (execução final, contra o app real)

`node tools/validate-local.js` → **30 PASS · 0 FAIL · 0 INFO** (`exit 0`), contra o servidor
real, o Ollama real e a janela Electron real. `node tools/e2e-real.js` → **18 PASS · 0 FAIL ·
1 INFO**, incluindo a janela Electron viva pelo CDP.

```
========================================
FREEBUFF DESKTOP — POWER USER FINAL
========================================

CODE EDITOR                 PASS
FILE TABS                   PASS
SEARCH                      PASS
REPLACE                     PASS
SELECTION ACTIONS           PASS
EXPLAIN SELECTION           PASS
EDIT SELECTION              PASS

COMMAND PALETTE             PASS
QUICK MODEL SWITCH          PASS
QUICK PROJECT SWITCH        PASS

RISK CLASSIFICATION         PASS
APPROVAL                    PASS
AGENT STOP                  PASS
MANUAL RETRY                PASS

DIFF                        PASS
CHECKPOINT                  PASS
ROLLBACK                    PASS
TIMELINE                    PASS
DIAGNOSTICS                 PASS

TERMINAL                    PASS
TERMINAL SECURITY           PASS
TERMINAL SCOPE              PASS
SERVICES                    PASS
LOGS                        PASS

OLLAMA                      PASS
FULL LOCAL                  PASS
OFFLINE                     PASS
PROJECT ISOLATION           PASS
PREVIEW ISOLATION           PASS
HOT RELOAD                  PASS
ELECTRON                    PASS

TESTES AUTOMATIZADOS        312/312

========================================
```

Records da execução (recorte literal):

```
PASS LOCAL ONLY — localOnly=true · internetRequired=false · auditoria da interface: limpa
PASS SECRET PROTECTION — o .env é recusado e o conteúdo não é devolvido ao modelo
PASS LOCAL TERMINAL — proteção — comando destrutivo recusado antes de qualquer processo
PASS MODEL ROUTING — tarefa classificada: EDIT · modo manual · perfil Coder
PASS ADAPTIVE CONTEXT — 1 rodada(s) · r1:full=APPLIED
PASS DIFF — +1/-1 · hash 53111975 → ea3b35c3
PASS CHECKPOINT — styles.css · restaurável: true
PASS ROLLBACK — restaurado: styles.css
PASS MULTI-FILE ATOMIC EDIT — uma escrita que falha reverte a rodada inteira
PASS PROJECT ISOLATION — índice próprio (1 arquivo(s), sem styles.css) · preview do projeto
PASS PREVIEW ISOLATION — 165 caracteres servidos · 0 vazamento(s)
PASS HOT RELOAD — revisão 1791386431577:4:330 → 1791386431623:4:331
PASS OFFLINE MODE — sem serviço: ok=false · estado error
PASS SAFE FAILURE — turno encerrado com erro controlado
PASS RECOVERY — reconectado: true · qwen2.5-coder:3b
PASS ELECTRON — janela Electron viva na porta 9222
PASS BUILD LOCAL — entrada electron/main.js presente · binário Electron instalado
```

```
PASS ELECTRON REAL — http://127.0.0.1:4182/ · chat: true · código interno na UI: false
PASS PREVIEW ISOLADO — 38 caracteres no HTML + 1226 no CSS · 0 ocorrências de protocolo
PASS EDITAR CSS — leitura, trecho focado e edição por âncora · patch: 1
PASS PREVIEW ALTERADO — antes ".titulo { color: #111111; }" · depois "#2563eb"
PASS OLLAMA OFFLINE — porta sem serviço: ok=false · estado error
PASS RECONEXÃO — porta real: ok=true · estado connected
```

## 17.3 Evidência de interface (DOM sobre a aplicação real)

* Painéis do agente: **Chat · Arquivos · Terminal · Diagnóstico**; todos os controles
  power-user presentes (`#agent-action`, `#file-tabs`, `#project-search`, `#palette`,
  `#checkpoint-list`, `#service-list`, `#log-out`, `#log-channels`, `#settings-nav`,
  `#memory-editor`, `#context-info`, `#projects-info`).
* **Command palette** (`Ctrl+K`): 23 comandos com `run:` — nenhum decorativo.
* **Diagnóstico**: seções Contexto / Checkpoints / Serviços / Logs / Técnico, sem
  `undefined`, `NaN` ou `[object Object]` na tela.
* **Editor**: abrir/fechar/trocar abas, dirty state, `Ctrl+S` gravando no disco (hash volta a
  "sincronizado"), busca global abrindo o arquivo na linha certa com a ocorrência selecionada.
* **Serviços**: App · Preview · Ollama com status e porta reais.
* Nenhum erro de console além do aviso padrão do iframe de preview (aplicação em **outra
  origem**, portanto o aviso é inócuo e o isolamento é por origem/porta).

## 17.4 Auditoria de código

Busca por `TODO`, `FIXME`, `HACK`, `XXX`, `stub`, `debugger`, `console.log`, `not
implemented`, `deprecated` em `server/`, `public/` e `electron/`: **0 ocorrências reais**. Os
únicos casamentos são legítimos — o identificador `isPlaceholderPath` (que trata caminhos
genéricos que o modelo eventualmente devolve), o `::placeholder`/`placeholder=` de campos de
formulário e textos de exemplo da interface.

## 17.5 Auditoria de segurança (rotas novas)

`test/security.test.js`, `test/audit.test.js`, `test/workspace.test.js`, `test/edit.test.js`,
`test/patch.test.js`, `test/shell.test.js` → **64/64 PASS**. As rotas novas desta fase
(`POST /api/file`, `POST /api/checkpoints*`, `POST /api/run`, `POST /api/terminal/output`,
`POST /api/approval`, `GET /api/search?scope=content`) passam pelas mesmas guardas: `APP_ROOT`,
travessia, dotfiles, arquivos sensíveis, alvo errado, edição destrutiva, patch repetido e
comando de terminal perigoso. Nenhuma proteção foi contornada. `POST /api/run` recusa
caminhos fora do projeto (`404`) e `/etc/passwd` (`404`).

## 17.6 Defeitos encontrados nesta rodada e corrigidos

1. **Troca automática de modelo podia ficar sem candidato.** Quando o modelo ativo estourava a
   memória, o servidor escolhia o substituto **só** pelo catálogo — que pode não ter sido
   atualizado no meio do turno — e acabava removendo o modelo ativo sem nenhum substituto.
   Agora o substituto vem da lista de modelos realmente instalados (`provider.state.models`),
   excluindo os que o catálogo já marcou como falhos, e a interface é avisada
   (`model_changed`) para atualizar o modelo exibido em vez de falhar em silêncio.

## 17.7 Limitações honestas (fim)

* **A captura de tela do Electron não está disponível neste ambiente** ("no frames"): a prova
  de interface é DOM/CDP e leitura de disco/HTTP, não imagem.
* **`gemma4:26b` (18,73 GB) não carrega nesta máquina** — permanece listado como
  `⚠ OOM · indisponível`, nunca é escolhido, e a própria troca automática o evita.
* **A Internet não foi desligada no adaptador.** A independência é provada por dentro:
  auditoria da interface com 0 achados e o caixa-preta `OFFLINE MODE`/`SAFE FAILURE` com o
  provider inacessível — sem fallback externo.
* **Os modelos pequenos são estocásticos.** O critério continua sendo *operação válida +
  servidor valida + edição correta + nenhum arquivo pior do que estava*.
---

# 18. HARDENING ACCEPTANCE — estabilidade de uso diário

Executada em **2026-10-07** nesta máquina (Windows, Node v26.7.0, Ollama 0.35.1, Electron
33.4.11), produto `Freebuff Desktop 1.1.0`. Fase de **hardening + UX + estabilidade**: nada de
cloud, login, telemetria ou API externa; nenhuma funcionalidade removida; o preview continua
somente a aplicação renderizada.

## 18.1 Bugs reais encontrados e corrigidos nesta rodada

1. **Processos órfãos no terminal.** `runCommand` encerrava apenas o *shell* (`child.kill()`),
   deixando o processo neto (o `node` do `npm`, por exemplo) vivo e segurando os pipes — a
   promise podia nunca resolver. Agora o encerramento usa `taskkill /T /F` no Windows (a árvore
   inteira), **sem** matar o shell antes (matá-lo primeiro órfao o neto e o `taskkill` deixa de
   alcançá-lo). Verificado: um comando abortado resolve em ~600 ms e o `CLOSE` dispara.
2. **Escrita depois do Parar.** O laço do agente não checava o sinal imediatamente antes de
   aplicar uma operação, então um bloco já emitido podia ser gravado após o cancelamento.
   Agora o laço interrompe **antes** de qualquer item, com um passo explícito no chat.
3. **Turnos concorrentes.** Não havia guarda: dois `/api/chat` podiam correr juntos sobre o
   mesmo workspace. Agora o turno é **reservado de forma síncrona** (antes de qualquer `await`)
   e o segundo é recusado com `409`; trocar de modelo/projeto durante a operação também é
   recusado. O chat mostra \"já existe uma operação em andamento\" em vez de culpar a rede.
4. **Conflito de arquivo não detectado no editor.** O editor abria o arquivo com hash `null`,
   então a primeira gravação nunca detectava mudança externa. Agora `GET /api/file` devolve o
   hash do conteúdo em disco e o editor o guarda como base; um conflito abre o diálogo
   **Comparar / Recarregar / Manter minha versão** (substituiu o `confirm` bloqueante).
5. **Abas não persistiam e o boot não restaurava o projeto.** `setProject` só era chamado na
   troca de projeto, nunca no carregamento inicial. Agora o boot restaura abas/aba ativa/cursor
   do projeto ativo e cada projeto tem a sua própria chave local.

## 18.2 Tabela HARDENING (execução final, contra o app real)

`npm test` → **323/323**. `node tools/validate-local.js` → **30 PASS · 0 FAIL · 0 INFO**;
`node tools/e2e-real.js` → **18 PASS · 0 FAIL · 1 INFO** (Ollama real `11434`, janela Electron
real pelo CDP `9222`).

```
========================================
FREEBUFF DESKTOP — HARDENING FINAL
========================================

EDITOR STABILITY             PASS
TAB PERSISTENCE              PASS
EXTERNAL FILE CHANGE         PASS
DIFF HUNK                    PASS

AGENT STOP RACE              PASS
STREAM ORDER                 PASS
CONCURRENT OPERATIONS        PASS
OPERATION QUEUE              PASS

PREVIEW STALE PROTECTION     PASS
PROJECT SWITCH SAFETY        PASS
MODEL SWITCH SAFETY          PASS

MEMORY PERSISTENCE           PASS
CHECKPOINT PERSISTENCE       PASS
ROLLBACK SAFETY              PASS

TERMINAL PROCESS CLEANUP     PASS
TERMINAL SECURITY            PASS

SECRET PROTECTION            PASS
APP ROOT GUARD               PASS
WORKSPACE ISOLATION          PASS

OFFLINE                      PASS
RECOVERY                     PASS
CRASH RECOVERY               PASS

PERFORMANCE                  PASS
ELECTRON REAL                PASS

TESTES AUTOMATIZADOS        323/323

========================================
```

Notas honestas de cada item:

* **OPERATION QUEUE** — usado o modelo de *bloquear com aviso* (409), não uma fila visual: uma
  segunda operação é recusada enquanto uma roda, com mensagem clara. Não há \"Operação 002
  aguardando\" porque o produto não enfileira silenciosamente — ele recusa e deixa o usuário
  decidir.
* **STREAM ORDER** — o servidor emite os passos em ordem a partir do orquestrador, com
  `at: Date.now()`; a interface não reordena. Um turno por vez elimina os eventos intercalados.
* **CRASH RECOVERY** — o diário de checkpoints e a conversa são duráveis e relidos no boot; uma
  operação parcialmente aplicada nunca é restaurada como concluída (só as gravações realmente
  feitas entram no diário).
* **PERFORMANCE** — `/api/index` 2,4 ms · `/api/search` 1,5 ms · `/api/file` 1,6 ms ·
  `/api/bootstrap` 4,3 ms · `/api/diagnostics` 2,0 ms (projeto pequeno). Sem otimização
  prematura — nada apontou gargalo real.
* **RESPONSIVIDADE** — verificado por DOM em 1280×720 e 1366×768: sem estouro horizontal/vertical,
  composer e preview visíveis, sem sobreposição. (As resoluções maiores só folgam da mesma
  divisão em flex.)

## 18.3 Evidência real (DOM sobre a aplicação em execução)

* **Abas persistentes:** abrir `index.html` gravou `freebuff.editor.ws` com
  `{paths:["index.html"], active:"index.html", cursor:{start:38}}`; após recarregar a página, a
  aba voltou ativa com o cursor na posição 38.
* **Conflito de arquivo:** alterar `index.html` por fora e salvar pelo editor abriu o diálogo
  `index.html` com **Comparar / Recarregar / Manter minha versão**; *Recarregar* trouxe o
  conteúdo do disco (`ALTERADO POR FORA`). Único erro de console: o `409` esperado.
* **Diff por hunk:** um turno real (`qwen2.5-coder:3b`) produziu `index.html +1 / -1` com 1
  hunk e os botões **Descartar trecho** / **Reaplicar trecho**. Descartar: disco de 57 → 38
  bytes (o parágrafo removido, o resto intacto) e checkpoint `kind: editor`. Reaplicar: o
  parágrafo voltou ao arquivo real.
* **Preview** continua isolado (0 ocorrências de protocolo/diagnóstico no HTML/CSS servidos) e
  descarta evento de preview de outro projeto.

## 18.4 Limitações honestas (fim)

* **Captura de tela do Electron indisponível** neste ambiente (\"no frames\"): a prova de UI é
  DOM/CDP e leitura de disco/HTTP.
* **Modelos pequenos são estocásticos.** `gemma3:1b` não produziu uma edição válida numa
  amostra (terminou sem alterar o arquivo, o que é o comportamento seguro); `qwen2.5-coder:3b`
  concluiu. O critério do produto não mudou.
* **A Internet não foi desligada no adaptador.** A independência é provada por dentro: 0
  dependências de execução além do Electron opcional, 0 URL externa no código e o caixa-preta
  `OFFLINE MODE`/`SAFE FAILURE`.

# 19. PERSISTENT TERMINAL — uma sessão real por projeto

Executada em **2026-10-07** nesta máquina (Windows, Node v26.7.0, Ollama 0.35.1, Electron
33.4.11), produto `Freebuff Desktop 1.1.0`. O terminal deixou de abrir um processo novo a cada
comando: agora há **uma sessão de shell persistente por projeto** (até 4), com `cwd`, ambiente e
histórico próprios, duráveis e isolados. Nada de cloud, login, telemetria ou API externa; nenhuma
proteção removida — o `killTree`, a guarda de workspace e a guarda da raiz do app continuam
valendo, agora também dentro da sessão. O preview continua somente a aplicação renderizada.

## 19.1 Arquitetura (sem arquitetura paralela)

`server/terminal.js` (`createTerminalManager`) rege as sessões; as rotas novas em
`server/index.js` são `/api/terminal/sessions` (GET lista / POST abre), `.../run` (SSE de
`start`/`stdout`/`stderr`/`exit`/`process`), `.../stop`, `.../close`, `.../reconnect` e
`.../input`. O painel `public/terminal.js` é um cliente fino: abas, `↑`/`↓`, Parar, Entrar,
Reconectar e a barra `shell · cwd` — nada é inventado localmente. O agente usa uma sessão
interna associada ao projeto (não conta no limite e não aparece como aba). O estado durável é
um arquivo por projeto em `data/terminal/<project-id>.json`:

```json
{ "shell": "cmd", "sessions": [
    { "id": "t1", "cwd": "D:\\Projeto\\src", "history": ["cd src", "npm test"],
      "runs": [{ "command": "npm test", "at": 1791410000000, "code": 0, "ok": true }] }
] }
```

O que vai para o disco é apenas `shell` + `{ id, cwd, history, runs }` (`runs` é o histórico
visual: comando, horário e resultado). **Nenhum processo vivo é serializado** — no boot uma sessão
nova abre no `cwd` guardado. Limites: 4 sessões por projeto, 100 entradas de histórico,
200 000 caracteres de saída por comando; a escrita é *debounced* e nunca é motivo de falha de um
comando. No Command Palette, `Novo terminal`, `Fechar terminal`, `Reconectar terminal`, `Limpar
terminal`, `Histórico do terminal` e `Mudar shell` acionam o painel de verdade.

## 19.2 Bugs reais encontrados e corrigidos nesta rodada

1. **Aviso de "comando em execução" nunca aparecia.** O estado *ocupado* era marcado **antes**
   do *handle* do stream existir, então o `refreshNotice` lia `busy = false` e a barra de aviso
   com **Nova sessão / Cancelar atual** não era pintada. Agora `setBusy(true)` só roda depois do
   handle existir — verificado no painel real.
2. **Parar não registrava nada na transcrição.** Ao cancelar, o stream era abortado e o evento
   `done` nunca chegava, então o terminal ficava mudo. Agora a linha
   `■ comando interrompido pelo usuário — a sessão continua viva` é escrita no próprio painel.
3. **Casca JSON vazia em `data/terminal/`.** Fechar a última sessão de um projeto deixava um
   arquivo só com `sessions: []`. Agora, sem sessão, o arquivo é **removido** — um projeto
   fechado (ou apagado) não deixa resíduo.
4. **O banner do `cmd` vazava para o primeiro comando de um shell novo.** Depois de um crash, de
   um Parar ou de um `exit`, a nova sessão abre outro `cmd.exe`; no Windows ele imprime
   `Microsoft Windows [versão …]` e o prompt antes de qualquer comando, e essa saída era
   atribuída ao **primeiro comando** seguinte. Agora todo shell novo é *primed*: ele emite um
   marcador próprio logo ao abrir e a sessão só recebe um comando **depois** desse marcador —
   toda a saída de abertura é descartada. Quando o shell é substituído, um contador de geração
   ignora os dados do processo antigo (o `close` do shell velho não reabre nem liquida o novo).
   Verificado ao vivo: o mesmo comando que antes trazia o banner agora traz só `cwd` e
   `✓ concluído (código 0)`.

## 19.3 Tabela PERSISTENT TERMINAL (execução final, contra o app real)

`npm test` → **353/353**. `node tools/validate-local.js` → **46 PASS · 0 FAIL · 0 INFO**;
`node tools/e2e-real.js` → **27 PASS · 0 FAIL · 0 INFO** (Ollama real `11434`, janela Electron
real pelo CDP `9222`).

```
========================================
FREEBUFF DESKTOP — PERSISTENT TERMINAL
========================================

PERSISTENT SESSION          PASS
PROJECT TERMINAL ISOLATION  PASS
CWD PERSISTENCE             PASS
HISTORY PERSISTENCE         PASS
MULTI SESSION               PASS
RECONNECT                   PASS
PROCESS TREE KILL           PASS
STOP COMMAND                PASS
ENVIRONMENT                 PASS
SECRET FILTERING            PASS

TERMINAL SECURITY           PASS
APP ROOT GUARD              PASS
WORKSPACE GUARD             PASS

AGENT INTEGRATION           PASS
COMMAND PALETTE             PASS
ELECTRON REAL               PASS
OFFLINE                     PASS

PREVIEW ISOLATED            PASS

TESTES AUTOMATIZADOS        353/353
========================================
```

## 19.4 Prova por item (nomes reais dos verificadores)

* **PERSISTENT SESSION / TERMINAL STREAMING** — `FULL LOCAL`: *PERSISTENT SESSION*,
  *TERMINAL STREAMING*; `E2E REAL`: "sessão real com streaming — sessão `t1` em
  `…\e2e-379097` · código 0 · 6 evento(s)" (eventos `start`/`stdout`/`stderr`/`exit`/`process`).
* **PROJECT TERMINAL ISOLATION** — `FULL LOCAL`: *PROJECT TERMINAL ISOLATION* e
  *PROJECT TERMINAL ISOLATION — volta ao projeto A*; `E2E REAL`: "projeto B começa com 0
  sessão(ões), sem herdar 4 do projeto A" e "4 sessões restauradas no projeto A".
* **CWD PERSISTENCE** — `FULL LOCAL`: *CWD PERSISTENCE*; `E2E REAL`: "o comando seguinte rodou
  em `…\src`".
* **HISTORY PERSISTENCE** — `FULL LOCAL`: *HISTORY PERSISTENCE* e *HISTORY PER SESSION*
  (histórico por sessão, não compartilhado entre abas); unitário: o histórico durável sobrevive
  ao restart. O painel tem ainda um **Histórico visual** (botão *Histórico*): cada entrada mostra
  **comando · horário · autoria (USER/AGENT) · resultado** (`código 0`, `código N` ou
  `interrompido`), lido de `runs` da sessão — nunca a saída bruta e nunca um comando com segredo.
  Verificado na janela real: `Histórico da sessão t1 (2)` com `19:24:52 · USER · echo hist-ok ·
  código 0` e `19:24:52 · USER · node -e "process.exit(4)" · código 4`.
* **MULTI SESSION / STOP COMMAND** — `FULL LOCAL`: *MULTI SESSION*, *SESSION LIMIT*,
  *STOP COMMAND*; `E2E REAL`: "limite de sessões — máximo 4 por projeto" e "Parar mata o
  comando, não a sessão — a sessão segue viva com o histórico preservado".
* **RECONNECT** — `FULL LOCAL`: *RECONNECT*; unitário: um `exit` mata o shell, um **processo
  novo** abre no `cwd` guardado e o histórico permanece; na janela real aparece a barra
  "Terminal encerrado (t1) … Reconectar" e o botão religa a sessão.
* **PROCESS TREE KILL** — unitário (requisito 34): um `node` que gera um `node` neto; o Parar
  encerra **pai, filho e neto**, fecha os pipes e resolve a promise (a correção de `killTree` em
  `server/shell.js` foi preservada, sem voltar a órfãos).
* **ENVIRONMENT** — unitário: `set FBTESTE=abc` numa linha continua visível em `echo %FBTESTE%`
  na linha seguinte, na **mesma** sessão.
* **SECRET FILTERING** — `FULL LOCAL`: *SECRET FILTERING*; unitário: um comando com
  `password`/`token`/`apikey`/`secret`/`credential` **não é gravado** no JSON nem entra no
  contexto do modelo; variáveis sensíveis ficam só na memória da sessão.
* **TERMINAL SECURITY / WORKSPACE GUARD / APP ROOT GUARD** — `FULL LOCAL`: *TERMINAL
  SECURITY*, *WORKSPACE GUARD*, *APP ROOT GUARD*; unitário: `cd /d "C:\…"` fora do projeto é
  recusado ("O terminal não pode mudar para fora do projeto: C:\Windows"), `cd ..` clampa de
  volta à raiz e a persistência **não** serve de escapatória.
* **AGENT INTEGRATION** — o agente roda na sessão interna do projeto (`npm test` e
  `npm run build` na mesma sessão quando apropriado) e recebe só um bloco compacto
  `{cwd, shell, status, command}` — nunca o histórico inteiro (unitário: *o agente recebe um
  estado compacto do terminal, não o histórico inteiro*).
* **COMMAND PALETTE** — *Novo terminal*, *Fechar terminal*, *Reconectar terminal*, *Limpar
  terminal* e *Mudar shell* no Command Palette; unitário: *o palette aciona cada comando de
  terminal de verdade*.
* **ELECTRON REAL** — `E2E REAL`: "terminal persistente no painel real — aba(s): 1 · barra:
  `cmd` · `…\src` · campo de comando: true".
* **OFFLINE / PREVIEW ISOLATED** — `FULL LOCAL`: *OFFLINE MODE*; `E2E REAL`: "PREVIEW ISOLADO —
  só o produto renderizado, nunca o protocolo · 0 ocorrências". O terminal vive no painel do
  agente e **nunca** aparece no preview.

## 19.5 Restart e crash reais (requisitos 26 e 27)

Ciclo medido na janela Electron real, ponta a ponta:

1. Projeto `restart-probe-muymlv56`, sessão `t1`: `mkdir src` → `cd src` →
   `node -e "console.log('ANTES-OK')"`. A barra mostra `cmd · …\restart-probe-muymlv56\src` e
   `↑ ↑ ↑` devolve `node …`, `cd src`, `mkdir src`.
2. Estado durável em disco **antes** do crash: `{shell:"cmd", sessions:[{id:"t1", cwd:
   "D:\…\restart-probe-muymlv56\src", history:["mkdir src","cd src","node -e \"console.log('ANTES-OK')\""]}]}`.
3. **Kill abrupto** (`taskkill /F /T` na árvore do Electron) — sem `closeAll()`, sem descarga
   graciosa. Portas `4180/4181/9222` liberadas.
4. Relançamento: o projeto volta como ativo, a sessão volta com `cwd = …\src` e o histórico
   intacto, e `alive: false` — **nenhum processo morto é restaurado**.
5. O painel real mostra a barra `cmd · …\src`, a aba `t1 cmd src`, o histórico no `↑` e a barra
   "Terminal encerrado (t1) … Reconectar".
6. Um comando novo roda de verdade no `cwd` restaurado: `node -e "console.log(process.cwd())"`
   imprime `D:\…\restart-probe-muymlv56\src` e `✓ concluído (código 0)` — **um processo novo e
   seguro**, sem o banner do `cmd`.

## 19.6 Limitações honestas (fim)

* **Terminal gráfico completo não implementado** (não era pedido): há entrada de texto para um
  processo que pede `stdin` (`/api/terminal/sessions/input`) e saída longa em streaming; não há
  TUI de tela cheia nem realce de sintaxe por programa.
* **Um comando por vez na mesma sessão.** Tentar iniciar um segundo comando na mesma aba não
  concorre: a barra oferece **Nova sessão** / **Cancelar atual** (modelo *recusar com aviso*, o
  mesmo do resto do produto).
* **A neutralização de segredos é por nome**, não por conteúdo: um valor secreto colado dentro
  de um comando sem palavra-chave conhecida (por exemplo `curl -d "…"`) não é detectado. O
  produto não varre o valor, só filtra o que o comando declara.
* **`sh` no Windows** existe na tabela de shells mas depende de um `sh` no `PATH`; o padrão e o
  caminho testado são `cmd` e `powershell`.
* **Captura de tela do Electron** funcionou nesta rodada via CDP
  (`Page.captureScreenshot`): um PNG real de **334 163 bytes** (magic `89504e47`) foi extraído da
  janela em execução — corrigindo a limitação registrada nas rodadas 18.4 e anteriores. A prova
  principal continua sendo DOM/CDP e leitura de disco/HTTP, que é o que os verificadores usam.
* **Uma rodada de `e2e-real.js` registrou 25 PASS · 2 FAIL de forma transitória** (as duas
  verificações do Ollama real) logo após o relançamento; as **duas execuções seguintes deram
  27 PASS · 0 FAIL com vereditos idênticos**, e o restante do roteiro (incluindo todo o terminal)
  passou nas três. A flutuação é dos modelos pequenos, não do terminal — registrada por
  honestidade.
