---
name: auto-piloto
description: "Owner do conhecimento de automação de UI Win32 por handles via CLI DriveAutoTeste 1.7 (bin/DriveAutoTeste.exe incluído nesta skill): localizar janelas e handles (find --hwnd 0 = todas as janelas abertas, busca por contém, filtro de visibilidade, parentclass), clicar (BM_CLICK), ler/escrever textos, teclado/mouse/paste/print por mensagem e workflows .wh autocontidos (prompt + vars defaults + params posicionais + steps com asserts, incluindo assert-empty que prova envio) reutilizáveis — sem escrever driver novo por sistema. Regra de ouro: nunca declarar impossível automatizar UI Win32."
metadata:
  systemone: ""
---

# auto-piloto

Alias humano e de chamada: `$auto-piloto` / `Auto Piloto`. **Alias legado:**
`drive-autoteste`.

Skill **owner do conhecimento** de automação de UI Win32 por handles. A
ferramenta é o CLI `bin/DriveAutoTeste.exe` 1.7, **incluído nesta skill**
(compilado com Delphi 13, dcc32 `-Q -B`); esta skill domina COMO usar e COMO
criar workflows. Referência completa: `docs/USAGE.md` (exemplos ilustrativos). Exemplos prontos: `examples/*.wh`.

## Instalação (Windows)

No Prompt de Comando (`cmd`), uma linha de cada vez:

```bat
git clone https://github.com/dex-agent/auto-piloto "%USERPROFILE%\.agents\skills\auto-piloto"
"%USERPROFILE%\.agents\skills\auto-piloto\bin\DriveAutoTeste.exe" --json schema
```

No PowerShell, troque `%USERPROFILE%` por `$HOME`. `~` e barra `/` não
funcionam no Prompt de Comando. Se a pasta já existir e **não** for deste
repositório (`git -C "<pasta>" remote get-url origin` imprime outra URL),
clone em `"%USERPROFILE%\auto-piloto"` e **não apague** a pasta existente.
Passo a passo completo: `README.md`, seção "Instalação".

## Como chamar o CLI

Os caminhos `bin\DriveAutoTeste.exe` e `examples\` são **relativos à pasta
desta skill**. Chame pelo caminho completo ou entre na pasta antes:

```bat
cd /d "%USERPROFILE%\.agents\skills\auto-piloto"
bin\DriveAutoTeste.exe --json schema
bin\DriveAutoTeste.exe play --file examples\key-notepad.wh
```

Nos exemplos abaixo, `DriveAutoTeste` significa esse caminho.

## Regra de ouro (origem)

Antes de declarar impossível automatizar qualquer UI Win32: **enumerar handles
e enviar a mensagem**. Funciona até em form invisível (`ShowMainForm=False`) —
handles existem após o `CreateForm`.

## Comandos (saída humana; `--json` = 1 linha `{"ok":..,"cmd":..,"data":{..}}`)

```
DriveAutoTeste [--json] <comando> [flags]
  run    --exe <path> [--dir <workdir>]           sobe o app e imprime pid
  find   (--pid N | --hwnd N | --hwnd 0) [--class C] [--caption S] [--caption-contains S]
                                                [--vis all|visible|invisible] [--titled]
                                                descendentes da raiz; --hwnd 0 = janelas abertas
  wait   --pid N --caption S [--class C] [--timeout ms]
  click  (--hwnd N | --pid N --caption S)        BM_CLICK
  text   (--hwnd N | --pid N [--class C])        le texto (WM_GETTEXT)
  settext --hwnd N --value S                     escreve texto
  key    (--hwnd N | --pid N --caption S) --keys k1,k2,.. [--mode press|raw|down|up|combo]
                                                teclas: enter esc tab setas f1-12 a-z 0-9 #vk; combo (1.7):
                                                ctrl+shift+v etc via SendInput com gate de foco (EXIGE foreground)
  mouse  (--hwnd N | --pid N --caption S) [--action move|down|up|left|right|dclick] [--x N --y N]
  paste  (--hwnd N | --pid N --caption S)       cola clipboard (WM_PASTE)
  close  --pid N                                 encerra o processo
  sleep  --ms N                                  pausa
  shot   --hwnd N --out f.png [--locate] [--color yellow|red] [--format png|bmp]
  wh     --file <x.wh>                           playbook do workflow (campo prompt)
  play   --file <x.wh> [--var k=v]... [posicionais]  workflow JSON com asserts; posicionais
                                                na ordem do bloco params do .wh (1.6)
  help | schema                                  ajuda (schema = JSON, 14 comandos)
