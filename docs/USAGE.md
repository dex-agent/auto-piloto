# USAGE — DriveAutoTeste

CLI de automação de UI Win32 por handles: enumera janelas e controles
(`EnumWindows`/`EnumChildWindows`) e envia mensagens Win32
(`BM_CLICK`, `WM_GETTEXT`/`WM_SETTEXT`, `WM_KEYDOWN/CHAR/UP`, `WM_LBUTTON*`,
`WM_PASTE`), além de tira print de handle com retângulo localizador.

> Todos os exemplos desta página são **ilustrativos**: pids, hwnds, retângulos
> e nomes de janela são inventados. Rode os comandos na sua máquina para ver
> os valores reais.

Versão do binário neste pacote: **1.7.3** (`bin\DriveAutoTeste.exe`).

> **Windows:** os exemplos escrevem `./DriveAutoTeste.exe` (notação de shell
> Unix). No Prompt de Comando, entre na pasta instalada e use a barra
> invertida:
>
> ```bat
> cd /d "%USERPROFILE%\.agents\skills\auto-piloto"
> bin\DriveAutoTeste.exe <comando> [flags>
> ```
>
> Instalação: seção "Instalação" do `README.md`.

## 1. Ajuda e schema

`help` (ou `schema`) se auto-descreve; com `--json` o `schema` devolve a
lista estruturada de comandos e flags (1 linha parseável):

```text
$ ./DriveAutoTeste.exe help
DriveAutoTeste 1.7.3 — automacao de UI Win32 por handles
uso: DriveAutoTeste [--json] <comando> [flags]
comandos:
  run    --exe <path> [--dir <workdir>]           sobe o app e imprime pid
  find   (--pid N | --hwnd N | --hwnd 0) [--class C] [--caption S] [--caption-contains S]
                                                [--vis all|visible|invisible] [--titled]
                                                descendentes da raiz; --hwnd 0 = janelas abertas
  wait   --pid N --caption S [--class C] [--timeout ms]
  click  (--hwnd N | --pid N --caption S)        BM_CLICK
  text   (--hwnd N | --pid N [--class C])        le texto (WM_GETTEXT)
  settext --hwnd N --value S                     escreve texto
  clear  --hwnd N                                limpa o campo (len=0)
  key    (--hwnd N | --pid N --caption S) --keys k1,k2,.. [--mode press|raw|down|up|posted|combo]
  mouse  (--hwnd N | --pid N --caption S) [--action move|down|up|left|right|dclick] [--x N --y N]
  paste  (--hwnd N | --pid N --caption S)       cola clipboard (WM_PASTE)
  close  --pid N                                 encerra o processo
  sleep  --ms N                                  pausa
  shot   --hwnd N --out f.png [--locate] [--color yellow|red] [--format png|bmp]
  wh     --file <x.wh>                           playbook do workflow (campo prompt)
  play   --file <x.wh> [--var k=v]... [posicionais]
  help | schema
flag global: --json  saida estruturada {"ok":..,"cmd":..,"data":..}
```

> Nota de console: em codepage não-UTF-8 o travessão do banner pode aparecer
> como outro caractere — é só exibição; o dado está correto.

## 2. Ciclo manual (notepad)

Subir um app, descobrir handles, ler texto, fechar:

```text
$ ./DriveAutoTeste.exe --json run --exe "C:/Windows/System32/notepad.exe"
{"ok":true,"cmd":"run","data":{"pid":4242}}

$ ./DriveAutoTeste.exe find --pid 4242
hwnd=111222 class=Edit caption= parent=333444 parentclass=Notepad visible=1 rect=200,200,1200,800
hwnd=555666 class=msctls_statusbar32 caption= parent=333444 parentclass=Notepad visible=1 rect=200,800,1200,823

$ ./DriveAutoTeste.exe --json text --hwnd 111222
{"ok":true,"cmd":"text","data":{"hwnd":111222,"text":""}}

$ ./DriveAutoTeste.exe close --pid 4242
close=ok
```

- `find` lista os **descendentes** da raiz: `--pid N` (as janelas top-level do
  processo) ou `--hwnd N` (drill-down no handle).
- `text` sem `--hwnd` procura sozinho o controle: `--pid N [--class C]`
  (default `TMemo`).
- `settext --hwnd N --value S` escreve no controle (`WM_SETTEXT`);
  `clear --hwnd N` esvazia e devolve `len=0`.
- `click` aceita `--hwnd N` direto ou `--pid N --caption S` (localiza e clica).
- `wait` bloqueia até a janela existir (default 15000 ms).

## 3. find em detalhe

