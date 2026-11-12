# BOT.IA — Assistente pessoal local e ambiente de construção com IA

Aplicação desktop local com dois modos separados: **Conversa**, para perguntas gerais e respostas em streaming, e **Construir**, para desenvolvimento de projetos com agente, workspace e preview isolado. O provedor padrão é **llama.app**, usando o serviço local configurado; **Ollama é uma alternativa explícita**, não um fallback silencioso. O BOT.IA usa o modelo selecionado pelo utilizador e não é o serviço ChatGPT nem declara paridade funcional com ele.

```
┌──────────────────────────────┬────────────────────────────────────────────┐
│  CONSTRUIR / AGENTE          │  PREVIEW DO PROJETO                        │
│  passos · arquivos · diff    │  somente a interface sendo construída      │
│  ---------------------------------------------------------------          │
│  [ Campo de comando        ] │  [Desktop] [Tablet] [Mobile]    [↻]        │
└──────────────────────────────┴────────────────────────────────────────────┘
```

## Como executar

O plano de evolução, as capacidades ainda ausentes e os critérios verificáveis de conclusão estão em [docs/roadmap/chatgpt-level-roadmap.md](docs/roadmap/chatgpt-level-roadmap.md). A documentação histórica de testes deve ser lida como evidência da execução indicada nela, não como resultado automático do commit actual.

```bash
npm start        # abre a interface (servidor local + navegador padrão)
npm run desktop  # janela desktop nativa (usa o Electron, se instalado)
npm run demo     # demonstração com um Ollama simulado (nenhum modelo necessário)
npm test         # suíte principal (.test.js e .test.mjs); o runner apresenta as contagens reais de cada execução
```

Requisitos: Node 22.12+ e, para uso real de IA, o serviço local do **llama.app** deve estar iniciado e acessível no endpoint padrão `http://127.0.0.1:9931`. O BOT.IA não presume que o serviço ou um modelo estejam prontos e não troca silenciosamente de provedor. `LLAMA_APP_BASE_URL` permite configurar o endpoint de loopback.

O provedor alternativo Ollama só fica disponível quando habilitado. Para selecionar apenas Ollama numa instalação, configure `BOTIA_PROVIDERS=ollama`. Para disponibilizar os dois adaptadores, configure `BOTIA_PROVIDERS=llama-app,ollama`; nesse caso, uma escolha de provedor já persistida continua tendo precedência. `BOTIA_PROVIDER` define o provedor padrão do ambiente quando não há outra seleção válida persistida.

Para a janela desktop nativa, use `npm ci` para instalar as dependências fixadas no lockfile. Se o binário do Electron não estiver disponível, `npm run desktop` segue a política de fallback definida pela aplicação. A auditoria de produção exige que o Electron seja verificado e que exista um instalador da versão actual.

## Arquitetura (responsabilidades separadas)

```
CONSTRUIR / AGENTE ──► ORQUESTRADOR ──► PROVIDER LOCAL ATIVO ──► MODELO ESCOLHIDO
 (public/)                   │                      │
                             ▼                      │
                    PROJECT WORKSPACE  ◄──── arquivos escritos
                             │
                             ▼
                       PREVIEW SERVER (porta própria)
                             │
                             ▼
                      PREVIEW RENDERER (iframe, origem separada)
```

| Camada | Arquivo | Responsabilidade |
| --- | --- | --- |
| Interface ativa (chat) | [public/index.html](public/index.html) + [public/chatgpt-shell.js](public/chatgpt-shell.js) | conversa, modos, fluxo de geração e shell carregada em runtime |
| Interface ativa (preview) | [public/index.html](public/index.html) + [server/preview.js](server/preview.js) | viewport, sincronização por revisão, recarga e renderização isolada |
| Modelos e preferências | [public/settings.js](public/settings.js) | seletor de modelos dinâmico, configurações, diagnóstico |
| API do app | [server/index.js](server/index.js) | rotas REST + stream SSE do agente, servidor das duas portas |
| Orquestrador | [server/agent.js](server/agent.js) | prompt, protocolo de arquivos, aplicação das mudanças, etapas |
| Protocolo | [server/parser.js](server/parser.js) | parser incremental dos blocos `file:` / `patch:` / `edit:` / `delete:` |
| Edição por âncora | [server/anchor-edit.js](server/anchor-edit.js) | FIND/REPLACE: localizar, validar unicidade, aplicar e verificar |
| Alvo autorizado | [server/targets.js](server/targets.js) | quais arquivos cada rodada pode alterar |
| Contexto por rodada | [server/context.js](server/context.js) | contexto de reparo limpo, assinatura de operação e escalonamento adaptativo |
| Provider | [server/provider.js](server/provider.js), [server/llama-app.js](server/llama-app.js), [server/ollama.js](server/ollama.js), [server/free-claude-code.js](server/free-claude-code.js) | seleção explícita; llama.app é o padrão, Ollama e FCC são opcionais; FCC usa somente IDs exatos verificados pelo operador via HTTP local |
| Workspace | [server/workspace.js](server/workspace.js) | sandbox de arquivos do projeto |
| Preview | [server/preview.js](server/preview.js) | servidor estático isolado do workspace |
| Configuração | [server/config.js](server/config.js) | preferências persistidas e padrões |
| LOCAL ONLY | [server/local-mode.js](server/local-mode.js) | audita a interface e prova que nenhuma superfície depende da Internet |
| Segredos | [server/secrets.js](server/secrets.js) | identifica, recusa e redige credenciais antes de qualquer prompt |
| Task routing | [server/routing.js](server/routing.js) | classifica a tarefa, monta o plano e escolhe o modelo local |
| Catálogo de modelos | [server/model-catalog.js](server/model-catalog.js) | instalado / disponível / carregado / OOM / erro, com TTL de falha |
| Perfis | [server/profiles.js](server/profiles.js) | Coder / Fast / Reasoning / UI e os modos MANUAL e AUTO |
| Memória | [server/memory.js](server/memory.js) | 4 camadas: global, projeto, conversa e operação (nunca persistida) |
| Índice do projeto | [server/project-index.js](server/project-index.js) | símbolos, seletores, dependências, rotas e busca local com cache |
| Checkpoints | [server/checkpoints.js](server/checkpoints.js) | snapshot, rollback por hash e edição multi-arquivo atômica |
| Terminal | [server/shell.js](server/shell.js) | preparação e recusa de comandos detectados; **a criação de processos está bloqueada pela política atual** |
| Terminal persistente | [server/terminal.js](server/terminal.js) | sessão registrada por projeto: cwd, ambiente e histórico próprios, até 4 abas, Reconectar — **sem início de processo real enquanto a política bloquear** |
| Diff | [server/diff.js](server/diff.js) | diferença por linhas mostrada dentro do chat |
| Explorador | [public/files.js](public/files.js) | árvore do projeto e visualizador de código no painel do agente |
| Terminal (UI) | [public/terminal.js](public/terminal.js) | abas por sessão, streaming em SSE, ↑/↓, Parar, Reconectar — só no painel do agente |