```

### find em detalhe (1.3+)

- Saída por item: `hwnd/class/caption/parent/parentclass/visible/rect` —
  **`parentclass` é a classe do pai**: identifica a janela sem decorar número
  (`parentclass=Shell_TrayWnd`, `#32770` = diálogo padrão do Win32); vazio =
  top-level.
- `--hwnd N` = drill-down: todos os descendentes recursivos do handle (o hwnd
  da janela principal aparece no campo `parent` dos filhos — use-o como
  próxima raiz).
- `--hwnd 0` (explícito) = **todas as janelas abertas** (top-level); com
  `--pid N` lista as top-level daquele processo.
- `--caption-contains S` = busca case-insensitive por "contém" (o casamento
  por `--caption` é exato/SameText).
- `--vis all|visible|invisible` (default `all`; valor inválido → erro JSON +
  exit 1). Invariante viva: `all = visible + invisible`.
- `--titled` (1.6, opt-in) = descarta caption vazia (controles-label).
  Default do find NÃO muda.

## Workflows .wh (reuso — 1 arquivo por regressão, zero código novo)

**Workflows autocontidos:** quando te entregarem só o nome de um `.wh` +
argumentos (`use $auto-piloto key-notepad.wh`), abra o arquivo e siga
o campo **`prompt`** — ele é o playbook completo daquele workflow (missão,
descoberta de handles, invocação, prova de efeito, avisos).
**Posicionais são NATIVOS (1.6):** o bloco `"params": ["msg","memo"]` declara
a ordem e `play --file x.wh "mensagem" <hwnd>` aceita os valores soltos
(posicional prevalece sobre `--var`; posicional em workflow sem `params`,
posicionais demais ou sem `--file` = exit 2 com erro nomeado). **Defaults
(1.6):** bloco `"vars": {"k":"v"}` = default que `--var`/posicional
sobrescreve. Superfície CLI: `wh --file <arquivo>.wh` imprime o playbook
(humano) ou devolve JSON parseável.

JSON: `{name, prompt, vars?, params?, steps:[{do, ...params}]}`. Steps:
`run(exe,dir,save)`, **`assert-empty(hwnd|pid+class, 1.6: PASS=vazio nomeia
hwnd; FAIL nomeia conteúdo; hwnd morto=FAIL sem falso verde)`**,
`wait(pid,class,caption,timeoutMs)`, `click(hwnd|pid+caption)`,
`settext(hwnd|pid+caption,value)`, `text(pid,class,save,out)`,
`key(pid/hwnd+class/caption,keys,mode)`,
`mouse(pid/hwnd+localização,x,y,action)`, `paste(pid/hwnd+localização)`,
`shot(pid/hwnd+localização,out,locate,color,format)`,
`assert-text(pid,class,contains,timeoutMs)`, `assert-file(path,timeoutMs)`,
`sleep(ms)`, `print(msg)`, `close(pid)`.
Vars: `--var k=v` + `${k}` em qualquer campo; `run save:` publica `${pid}`;
var salva com valor VAZIO expande para vazio (1.6 — antes saía literal por
pegadinha VCL). Falha aborta, **mata os PIDs abertos pelo workflow** e sai 1;
sucesso fecha os PIDs no `finally` (1.6) e sai 0
(`WORKFLOW=PASS pass=N fail=0`). Workflow que envia algo deve TERMINAR com
`assert-empty` do campo de origem: WORKFLOW=PASS = envio provado.