- Formato por item: `hwnd/class/caption/parent/parentclass/visible/rect`.
- **`parentclass`** é a classe da janela pai: identifica o controle sem
  decorar número (`Edit parentclass=Notepad`; diálogos padrão do Win32
  aparecem como `#32770`; vazio = janela top-level).
- `--hwnd N`: raiz = o handle dado (tem precedência sobre `--pid`); devolve
  todos os descendentes recursivos.
- `--hwnd 0` (explícito) = todas as **janelas abertas** (top-level,
  `parent=0`); com `--pid N`, só as do processo.

```text
$ ./DriveAutoTeste.exe find --hwnd 0 --pid 4242
hwnd=333444 class=Notepad caption=Sem titulo - Bloco de Notas parent=0 parentclass= visible=1 rect=200,150,1200,850
```

- `--caption S` casa exato (case-insensitive); `--caption-contains S` casa
  por "contém", case-insensitive:

```text
$ ./DriveAutoTeste.exe settext --hwnd 111222 --value MarcadorAlvoXyZ
settext=ok

$ ./DriveAutoTeste.exe find --hwnd 333444 --caption-contains alvoxyz
hwnd=111222 class=Edit caption=MarcadorAlvoXyZ parent=333444 parentclass=Notepad visible=1 rect=200,200,1200,800
```

- `--vis all|visible|invisible` (default `all`; invariante:
  `all = visible + invisible`). Valor inválido = erro JSON + exit 1:

```text
$ ./DriveAutoTeste.exe --json find --hwnd 333444 --vis bogus
{"ok":false,"cmd":"find","data":{"error":"vis invalido (all|visible|invisible): bogus"}}
```

- `--titled` (opt-in) descarta caption vazia (controles-label). O default do
  `find` não muda.
- Limite: `--hwnd` com o HWND **numérico** do desktop (`GetDesktopWindow`)
  devolve 0 itens — para "todas as janelas" use `--hwnd 0`.

## 4. Teclado, mouse, colar e print

### `key` — digitação por eventos

Tokens: `enter esc tab space up down left right del bs home end pgup pgdn ins
f1..f12 a-z 0-9 #VKcodigo` e modificadores `ctrl alt shift win` (vírgula
separa tokens). A vírgula é **separador**, nunca tecla: pontuação literal usa
o código VK (`#188` = VK_OEM_COMMA, `#190` = ponto; códigos OEM dependem do
layout). Modos:

- `press` (default): keydown + char + keyup.
- `raw`: sem char. `down`/`up`: segurar/soltar.
- `combo`: atalho composto, `--keys "ctrl+shift+v"` (`+` junta a tecla do
  combo, `,` separa combos; `+` literal = `#BB`). Injeta por `SendInput` com
  a janela raiz em primeiro plano; se a janela em foco não for a alvo, aborta
  **sem enviar nada** (erro nomeado). Use para Electron/Chromium, que ignoram
  teclas por mensagem.

```text
$ ./DriveAutoTeste.exe key --hwnd 111222 --keys "1,2,left,left,X"
key=hwnd=111222 keys=1,2,left,left,X

$ ./DriveAutoTeste.exe --json text --hwnd 111222
{"ok":true,"cmd":"text","data":{"hwnd":111222,"text":"X12"}}
```

#### `--activate` — ativar o alvo antes do combo (1.7.2)

Quando o alvo está atrás de outra janela, em outro monitor ou numa sessão RDP,
o combo era recusado pelo gate de foco. `--activate` (só com `--mode combo`)
pede a ativação explícita: registra o foreground anterior, ativa o alvo,
envia o combo e **restaura o foco anterior no fim**, inclusive quando falha.

```text
$ ./DriveAutoTeste.exe --json key --hwnd 111222 --keys ctrl+s --mode combo --activate
{"ok":true,"cmd":"key","data":{"hwnd":111222,"keys":"ctrl+s","focus":{"mode":"combo",
 "activate":true,"activated":true,"held":true,"restored":true,
 "target":{"hwnd":111222,"pid":4242,"title":"Editor","iconic":false,"monitor":2,...},
 "foreground":{"hwnd":777888,"pid":5150,"title":"Terminal","iconic":false,"monitor":1,...}}}}
```

O recibo diz o que aconteceu em vez de prometer: `activated` (o alvo chegou a
primeiro plano), `held` (continuou na frente **até o fim do envio** — outro app
pode roubar o foco no meio) e `restored` (o foreground anterior voltou).

```text
{"ok":false,"cmd":"key","data":{"error":"combo: ativacao falhou (alvo=... foreground=...)","focus":{...}}}
{"ok":false,"cmd":"key","data":{"error":"combo: foco perdido durante o envio (alvo=... foreground=...)"}}
{"ok":false,"cmd":"key","data":{"error":"--activate exige --mode combo"}}     exit 2
```