**Fonte de verdade do runtime da interface:** `public/index.html` contém o script inline principal; a shell carrega `public/sse.js`, `public/chatgpt-shell.js`, `public/shell.css` e `public/chatgpt-shell.css`. Os módulos como `public/chat.js`, `public/preview.js`, `public/app.js` e `public/settings.js` permanecem no repositório para testes e ferramentas, mas não são carregados pela interface empacotada atual. As correções de interface precisam mirar o caminho de runtime, não presumir que esses módulos legados são executados. A shell ativa consulta a revisão do workspace periodicamente para refletir alterações externas sem recarregar quando nada mudou.

**O preview é um servidor HTTP separado, com raiz exatamente no workspace.** Ele roda em
outra origem (outra porta), então nada do chat pode aparecer nele: nem código, nem
terminal, nem árvore de arquivos, nem conversa. Navegadores bloqueiam o acesso da
interface ao conteúdo do iframe (verificado nos testes: `contentDocument` é inacessível).

## Fluxo de um pedido

1. O usuário escreve no chat → `POST /api/chat` (resposta em SSE).
2. O orquestrador injeta o contexto do workspace (arquivos + conteúdo relevante) e o
   protocolo de escrita no prompt do modelo selecionado.
3. O modelo responde em prosa e blocos de arquivo. Para uma alteração pequena ele
   responde com o **trecho**, não com o arquivo:

   ```edit:caminho/arquivo.ext
   FIND:
   linhas exatas que já existem no arquivo (poucas, únicas)
   REPLACE:
   linhas novas
   ```
   ```patch:caminho/arquivo.ext      (formato antigo, continua valendo)
   <<<<<<< ATUAL
   trecho exato que já existe
   =======
   trecho novo
   >>>>>>> NOVO
   ```
   ```file:caminho/arquivo.ext
   conteúdo completo
   ```
   ```delete:caminho/arquivo.ext
   ```

4. **O destino é validado antes da escrita.** Para cada rodada o orquestrador deriva os
   arquivos *autorizados* (`server/targets.js`) e recusa qualquer bloco destinado a outro
   arquivo — o patch não chega a ser lido nem gravado (ver “Alvo autorizado” abaixo).
5. O parser consome o stream em tempo real, os arquivos são aplicados no workspace e cada
   evento (`text`, `file`, `step`, `deleted`, `file_error`, `done`) vai para o chat.
6. O preview detecta a nova revisão e recarrega apenas o resultado renderizado.

Erros técnicos (conexão recusada, `ENOENT`, stack traces) aparecem **no chat e no painel
de diagnóstico**, nunca dentro da interface renderizada.

## Provedor padrão: llama.app

- O adaptador padrão fala com o serviço local no endpoint `http://127.0.0.1:9931` (`LLAMA_APP_BASE_URL` pode substituí-lo por outro endpoint de loopback).
- A descoberta de modelos usa `GET /v1/models`; as respostas em streaming usam `POST /v1/chat/completions`.
- `LLAMA_APP_MODEL` permite indicar o modelo servido; `LLAMA_APP_API_KEY` é opcional e, quando configurada, é enviada como credencial sem ser impressa nos logs; `LLAMA_APP_CONTEXT_LIMIT` define o limite conservador usado pelo aplicativo.
- O limite de contexto usado pelo adaptador começa em 4096 e tem teto de configuração de 6144. Isso não aumenta por si só a janela real do modelo já carregado no serviço.
- Falhas de conexão ou ausência de modelo devem ser tratadas como indisponibilidade. O BOT.IA não deve trocar para outro provedor sem seleção/configuração explícita.

## Free Claude Code como catálogo opcional de modelos verificados