Exemplos em `examples/` (ambos rodam em qualquer Windows e fecham o notepad
ao terminar): `key-notepad.wh` (7/7, teclado por eventos + asserts de texto) e
`shot-notepad.wh` (7/7, `settext` + print com retângulo localizador + assert
de arquivo). Use-os como molde: troque `exe`, `class` e `caption` pelos do
seu app. Os valores de pid/hwnd mudam a cada execução — descubra com `find`,
nunca reutilize de uma sessão anterior.

## Gotchas pagos (consulte antes de escrever driver novo)

- `lpCurrentDirectory` vazio é INVÁLIDO — passar `nil`, nunca `PChar('')`.
- Parser de flags em cadeia if/else-if: incrementar I **só no else final**
  (NextStr que falha não pode consumir a posição).
- `TJSONArray.Items[I]` (não `.Objects`); eventos do .dfm exigem handler em
  `published`; comentário `{ }` Pascal **não aninha** (nunca `{...}` dentro).
- `--json` do `play` silencia logs de step (saída = linha única parseável).
- `--json` com caption acentuada sai no console em CP1252: em python,
  `sys.stdin.buffer.read().decode('cp1252')` antes do `json.loads`.
- `--hwnd` com o HWND **numérico** do desktop (`GetDesktopWindow`) devolve 0
  itens — para todas as janelas use `--hwnd 0`.
- Vírgula é **separador**, nunca tecla: literal é `#188` (VK_OEM_COMMA,
  layout-dependente; ponto `#190`).
- Botão de suite VCL (ex.: `TsuiButton`) pode **ignorar `BM_CLICK`**: o `click`
  retorna ok e nada acontece (falso verde, provado ao vivo 2026-10-07).
  Rota B: `mouse --hwnd N --action left` (eventos no centro).
  **Sempre verifique o efeito real** (campo limpo / linha no histórico), nunca
  só o exit do comando.
- Campo que só reage a teclado APÓS clique (provado ao vivo): sem um
  `mouse --action left` NO PRÓPRIO campo, NENHUMA tecla sintetizada entra
  (nem WM_CHAR digita; foco sintético/attach não resolveu). Receita de
  "enter envia": `settext msg` → `mouse left` no campo → `key z` (arma o
  OnKey) → `key #8` (apaga o z) → `key #13` (envia; o app limpa o campo).
  Limpar antes: `clear --hwnd N` (devolve `len=0` como prova).
- pid de step `run` com `save:` grava em Vars **E** Pids (expansão `${pid}` +
  kill de emergência).
- Receita Git Bash: pid via `sed 's/.*"pid":\([0-9]*\).*/\1/'`; hwnd do
  primeiro item via sed **ancorado** `sed 's/^hwnd=\([0-9]*\).*/\1/'` (com
  espaço antes de `hwnd` não casa); pid/hwnd vindos de PowerShell precisam
  `tr -d '\r'` (CRLF quebra StrToIntDef).
- dcc32: UTF-8 **com BOM**; `-B` para build all; verde = exit 0 **com**
  output esperado.
- VCL: `TStringList.Values[nome] := ''` com nome INEXISTENTE não registra o
  par (silencioso) — para publicar var possivelmente vazia use `Add` explícito
  (`IndexOfName < 0`); pago no 1.6 (print `${var}` literal).

## Quando usar / não usar

- USE: validar/clicar qualquer app Win32 com HWND (VCL, WinForms clássico,
  Qt com HWND nativo), regressão de harness, localizar janelas abertas,
  capturar textos de controles, teclado/mouse sem foco (SendMessage direto,
  dispensa foreground — não trava CDP nem rouba foco).
- NÃO USE: conteúdo de página WebView/Electron/Chrome (a página inteira é UM
  HWND — elementos do DOM não são filhos Win32; use automação de browser/CDP).
  **CDP pra página, handles pra janela.** Também não é para testes unitários
  Delphi nem diálogo pontual de abrir arquivo.
