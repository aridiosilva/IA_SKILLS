# Agent + Skills + Computer: Como Funcionam as Skills do Claude AI

[English](README.md) | [Português](README.pt.md)

---

## Sumário

1. [Arquitetura Agent + Skills + Computer](#1-arquitetura-agent--skills--computer)
2. [Onde ficam os SKILL.md: Chrome vs. Excel vs. Claude Code](#2-onde-ficam-os-skillmd-chrome-vs-excel-vs-claude-code)
3. [Especificação do SKILL.md — anatomia geral](#3-especificação-do-skillmd--anatomia-geral)
4. [Conteúdo completo: Skill `docx`](#4-skill-docx)
5. [Conteúdo completo: Skill `xlsx`](#5-skill-xlsx)
6. [Conteúdo completo: Skill `pptx`](#6-skill-pptx)
7. [Scripts de verificação obrigatória](#7-scripts-de-verificação-obrigatória)
8. [Síntese: padrões de design que atravessam tudo](#8-síntese-padrões-de-design-que-atravessam-tudo)

---

## 1. Arquitetura Agent + Skills + Computer

![](https://github.com/aridiosilva/IA_SKILLS/blob/main/Claude_AI/images/agent-skills-computer-diagram-no-CLAUDE-AI.png)

O diagrama de referência ("Agent + Skills + Computer") divide a arquitetura em dois blocos principais:

- **Agent configuration** (esquerda) — o que está "dentro da cabeça" do Claude: o prompt de sistema, as Skills equipadas e os servidores MCP conectados.
- **Agent virtual machine** (direita) — o ambiente computacional real (uma VM/sandbox) onde o Claude efetivamente executa código e manipula arquivos.

A seta central rotulada **"use computer"** é a ponte entre os dois: é assim que o agente sai da "configuração" e efetivamente age no mundo (rodando bash, Python, Node.js).

![Diagrama: Agent + Skills + Computer](agent-skills-computer-diagram.png)

### 1.0 Fluxo de funcionamento descrito em palavras

A imagem acima não é um fluxograma de "passo 1, passo 2, passo 3" no sentido estrito — é um diagrama de **arquitetura estática**, mostrando onde cada peça mora e como se conecta às demais. Mas dá para narrar o fluxo de funcionamento que ela implica, na ordem em que ele acontece na prática:

**Etapa 1 — Preparação (antes de qualquer tarefa do usuário).**
Quando uma sessão do agente começa, o **Core system prompt** é carregado primeiro — é a base de comportamento do Claude. Junto com ele, os metadados (nome + `description`) de cada uma das **Equipped Skills** são pré-carregados no mesmo prompt de sistema. Nenhum conteúdo completo de skill é lido ainda — só o suficiente para o agente saber que aquelas capacidades existem e quando cada uma se aplica. Em paralelo, os **Equipped MCP servers** configurados também ficam disponíveis como conexões prontas, mas ainda não usadas.

**Etapa 2 — O usuário faz um pedido.**
O agente interpreta a tarefa e decide, com base nas `description`s pré-carregadas, se alguma Skill se encaixa. Se sim, ele lê o `SKILL.md` completo daquela skill — isso é a "ativação" de segundo nível de divulgação progressiva.

**Etapa 3 — A skill instrui uma ação no computador.**
É aqui que a seta central **"use computer"** entra em cena: para executar o que o `SKILL.md` recomenda (rodar um script, escrever um arquivo, chamar uma biblioteca), o agente atravessa a fronteira entre a "configuração" (onde só existiam instruções) e a **Agent virtual machine** (onde as coisas de fato acontecem) — a VM com Bash, Python e Node.js.

**Etapa 4 — Os arquivos da skill já estão lá, esperando.**
A caixa verde do diagrama ("Contents of Skill directories live in the agent computer's file system") documenta uma pré-condição, não uma etapa em si: o conteúdo de cada Skill equipada (o `SKILL.md` e todos os arquivos auxiliares — `.md` de referência, scripts `.py`) já foi materializado como pastas reais dentro do sistema de arquivos da VM, antes mesmo de a tarefa começar. Por isso, quando o agente decide "vou usar a skill `pdf`", ele não precisa baixar nada — ele simplesmente navega até `skills/pdf/` com os mesmos comandos que usaria para qualquer outra pasta, e executa o que encontra lá (por exemplo, `extract_fields.py`).

**Etapa 5 — Se a tarefa exigir dados ou ferramentas externas.**
Caso a skill sozinha não seja suficiente — por exemplo, se for necessário consultar um banco de dados ou uma API de terceiros — o agente recorre aos **MCP servers** configurados, que por sua vez se conectam aos **Remote MCP servers** na internet (mostrados na parte inferior esquerda do diagrama). Essa é uma via paralela e independente da via das Skills: MCP traz dados/ferramentas de fora; Skills trazem know-how de como usar tudo isso corretamente.

**Etapa 6 — Execução e retorno.**
O resultado da ação executada na VM (arquivo criado, dado consultado, código rodado) volta para o fluxo de conversa do agente, que então responde ao usuário — encerrando o ciclo até o próximo pedido.

Em resumo, o fluxo é: **carregar metadados leves → identificar a skill certa pela `description` → carregar as instruções completas → atravessar para a VM via "use computer" → executar usando os arquivos da skill já presentes no sistema de arquivos → opcionalmente estender a ação com MCP para dados/ferramentas externas → devolver o resultado.**

### 1.1 Core system prompt

O prompt-base do agente — instruções fundamentais de comportamento, papel e regras carregadas antes de qualquer tarefa.

### 1.2 Equipped Skills

Skills disponíveis (ex.: `bigquery`, `docx`, `nda-review`, `pdf`, `pptx`, `xlsx`). O conceito central é a **divulgação progressiva (progressive disclosure)**, em três níveis:

1. **Nível 1 (sempre carregado):** apenas nome + `description` de cada skill — poucas dezenas de tokens cada, permitindo dezenas de skills coexistirem sem estourar o contexto.
2. **Nível 2 (carregado ao ativar):** quando uma tarefa bate com a `description`, o `SKILL.md` completo é carregado — instruções procedurais detalhadas.
3. **Nível 3 (carregado sob demanda):** arquivos de referência adicionais dentro da skill (`FORMS.md`, `REFERENCE.md`), lidos apenas se a tarefa específica exigir.

### 1.3 Equipped MCP servers

MCP (**Model Context Protocol**) conecta o agente a ferramentas e fontes de dados externas — bancos de dados, APIs, serviços de terceiros — apontando para **servidores MCP remotos** na internet.

| Dimensão | Skills | MCP |
|---|---|---|
| Papel principal | Conhecimento procedural (como fazer) | Conectividade com ferramentas (com o quê fazer) |
| Unidade | Pasta com `SKILL.md` | Servidor com endpoints |
| Onde vive | Sistema de arquivos do agente | Sessão/configuração de cliente |
| Persistência | Baseada em arquivo | Baseada em sessão |

**MCP dá acesso a ferramentas externas; Skills dão know-how sobre como usar bem essas ferramentas** (ou realizar tarefas complexas em geral).

### 1.4 Agent virtual machine

A "máquina" onde o Claude realmente trabalha, com acesso a **Bash**, **Python** e **Node.js**, e um sistema de arquivos real onde cada Skill equipada é materializada como uma pasta:

```
skills/bigquery/
├── SKILL.md
├── datasources.md
└── rules.md

skills/docx/
├── SKILL.md
└── ooxml/
    ├── spec.md
    └── editing.md

skills/nda-review/
└── SKILL.md

skills/pdf/
├── SKILL.md
├── forms.md
├── reference.md
└── extract_fields.py
```

Cada skill pode conter, além do `SKILL.md` obrigatório: **arquivos de referência** (`.md`), **scripts executáveis** (`.py`) e **subpastas** organizando conhecimento por subtópico.

### 1.5 A ponte entre os dois mundos

A ideia-chave: as Skills não são apenas "instruções abstratas" — elas **existem literalmente como arquivos reais** dentro da VM do agente. O Claude as acessa com os mesmos comandos que usaria para navegar em qualquer sistema de arquivos, não como algo embutido magicamente no modelo.

### 1.6 Por que essa arquitetura importa

1. **Escalabilidade de contexto** — centenas de skills sem sobrecarregar a janela de contexto, graças à divulgação progressiva.
2. **Separação de responsabilidades** — Skills = conhecimento processual; MCP = conectividade; VM = ambiente de execução.
3. **Modularidade e reuso** — Skills são pastas versionáveis e compartilháveis, seguindo um padrão aberto emergente (Agent Skills Specification).

---

## 2. Onde ficam os SKILL.md: Chrome vs. Excel vs. Claude Code

Uma constatação importante: **nem toda superfície do Claude usa a arquitetura de `SKILL.md` local em disco.** Ela é específica de ambientes com acesso a "computador" (VM com bash/Python/Node).

### 2.1 Claude for Chrome (extensão do navegador)

**Não há pasta `skills/` nem `SKILL.md` local para essa extensão.**

- É uma extensão Manifest V3 que abre como painel lateral ao lado da aba ativa.
- Usa o SDK da Anthropic diretamente no navegador, autenticando via OAuth.
- Em vez de Skills em arquivo, opera com uma ferramenta de **"computer use"** (mouse/teclado/screenshot via Chrome DevTools Protocol): o modelo "vê" a tela via screenshot, decide uma ação, a extensão executa, tira novo screenshot, repete o loop.
- Não existe filesystem de skills exposto localmente.

📁 **No Windows:** o único artefato local é o próprio pacote da extensão instalado pelo Chrome (`...\User Data\Default\Extensions\<id>\`), contendo apenas código-fonte da extensão (JS/HTML) — não Skills no sentido do diagrama.

### 2.2 Claude for Excel (add-in do Office 365)

Aqui as Skills **existem e são aplicadas automaticamente**, mas ainda **não como arquivos soltos no disco Windows**:

> "As Skills habilitadas nas configurações da sua conta Claude também ficam disponíveis nos add-ins do Excel, PowerPoint, Word e Outlook. O Claude aplica as skills relevantes automaticamente enquanto você trabalha." — documentação oficial

- O add-in é um *task pane* (interface web) rodando dentro do Excel, comunicando-se com os servidores da Anthropic na nuvem.
- As Skills habilitadas ficam gerenciadas **na conta claude.ai**, em **Customize → Skills** — não em pasta local.
- Digitar `/` na barra lateral lista as Skills disponíveis (vindas da nuvem); o conteúdo do `SKILL.md` "roda" do lado do servidor, não no `C:\`.

📁 **No Windows:** não há `SKILL.md` local para o add-in do Excel — apenas o manifesto do add-in do Office instalado via AppSource, sem relação com o conteúdo das Skills.

### 2.3 Claude Code / Desktop (onde o `SKILL.md` local de fato existe)

```
C:\Users\<seu_usuário>\.claude\skills\          ← Skills pessoais (globais, todos os projetos)
<pasta_do_projeto>\.claude\skills\               ← Skills específicas de um projeto
```

Cada skill em sua própria subpasta, com `SKILL.md` na raiz:

```
C:\Users\aridio\.claude\skills\
├── minha-skill\
│   └── SKILL.md
└── outra-skill\
    ├── SKILL.md
    └── scripts\
```

### 2.4 Resumo comparativo

| Superfície | Skills locais em disco? | Onde ficam de fato |
|---|---|---|
| **Claude for Chrome** | Não | Não usa Skills no sentido do diagrama; usa computer-use puro |
| **Claude for Excel (add-in)** | Não | Gerenciadas na nuvem, via claude.ai → Customize → Skills |
| **Claude Code / Desktop (Code)** | **Sim** | `C:\Users\<usuário>\.claude\skills\` |

---

## 3. Especificação do SKILL.md — anatomia geral

Estrutura padrão de uma skill (exemplo real: skill `pdf`, equipada no ambiente de execução):

```
/mnt/skills/public/pdf/
├── SKILL.md                          ← obrigatório, ponto de entrada
├── FORMS.md                          ← referência sob demanda
├── REFERENCE.md                      ← referência sob demanda
├── LICENSE.txt                       ← termos de licença
└── scripts/                          ← código executável
    ├── check_bounding_boxes.py
    ├── check_fillable_fields.py
    ├── convert_pdf_to_images.py
    ├── extract_form_field_info.py
    ├── extract_form_structure.py
    ├── fill_fillable_fields.py
    └── fill_pdf_form_with_annotations.py
```

### 3.1 Campos do YAML frontmatter

```yaml
---
name: pdf
description: Use this skill whenever the user wants to do anything with PDF files...
license: Proprietary. LICENSE.txt has complete terms
---
```

| Campo | Função |
|---|---|
| `name` | Identificador único da skill (minúsculas, hífens, ≤64 caracteres) |
| `description` | **O mais crítico.** Único texto pré-carregado no contexto o tempo todo, usado para decidir se a skill se aplica à tarefa. Escrita como regra de ativação ("Use this skill whenever...", "If the user mentions...") |
| `license` | Opcional — termos de uso do conteúdo da skill |

### 3.2 Corpo do SKILL.md

Segue um padrão pragmático comum: **Quick Start** → bibliotecas/ferramentas específicas com snippets prontos → avisos operacionais concretos ("gotchas") → referências ao próximo nível de detalhe (`REFERENCE.md`, `FORMS.md`).

### 3.3 A pasta `scripts/`

Quando a skill empacota **código Python/JS pronto** em vez de apenas instruir, o agente executa o script já testado via bash em vez de gerar código do zero a cada vez — reduzindo variância e superfície de erro. Do ponto de vista de GRC, a proveniência e integridade desses scripts importa tanto quanto qualquer dependência de terceiros em um pipeline de software.

---

## 4. Skill `docx`

**Caminho:** `/mnt/skills/public/docx/SKILL.md` · **Tamanho:** ~7K

### Estrutura de diretório

```
docx/
├── SKILL.md (7K)
├── LICENSE.txt
└── scripts/
    ├── accept_changes.py
    ├── comment.py
    ├── merge_runs.py
    └── office/ (1.1M — LibreOffice empacotado)
```

### Conteúdo completo

```yaml
---
name: docx
description: "Use this skill whenever the user wants to create, read, edit, or manipulate Word documents (.docx) or Word templates (.dotx). Triggers include: any mention of 'Word doc', 'word document', '.docx', '.dotx', or requests to produce professional documents with formatting like tables of contents, page numbers, or letterheads. Also use when extracting or reorganizing content from .docx or .dotx files, inserting or replacing images in documents, find-and-replace in Word files, working with tracked changes or comments, or converting content into a polished Word document. If the user asks for a 'report', 'memo', 'letter', 'template', or similar deliverable as a Word or .docx file (to download, email or print), use this skill. However, if they ask for a document, page, report, memo, or notes WITHOUT naming a file format and the session offers a dedicated document or page skill or connector, use that instead. Do NOT use for PDFs, spreadsheets, Google Docs, or coding unrelated to document generation."
license: Proprietary. LICENSE.txt has complete terms
---
```

# DOCX creation, editing, and analysis

A `.docx` is a ZIP archive of XML files. Choose your approach by task:

| Task | Approach |
|---|---|
| **Create** a new document | Write a `docx` (npm) script — see gotchas below |
| **Edit** an existing document | `unzip` → edit `word/document.xml` → `zip` (docx-js cannot open existing files) |
| **Read** content | `pandoc -t markdown file.docx` |

> Script paths below are relative to this skill's directory.

## Creating with docx-js — gotchas

`docx` is preinstalled — do not run `npm install` first; write the script and `require('docx')` directly. Only if that require fails: `npm install docx`. The model knows the API; these are the footguns:

- **Page size defaults to A4.** For US Letter set `page: { size: { width: 12240, height: 15840 } }` (DXA; 1440 = 1″).
- **Landscape:** pass portrait dimensions and `orientation: PageOrientation.LANDSCAPE` — docx-js swaps width/height internally.
- **Tables need dual widths:** set `columnWidths` on the table AND `width` on every cell, both in `WidthType.DXA` (PERCENTAGE breaks in Google Docs). Column widths must sum to the table width.
- **Table shading:** use `ShadingType.CLEAR`, never `SOLID` (renders black).
- **Lists:** never insert `•` literally; use a `numbering` config with `LevelFormat.BULLET`.
- **`ImageRun` requires `type:`** (`"png"`, `"jpg"`, …).
- **`PageBreak` must be inside a `Paragraph`.**
- **Never use `\n`** — use separate `Paragraph` elements.
- **TOC:** headings must use built-in `HeadingLevel.*`; custom heading styles need `outlineLevel` set or they won't appear.
- **Don't use a table as a horizontal rule** — use a paragraph bottom border instead.
- **Dot-leader / right-aligned-on-same-line:** use `PositionalTab` (`alignment: PositionalTabAlignment.RIGHT`, `leader: PositionalTabLeader.DOT`) inside a `TextRun`, not literal `.` or space padding.

## Verify the output

After writing a `.docx`, render it and look at it:

```bash
python scripts/office/soffice.py --headless --convert-to pdf output.docx
pdftoppm -jpeg -r 100 output.pdf page
ls page-*.jpg   # then Read the images
```

`pdftoppm` zero-pads page numbers to the width of the page count (`page-01.jpg`…`page-12.jpg`).

## Editing existing documents

Legacy `.doc` files must be converted first: `python scripts/office/soffice.py --headless --convert-to docx file.doc`.

```bash
unzip -q doc.docx -d unpacked/
find unpacked -type l -delete   # strip symlink entries — docx from external parties is untrusted
python scripts/merge_runs.py unpacked/   # coalesce fragmented runs so text is findable
# edit unpacked/word/document.xml in place — do NOT reformat or pretty-print
(cd unpacked && rm -f ../out.docx && zip -Xr ../out.docx .)
python scripts/office/validate.py out.docx --original doc.docx   # XSD checks; --auto-repair fixes common issues
# redlining? add --author "<the name you redlined under>" to check every edit is tracked
```

Word splits text across many `<w:r>` runs (revision ids, spell-check markers), so a phrase you can see in the document often doesn't exist as a contiguous string in the XML. `merge_runs.py` merges adjacent identically-formatted runs in `word/document.xml` without changing content or rendering; it also accepts a `.docx` directly (`python scripts/merge_runs.py doc.docx -o merged.docx`).

**Tracked changes:** when redlining, validate with `--author "<the name you redlined under>"` (needs `--original`) — it reports any text you changed without a `<w:ins>`/`<w:del>` around it, which is easy to do by accident and invisible in the accepted view. Wrap runs in `<w:ins>`/`<w:del>` with `w:id`, `w:author`, `w:date` attributes. Inside `<w:del>`, the text element is `<w:delText>`, not `<w:t>`. A deleted paragraph mark (`<w:pPr><w:rPr><w:del w:id=".." w:author=".." w:date=".."/></w:rPr></w:pPr>`) means "merge this paragraph into the next" — so deleting a paragraph outright is that plus a `<w:del>` around every run. The `<w:del/>` must come before the rPr's other children; their order is schema-enforced.

To produce a clean copy with all tracked changes accepted: `python scripts/accept_changes.py in.docx out.docx`.

Accepting a deleted paragraph mark should join that paragraph to the one below it, so a paragraph whose runs are *all* deleted vanishes. Word does this; `accept_changes.py` and `pandoc --track-changes=accept` don't always. Both fail the same way — they strip the deleted text but leave the emptied paragraph behind, which reads as a stray empty bullet when it was auto-numbered:

- `pandoc --track-changes=accept` never joins the paragraphs.
- `accept_changes.py` (LibreOffice) joins them correctly, except when the deleted paragraph is followed by an empty spacer paragraph.

An empty bullet in either view is an artifact of that view, not a defect in the document. Check paragraph deletions in the XML.

## Comments

Comments require six cross-linked files. Use the helper — directory mode when you'll also be editing `document.xml` (saves an unzip/rezip cycle), `.docx`-direct mode otherwise:

```bash
# Against an already-unpacked directory (preferred when also placing markers)
python scripts/comment.py unpacked/ "Fees & expenses cap is too low"
python scripts/comment.py unpacked/ "Agreed" --parent 0

# Against a .docx directly
python scripts/comment.py contract.docx "This cap is too low" -o annotated.docx
```

The script writes `comments.xml`, `commentsExtended.xml`, `commentsIds.xml`, `commentsExtensible.xml`, the relationships, and the content-type overrides. Comment IDs are auto-assigned. It then prints the `<w:commentRangeStart>`/`<w:commentRangeEnd>`/`<w:commentReference>` snippet to add to `word/document.xml` so the comment anchors to specific text — until you place those markers, the comment exists but is not visible.

## Dependencies

`docx` (npm, preinstalled — install only if `require('docx')` fails) · `pandoc` · LibreOffice (`soffice`) · `pdftoppm` (Poppler)

---

## 5. Skill `xlsx`

**Caminho:** `/mnt/skills/public/xlsx/SKILL.md` · **Tamanho:** ~8.5K

### Estrutura de diretório

```
xlsx/
├── SKILL.md (8.5K)
├── LICENSE.txt
└── scripts/
    ├── recalc.py
    └── office/ (1.1M — LibreOffice empacotado)
```

### Conteúdo completo

```yaml
---
name: xlsx
description: "Use this skill any time a spreadsheet file is the primary input or output. This means any task where the user wants to: open, read, edit, or fix an existing .xlsx, .xlsm, .xltx, .csv, or .tsv file (e.g., adding columns, computing formulas, formatting, charting, cleaning messy data); create a new spreadsheet from scratch or from other data sources; or convert between tabular file formats. Trigger especially when the user references a spreadsheet file by name or path — even casually (like \"the xlsx in my downloads\") — and wants something done to it or produced from it. Also trigger for cleaning or restructuring messy tabular data files (malformed rows, misplaced headers, junk data) into proper spreadsheets. The deliverable must be a spreadsheet file. Do NOT trigger when the primary deliverable is a Word document, HTML report, standalone Python script, database pipeline, or Google Sheets API integration, even if tabular data is involved."
license: Proprietary. LICENSE.txt has complete terms
---
```

# XLSX creation, editing, and analysis

| Task | Approach |
|---|---|
| **Create** or **edit** with formulas/formatting | `openpyxl` — see gotchas below |
| **Bulk data** in or out | `pandas` (`read_excel`, `to_excel`) |
| **Quick look** at a sheet | `markitdown file.xlsx` — `## SheetName` per sheet; reads `.xlsm` too. No cell coordinates, so don't plan edits from it |
| **Read** a model (formulas *and* values) | two `load_workbook` passes — see gotchas |

> `openpyxl`, `pandas`, and `markitdown` are preinstalled — do not run `pip install` first; write the script and import directly. Only if an import fails (or the `markitdown` command is missing): `pip install` the missing package.

> Script paths below are relative to this skill's directory.

## Requirements for every output

- **Professional font** (Arial, Times New Roman) throughout, unless the user says otherwise.
- **Zero formula errors.** Never ship while `recalc.py` reports `errors_found`. If you think an error predates you, prove it: load the *original* with `data_only=True` and look at that cell. An error you introduced looks exactly like one you inherited.
- **Use formulas, never hardcoded results.** Write `sheet['B10'] = '=SUM(B2:B9)'`, not the Python-computed total. The sheet must recalculate when its inputs change.
- **Follow the user's spec literally.** Exact tab names, exact column headers, and the formula they spelled out. A redesign that computes something else fails, however elegant.
- **Document every assumption and hardcoded number** where the reader will see it — a cell comment, or an adjacent cell at a table's end. Cite a real source when one exists (`Source: Company 10-K, FY2024, Page 45, Revenue Note, [SEC EDGAR URL]`); when the number came from the user, say so plainly.
- **A workbook *you create* for someone to fill in** needs a short legend naming which cells to edit, and one example row of realistic values showing the expected format. Never add such a row to a file you were asked to edit.
- **Editing an existing file: match its conventions exactly.** They override every guideline here. Find its designated input cells first — a distinct font color, fill, or shading marks them — write only there, and leave every existing formula untouched.

## Recalculate (mandatory whenever the file contains formulas)

openpyxl writes formulas as strings with **no cached values**. Until you recalculate, every
formula cell reads back as `None` to anything reading cached values — `pandas`,
`load_workbook(data_only=True)`, and most previewers.

```bash
python scripts/recalc.py output.xlsx [timeout_seconds]   # default 30
```

LibreOffice computes every formula, the file is **rewritten in place**, and you get JSON:
`status` (`success` | `errors_found`), `total_formulas`, `total_errors`, and an
`error_summary` naming up to 100 cells per error type (`locations_truncated` says how many it
withheld — trust `total_errors`, not the length of the list). Fix what it names and run it
again. **JSON with an `error` key instead of a `status` means nothing was recalculated**, and
only that case exits non-zero — `errors_found` exits 0, so never treat a clean exit as a clean
workbook.

**A green recalc proves your formulas *evaluate*, not that they are *right*.** An off-by-one
range or a reference to the wrong row yields a clean, error-free file with wrong numbers.
Write 2–3 formulas first and check they pull the values you expect, before building out a grid.

**A workbook that links to another file loses those links** if you re-save it with openpyxl and
then recalculate. Such a formula reads `='[1]Returns Analysis'!$B$2` — the `[1]` is an index
into the workbook's external-reference list, naming a *separate file on disk*, not a sheet.
That file is rarely present here, so the cell's cached value is the only thing holding its
data. openpyxl strips that value on save; LibreOffice then has to resolve the reference for
real, fails, writes `#NAME?`, and deletes every link. `recalc.py` refuses to run in that state
— copy those cells' values out of the original before you save over them (`--force` overrides,
and accepts the loss).

## Choosing formulas that survive verification

LibreOffice implements fewer functions than Excel, and one it cannot evaluate becomes a
literal `#NAME?` baked into the file you deliver.

- **Prefer Excel-2007-era functions** — `SUMIFS`, `INDEX`, `MATCH`, `IFERROR`, `SUMPRODUCT` — which need no prefix.
- **Six post-2007 functions work, but only with an `_xlfn.` prefix**, because openpyxl writes your formula into the XML verbatim and Excel stores post-2007 names prefixed (its UI hides the prefix): `_xlfn.TEXTJOIN`, `_xlfn.CONCAT`, `_xlfn.IFS`, `_xlfn.SWITCH`, `_xlfn.MAXIFS`, `_xlfn.MINIFS`. Written bare, each yields `#NAME?`.
- **Never use `XLOOKUP`, `XMATCH`, `SORT`, `FILTER`, `UNIQUE`, or `SEQUENCE`.** The runtime's LibreOffice cannot evaluate them under *any* prefix. Newer builds do evaluate them, but they are spilling array functions and an openpyxl-written file has no spill metadata, so only the top-left cell of the range gets a value — and `recalc.py` reports `total_errors: 0` on the truncated result. Use `INDEX`/`MATCH` for lookups, and sort, filter, and de-duplicate in Python before writing the cells.
- A formula LibreOffice could not parse is written back **lowercased** — a quick tell beside a `#NAME?`.

## openpyxl gotchas

- **Reading a model takes two loads.** `data_only=True` yields cached values with the formulas gone; the default yields formula strings with no values. One pass cannot give you both.
- **`data_only=True` is destructive if you save.** That workbook has no formulas left, so saving replaces every one with a literal — permanently.
- **`data_only=True` on a file openpyxl just wrote returns `None` everywhere** — run `recalc.py` first. (A formula whose result is `""` also reads back as `None`.)
- **Merged cells: write the top-left anchor only.** Every other cell in the range is a `MergedCell` whose `.value` is read-only.
- **`.xlsm` loses its macros unless you pass `keep_vba=True`** to `load_workbook`.
- **A sheet name containing a space must be quoted** in a cross-sheet reference: `='Assumptions Inputs'!$B$5`. Unquoted, it evaluates to `#VALUE!`.

## Financial models

Unless the user says otherwise, or the existing file already does something else.

**Color:** blue text (`0,0,255`) for hardcoded inputs and scenario levers · black for formulas ·
green (`0,128,0`) for links to another sheet · red (`255,0,0`) for links to another file ·
yellow fill (`255,255,0`) for key assumptions and cells the user should fill in.

**Numbers:** currency `$#,##0`, with the unit named in the header (`Revenue ($mm)`) · zeros
render as `-`, including in percentages (`$#,##0;($#,##0);-`) · negatives in parentheses ·
percentages `0.0%`, **stored as fractions** (`0.15` renders `15.0%`; storing `15` renders
`1500.0%`) · valuation multiples `0.0x` · years as text (`"2024"`, never `2,024`).

**Structure:** every assumption in its own labeled cell, referenced by the formulas that use it
(`=B5*(1+$B$6)`, never `=B5*1.05`) · formulas consistent across every projection period, since a
lone edited cell mid-row is the commonest silent error · guard denominators that can be zero.

## Dependencies

`openpyxl`, `pandas`, `markitdown` (pip, preinstalled — install only if an import fails or the command is missing) · LibreOffice (`soffice`, auto-configured for sandboxed environments via `scripts/office/soffice.py`)

---

## 6. Skill `pptx`

**Caminho:** `/mnt/skills/public/pptx/SKILL.md` · **Tamanho:** ~22K (a mais extensa das três)

### Estrutura de diretório

```
pptx/
├── SKILL.md (22K)
├── LICENSE.txt
└── scripts/
    ├── add_slide.py
    ├── clean.py
    ├── thumbnail.py
    └── office/ (1.1M — LibreOffice + validadores)
```

### Conteúdo completo

```yaml
---
name: pptx
description: "Use this skill any time a .pptx or .potx file is involved in any way — as input, output, or both. This includes: creating slide decks, pitch decks, or presentations as PowerPoint (.pptx) files; reading, parsing, or extracting text from any .pptx or .potx file (even if the extracted content will be used elsewhere, like in an email, summary, or creating a different type of slide deck); editing, modifying, or updating existing presentations; combining or splitting slide files; working with templates (.potx), layouts, speaker notes, or comments. Trigger whenever the user asks for a PowerPoint or .pptx file, or references a .pptx or .potx filename, regardless of what they plan to do with the content afterward. However, when the user asks for a deck, slides, a slide deck, or a presentation without naming a file format, default to using a dedicated slide-deck artifact type or a separate slides skill if this session offers one; otherwise, use this skill."
license: Proprietary. LICENSE.txt has complete terms
---
```

# PPTX creation, editing, and analysis

If this session offers a dedicated slide-deck artifact type or a separate slides skill, and the user has neither asked for a PowerPoint/.pptx file nor supplied a .pptx/.potx file, build the deck with that type or skill instead; this skill remains the right tool for producing .pptx files and for reading, editing, templating, or converting existing .pptx/.potx files.

A `.pptx` is a ZIP archive of XML files. Choose your approach by task:

| Task | Approach |
|---|---|
| **Create** a new deck | Write a `pptxgenjs` script — see gotchas below |
| **Edit** an existing deck, or build from a template | unzip → edit `ppt/slides/slideN.xml` → zip |
| **Read** content | `markitdown deck.pptx` (one block per slide under `<!-- Slide number: N -->` markers); visual grid: `python scripts/thumbnail.py deck.pptx` |

## Scripts

Paths are relative to this skill's directory. Everything else is plain Python, `node`, or shell.

| Script | What it does |
|---|---|
| `scripts/thumbnail.py deck.pptx [prefix]` | Labeled grid of every slide, for picking template layouts. `.pptx` only. Pass `prefix` — it defaults to `thumbnails`, which overwrites the grids of any other deck done in the same directory |
| `scripts/add_slide.py unpacked/ slide2.xml [--after slideN.xml]` | Duplicate a slide (or a `slideLayoutN.xml`) with all the package bookkeeping. Also takes a `.pptx` directly with `-o out.pptx` |
| `scripts/clean.py unpacked/` | Delete slides, media, and rels no longer referenced. Run **after** `<p:sldIdLst>` is final |
| `scripts/office/validate.py deck.pptx [--original src.pptx]` | Schema, relationship, content-type, chart and slide checks; each failure names its fix. Pass `--original` for any template-derived deck — it baselines the schema checks against the template, so the template's own XSD errors don't read as yours |
| `scripts/office/soffice.py --headless --convert-to pdf deck.pptx` | LibreOffice wrapper — bare `soffice` hangs in this sandbox |

## Creating with pptxgenjs — gotchas

`pptxgenjs` is preinstalled — do not run `npm install` first; write the script and `require('pptxgenjs')` directly. Only if that require fails: `npm install pptxgenjs`. The model knows the API; these are the footguns:

- **Set `pres.layout` before adding slides.** The default canvas is `LAYOUT_16x9` = **10" × 5.625"**, not 13.3" wide. Coordinates past the edge are written, not clamped — the shape just isn't on the slide. (`LAYOUT_WIDE` is 13.3" × 7.5".)
- **Hex colors: never `#`, never 8 digits.** `color: "FF0000"`. Both `"#FF0000"` and alpha baked into the hex (`"00000020"`) **corrupt the file**. For translucency: `transparency: 0-100` on fills and images, `opacity: 0.0-1.0` on shadows — each is silently ignored on the other.
- **pptxgenjs mutates option objects in place** (converts values to EMU on first use). Never share one `shadow`/options object across two `add*` calls — build a fresh object each time.
- **Shadow `offset` must be ≥ 0** — a negative offset corrupts the file. To cast a shadow upward, use `angle: 270` with a positive offset.
- **`letterSpacing` is silently ignored** — the real option is `charSpacing`.
- **Lists:** `bullet: true` on each item, never a literal `•` (renders double bullets). Set `breakLine: true` on every array item except the last. Space bulleted paragraphs with `paraSpaceAfter`, not `lineSpacing` (huge gaps).
- **One `new pptxgen()` per output file** — never reuse an instance.
- **`rectRadius` only works on `ROUNDED_RECTANGLE`**, not `RECTANGLE`.
- **Gradient fills aren't supported** — use a gradient image as the background instead.
- **Every `addText` call needs `isTextBox: true`** — without it the shape lacks `txBox="1"`, so screen readers announce the text as a "graphic" instead of a text box. No visual change.
- **Text boxes have built-in internal padding** — set `margin: 0` whenever text must align with a shape, line, or icon at the same x.
- **Speaker notes go in `slide.addNotes("...")`** (plain text, once per slide), never in a text box on the slide.
- **Keep charts native.** Use `addChart()` for everything PowerPoint can chart (pass an array of `{type, data, options}` for combos). For PowerPoint-native features the library doesn't expose (trendlines, error bars), compute the extra series yourself or post-process the generated OOXML — do not fall back to a rendered image. Only chart types PowerPoint has no native form for (Sankey, network, chord) go in as images.
- **Default charts render bare** — no title, no data labels, dated palette. Set `showTitle` + `title`, `showValue: true` + `dataLabelPosition`, `chartColors: [...]` from your palette, and quiet the frame (`catAxisLabelColor`/`valAxisLabelColor`, `valGridLine: { color, size }`, `catGridLine: { style: "none" }`, `showLegend: false` for a single series).
- **On a stacked bar or column chart, `dataLabelPosition` must be `ctr`, `inEnd`, or `inBase`.** `outEnd` **corrupts the file**.
- **A combo series using `secondaryValAxis`/`secondaryCatAxis` needs both `valAxes` and `catAxes` on the chart options, two entries each.** Without them pptxgenjs writes axis *ids* it never declares, and PowerPoint **discards that chart** and reports the file as corrupt. Supplying only `valAxes` is not enough.
- **After `writeFile()`, run `python scripts/office/validate.py deck.pptx`.** It reports the two chart faults above and the slide-XML defects PowerPoint refuses, and names the fix for each. Fix them in your generator, not by hand-editing the packed XML.
- **Never reorder the children of `<p:presentation>`.** pptxgenjs writes `<p:notesMasterIdLst>` right after `<p:sldIdLst>` and points both masters at one theme part. PowerPoint reads that happily — move the element and the same deck becomes unopenable.
- **Icons:** render `react-icons` to SVG (`ReactDOMServer.renderToStaticMarkup`), rasterize with `sharp` at ≥256px, and insert via `addImage({ data: "image/png;base64," + buf.toString("base64") })` — the `image/png;base64,` prefix is required (`react-icons`, `react`, `react-dom`, and `sharp` are preinstalled — `npm install react-icons react react-dom sharp` only if a require fails).

## Editing existing decks and templates

Pick layouts first: `python scripts/thumbnail.py template.pptx template-thumbs` writes a labeled grid of every slide and prints the file(s) it created — `template-thumbs.jpg`, split into `template-thumbs-N.jpg` past 12 slides. **Always pass that second argument, named after the deck.** It defaults to `thumbnails`, so two decks thumbnailed in one directory silently overwrite each other's grids — the first deck's are simply gone (template analysis only — visual QA needs the full-resolution renders from Converting to Images; it only accepts `.pptx`, so copy a `.potx` to a `.pptx` name first). Use it with `markitdown` to map each content section onto a template slide, and vary the layouts — don't put every section on the same title-and-bullets slide.

```bash
python3 -c "import sys,zipfile; zipfile.ZipFile(sys.argv[1]).extractall('unpacked')" deck.pptx
python scripts/add_slide.py unpacked/ slide2.xml --after slide2.xml   # duplicate a slide (or slideLayoutN.xml); prints the new slide's path
# reorder / delete slides = edit <p:sldIdLst> in ppt/presentation.xml
python scripts/clean.py unpacked/                                     # after deletions: removes orphaned slides, media, rels
# edit slide content in ppt/slides/slideN.xml
(cd unpacked && rm -f ../out.pptx && zip -Xr ../out.pptx .)           # zip from INSIDE the dir; rm first or deleted parts survive
python scripts/office/validate.py out.pptx --original deck.pptx
```

- **Do all structural work — add, delete, reorder — before editing any slide's content.** `add_slide.py` copies a slide file verbatim, so duplicating after you edit clones the edited content; and `clean.py` deletes any slide missing from `<p:sldIdLst>`, including one you just wrote.
- **Never copy a slide file by hand** — `add_slide.py` does every registration a new slide needs and reports what it made (`Created ppt/slides/slide17.xml from slide2.xml`). It also works directly on a file: `add_slide.py deck.pptx slide2.xml -o out.pptx` — **pass `-o`, or it rewrites the input deck in place.** A duplicated slide still *references* its source's chart/SmartArt/embedded-object parts rather than cloning them, so editing one slide's chart changes the other's.
- **If you use `python-pptx`**, three things it won't do: duplicate a slide (its only entry point is `add_slide(layout)`), preserve formatting through `text_frame.text = "..."` (that collapses the paragraph to a single unstyled run — assign `run.text` instead), or read the SVG/EMF most template art uses (`add_picture` raises `UnidentifiedImageError`).
- Legacy `.ppt` must be converted first: `python scripts/office/soffice.py --headless --convert-to pptx file.ppt`. `.potx` templates unpack and pack identically — keep the `.potx` extension on the output.
- To reuse a template icon or image, duplicate a slide or layout that already contains it.

When filling in a template:

- If you script an XML transform, parse with `defusedxml.minidom` — round-tripping OOXML through `xml.etree.ElementTree` rewrites namespace prefixes and corrupts the deck.
- **Template slots ≠ source items.** If the template shows 4 team members and you have 3, delete the 4th member's entire group (image + text boxes), not just its text — then check for orphaned visuals in QA.
- One `<a:p>` per list item — never concatenate items into a single paragraph. Copy the sibling `<a:pPr>` to preserve spacing, and put `b="1"` on the `<a:rPr>` of titles, section headers, and inline labels (`Status:`, `Owner:`).
- Let bullets inherit from the layout; only add `<a:buChar>`, `<a:buAutoNum>` (numbered), or `<a:buNone>` to override — never a literal `•` in the text.
- Text with leading or trailing spaces needs `xml:space="preserve"` on its `<a:t>`.

## Design Ideas

**Don't create boring slides.** Plain bullets on a white background won't impress anyone. Consider ideas from this list for each slide.

### Before Starting

- **Pick a bold, content-informed color palette**: The palette should feel designed for THIS topic. If swapping your colors into a completely different presentation would still "work," you haven't made specific enough choices.
- **Dominance over equality**: One color should dominate (60-70% visual weight), with 1-2 supporting tones and one sharp accent. Never give all colors equal weight.
- **Dark/light contrast**: Dark backgrounds for title + conclusion slides, light for content ("sandwich" structure). Or commit to dark throughout for a premium feel.
- **Commit to a visual motif**: Pick ONE distinctive element and repeat it — rounded image frames, icons in colored circles. Carry it across every slide. **Do not use a color bar or accent stripe as your motif** (see Avoid list).

### Color Palettes

Choose colors that match your topic — don't default to generic blue. Use these palettes as inspiration:

| Theme | Primary | Secondary | Accent |
|-------|---------|-----------|--------|
| **Midnight Executive** | `1E2761` (navy) | `CADCFC` (ice blue) | `FFFFFF` (white) |
| **Forest & Moss** | `2C5F2D` (forest) | `97BC62` (moss) | `F5F5F5` (cream) |
| **Coral Energy** | `F96167` (coral) | `F9E795` (gold) | `2F3C7E` (navy) |
| **Warm Terracotta** | `B85042` (terracotta) | `E7E8D1` (sand) | `A7BEAE` (sage) |
| **Ocean Gradient** | `065A82` (deep blue) | `1C7293` (teal) | `21295C` (midnight) |
| **Charcoal Minimal** | `36454F` (charcoal) | `F2F2F2` (off-white) | `212121` (black) |
| **Teal Trust** | `028090` (teal) | `00A896` (seafoam) | `02C39A` (mint) |
| **Berry & Cream** | `6D2E46` (berry) | `A26769` (dusty rose) | `ECE2D0` (cream) |
| **Sage Calm** | `84B59F` (sage) | `69A297` (eucalyptus) | `50808E` (slate) |
| **Cherry Bold** | `990011` (cherry) | `FCF6F5` (off-white) | `2F3C7E` (navy) |

### For Each Slide

**Every slide needs a visual element** — image, chart, icon, or shape. Text-only slides are forgettable.

**Layout options:**
- Two-column (text left, illustration on right)
- Icon + text rows (icon in colored circle, bold header, description below)
- 2x2 or 2x3 grid (image on one side, grid of content blocks on other)
- Half-bleed image (full left or right side) with content overlay

**Data display:**
- Large stat callouts (big numbers 60-72pt with small labels below)
- Comparison columns (before/after, pros/cons, side-by-side options)
- Timeline or process flow (numbered steps, arrows)

**Visual polish:**
- Icons in small colored circles next to section headers
- Italic accent text for key stats or taglines

### Typography

**Font names you write into the .pptx are rendered by the user's PowerPoint, not by this environment.** Your visual QA renders via LibreOffice, which substitutes fonts it doesn't have — and for some fonts the substitute has different widths, so your QA preview can show text overflow (or fit) that the real deck won't have. To keep your QA trustworthy:

- **Safe fonts** (render true-to-width in QA *and* ship with Office): **Arial, Calibri, Cambria, Times New Roman, Courier New, Bookman Old Style, Century Schoolbook**. Use these for body text and anything where fit matters.
- **Headers with personality at zero QA risk**: pair a safe-list serif header (Cambria, Bookman Old Style, Century Schoolbook) with a safe-list sans body (Calibri or Arial). You get visual contrast without giving up reliable overflow checks.
- **If the user asks for a font outside the safe list** (e.g. Georgia or Trebuchet MS): use it where the user asked, but size those containers with extra slack (~10%) and don't trust QA text-fit on those elements — the preview of that font is approximate. If the user hasn't specified, prefer safe-list fonts for body text.
- **QA-unreliable fonts** (substitute has different widths — overflow checks can be wrong): Georgia, Trebuchet MS, Impact, Arial Black, Garamond, Consolas, Palatino Linotype. Calibri Light substitution varies by environment; treat as QA-unreliable. Fine for titles/accents with slack; don't trust QA text-fit on these.
- **Never default to Aptos** — Office's post-2023 default has no metric-compatible substitute here *and* is missing from older Office installs, so it's unreliable on both ends.

| Element | Size |
|---------|------|
| Slide title | 36-44pt bold |
| Section header | 20-24pt bold |
| Body text | 14-16pt |
| Captions | 10-12pt muted |

### Spacing

- 0.5" minimum margins
- 0.3-0.5" between content blocks
- Leave breathing room—don't fill every inch

### Avoid (Common Mistakes)

- **Don't repeat the same layout** — vary columns, cards, and callouts across slides
- **Don't center body text** — left-align paragraphs and lists; center only titles
- **Don't skimp on size contrast** — titles need 36pt+ to stand out from 14-16pt body
- **Don't default to blue** — pick colors that reflect the specific topic
- **Don't mix spacing randomly** — choose 0.3" or 0.5" gaps and use consistently
- **Don't style one slide and leave the rest plain** — commit fully or keep it simple throughout
- **Don't create text-only slides** — add images, icons, charts, or visual elements; avoid plain title + bullets
- **Don't forget text box padding** — when aligning lines or shapes with text edges, set `margin: 0` on the text box or offset the shape to account for padding
- **Don't use low-contrast elements** — icons AND text need strong contrast against the background; avoid light text on light backgrounds or dark text on dark backgrounds
- **NEVER use accent lines under titles** — these are a hallmark of AI-generated slides; use whitespace or background color instead
- **NEVER add decorative color bars or accent stripes** — this includes: header/footer bars spanning the slide width, vertical sidebar stripes down one edge of the slide, thin accent stripes along one edge of a card or content block, and "single-side borders" on rectangles. These read as AI-generated filler. If you want to set a card apart, use a subtle background tint, a drop shadow, or an icon — not an edge stripe.
- **Don't default to cream/beige backgrounds** — when no background is specified, use white (`FFFFFF`) or the user's brand palette; avoid warm-neutral defaults like `F5F5DC`, `FAF0E6`, `FAEBD7`, `FFF8E1`
- **Don't ship text that overflows its shape** — if text doesn't fit, reduce font size, split across slides, or enlarge the container; never leave content cut off or spilling past bounds

## QA (Required)

Your first render usually has a few real issues — overlaps, overflow, misalignment. Find and fix those, re-render only the slides you changed, and stop.

### Content QA

```bash
markitdown output.pptx
```

Check for missing content, typos, wrong order.

**When using templates, check for leftover placeholder text:**

```bash
markitdown output.pptx | grep -iE "\bx{3,}\b|lorem|ipsum|\bTODO|\[insert|this.*(page|slide).*layout"
```

If grep returns results, fix them before declaring success.

### File QA (required)

```bash
python scripts/office/validate.py output.pptx                      # built from scratch
python scripts/office/validate.py output.pptx --original src.pptx  # built from a template
```

**If the deck came from a template, always pass `--original`.** A template may itself
contain parts the XSD rejects, so a bare run can report failures you never caused — and
a genuine regression can hide among them. `--original` baselines
the schema and slide checks against the template, suppressing errors it already had.
The structural checks — relationships, content types, charts — ignore `--original` and
report template-inherited problems either way, so read those on their own merits.

pptxgenjs emits chart XML PowerPoint refuses to open, and every other tool
accepts: python-pptx opens those decks, LibreOffice renders them, the XSD
passes them. Every failure names its fix. Fix it in the generator and rebuild.

### Visual QA

Convert the slides to images (see Converting to Images) and inspect every one. After staring at the generating code you tend to see what you expect rather than what rendered, so look at the images fresh (a subagent works well for this if you have one). User-visible defects to look for:

- **Text overflow or text cut off at a box or slide boundary — check this first.** It is the most common defect and always user-visible. (For a font the previewer renders unreliably per Typography, the preview is approximate: trust the ~10% slack you left, not its apparent fit.)
- Overlapping elements (text through shapes, lines through words, stacked elements)
- Source citations or footers colliding with content above
- Elements too close (< 0.3" gaps) or cards/sections nearly touching
- Uneven gaps (large empty area in one place, cramped in another)
- Insufficient margin from slide edges (< 0.5")
- Columns or similar elements not aligned consistently
- Low-contrast text (e.g., light gray text on cream-colored background)
- Template decoration mispositioned after text replacement — e.g., a title underline positioned for one line, but the replaced title wrapped to two
- Low-contrast icons (e.g., dark icons on dark backgrounds without a contrasting circle)
- Text boxes too narrow causing excessive wrapping
- Leftover placeholder content

## Converting to Images

Convert presentations to individual slide images for visual inspection:

```bash
python scripts/office/soffice.py --headless --convert-to pdf output.pptx
rm -f slide-*.jpg
pdftoppm -jpeg -r 150 output.pdf slide
ls -1 "$PWD"/slide-*.jpg
```

**Pass the absolute paths printed above directly to the view tool.** The `rm` clears stale images from prior runs. `pdftoppm` zero-pads based on page count: `slide-1.jpg` for decks under 10 pages, `slide-01.jpg` for 10-99, `slide-001.jpg` for 100+.

**After fixes, rerun all four commands above** — the PDF must be regenerated from the edited `.pptx` before `pdftoppm` can reflect your changes.

## Dependencies

`pptxgenjs` (npm, preinstalled — install only if `require('pptxgenjs')` fails) · `markitdown[pptx]`, `Pillow`, `defusedxml`, `lxml` (pip — text dump, thumbnail, clean, validate) · LibreOffice (`soffice`, auto-configured for sandboxed environments via `scripts/office/soffice.py`) · `pdftoppm` (Poppler)

---

## 7. Scripts de verificação obrigatória

Dois scripts que atuam como controle de qualidade **obrigatório**, rodados sempre após criação/edição de arquivos, antes de qualquer entrega ser considerada concluída.

### 7.1 `recalc.py` — recalcula fórmulas do Excel e verifica erros

**Caminho:** `/mnt/skills/public/xlsx/scripts/recalc.py`
**Por quê existe:** o `openpyxl` grava fórmulas como texto puro, sem calcular o resultado. Até alguém recalcular, toda célula com fórmula "lê" como vazia (`None`).

**Passo a passo:**

1. Verifica se o arquivo existe e é gravável.
2. **Checagem de segurança contra perda de dados:** varre o workbook procurando fórmulas que apontam para outro arquivo externo com valor em cache já perdido — se encontrar, **recusa recalcular** (só prossegue com `--force`).
3. Cria um perfil temporário e isolado do LibreOffice, injetando uma macro Basic mínima (`calculateAll()` + `store()` + `close()`).
4. Executa o LibreOffice em modo headless via essa macro, com timeout e watchdog externo (`timeout`/`gtimeout`).
5. Confirma que o arquivo realmente mudou (compara timestamp/tamanho) — "saída limpa sem mudança" também é falha.
6. Reabre o arquivo e varre por 7 erros clássicos do Excel: `#VALUE!`, `#DIV/0!`, `#REF!`, `#NAME?`, `#NULL!`, `#NUM!`, `#N/A`.
7. Devolve JSON estruturado com `status`, `total_errors`, `total_formulas`, `error_summary`.

**Código completo:**

```python
"""
Excel Formula Recalculation Script
Recalculates all formulas in an Excel file using LibreOffice
"""

import contextlib
import json
import os
import platform
import re
import shutil
import subprocess
import sys
import tempfile
import time
import zipfile
from pathlib import Path

from office.soffice import get_soffice_env, run_soffice

from openpyxl import load_workbook
from openpyxl.worksheet.formula import ArrayFormula

MACRO_FILENAME = "Module1.xba"
SOFFICE_MISSING = "soffice not found on PATH; LibreOffice is required to recalculate"

MAX_LOCATIONS = 100

EXTERNAL_REF_RE = re.compile(r"""(?<![\w"\[])'?\[\d+\][^!"\[\]]*'?!""")

RECALCULATE_MACRO = """<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE script:module PUBLIC "-//OpenOffice.org//DTD OfficeDocument 1.0//EN" "module.dtd">
<script:module xmlns:script="http://openoffice.org/2000/script" script:name="Module1" script:language="StarBasic">
    Sub RecalculateAndSave()
      ThisComponent.calculateAll()
      ThisComponent.store()
      ThisComponent.close(True)
    End Sub
</script:module>"""


def has_gtimeout():
    try:
        subprocess.run(
            ["gtimeout", "--version"], capture_output=True, timeout=1, check=False
        )
        return True
    except (FileNotFoundError, subprocess.TimeoutExpired):
        return False


def _stamp(path):
    st = os.stat(path)
    return st.st_mtime_ns, st.st_size


def setup_libreoffice_macro(profile_dir: Path, timeout=30):
    url = profile_dir.as_uri()
    try:
        run_soffice(
            ["--headless", "--terminate_after_init", f"-env:UserInstallation={url}"],
            capture_output=True,
            timeout=timeout,
        )
    except FileNotFoundError:
        return None, SOFFICE_MISSING
    except subprocess.TimeoutExpired:
        return None, "LibreOffice timed out creating its profile; formulas were NOT recalculated"

    macro_dir = profile_dir / "user" / "basic" / "Standard"
    if not macro_dir.exists():
        return None, "LibreOffice did not create a usable profile; formulas were NOT recalculated"

    try:
        (macro_dir / MACRO_FILENAME).write_text(RECALCULATE_MACRO)
    except OSError as e:
        return None, f"Could not install the recalculation macro: {e}"

    return url, None


def _names_pattern(names):
    if not names:
        return None
    return re.compile(r"\b(" + "|".join(re.escape(n) for n in sorted(names)) + r")\b")


def _existing_cells(ws):
    if not hasattr(ws, "iter_rows"):
        return []
    return [ws._cells[key] for key in sorted(ws._cells)]


def external_links_at_risk(filename):
    try:
        with zipfile.ZipFile(filename) as archive:
            names = archive.namelist()
    except (zipfile.BadZipFile, OSError):
        return []
    if not any(n.startswith("xl/externalLinks/") for n in names):
        return []

    with contextlib.ExitStack() as stack:
        formulas = load_workbook(filename, data_only=False)
        stack.callback(formulas.close)
        values = load_workbook(filename, data_only=True)
        stack.callback(values.close)

        defined_names = list(formulas.defined_names.items())
        for scope_ws in formulas.worksheets:
            defined_names.extend(scope_ws.defined_names.items())
        name_texts = [
            (name, dn.value)
            for name, dn in defined_names
            if isinstance(getattr(dn, "value", None), str)
        ]
        external_names = {name for name, text in name_texts if EXTERNAL_REF_RE.search(text)}
        while True:
            name_re = _names_pattern(external_names)
            if name_re is None:
                break
            added = {
                name
                for name, text in name_texts
                if name not in external_names and name_re.search(text)
            }
            if not added:
                break
            external_names |= added
        name_re = _names_pattern(external_names)

        at_risk = []
        for sheet in formulas.sheetnames:
            ws = formulas[sheet]
            cached = values[sheet]
            for cell in _existing_cells(ws):
                v = cell.value
                if isinstance(v, ArrayFormula):
                    v = v.text
                if not (isinstance(v, str) and v.startswith("=")):
                    continue
                reaches_out = EXTERNAL_REF_RE.search(v) or (name_re and name_re.search(v))
                if reaches_out and cached[cell.coordinate].value is None:
                    at_risk.append(f"{sheet}!{cell.coordinate}")
        return at_risk


def recalc(filename, timeout=30, force=False):
    if not Path(filename).exists():
        return {"error": f"File {filename} does not exist"}

    abs_path = str(Path(filename).absolute())

    if not os.access(abs_path, os.W_OK):
        return {"error": f"{filename} is not writable; recalculation rewrites the file in place"}

    try:
        get_soffice_env()
    except Exception as e:
        return {"error": f"Could not prepare the LibreOffice environment: {e}"}

    if not force:
        try:
            at_risk = external_links_at_risk(filename)
        except Exception as e:
            return {"error": f"Could not inspect {filename} for external links: {e}"}
        if at_risk:
            shown = at_risk[:MAX_LOCATIONS]
            return {
                "error": (
                    "Refusing to recalculate: this workbook links to another workbook, and "
                    f"{len(at_risk)} linked cell(s) have lost their cached value (openpyxl strips "
                    "these on save). Recalculating would resolve them to #NAME? and delete the "
                    "external links for good. Copy those cells' values from the original file "
                    "before saving, or pass --force to accept the loss. Charts and conditional "
                    "formats can hold external references too, so this list may not be exhaustive."
                ),
                "external_link_cells": shown,
                "external_link_cells_truncated": max(0, len(at_risk) - len(shown)),
            }

    with tempfile.TemporaryDirectory(
        prefix="recalc-lo-profile-", ignore_cleanup_errors=True
    ) as profile_dir:
        return _recalc_with_profile(filename, abs_path, timeout, Path(profile_dir))


def _recalc_with_profile(filename, abs_path, timeout, profile_dir: Path):
    started = time.monotonic()
    profile_url, err = setup_libreoffice_macro(profile_dir, timeout=timeout)
    if err:
        return {"error": err}

    timeout = max(5, int(timeout - (time.monotonic() - started)))

    before = _stamp(abs_path)

    cmd = [
        "soffice",
        "--headless",
        "--norestore",
        f"-env:UserInstallation={profile_url}",
        "vnd.sun.star.script:Standard.Module1.RecalculateAndSave?language=Basic&location=application",
        abs_path,
    ]

    if platform.system() == "Linux" and shutil.which("timeout"):
        cmd = ["timeout", str(timeout)] + cmd
    elif platform.system() == "Darwin" and has_gtimeout():
        cmd = ["gtimeout", str(timeout)] + cmd

    timed_out = f"LibreOffice timed out after {timeout}s; formulas were NOT recalculated. Re-run with a longer timeout."

    try:
        result = subprocess.run(
            cmd, capture_output=True, text=True, env=get_soffice_env(), timeout=timeout + 15
        )
    except subprocess.TimeoutExpired:
        return {"error": timed_out}
    except FileNotFoundError:
        return {"error": SOFFICE_MISSING}

    if result.returncode == 124:
        return {"error": timed_out}

    if result.returncode != 0:
        detail = (result.stderr or "").strip() or f"soffice exited {result.returncode}"
        return {"error": f"LibreOffice failed to recalculate: {detail}"}

    if _stamp(abs_path) == before:
        return {
            "error": (
                "LibreOffice exited cleanly but never rewrote the file, so nothing was "
                "recalculated. Check that no other LibreOffice instance is running, then retry."
            )
        }

    try:
        wb = load_workbook(filename, data_only=True)

        excel_errors = [
            "#VALUE!",
            "#DIV/0!",
            "#REF!",
            "#NAME?",
            "#NULL!",
            "#NUM!",
            "#N/A",
        ]
        error_details = {err: [] for err in excel_errors}
        total_errors = 0

        for sheet_name in wb.sheetnames:
            ws = wb[sheet_name]
            for cell in _existing_cells(ws):
                if isinstance(cell.value, str):
                    for err in excel_errors:
                        if err in cell.value:
                            location = f"{sheet_name}!{cell.coordinate}"
                            error_details[err].append(location)
                            total_errors += 1
                            break

        result = {
            "status": "success" if total_errors == 0 else "errors_found",
            "total_errors": total_errors,
            "error_summary": {},
        }

        for err_type, locations in error_details.items():
            if locations:
                entry = {"count": len(locations), "locations": locations[:MAX_LOCATIONS]}
                if len(locations) > MAX_LOCATIONS:
                    entry["locations_truncated"] = len(locations) - MAX_LOCATIONS
                result["error_summary"][err_type] = entry

        wb.close()

        wb_formulas = load_workbook(filename, data_only=False)
        formula_count = 0
        for sheet_name in wb_formulas.sheetnames:
            ws = wb_formulas[sheet_name]
            for cell in _existing_cells(ws):
                v = cell.value
                if isinstance(v, ArrayFormula):
                    v = v.text
                if isinstance(v, str) and v.startswith("="):
                    formula_count += 1
        wb_formulas.close()

        result["total_formulas"] = formula_count

        return result

    except Exception as e:
        return {"error": str(e)}


def main():
    args = [a for a in sys.argv[1:] if a != "--force"]
    force = "--force" in sys.argv[1:]

    if not args:
        print("Usage: python recalc.py <excel_file> [timeout_seconds] [--force]")
        print("\nRecalculates all formulas in an Excel file using LibreOffice")
        print("\nReturns JSON with error details:")
        print("  - status: 'success' or 'errors_found'")
        print("  - total_errors: Total number of Excel errors found")
        print("  - total_formulas: Number of formulas in the file")
        print("  - error_summary: Breakdown by error type with locations")
        print("    - #VALUE!, #DIV/0!, #REF!, #NAME?, #NULL!, #NUM!, #N/A")
        print("\nOn any failure the JSON has an 'error' key and no 'status'.")
        print("--force recalculates even when it would destroy external links.")
        sys.exit(1)

    filename = args[0]
    try:
        timeout = int(args[1]) if len(args) > 1 else 30
    except ValueError:
        result = {"error": f"timeout must be an integer number of seconds, got {args[1]!r}"}
        print(json.dumps(result, indent=2))
        sys.exit(1)

    result = recalc(filename, timeout, force=force)
    print(json.dumps(result, indent=2))
    sys.exit(1 if "error" in result else 0)


if __name__ == "__main__":
    main()
```

### 7.2 `validate.py` — valida a estrutura XML de docx/pptx contra o schema oficial

**Caminho:** `/mnt/skills/public/pptx/scripts/office/validate.py` (módulo compartilhado com a skill `docx`)
**Por quê existe:** ferramentas como `python-pptx`, LibreOffice ou até o Word/PowerPoint às vezes abrem um arquivo malformado sem reclamar — mas o formato real (ISO/IEC 29500) é mais estrito. "Abre" não é o mesmo que "estruturalmente correto".

**Passo a passo:**

1. Aceita arquivo empacotado (`.docx`/`.pptx`) ou diretório já descompactado.
2. Detecta a família do arquivo e escolhe o validador certo (`DOCXSchemaValidator`, `PPTXSchemaValidator`); para `xlsx`, recusa validar e redireciona para `recalc.py`.
3. Valida contra os schemas XSD oficiais (ISO/IEC 29500).
4. **Modo `--original`:** usa o arquivo original/template como linha de base, para não acusar como "seu erro" um problema herdado do template.
5. **Checagem de tracked changes** (`--author`, só docx): verifica se toda diferença de texto está devidamente marcada com `<w:ins>`/`<w:del>`.
6. **Modo `--auto-repair`:** corrige automaticamente problemas comuns (IDs fora do limite, falta de `xml:space="preserve"`) e re-empacota.
7. Código de saída: `0` (tudo passou), `1` (falhou), `2` (erro de uso).

**Código completo:**

```python
"""
Command line tool to validate Office document XML files against XSD schemas and tracked changes.

Usage:
    python validate.py <path> [--original <original_file>] [--auto-repair] [--author NAME]

The first argument can be either:
- An unpacked directory containing the Office document XML files
- A packed Office file (.docx/.pptx/.xlsx or .dotx/.potx/.xltx template) which will be unpacked to a temp directory

Auto-repair fixes:
- paraId/durableId values that exceed OOXML limits
- Missing xml:space="preserve" on w:t elements with whitespace
"""

import argparse
import sys
import tempfile
import zipfile
from pathlib import Path

import defusedxml.ElementTree as ET
from defusedxml.common import DefusedXmlException

from helpers import OOXML_FAMILY, rezip, safe_extract
from validators import DOCXSchemaValidator, PPTXSchemaValidator, RedliningValidator

WORD_NS = "http://schemas.openxmlformats.org/wordprocessingml/2006/main"


def _fail(message: str):
    print(f"Error: {message}", file=sys.stderr)
    sys.exit(2)


def _has_tracked_changes(unpacked_dir: Path) -> bool:
    document = unpacked_dir / "word" / "document.xml"
    if not document.is_file():
        return False
    try:
        root = ET.parse(document).getroot()
    except (ET.ParseError, DefusedXmlException):
        return False
    tracked = {f"{{{WORD_NS}}}ins", f"{{{WORD_NS}}}del"}
    return any(elem.tag in tracked for elem in root.iter())


def main():
    parser = argparse.ArgumentParser(description="Validate Office document XML files")
    parser.add_argument(
        "path",
        help="Path to unpacked directory or packed Office file (.docx/.pptx/.xlsx or .dotx/.potx/.xltx)",
    )
    parser.add_argument(
        "--original",
        required=False,
        default=None,
        help="Path to original file (.docx/.pptx/.xlsx or .dotx/.potx/.xltx). If omitted, all XSD errors are reported and redlining validation is skipped.",
    )
    parser.add_argument(
        "-v",
        "--verbose",
        action="store_true",
        help="Enable verbose output",
    )
    parser.add_argument(
        "--auto-repair",
        action="store_true",
        help="Automatically repair common issues (hex IDs, whitespace preservation). "
        "Modifies the input in place: repairs to a packed file are written back to it.",
    )
    parser.add_argument(
        "--author",
        default=None,
        help="The name you are redlining under. Passing it turns on the "
        "tracked-change check: any text differing from --original without a "
        "<w:ins>/<w:del> recording it is reported. Untracked edits carry no "
        "author, so the check covers them whoever made them — the name marks "
        "the run as redlining work and is not used to filter. Requires "
        "--original; docx only.",
    )
    args = parser.parse_args()

    if args.author is not None and not args.original:
        _fail("--author requires --original")

    path = Path(args.path)
    if not path.exists():
        _fail(f"{path} does not exist")

    original_file = None
    if args.original:
        original_file = Path(args.original)
        if not original_file.is_file():
            _fail(f"{original_file} is not a file")
        if original_file.suffix.lower() not in OOXML_FAMILY:
            _fail(f"{original_file} must be one of: {', '.join(sorted(OOXML_FAMILY))}")

    family = OOXML_FAMILY.get((original_file or path).suffix.lower())
    if family is None:
        _fail(
            f"Cannot determine file type from {path}. Use --original or provide one of: {', '.join(sorted(OOXML_FAMILY))}."
        )

    if args.author is not None and family != "docx":
        _fail(f"--author only applies to docx files, not {family}")

    packed_file = None
    temp_dir_ctx = None
    if path.is_file() and path.suffix.lower() in OOXML_FAMILY:
        packed_file = path
        temp_dir_ctx = tempfile.TemporaryDirectory()
        unpacked_dir = Path(temp_dir_ctx.name)
        try:
            with zipfile.ZipFile(path, "r") as zf:
                safe_extract(zf, unpacked_dir)
        except (zipfile.BadZipFile, ValueError, OSError) as e:
            _fail(f"cannot unpack {path}: {e}")
    else:
        if not path.is_dir():
            _fail(f"{path} is not a directory or Office file")
        unpacked_dir = path

    match family:
        case "docx":
            validators = [
                DOCXSchemaValidator(unpacked_dir, original_file, verbose=args.verbose),
            ]
            if args.author is not None:
                validators.append(
                    RedliningValidator(unpacked_dir, original_file, verbose=args.verbose)
                )
            elif original_file and _has_tracked_changes(unpacked_dir):
                print(
                    "Note: this document has tracked changes; they were not "
                    "checked against the original (pass --author to check)."
                )
        case "pptx":
            validators = [
                PPTXSchemaValidator(unpacked_dir, original_file, verbose=args.verbose),
            ]
        case "xlsx":
            exts = ", ".join(k for k, v in sorted(OOXML_FAMILY.items()) if v == "xlsx")
            print(
                f"No XSD schema validation is performed for xlsx-family files ({exts}). "
                "For formula-error checking, use scripts/recalc.py instead."
            )
            sys.exit(0)
        case _:
            print(f"Error: Validation not supported for file type {family}")
            sys.exit(1)

    if args.auto_repair:
        total_repairs = sum(v.repair() for v in validators)
        if total_repairs:
            print(f"Auto-repaired {total_repairs} issue(s)")
            if packed_file is not None:
                rezip(unpacked_dir, packed_file)
                print(f"Wrote repaired file to {packed_file}")

    success = all([v.validate() for v in validators])

    if temp_dir_ctx is not None:
        temp_dir_ctx.cleanup()

    if success:
        print("All validations PASSED!")

    sys.exit(0 if success else 1)


if __name__ == "__main__":
    main()
```

### 7.3 Comparativo entre os dois scripts

| Aspecto | `recalc.py` | `validate.py` |
|---|---|---|
| **O que verifica** | Corretude numérica (fórmulas calculam sem erro) | Corretude estrutural (XML válido conforme o padrão ISO) |
| **Motor por trás** | LibreOffice (headless, via macro Basic) | Parser XML + schemas XSD oficiais |
| **Saída** | JSON estruturado, sempre | Texto + código de saída (0/1/2) |
| **Nunca faz** | Reportar sucesso sem confirmar que o arquivo mudou de fato | Validar contra o original sem que o usuário peça (`--original`) |
| **Filosofia** | "Um exit limpo não prova nada — leia o conteúdo" | "Avise quando uma checagem NÃO foi feita, não finja que foi" |

---

## 8. Síntese: padrões de design que atravessam tudo

1. **Divulgação progressiva em três níveis** — `description` sempre carregada → `SKILL.md` sob ativação → arquivos de referência sob demanda dentro da tarefa.
2. **Separação clara de papéis** — Skills = conhecimento procedural; MCP = conectividade externa; VM = ambiente de execução.
3. **`description` como regra de ativação, não resumo** — escrita para dizer explicitamente quando usar *e* quando não usar, evitando ativação da skill errada em casos ambíguos.
4. **Skills empacotam código, não só instrução** — reduz variância e risco de erro ao reusar scripts já testados em vez de gerar tudo do zero a cada execução.
5. **Verificação obrigatória e programática pós-execução** — nenhuma das três skills (docx/xlsx/pptx) considera uma tarefa concluída sem rodar um script de validação; nenhuma confia apenas no "código de saída limpo" como prova de sucesso.
6. **Fail-safe por padrão, não fail-open** — exemplo central: `recalc.py` recusa recalcular quando isso destruiria links externos, exigindo `--force` explícito para aceitar a perda.
7. **Transparência sobre o que NÃO foi verificado** — `validate.py` avisa explicitamente quando uma checagem (como tracked changes) não foi feita, em vez de ficar em silêncio.

Esses padrões, lidos em conjunto, descrevem uma arquitetura pensada para produzir entregáveis confiáveis mesmo quando o agente que os produz também é quem os audita — um ponto de atenção relevante para quem trabalha com GRC e controles de segregação de funções aplicados a agentes de IA.
