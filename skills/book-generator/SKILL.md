---
name: book-generator
description: Gera ebook (epub, pdf, html, markdown, mobi, azw3) e audiobook (mp3, m4b) a partir de um prompt de texto, de um arquivo .md ou de uma pasta com vários .md. Use quando o usuário digitar /book-generator, pedir para gerar ebook ou audiobook, transformar um .md ou uma pasta de capítulos em epub/pdf/mobi/audiobook, ou citar os formatos do book-generator. Por padrão o texto dos .md é preservado exatamente (modo estrito); use --strict-mode=false para permitir edição.
---

# /book-generator

Transforma texto em livro. Duas origens: um **prompt** (eu escrevo os capítulos) ou **arquivos .md** (o texto deles é usado como está). Saídas: ebook e audiobook, em um ou vários formatos, escolhidos por parâmetro.

O motor é `scripts/book_engine.py`, ao lado deste arquivo. Ele não reescreve nada: só monta o manuscrito, verifica o texto e chama pandoc, Calibre, LaTeX, ffmpeg e edge-tts.

## Parâmetros

| Parâmetro | Padrão | Significado |
|---|---|---|
| `--source <caminho>` | — | Um `.md` ou uma pasta com `.md` (ordem natural, sem subpastas, ignora nomes que começam com `_`). |
| `--prompt "<texto>"` | — | Escrevo os capítulos a partir do prompt. Use `--source` **ou** `--prompt`, não os dois. |
| `--chapters <n>` | 8 | Só com `--prompt`: quantidade de capítulos. |
| `--formats <lista>` | `ebook` | Lista separada por vírgula. Formatos: `epub`, `pdf`, `html`, `markdown`, `mobi`, `azw3`, `mp3`, `m4b`. Grupos: `ebook` (os 6 primeiros), `audiobook` (`mp3`, `m4b`), `all` (tudo). Ex.: `--formats=epub,m4b`. |
| `--strict-mode <true\|false>` | `true` | Ver "Regra do texto" abaixo. |
| `--title`, `--subtitle`, `--author`, `--publisher`, `--isbn`, `--year` | título do primeiro H1 ou nome da pasta | Metadados. Autor e editora ficam vazios se você não informar. |
| `--language <pt-BR\|en-US>` | `pt-BR` | Idioma dos metadados, rótulos do sumário e voz padrão. |
| `--cover <imagem>` | — | Capa para epub, mobi e azw3. |
| `--voice <voz>` | `pt-BR-FranciscaNeural` (en: `en-US-AriaNeural`) | Voz do edge-tts. |
| `--rate <+10%>` | `+0%` | Velocidade da narração. |
| `--output <pasta>` | `book_outputs/<slug>/exports_<data>` | Onde gravar. |
| `--fast` | — | Pula mobi e azw3, como o `compile_book.sh` do repositório. |
| `--dry-run` | — | Só monta o manuscrito e o manifest. Serve para conferir antes de gastar tempo com áudio. |

## Regra do texto

**Origem .md, modo estrito (padrão):** o texto de cada `.md` sai exatamente como está. Eu não corrijo, não reescrevo, não completo. O motor só acrescenta metadados, quebras de página entre arquivos e o sumário gerado pelo pandoc. Antes de exportar, ele confere que cada arquivo aparece inteiro e sem alteração no manuscrito. Se algum não aparecer, nada é exportado (código de saída 3).

**Origem .md, `--strict-mode=false`:** eu posso editar o texto (corrigir erros, reescrever trechos, completar capítulos, adaptar a formatação). Regras:
1. Trabalho sempre numa cópia: `book_outputs/<slug>/source/`. Os `.md` originais não mudam.
2. Escrevo `CHANGES.md` na pasta de saída, com o que mudei em cada arquivo e por quê.
3. O motor também remove sobras de IA (`NOTE TO WRITER`, `Word count:` e similares) e registra cada linha removida no manifest.

**Origem prompt:** eu escrevo os `.md` em `book_outputs/<slug>/source/` (`01-...md`, `02-...md`, ...), cada um começando com `# Capítulo N: Título`. Depois o texto desses arquivos vale como fonte, com a mesma verificação de texto. Os capítulos são meus: não há modo estrito nesse caso, mas a saída fica igual ao que escrevi.

## Fluxo

1. Lê os parâmetros. Sem `--formats`, usa `ebook`.
2. Origem prompt: escreve os capítulos e segue para o passo 3 com `--origin prompt`.
3. Roda o motor (ou `--dry-run` primeiro, se o livro for grande):

```powershell
python "$HOME\.claude\skills\book-generator\scripts\book_engine.py" --source "C:\caminho\do\livro" --formats epub,pdf
```

4. Lê o `manifest.json` da saída. Se algum formato falhou, mostro o erro e a instalação que falta (abaixo).
5. Entrego os caminhos dos arquivos gerados e o resumo do manifest (formatos ok, falhas, verificação de texto).

## Requisitos por formato

- **epub, html, pdf:** pandoc. Para pdf também é preciso um motor LaTeX (lualatex ou xelatex).
- **mobi, azw3:** Calibre (`ebook-convert`). Dependem do epub.
- **mp3, m4b:** edge-tts (Python) e internet. m4b também precisa do ffmpeg.
- **markdown:** nada extra.

Instalação (uma linha por bloco):

```powershell
winget install JohnMacFarlane.Pandoc
```

```powershell
winget install Gyan.FFmpeg
```

```powershell
python -m pip install edge-tts
```

```powershell
winget install MiKTeX.MiKTeX
```

```powershell
winget install calibre.calibre
```

## Privacidade

- O áudio usa o serviço de leitura do Microsoft Edge (edge-tts). **O texto do livro é enviado a esse serviço.** Para texto que não pode sair do computador, peça `--formats=ebook`.
- Nenhuma API paga é usada. O motor não envia nada além do que o edge-tts precisa para narrar.

## Saída

```
exports_<data>/
  manuscript.md          texto montado (o mesmo usado em todos os formatos)
  book.epub  book.pdf  book.html  book.mobi  book.azw3   (conforme pedido)
  book.css
  audiobook/
    01-<capitulo>.mp3    um arquivo por arquivo .md
    book.mp3             livro inteiro
    book.m4b             com marcas de capítulo
  manifest.json          fontes com sha256, modo, verificação, ferramentas e erros
```

## Notas técnicas

- O `compile_book.sh` do repositório não é usado para o texto. O filtro awk dele apaga tudo o que vem depois de uma linha que começa com "In this chapter", e o `sed` remove negrito. No modo estrito isso não pode acontecer.
- Os formatos de saída seguem os mesmos comandos do `compile_book.sh` (pandoc epub3, pandoc pdf com lualatex, pandoc html5, ebook-convert). Não criei capa automática: use `--cover`.
- Código de saída: `0` tudo certo, `1` algum formato falhou, `2` parâmetro ou origem inválidos, `3` verificação de texto falhou.
