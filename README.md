# auto-piloto

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

## Instalação

```bash
git clone https://github.com/dex-agent/auto-piloto ~/.agents/skills/auto-piloto
~/.agents/skills/auto-piloto/bin/DriveAutoTeste.exe --json schema
```

A raiz do repositório é a skill; `bin/` vai junto.

## Uso rápido

```bash
# listar as janelas abertas
./DriveAutoTeste.exe find --hwnd 0 --titled

# subir um app e descobrir os controles
./DriveAutoTeste.exe --json run --exe "C:/Windows/System32/notepad.exe"
./DriveAutoTeste.exe find --pid <pid>

# exemplo que roda em qualquer Windows
./DriveAutoTeste.exe play --file examples/key-notepad.wh
```

Referência completa de comandos, steps e exit codes: [docs/USAGE.md](docs/USAGE.md).
Regras de uso para agentes: [SKILL.md](SKILL.md).

## Exemplos

| Arquivo | Mostra |
|---|---|
| `examples/key-notepad.wh` | teclado por eventos (setas movem o cursor de verdade), asserts de texto |
| `examples/shot-notepad.wh` | `settext` + print com retângulo localizador + assert de arquivo |

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