O BOT.IA pode consultar uma instalação separada do [Free Claude Code](https://github.com/Alishahryar1/free-claude-code) por HTTP local. O FCC não substitui o agente do BOT.IA: workspace, ferramentas, aprovações, checkpoints e política de edição continuam sob controle do BOT.IA. O FCC não é incluído no instalador do BOT.IA e as sessões de execução de código do FCC não são invocadas pelo agente.

A integração está desativada por padrão. **A conexão FCC falha fechada sem `BOTIA_FCC_API_KEY`**. Antes de consultar o catálogo, o BOT.IA também testa a proteção do FCC com um token propositalmente inválido e recusa continuar se o servidor o aceitar; isso detecta `PROXY_AUTH_ENABLED` desligado no endpoint de catálogo. Configure o serviço FCC para escutar em `127.0.0.1:8082`, ative `PROXY_AUTH_ENABLED=true` e defina `ANTHROPIC_AUTH_TOKEN` com um token aleatório de pelo menos 32 caracteres (não use o padrão conhecido `freecc`). O BOT.IA deve receber exatamente o mesmo valor em `BOTIA_FCC_API_KEY`. Mudanças nas variáveis só afetam processos iniciados depois delas; encerre e reinicie o FCC após alterar sua configuração.

Exemplo de geração de token no PowerShell (execute antes de iniciar o serviço FCC):

```powershell
$env:HOST = "127.0.0.1"
$env:PORT = "8082"
$env:PROXY_AUTH_ENABLED = "true"
$secretBytes = New-Object byte[] 32
$rng = [System.Security.Cryptography.RandomNumberGenerator]::Create()
$rng.GetBytes($secretBytes)
$env:ANTHROPIC_AUTH_TOKEN = [Convert]::ToBase64String($secretBytes)
$rng.Dispose()
```

No mesmo ambiente, configure o BOT.IA com o mesmo token. Substitua o valor de exemplo da lista por IDs completos que você confirmou manualmente como gratuitos no provedor. **Antes**, abra o painel do FCC e confirme que *Model Fallbacks* está vazio ou contém somente os mesmos IDs gratuitos verificados. O BOT.IA não consegue ler essa configuração por API; o sinalizador abaixo é uma confirmação explícita do operador, não uma verificação automática:

```powershell
$env:BOTIA_PROVIDERS = "llama-app,ollama,free-claude-code"
$env:BOTIA_FCC_BASE_URL = "http://127.0.0.1:8082"
$env:BOTIA_FCC_API_KEY = $env:ANTHROPIC_AUTH_TOKEN
$env:BOTIA_FCC_FREE_MODEL_IDS = "provider/exact-model-id-verified-free"
# Só depois de confirmar os fallbacks no painel FCC, descomente a linha seguinte:
# $env:BOTIA_FCC_FALLBACKS_CONFIRMED = "true"
npm start
```

O adaptador rejeita endpoints não locais, recusa chaves ausentes ou com menos de 32 caracteres, verifica que o FCC rejeita tokens inválidos e exige `BOTIA_FCC_FALLBACKS_CONFIRMED=true` antes de consultar o catálogo. Só apresenta IDs exatos existentes em `BOTIA_FCC_FREE_MODEL_IDS`. Não confia automaticamente nos sufixos `:free` ou `/free`. Se a lista estiver vazia, nenhum modelo FCC será disponibilizado. O rótulo da interface indica que o modelo foi colocado na lista verificada; não promete que preços ou cotas de terceiros nunca mudarão.

**Para evitar fallbacks pagos:** no painel de configuração do próprio FCC, deixe a lista **Model Fallbacks** vazia ou restrinja-a aos mesmos IDs verificados. O contrato HTTP usado pelo BOT.IA não expõe informação suficiente para confirmar ou desativar os fallbacks internos do FCC; por isso, a configuração deve ser conferida no FCC. Se o FCC encaminhar uma requisição para outro modelo internamente, o BOT.IA não consegue impedir ou detectar essa cobrança a partir do catálogo. Para evitar qualquer inferência remota, não habilite o FCC.

O BOT.IA limita o tamanho do catálogo e dos corpos de erro enquanto lê o stream, cancela a leitura ao exceder o limite e aplica timeout durante a leitura do catálogo. A recuperação automática de modelo por erro de memória está restrita a assinaturas explícitas de esgotamento de memória do Ollama; erros remotos do FCC não fazem o BOT.IA trocar de modelo automaticamente.

## Ollama como alternativa explícita

- Ollama permanece disponível como adaptador opcional (`server/ollama.js`), usando normalmente `127.0.0.1:11434`.
- Para usar somente Ollama, configure `BOTIA_PROVIDERS=ollama`; `BOTIA_PROVIDER=ollama` também o seleciona quando não há vários provedores habilitados. Para habilitar ambos os adaptadores, configure `BOTIA_PROVIDERS=llama-app,ollama`. Uma seleção válida persistida pelo utilizador tem precedência; sem ela, `BOTIA_PROVIDER` define a seleção inicial.
- O demo (`npm run demo`) usa um Ollama simulado para exercitar fluxos do construtor; isso não torna Ollama o provedor padrão de uso real.

## Preferências e persistência

Salvas em `data/` (fora do controle de versão): preferências do provedor/modelo ativo, parâmetros do agente, nome do projeto, viewport e a conversa. Ao iniciar:
**configurações → conexão → modelos → modelo escolhido → preferências → pronto**.

## Segurança do workspace

Todo caminho vindo do modelo passa por `Workspace#resolve`: caminhos absolutos, `..`,
arquivos ocultos (exceto `.env`/`.gitignore`) e diretórios reservados são recusados. O
preview nunca lista diretórios e nunca serve dotfiles, e a interface do app só entrega
arquivos de `public/` com extensões conhecidas.

### Alvo autorizado: o modelo pode errar, o servidor não obedece

Um modelo pequeno costuma acertar o conteúdo e errar o destino — pedido para editar
`styles.css`, envia `patch:index.html` porque o exemplo do protocolo mostra esse nome.
Antes de qualquer leitura ou escrita, o servidor decide quais arquivos a rodada pode
alterar (`server/targets.js`), na ordem de autoridade:

1. o arquivo que o usuário citou no pedido;
2. o arquivo lido na rodada (`read:` ou leitura automática antes de editar);
3. o nome no pedido que casa com um único arquivo existente do projeto;
4. a inferência do modelo — só quando nada acima aponta para um arquivo.

Um destino fora da lista **não é gravado**: a rodada emite
`PATCH_REJECTED_TARGET_MISMATCH` (motivo `target-mismatch`), relê o arquivo correto, devolve
o conteúdo real ao modelo e pede o patch de novo — sem repetir o mesmo prompt. A comparação
é de *identidade de arquivo*, não de texto: `styles.css`, `./styles.css` e
`assets/../styles.css` são o mesmo arquivo, `public/styles.css` não é, e `../styles.css`
nunca é autorizado. Uma proibição explícita (“não altere o `index.html`”) vence todas as
outras regras, e reescrever (`file:`) ou apagar (`delete:`) um arquivo existente fora do
alvo também é recusado. Criar um arquivo **novo** de apoio continua permitido: criar não
destrói nada.

### Edição por âncora: o modelo não precisa reproduzir o arquivo

Um CSS de 13,7 KB não cabe no orçamento do prompt e um modelo pequeno que recebe o arquivo
inteiro tende a **inventar** o corpo da regra. O bloco `edit:` evita isso: o modelo devolve
apenas o trecho que muda.

```edit:styles.css
FIND:
.hero h1 { font-size: 20px; margin: 0 0 8px; }
REPLACE:
.hero h1 { font-size: 28px; margin: 0 0 8px; }
```

O servidor (e não o modelo) faz o trabalho pesado, sempre antes de gravar:

1. **localiza** o trecho literalmente no arquivo;
2. **exige unicidade** — duas ou mais ocorrências recusam com `EDIT_REJECTED_AMBIGUOUS`,
   informando o número de ocorrências e as linhas, e a rodada corretiva pede uma âncora mais
   específica (o modelo inclui o seletor/bloco vizinho);
3. **integridade** — `hash_before`/`hash_after`, e a conferência
   `conteúdo anterior sem o trecho + substituição = conteúdo final`; a gravação é atômica e
   restaura o original se os bytes gravados divergirem do que foi validado;
4. **estrutura** — a substituição não pode deixar chaves inválidas (`}` seguido de `{`, ou um
   `{` sobrando), o erro real de ancorar um seletor pelado e reescrever a regra inteira;
5. **destrutivo** — a guarda de sempre: um `NOVO` que apaga a maior parte do arquivo é
   recusado (`EDIT_REJECTED_DESTRUCTIVE`), igual ao `patch:` antigo.

Nada disso depende de o modelo copiar o arquivo. Quando o trecho **não** casa — inclusive
quando o modelo cita uma forma antiga da regra — o orquestrador não devolve o arquivo
inteiro: ele usa o próprio `FIND` do modelo para achar a região (pelo texto exato, ou pelo
seletor) e manda de volta só as poucas linhas em volta. Nas execuções reais é exatamente
isso que converte um palpite errado em uma edição correta: o modelo devolve um trecho de
~100 bytes, o servidor valida, o arquivo muda em uma linha e todo o resto fica **byte a
byte** igual.

Códigos de recusa: `EDIT_REJECTED_NOT_FOUND`, `EDIT_REJECTED_AMBIGUOUS`,
`EDIT_REJECTED_DESTRUCTIVE`, `EDIT_REJECTED_MARKERS`, `EDIT_REJECTED_OVERLAP`,
`EDIT_REJECTED_EMPTY`, `EDIT_REJECTED_FORMAT`, `EDIT_REJECTED_UNBALANCED` e — antes de
qualquer leitura ou escrita — `PATCH_REJECTED_TARGET_MISMATCH`.

### Higiene de contexto: o histórico do chat não é o contexto da operação

Numa conversa longa o modelo tem uma falha que nenhum prompt resolveu: ele
**regurgita**. Já respondeu algo parecido, então repete a resposta anterior palavra por
palavra — inclusive um bloco que o servidor acabou de recusar — e ignora a correção. A
causa era arquitetural: reenviar a conversa inteira em cada retry deixava o próprio erro do
modelo em frente dele.

Duas regras resolvem isso:

1. **histórico visual ≠ contexto de execução.** Só a primeira rodada de um pedido recebe a
   conversa. Toda rodada de reparo (retry, leitura respondida, destino recusado) recebe um
   contexto construído para **uma** operação, e só ele:

   ```
   SOLICITAÇÃO ORIGINAL          a autoridade da operação
   ARQUIVO AUTORIZADO            o destino, dito pelo servidor
   ESTADO ATUAL / TRECHO         o arquivo, ou só as linhas em volta
   ERRO DA RODADA ANTERIOR       o motivo, dito semanticamente
   TAREFA + formato              o que produzir
   ```

   Ficam de fora: o histórico, as respostas anteriores, os patches antigos, os arquivos não
   relacionados, logs e saídas de ferramentas. O bloco recusado **nunca** é reenviado — a
   recusa é descrita (*“destino incompatível”*, *“destino recusado: index.html”*), nunca
   reexibida como exemplo.

2. **repetir uma operação recusada é um evento nomeado.** Se a nova resposta é a mesma
   operação já recusada — mesmo destino, mesmo FIND, mesmo REPLACE (comparação estrutural,
   insensível a indentação) —, nada é aplicado e a rodada é registrada como
   `MODEL_REPEATED_PREVIOUS_OPERATION`, com o contexto reconstruído para a próxima.

Cada rodada deixa um diagnóstico (`round`, `operation`, `target`, `result`, `failure`,
`context_mode`) no evento final, e um pedido que esgota o orçamento de rodadas termina com
um aviso claro em vez de silêncio: *“Não foi possível concluir a alteração com segurança.
Nenhuma alteração adicional foi gravada.”*

A garantia continua a mesma: **uma rodada ruim pode consumir uma rodada, nunca pode
corromper o projeto.**

### Contexto adaptativo: o arquivo cresce com a falha

Higiene responde *“o histórico entra na rodada?”* (não). A pergunta seguinte é *“quanto do
arquivo esta rodada precisa?”* — e a resposta depende da falha, não de um palpite. Se o
mesmo erro (mesmo código, mesmo arquivo) volta, o trecho anterior foi demonstravelmente
pequeno demais: **reenviar exatamente as mesmas linhas é o modo como um turno queima o
orçamento.** O contexto então sobe um nível por rodada, e para no arquivo autorizado:

```
NÍVEL 0  minimal             pedido + alvo + erro (nada do arquivo)
NÍVEL 1  focused-excerpt     ±8 linhas em volta da âncora
NÍVEL 2  expanded-excerpt    ±24 linhas em volta da âncora
NÍVEL 3  structural-context  o bloco que contém a âncora + vizinhos
NÍVEL 4  full-file           o arquivo autorizado inteiro
```

Um erro **diferente** reinicia no nível natural daquele erro (escalar não é somar um
contador); o teto é sempre o nível 4. Cada rodada registra `context_level` e
`context_escalated` no diagnóstico, e **toda nova operação recomeça no nível 0** — nada é
herdado do pedido anterior.

Quem decide é o servidor, a partir do código da falha e do alvo. O modelo pode errar: o que
não pode é a memória do erro aumentar a chance de repeti-lo.

### Fronteira: a aplicação nunca é um projeto

`server/app-guard.js` decide, **por caminho canônico** (resolvido, symlinks seguidos,
caixa normalizada no Windows), se uma pasta pode virar projeto. São recusados:

* a própria raiz da aplicação;
* qualquer pasta interna (`server/`, `public/`, `electron/`, `tools/`, `test/`, `data/`,
  `workspace/`, `node_modules/`, configuração e dotfiles);
* qualquer ancestral da raiz (ex.: `D:\`) — a árvore da aplicação ficaria alcançável
  dentro do preview.

Pasta externa legítima continua aceita (`D:\Projetos\Login`, `D:\Sites\MeuSite`, …). A
única exceção *dentro* da aplicação é a área gerenciada `projects/`, onde nascem os
projetos criados pelo próprio app.

A barreira roda em três lugares, sempre com a mesma mensagem amigável (sem stack trace):

1. **Registro** — `ProjectRegistry#open` recusa antes mesmo de tocar no disco;
2. **Carregamento** — um `projects.json` adulterado (ou escrito antes desta barreira)
   tem a entrada inválida descartada em vez de virar raiz do preview;
3. **Preview** — o servidor estático responde `403` para qualquer coisa dentro da
   árvore da aplicação, mesmo que a raiz apontasse para lá.

Teste a fronteira na máquina real: `node tools/isolation-probe.js`.

## Alvo autorizado no relatório

As linhas `TARGET AUTHORIZATION`, `WRONG-FILE PATCH REJECTION`, `READ → RETRY`,
`CORRECT FILE PATCH` e `NO UNAUTHORIZED FILE CHANGES` de `VALIDACAO-FINAL.md` (item 12) têm
a prova com hashes, os testes determinísticos dos casos A–F e os turnos reais de
`tools/validate-real.js`.

O bloco **CONTEÚDO / EDIÇÃO** (item 13) cobre a edição por âncora: arquivo de 14,2 KB
alterado em **uma única linha**, `diff` provando que o resto ficou byte a byte igual, as
recusas (ambígua, inexistente, destrutiva, estrutura) e o turno real dentro do Electron.

O bloco **CONTEXT HYGIENE** (item 14) cobre a memória do erro: contexto de reparo limpo,
bloco recusado nunca reenviado, isolamento entre operações, guarda de regurgitação, trecho
focado no retry, pedido original preservado e falha segura no limite de rodadas.

O bloco **ADAPTIVE CONTEXT** (item 15) cobre o escalonamento: os cinco níveis, a escada de
ampliação por estagnação, o reinício por operação e o teto no arquivo autorizado — com o
resultado real de `tools/validate-real.js`.

A tabela **FINAL ACCEPTANCE** (item 15) fecha o produto inteiro — Ollama, agente, projetos,
fronteira, preview, viewports, Electron, build e testes — com os números da execução final
e as limitações honestas separadas do que passou.

## FULL LOCAL — as camadas locais

O desenho do produto prioriza inferência local: os adaptadores de modelo descritos nesta documentação usam serviços de loopback e não devem recorrer silenciosamente a APIs remotas. Isso reduz o envio de conversas a serviços externos, mas **não significa que os ficheiros locais estejam cifrados em repouso** nem constitui uma garantia absoluta contra acesso por outros processos ou contas com permissões na máquina.

```
BOT.IA
      │
      ├─ servidor local (app 4180 · preview 4181)
      ├─ llama.app local (127.0.0.1:9931) ──► provedor padrão e modelo selecionado
      ├─ Ollama local (127.0.0.1:11434) ──► alternativa somente se habilitada
      ├─ workspace local (arquivos do projeto no disco)
      └─ preview local (a aplicação renderizada, sem nada do agente)
```

O selo **LOCAL ONLY** no cabeçalho não é decorativo: `GET /api/local` audita a própria
interface (nenhum CDN, fonte remota, script externo ou telemetria), lista as seis
superfícies locais (agente, modelos, projetos, preview, histórico, configuração) e mostra o
estado de cada uma. Com o provedor local ativo indisponível, o app continua servindo a interface e os dados locais, mas a geração por IA fica indisponível. Não há fallback remoto silencioso.

**Perfis e MANUAL/AUTO.** Quatro perfis locais (Coder, Fast, Reasoning, UI) definem
modelo, temperatura, contexto, limite de saída e instrução de sistema. No modo **MANUAL** o
modelo é o escolhido por você. No modo **AUTO** o roteador local classifica a tarefa
(CREATE, EDIT, DEBUG, REFACTOR, EXPLAIN, ANALYZE, UI, DOCUMENTATION) e escolhe entre os
modelos **instalados e utilizáveis**: trabalho de código prefere um modelo de código (e o
maior entre esses), resposta rápida prefere o menor, análise estrutural prefere o maior.
Modelo com falha registrada (OOM/erro) **nunca** é escolhido — se não houver nenhum
utilizável, o turno falha com uma mensagem clara em vez de tentar um modelo quebrado. Essa
memória de falha é local e durável (`data/models.json`): `gemma4:26b`, que realmente estoura
memória nesta máquina, continua listado como `⚠ OOM · indisponível nesta máquina` depois de
fechar e abrir o app (e o registro expira sozinho, para uma máquina com mais memória poder
tentar de novo).

**Memória em quatro camadas.** Global (preferências), projeto (framework, estrutura,
comandos — derivados do disco e nunca sobrescrevendo o que você escreveu), conversa (o
chat daquele projeto) e operação (só a rodada atual, nunca persistida). Os níveis são
arquivos separados: um fato de um projeto não vaza para outro.

**Índice e busca.** Ao abrir (e a cada mudança em disco) o projeto é varrido uma vez:
tipos, símbolos, seletores CSS, dependências, rotas e termos. O agente **localiza** o
arquivo antes de editar em vez de adivinhar o caminho, e o índice responde do cache
enquanto nada mudou — o que mudou é sempre relido (fingerprint por tamanho + mtime).

**Diff, checkpoint e rollback.** Toda alteração aparece no chat com as linhas
`- antiga` / `+ nova` e os hashes antes/depois. Antes de escrever, os arquivos envolvidos
são fotografados num diário local; `Desfazer última alteração` restaura byte a byte pelo
conteúdo guardado (não pelo texto do chat) — inclusive **depois de reiniciar o app**, e uma
rodada multi-arquivo que falha reverte inteira, sem estado parcial.

**Explorador, terminal e segredos.** Os quatro painéis do agente são Chat, Arquivos,
Terminal e Diagnóstico. O explorador abre o código do arquivo clicado; o terminal **apresenta**
os comandos **detectados** no projeto (nada de assumir npm) e **recusa** comandos destrutivos
antes de criá-los — a criação em si está bloqueada pela política atual;
`.env`, `*.pem`, `*.key` e `credentials.json` são protegidos — o
conteúdo nunca entra em um prompt. Cada projeto tem a sua própria **sessão de terminal
registrada** (ver a seção *Terminal persistente*), então o `cwd` e o histórico de um
projeto nunca aparecem no outro.

**Validação histórica desta fase (não é o resultado atual da CI).** O relatório antigo informa que `node tools/validate-local.js` foi executado contra um ambiente real (servidor, Ollama e Electron): **30 PASS · 0 FAIL · 0 INFO** naquela validação, abrangendo LOCAL ONLY, catálogo de modelos, roteamento, memória de projeto, índice, busca, explorador, proteção de segredos, terminal, diff, checkpoint, rollback, edição multi-arquivo, isolamento, preview, recuperação e build local. Consulte `VALIDACAO-FINAL.md` (item 16) como evidência histórica. Para os resultados da CI e os limites da revisão posterior, consulte [a auditoria de prontidão](docs/audits/2026-10-23-assistente-local-prontidao.md).

## POWER USER — a IDE local

Esta fase transformou o painel do agente numa bancada de trabalho diária. **Tudo continua
local**; nada aqui abre uma conexão externa. O preview à direita continua sendo **somente a
aplicação renderizada** — editor, chat, terminal, diff, checkpoints, logs e diagnóstico
vivem exclusivamente no painel do agente, na mesma origem isolada de sempre.

**Editor de código real.** Abas com indicador de alteração, numeração de linhas, realce de
sintaxe, busca/substituir (individual e global), ir para linha, desfazer/refazer e seleção de
texto. Salvar (`Ctrl+S`) grava o **arquivo real** no workspace passando pelo servidor, que
valida o hash (edição concorrente → `409`), o caminho (travessia → `400`) e os arquivos
protegidos (`.env`, chaves → `403`), e registra um checkpoint `editor`. Fechar uma aba com
alterações pergunta **Salvar / Descartar / Cancelar** — nada é perdido em silêncio.

**Seleção: explicar vs. editar.** `Explicar seleção` é **read only**: manda só o trecho, não
cria checkpoint, não chama escrita (o agente recusa `write` com `READ_ONLY`). `Editar seleção`
usa o arquivo atual + o range selecionado + o pedido, e o alvo continua **autorizado pelo
servidor**: se o modelo tentar editar outro arquivo, a edição é rejeitada.

**Busca global (`Ctrl+Shift+F`).** Busca no projeto inteiro por conteúdo e devolve **arquivo,
linha e trecho**; clicar abre o arquivo no editor e seleciona a ocorrência. É local e não
depende do Ollama (ignora binários e arquivos protegidos).

**Command palette (`Ctrl+K`).** 28 comandos, todos registrados (nenhum item decorativo):
novo projeto, abrir projeto, abrir arquivo, buscar arquivo, buscar no projeto, novo terminal,
criar checkpoint, rollback, trocar modelo, trocar perfil, executar testes, diagnóstico,
configurações, salvar, explicar/editar seleção, parar agente, copiar/salvar saída do terminal,
recarregar preview e limpar conversa. **Atenção:** o comando *Executar testes* abre o fluxo de
aprovação, mas a criação do processo continua **bloqueada** pela política atual
(ver *Estado da execução*).

**Troca rápida de modelo (`Ctrl+M`) e de projeto (`Ctrl+Shift+O`).** O seletor lista primeiro
os modelos utilizáveis e separa um grupo `⚠ indisponíveis`; um modelo com OOM ou erro
**nunca** vira ativo (a API responde `409`). E, se o modelo ativo estourar a memória no meio
de um turno, o servidor troca sozinho para um modelo utilizável e **avisa a interface**
(evento `model_changed`) em vez de falhar em silêncio. Trocar de projeto muda, juntos,
memória, histórico, arquivos, editor e preview.

**Risco e aprovação.** Toda operação é classificada em `LOW` / `MEDIUM` / `HIGH` (1 linha de
CSS → LOW; 3 arquivos → MEDIUM; apagar/mover/grande refatoração → HIGH). Com `approvalMode`
ligado, as operações que exigem decisão **param e esperam** o usuário: aprovar aplica,
cancelar **não escreve nada**.

**Parar, tentar de novo, diff, checkpoint e timeline.** `Parar` interrompe o streaming,
cancela a execução e as aprovações pendentes e não deixa escrita indevida. `Tentar novamente`
começa uma **operação limpa**, sem reaproveitar o contexto da falha. `Ver alterações` calcula o
diff a partir dos **arquivos reais no disco** (arquivo, linhas, adições, remoções, quantidade)
em vez de confiar no que o modelo diz ter feito. O painel de checkpoints cria e restaura
snapshots (`agent`, `editor`, `manual`), com o diário durável atravessando reinícios. A
timeline do chat é montada dos `traces` reais (pedido recebido → arquivo localizado → trecho
lido → alteração aplicada → teste → preview), sem mostrar campos internos crus.

**Diagnóstico, terminal, serviços e logs.** O diagnóstico mostra `operation`, `model`,
`target`, `context level`, `round`, `result` e `bytes` (ocultando o que não existe, sem
inventar dado). O terminal tem abas, executar, cancelar, limpar, copiar e salvar a saída, e
roda **apenas no escopo permitido** (workspace / projeto / arquivo), preservando a guarda de
comando destrutivo e do `APP_ROOT`. Cada aba é uma **sessão de terminal registrada** do
projeto (a mesma usada pelo agente quando apropriado), com `cwd`, ambiente e histórico
próprios — ver a seção *Terminal persistente*. O painel de serviços mostra **App, Preview e Ollama** com
status e porta reais. Os logs são locais, com cinco canais (agent, ollama, preview, terminal,
system) e gravação em disco **desligada por padrão**. As configurações ficam divididas em
seções (Local, Agent, Models, Projects, Security, Preview, Terminal, Memory, Context).

**Aceitação desta fase.** `node tools/validate-local.js` → **30 PASS · 0 FAIL · 0 INFO** e
`node tools/e2e-real.js` → **18 PASS · 0 FAIL · 1 INFO** contra o Ollama real e a janela
Electron real; os 312 testes automatizados passam. O relatório está em `VALIDACAO-FINAL.md`
(item 17).

## HARDENING — estabilidade de uso diário

Uma passada de robustez sobre o que já existia, **sem cloud, sem login, sem telemetria e sem
API externa**. Nenhuma funcionalidade foi removida; o preview continua somente a aplicação
renderizada.

**Uma operação escrevendo por vez.** Um segundo turno iniciado enquanto outro roda é
**recusado** (`409`) em vez de disputar o mesmo workspace; trocar de modelo ou de projeto no
meio de uma operação também é recusado, porque mudaria o estado por baixo dela. No chat, isso
aparece como um aviso claro — não como "falha de rede".

**Parar nunca deixa nada para trás.** O guarda de cancelamento é checado antes de **qualquer**
escrita, então um bloco que o modelo já emitiu não é aplicado depois do Parar. Quando a execução
de processos está autorizada, os comandos do terminal rodam num grupo de processos e são
encerrados **com a árvore inteira** (no Windows, `taskkill /T`), o que antes deixava o processo
neto órfão; ao fechar o app, os comandos em execução e as aprovações pendentes são cancelados.
**No estado atual essa execução está bloqueada pela política** — o painel continua mostrando o
estado real da sessão e a ação **Reconectar**.

**Editor por projeto e contra o disco.** As abas, a aba ativa, o cursor e o scroll são
lembrados por projeto (local) e restaurados ao reabrir — dois projetos nunca compartilham
abas. Se o arquivo muda no disco enquanto está aberto, o editor mostra um diálogo de
**conflito** com **Comparar / Recarregar / Manter minha versão** e nunca sobrescreve sozinho.

**Diff por arquivo e por trecho.** `ver alterações` lista os arquivos (cada um com o seu
`+/−`) de forma expansível, e cada trecho (hunk) tem **Descartar trecho** e **Reaplicar
trecho**, que operam no **arquivo real** com validação de hash: se o arquivo mudou desde o
diff, a ação é recusada (`409`) em vez de sobrescrever.

**Preview e terminal presos ao projeto ativo.** Um evento de preview atrasado, vindo do
projeto anterior, é descartado; o preview recarrega quando o projeto muda. O terminal mostra a
pasta real usada em cada comando e recarrega os comandos ao trocar de projeto.

**Aceitação do hardening:** `npm test` → **323/323**, `node tools/validate-local.js` →
**30 PASS · 0 FAIL · 0 INFO**, `node tools/e2e-real.js` → **18 PASS · 0 FAIL · 1 INFO**,
segurança **64/64**, janela Electron real verificada pelo CDP. Endpoints locais respondem em
**1,5–4,3 ms** e a interface não estoura em 1280×720 nem 1366×768. Detalhes em
`VALIDACAO-FINAL.md` (item 18).

## Terminal persistente — uma sessão registrada por projeto

> **Estado atual da execução de processos.** A partir desta versão, o BOT.IA **não inicia
> processos** em nome do usuário: a política central `server/process-execution-policy.js`
> permite apenas a verificação de sintaxe (`node --check`) e recusa qualquer criação de
> shell, de terminal interativo e de execução de testes (`EXECUTION_NOT_AUTHORIZED`).
> Uma proposta revisada no menu nativo **não** autoriza execução. O painel Terminal, o
> registro de sessões, o `cwd`, o histórico e o botão **Reconectar** continuam
> funcionando como **interface**; o que está bloqueado é a criação do processo real.
> Veja a seção *Estado da execução* mais abaixo.

O terminal mantém **uma sessão registrada por projeto** (até 4 abas), e ela é o que o painel
mostra:

```text
Projeto A → Terminal A          Projeto B → Terminal B
  cwd, ambiente e histórico       cwd, ambiente e histórico
  próprios                        próprios
```

* **O `cwd` é preservado.** `cd src` muda a pasta da sessão e o próximo comando já roda em
  `…\Projeto\src` — sem novo shell. A barra do painel mostra sempre o diretório real que o
  servidor usou, nunca um palpite da interface.
* **O histórico é da sessão.** `↑`/`↓` percorrem os comandos daquela aba, com autoria
  (**USER** ou **AGENT**) e horário; dois projetos nunca compartilham histórico, e duas abas do
  mesmo projeto têm históricos independentes. Comandos que só fazem sentido com um segredo
  (`password`, `token`, `apikey`, `secret`, `credential`) **não são gravados**.
* **A variável de ambiente sobrevive** dentro da sessão: `set TESTE=123` numa linha, `echo
  %TESTE%` na seguinte. Nada sensível vai para o disco.
* **Várias abas.** `+ sessão` abre outra sessão no mesmo projeto, cada uma com o seu `cwd`; o
  limite é de **4 por projeto** e a recusa é explícita ("Limite de sessões atingido"), para
  nunca multiplicar processos sem teto.
* **Parar mata o comando, não o terminal.** O botão encerra a **árvore inteira** do processo
  (pai, filho e neto, preservando a correção de `killTree`) e a sessão continua viva: o próximo
  comando roda na mesma aba, no mesmo `cwd`, com o histórico intacto.
* **Reconectar.** Se o shell morrer, o painel avisa ("Terminal encerrado. Reconectar"); o botão
  abre um **processo novo no `cwd` guardado**, mantendo o histórico.
* **Histórico visual.** O botão *Histórico* (e o comando *Histórico do terminal* na palette)
  lista os comandos da sessão com **horário**, **autoria** (USER/AGENT) e **resultado**
  (`código 0`, `código N` ou `interrompido`) — sem saída bruta e sem comando que carregue segredo.
* **Restart e crash.** O que é persistido é só `{ shell, cwd, history }` por projeto, em
  `data/terminal/<projeto>.json`. Ao reabrir o app — ou depois de um crash — o projeto, o `cwd`,
  as abas e o histórico voltam, e uma **sessão nova e segura** é aberta: **nenhum processo morto
  é restaurado**.
* **As mesmas guardas de sempre.** A sessão persistente não é uma escapatória: um `cd` para fora
  do projeto é recusado, o `cwd` é clampado de volta à raiz, e a guarda da **raiz do app**
  (`D:\PREVIEW.IA`) continua valendo dentro da sessão. O agente recebe só um resumo compacto
  (`cwd`, `shell`, `status`, comando) — nunca o histórico inteiro.
* **O preview continua só a aplicação renderizada.** Nada do terminal aparece nele.

### Estado da execução — o que funciona hoje

| Recurso | Estado | Observação |
|---|---|---|
| Painel Terminal (abas, `cwd`, histórico, serviço) | **disponível** | interface e registro de sessão funcionam |
| `data/terminal/<projeto>.json` | **disponível** | persiste `{ shell, cwd, history }` entre execuções |
| **Reconectar** | **disponível** | reabre a sessão no `cwd` guardado |
| Recusa de comandos destrutivos | **disponível** | a validação ocorre antes de qualquer criação |
| **Criar um processo de shell** | **bloqueado** | `EXECUTION_NOT_AUTHORIZED` / `TRUSTED_AUTHORIZATION_UNAVAILABLE` |
| **Executar testes do projeto (`run_tests`)** | **bloqueado** | idem; uma proposta revisada no menu nativo **não** libera a execução |
| **Verificar sintaxe (`check_syntax`)** | **disponível** | única criação de processo permitida pela política |

Em outras palavras: o terminal **descreve e registra** o trabalho; ele **não executa** comandos
no estado atual. A revisão de uma proposta no menu nativo é uma confirmação humana, não uma
autorização de execução.

**Aceitação desta fase:** `npm test` → **353/353**, `node tools/validate-local.js` →
**46 PASS · 0 FAIL · 0 INFO**, `node tools/e2e-real.js` → **27 PASS · 0 FAIL · 0 INFO** contra o
Ollama real e a janela Electron real, mais o ciclo real de restart/crash (cwd, histórico e abas
restaurados, processo novo). Detalhes em `VALIDACAO-FINAL.md` (item 19).

## Modo demonstração

`npm run demo` sobe um Ollama simulado com quatro modelos e respostas prontas para
"tela de login", "dashboard" e "kanban". É o jeito de explorar interface, troca de modelo
e pipeline de preview sem instalar nenhum modelo — e também como os testes de ponta a ponta
exercitam o produto real.

## Experimento de interface

Para conferir a UI sem esforço: `npm start`, abrir o app e usar os atalhos
`Ctrl/Cmd+K` (seletor de modelos), `Ctrl/Cmd+Shift+R` (recarregar preview) e
`Ctrl/Cmd+Shift+L` (focar o campo de comando). A divisão entre os painéis é arrastável e
fica lembrada entre sessões.


## Modo Conversa — assistente geral

O cabeçalho agora permite alternar entre **Construir** e **Conversa**. Conversa usa o modelo local conectado para chat de uso geral, respostas em streaming, Markdown básico e histórico persistente de tópicos separados do agente de programação. Para iniciar, conecte um provedor local e escolha um modelo real no seletor; o modo simulado não gera respostas de IA neste modo.

Para anexar arquivos, analisar imagens, editar o workspace e utilizar o preview, volte a **Construir**. A versão atual do modo Conversa não inclui pesquisa na internet, voz, sincronização em nuvem ou anexos. A arquitetura e as rotas locais estão documentadas em [docs/general-chat-mode.md](docs/general-chat-mode.md).


## Vídeo local opcional com LTX-2.5

O BOT.IA tem uma ponte opt-in para gerar vídeo com o LTX-2.5 no computador local. A funcionalidade fica desativada por padrão, não descarrega pesos e não substitui o provedor conversacional. Requer um ambiente LTX-2, cinco componentes de modelo, CUDA/NVIDIA e memória compatível.

Consulte [Instalação e requisitos da integração LTX-2](docs/integrations/ltx2.md) antes de ativar. A ponte bloqueia a execução se os requisitos não forem confirmados.

## Consolidação técnica — 26/10/2026

[Relatório de consolidação e prontidão para novas ordens](RELATORIO_CONSOLIDACAO_RELEASE_2026-10-26.md).
