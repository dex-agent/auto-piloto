# auto-piloto

**Versão do CLI neste pacote: 1.9.0** (16 comandos, incluindo `wait` por
condição e o sensor `probe` para vigias).

Automação de UI Win32 por handles, sem escrever driver novo por sistema. Este
pacote junta a **skill `auto-piloto`** (o conhecimento de uso) e o
**CLI `DriveAutoTeste.exe`** (a ferramenta), prontos para instalar com um
clone.

## O que faz

Enumera janelas e controles Win32 (`EnumWindows`/`EnumChildWindows`) e envia
mensagens (`BM_CLICK`, `WM_GETTEXT`/`WM_SETTEXT`, `WM_KEYDOWN/CHAR/UP`,
`WM_LBUTTON*`, `WM_PASTE`), além de tira print de handle com retângulo
localizador. Funciona até em formulário invisível.

**Regra de ouro:** antes de declarar impossível automatizar uma UI Win32,
enumerar os handles e enviar a mensagem.

## Instalação (Windows)

Requisito: [Git for Windows](https://git-scm.com/download/win). A raiz do
repositório é a skill; `bin\` vai junto.

### Prompt de instalação

Copie e cole **uma linha de cada vez** no **Prompt de Comando** (`cmd`):

```bat
git clone https://github.com/dex-agent/auto-piloto "%USERPROFILE%\.agents\skills\auto-piloto"
"%USERPROFILE%\.agents\skills\auto-piloto\bin\DriveAutoTeste.exe" --json schema
```

No **PowerShell**:

```powershell
git clone https://github.com/dex-agent/auto-piloto "$HOME\.agents\skills\auto-piloto"
& "$HOME\.agents\skills\auto-piloto\bin\DriveAutoTeste.exe" --json schema
```

Deu certo quando o segundo comando imprime uma linha começando com
`{"ok":true,"cmd":"schema",...`.

> Use `%USERPROFILE%` (cmd) ou `$HOME` (PowerShell) com **barra invertida**.
> O `~` do Linux e a barra `/` não funcionam no Prompt de Comando.

### Se a pasta já existir

`fatal: destination path ... already exists and is not an empty directory`
significa que `%USERPROFILE%\.agents\skills\auto-piloto` já existe (outra
instalação da skill). Descubra de quem é a pasta:

```bat
git -C "%USERPROFILE%\.agents\skills\auto-piloto" remote get-url origin
```

- Imprime `https://github.com/dex-agent/auto-piloto` → é este repositório;
  atualize com `git -C "%USERPROFILE%\.agents\skills\auto-piloto" pull`.
- Imprime outra coisa ou dá erro → a pasta é de outro projeto. **Não apague**:
  clone em outra pasta e use o exe de lá:

```bat
git clone https://github.com/dex-agent/auto-piloto "%USERPROFILE%\auto-piloto"
"%USERPROFILE%\auto-piloto\bin\DriveAutoTeste.exe" --json schema
```

### Para agentes (Claude Code, Codex, OpenCode e outros)

Peça ao agente:

```text
Instale a skill auto-piloto: clone https://github.com/dex-agent/auto-piloto
na pasta de skills do agente (Windows: %USERPROFILE%\.agents\skills\auto-piloto)
e valide com bin\DriveAutoTeste.exe --json schema. Se a pasta já existir e não
for deste repositório, clone em %USERPROFILE%\auto-piloto e não apague a
pasta existente.
```

## Uso rápido

No Prompt de Comando, dentro da pasta instalada:

```bat
cd /d "%USERPROFILE%\.agents\skills\auto-piloto"

rem listar as janelas abertas
bin\DriveAutoTeste.exe find --hwnd 0 --titled

rem exemplo que roda em qualquer Windows
bin\DriveAutoTeste.exe play --file examples\key-notepad.wh
```

Referência completa de comandos, steps e exit codes: [docs/USAGE.md](docs/USAGE.md).
Regras de uso para agentes: [SKILL.md](SKILL.md).

## Exemplos

| Arquivo | Mostra |
|---|---|
| `examples/key-notepad.wh` | teclado por eventos (setas movem o cursor de verdade), asserts de texto |
| `examples/shot-notepad.wh` | `settext` + print com retângulo localizador + assert de arquivo |
| `examples/mouse-wheel-notepad.wh` | roda do mouse (1.7.5): escreve 40 linhas, print de antes, 5 entalhes para baixo, print de depois |

Workflow = arquivo JSON com `name`, `prompt` (playbook) e `steps`; um arquivo
por regressão, zero código novo.

## Limites

- Não serve para conteúdo de página WebView/Electron/Chrome: a página
  inteira é um HWND só (use automação de browser/CDP). **CDP pra página,
  handles pra janela.**
- Sem garantia. Sempre confira o efeito real, nunca só o exit code.

## Licença

Skill, documentação e exemplos: [MIT](LICENSE).

O binário `bin/DriveAutoTeste.exe` é distribuído **para uso livre**, sem
código-fonte e sem garantia; os direitos sobre ele ficam reservados pelo
autor.