**Limite:** o Windows pode negar a ativação mesmo com tudo certo, e a ativação
não atravessa desktop. `--activate` **tenta e reporta**; não garante. Sem a
flag, o comportamento e o recibo são os de 1.7 (`focus.activate:false`). Em
workflow `.wh`, o passo `key` aceita `"activate": true`.

`X12` (e não `12X`) mostra que as setas moveram o cursor de verdade.

#### Vocabulario de teclas e atalhos (1.7.3)

Tabela fixa: `a-z 0-9 f1..f24 #VK`, nomes (`enter esc tab space home end pgup
pgdn ins del bs ctrl(^) alt shift win caps prtsc pause menu`), `numpad0-9` +
`numpad_add/subtract/multiply/divide/decimal`, `oem_plus/minus/comma/period`
e `oem_1..oem_7`. Token invalido lista os aceitos; `ctrl+alt+del` falha como
reservada sem enviar nada.

- `--sequence` (a virgula ja sequencia), `--repeat N`, `--delay-ms N`.
- `--dry-run` resolve os tokens e imprime o plano sem enviar:
  `plan: mode=combo steps=2 -> numpad0(96); oem_plus(187)`.
- No `.wh`: `"shortcuts": {"salvar": "ctrl+s"}` e `"keys": "salvar"`.

### `mouse` — eventos de mouse no handle

`--action move|down|up|left|right|dclick` (default `left`; `left`/`right` =
down + up). `--x/--y` em coordenadas de client; sem elas, o centro do handle.

```text
$ ./DriveAutoTeste.exe mouse --pid 4242 --caption "OK" --action left
mouse=hwnd=777888 x=40 y=12 action=left
```

### `paste` — colar do clipboard (WM_PASTE)

```text
$ ./DriveAutoTeste.exe --json paste --hwnd 111222
{"ok":true,"cmd":"paste","data":{"hwnd":111222}}
```

### `shot` — print do handle + localizador

`--out` (obrigatório), `--locate` (captura a janela raiz e desenha o
retângulo sobre o handle alvo), `--color yellow|red` (default amarelo),
`--format png|bmp` (default PNG via GDI+):

```text
$ ./DriveAutoTeste.exe --json shot --hwnd 111222 --out locate.png --locate
{"ok":true,"cmd":"shot","data":{"hwnd":111222,"out":"locate.png","locate":true,"warning":""}}
```

Abra o PNG para conferir: o retângulo amarelo deve marcar o controle alvo.
Shot de janela oculta pode sair preto.

## 5. Workflows `.wh`

`play` executa um workflow `.wh`: um JSON com steps e asserts. Exemplo real
que roda em qualquer Windows — `examples/key-notepad.wh`:

```text
$ ./DriveAutoTeste.exe play --file examples/key-notepad.wh
== workflow: key-notepad ==
[PASS] step 1 run: pid=4242 (/Windows/System32/notepad.exe)
[PASS] step 2 sleep: 800ms
[PASS] step 3 key: hwnd=111222 keys=1,2,left,left,X
[PASS] step 4 assert-text: contem X12
[PASS] step 5 key: hwnd=111222 keys=bs,home,enter
[PASS] step 6 assert-text: contem
[PASS] step 7 close: pid=4242
WORKFLOW=PASS steps=7 pass=7 fail=0
```

Em modo `--json` a saída é 1 linha única (prints e logs de step
silenciados; só steps com resultado entram no array).

Workflow = arquivo JSON `{ "name", "prompt", "vars"?, "params"?, "steps": [ { "do": ..., ...params } ] }`.

- **Campo `prompt` = workflow autocontido**: o playbook de operação (missão,
  descoberta de handles, invocação, prova de efeito, avisos) viaja dentro do
  `.wh`. O player ignora esse campo (só lê `name`/`vars`/`params`/`steps`);
  `wh --file x.wh` imprime o playbook em modo humano ou JSON parseável.
- **Posicionais**: o bloco `"params": ["msg", "memo"]` declara a ordem e o
  `play` aceita os valores soltos na linha de comando. O posicional prevalece
  sobre `--var`; posicional em workflow sem `params`, posicionais demais ou
  sem `--file` = exit 2 com erro nomeado.
- **Defaults**: o bloco `"vars": {"k":"v"}` dá o valor default; `--var` e
  posicionais sobrescrevem.
- Um arquivo por regressão, zero código novo.

```bat
cd /d "%USERPROFILE%\.agents\skills\auto-piloto"
bin\DriveAutoTeste.exe play --file examples\key-notepad.wh
bin\DriveAutoTeste.exe play --file examples\shot-notepad.wh --var out=caminho\shot.png
```

### Steps

| `do` | Params | Default | Efeito |
|---|---|---|---|
| `run` | `exe`\*, `dir`, `save` | save=`"pid"` | sobe o exe; grava o pid na var `save` **e** na tabela de pids do workflow |
| `wait` | `pid`, `class`, `caption`, `timeoutMs` | timeout=15000 | espera a janela casar (class/caption por `SameText`) e retorna hwnd |
| `click` | `hwnd` **ou** `pid`+`class`+`caption`, `timeoutMs` | timeout=5000 | `BM_CLICK` no controle |
| `settext` | `hwnd` **ou** `pid`+`class`+`caption`+`timeoutMs`, `value` | timeout=5000 | `WM_SETTEXT` |
| `text` | `hwnd` **ou** `pid`+`class`+`timeoutMs`, `save`, `out` | class=`TMemo`, timeout=5000 | lê texto; `save` publica em var `${k}`; `out` grava em arquivo |
| `key` | `pid`/`hwnd`+`class`/`caption`+`timeoutMs`, `keys`\*, `mode` | timeout=5000, mode=`press` | sequência de teclas (WM_KEYDOWN/WM_CHAR/WM_KEYUP) |
| `mouse` | `pid`/`hwnd`+localização, `x`, `y`, `action` | x/y=centro do handle, action=`left` | eventos de mouse (move/down/up/left/right/dclick) |
| `paste` | `pid`/`hwnd`+localização | — | cola o clipboard (WM_PASTE) |
| `shot` | `pid`/`hwnd`+localização, `out`\*, `locate`, `color`, `format` | color=`yellow`, format=`png` | print do handle; `locate` = janela raiz + retângulo |
| `assert-text` | `pid`, `class`, `contains`\*, `timeoutMs` | class=`TMemo`, timeout=10000 | repoll a cada 250 ms até o texto do controle conter `contains` |
| `assert-file` | `path`\*, `timeoutMs` | timeout=10000 | repoll a cada 250 ms até o arquivo existir |
| `assert-empty` | `hwnd` **ou** `pid`+`class`+`caption`+`timeoutMs` | timeout=5000 | PASS se o controle está vazio (nomeia hwnd); FAIL nomeia o conteúdo; hwnd morto = FAIL `hwnd inexistente` |
| `sleep` | `ms` | 500 | pausa |
| `print` | `msg` | — | echo no modo humano (não entra no JSON; conta como pass) |
| `close` | `pid` | — | encerra o pid (aceita nome de var, ex. `${pid}` resolvido) |

\* obrigatório. Step com `do` desconhecido → FAIL ("step desconhecido").

### Vars

- `--var k=v` (múltiplas) alimentam `${k}` — expansão em **qualquer campo**
  de string de qualquer step (exe, caption, path, contains, value...).
- `run` com `save: "pid"` publica `${pid}` nos steps seguintes.
- Var salva com valor vazio expande para vazio.

### Semântica de falha

- Qualquer step que falha **aborta na hora**, mata todos os PIDs abertos
  pelos steps `run` do workflow e o processo sai **1**
  (`WORKFLOW=FAIL (step N) pass=X fail=Y` em modo humano; em `--json`, campo
  `failed_step`).
- **Exceção em step**: mesmo contrato — PIDs mortos + 1 linha
  `{"ok":false,...,"error":"<mensagem>"}` + exit 1 (nada de texto solto no
  `--json`). **Validação de entrada**: seletor vazio (sem hwnd e sem
  pid+class/caption), `keys` vazio e `x` sem `y` falham imediato com erro
  nomeando — nunca varrem o desktop.
- Sucesso: percorre tudo, fecha os PIDs subidos (close no `finally`) e sai
  **0** (`WORKFLOW=PASS steps=N pass=N fail=0`).
- **Sempre confira o efeito real** (campo limpo, arquivo gerado), nunca só o
  exit code. Um handle morto pode responder `ok:true` sem efeito algum.

## 6. Exit codes

| Código | Quando |
|---|---|
| 0 | sucesso do comando; `help`/`schema`; `find` mesmo com 0 itens; workflow PASS |
| 1 | falha do comando (run/wait/click/text/settext/clear/key/mouse/paste/shot/close) — inclui **hwnd inexistente** (`hwnd inexistente: N`) e tecla inválida (nomeia o token) — ou workflow FAIL |
| 2 | nenhum argumento (modo humano), **comando desconhecido** (em `--json`: 1 linha `ok:false`), **flag desconhecida** (`flag desconhecida: --x`) ou **posicional ilegal** (sem `--file`, workflow sem `params`, posicionais demais) |
