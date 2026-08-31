# SKILLs para LLMs da META AI

[English](README.md) | [Português](README.pt.md)

### O que são, como funcionam no Meta AI e como implementar geração de arquivos

---

## 1. O que é uma SKILL para LLMs

Uma **SKILL** em uma LLM é um pacote de capacidade plugável que transforma o modelo de um gerador de texto para um agente que age.

O modelo base, os pesos, sabe muito, mas por padrão ele só sabe *falar*. Ele não sabe como criar um slide, como pesquisar seus emails, como gerar uma imagem transparente, como agendar uma tarefa. A **SKILL** é a camada de **harness** — a parte dura, de engenharia, em torno do modelo — que ensina a ele COMO agir.

**Definição curta:**

> **Modelo = cérebro. SKILL = treinamento profissional + caixa de ferramentas + manual de procedimento.**

Sem skills, toda LLM é generalista. Com skills, ela vira especialista sob demanda.

### 1.1 Anatomia de uma SKILL

Tecnicamente, uma SKILL não é um prompt. É um artefato de software com 4 partes:

1.  **Manifesto (`manifest.yaml`):** Nome, descrição, quando deve ser ativada. É o que o orquestrador lê para decidir "essa pergunta precisa da skill X?". Ex: `shopping` -> "Use quando o usuário quer encontrar, comprar, comparar produtos..."

2.  **Instruções especializadas (`SKILL.md`):** Um system prompt de alta prioridade que é injetado SÓ quando a skill é carregada. Ele sobrescreve o comportamento genérico. Ele diz as regras, o fluxo de trabalho, os erros a evitar, o formato de saída.

3.  **Ferramentas autorizadas:** Quais funções ela pode chamar. Uma skill de `gmail-search` pode chamar busca no Gmail. Uma skill de `slides` pode chamar um gerador de apresentação. Uma skill de `transparent-background-image` pode chamar `image_gen` com parâmetros específicos para PNG RGBA.

4.  **Schemas e exemplos (`templates/` e `examples/`):** Como interpretar a intenção e como formatar a resposta final.

---

## 2. Conceitos Básicos ao Redor

### a) Harness vs. Weights
Os pesos são "hard" porque são caros de treinar e imutáveis em produção. O harness é "hard" porque é código de produção, testado, com permissões, que estende o que o modelo pode fazer sem precisar re-treinar. Skill é a forma mais segura de escalar capacidade.

Fórmula consagrada na indústria:

```
Agent = Model + Harness
```

### b) Roteamento e Ativação
O modelo não carrega todas as skills de uma vez — isso estouraria a janela de contexto e criaria conflito. Existe um roteador que lê sua pergunta e decide: `preciso carregar local? shopping? deep-research-report? nenhuma?`. Só depois disso as instruções da skill entram.

Fluxo no Meta AI:
1. Roteamento: Lê a descrição de todas as skills
2. Load: Chama `skills.load_skill({"skill_name": "slides"})`
3. Execução: O modelo é obrigado a seguir o protocolo da skill

### c) Skill vs. Tool vs. Plugin
*   **Tool:** é uma função atômica. Ex: `web_search(query)`.
*   **Plugin (antigo):** era basicamente expor uma tool para o modelo.
*   **Skill:** é orquestração de alto nível. Ela decide QUAL sequência de tools chamar, COMO pensar, COMO validar, e COMO apresentar. Uma skill pode usar 5 tools diferentes em loop.

Exemplo: a skill `deep-research-report` usa `browser.search` + `browser.open` + síntese + citações + formatação de relatório.

### d) MCP - Model Context Protocol
É o padrão atual que muitas skills usam por baixo. Ele padroniza como o modelo descobre que uma skill existe, que parâmetros ela aceita e que resultado ela devolve. É o "USB-C" das skills. Criado pela Anthropic em 25 de novembro de 2024 e hoje é open standard.

### e) Estado e Memória
Skills boas são stateless por padrão, mas podem ler contexto: seu perfil, suas conversas passadas, arquivos que você subiu. A skill `slides` que cria um deck sobre seu histórico de vendas precisa disso.

---

## 3. Como é no Meta AI - Estrutura de Pastas

### 3.1 Esqueleto padrão

```
/skills/
└── nome-da-skill/
    ├── manifest.yaml          # Quem sou eu, quando ativar, que tools posso usar
    ├── SKILL.md               # A instrução principal - 90% da inteligência está aqui
    ├── handlers/              # Código Python/TS que a skill pode chamar
    │   ├── main.py
    │   └── validators.py
    ├── templates/             # Templates base para os arquivos
    │   ├── base.html
    │   ├── base.docx
    │   └── style.css
    └── examples/              # Few-shots para o modelo não alucinar
        └── example_1.md
```

Exemplo de `manifest.yaml`:

```yaml
name: xlsx-report
description: Use when user asks to create, build, generate, edit spreadsheet, excel, xlsx, csv, planilha
version: 2.1
author: meta
allowed_tools:
  - container.python_execution
  - container.create
  - browser.search
requires_code_execution: true
output_mimetypes:
  - application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
```

### 3.2 As 8 Skills de Geração de Arquivos Implementadas

Esta conversa gerou uma estrutura real em `/mnt/data/meta-ai-skills-showcase/` com:

| Tipo de arquivo | Skill | Tool que a skill chama | O que o harness faz |
| --- | --- | --- | --- |
| **html** | `html-artifact` | `create_web_artifact_agent` | Sobe um sub-agente React que faz build e devolve `preview.html` |
| **docx** | `docx-generator` | `python_execution` + `python-docx` | Executa python que escreve binário .docx |
| **pdf** | `pdf-generator` | `python_execution` + `reportlab` | Canvas PDF nativo |
| **png** | `png-transparent` | `image_gen` | Chama gerador com `RGBA + transparent background` forçado |
| **md** | `md-report` | `container.create` | Escrita direta de markdown |
| **xlsx** | `xlsx-report` | `python_execution` + `pandas` | `df.to_excel()` |
| **py** | `py-script` | `container.create` | Cria arquivo e testa com `python_execution` |
| **bash** | `bash-task` | `container.create` + `chmod +x` | Cria `.sh` e valida comandos permitidos |

#### Exemplo de handler - docx
```python
from docx import Document
doc = Document()
doc.add_heading('Relatório Gerado por Skill', 1)
doc.add_paragraph('Este DOCX foi gerado pela skill docx-generator.')
doc.save('/mnt/data/output.docx')
```

#### Exemplo de handler - xlsx
```python
import pandas as pd
df = pd.DataFrame([{'Produto':'A','Vendas':100},{'Produto':'B','Vendas':200}])
df.to_excel('/mnt/data/output.xlsx', index=False)
```

#### Exemplo de handler - pdf
```python
from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas
c = canvas.Canvas('/mnt/data/output.pdf', pagesize=A4)
c.drawString(100, 750, 'PDF gerado pela skill pdf-generator - Meta AI Harness')
c.save()
```

---

## 4. Por que Skills são tão Importantes?

1.  **Confiabilidade:** Sem skill, a LLM improvisa. Com skill, segue procedimento validado.
2.  **Segurança e Permissão:** A skill delimita o que pode ser feito via `allowed_tools` e `allowed_commands`.
3.  **Composabilidade:** Você pode encadear skills. "Pesquise concorrentes (rival-watch-2) e crie pitch deck (slides)".
4.  **Evolução sem re-treino:** Quer que a IA aprenda a fazer reservas? Cria a skill `opentable` e pluga, sem re-treinar o modelo de 400B.

> Se a LLM é um ator muito inteligente, a SKILL é o roteiro, o figurino e o cenário que permitem que ele realmente desempenhe um papel específico.

---

## 5. Bibliografia

### 1. O que é uma SKILL em LLM
- **Anthropic - Documentação oficial de Skills:** https://docs.anthropic.com/en/docs/build-with-claude/skills
- **Anthropic lança Skills - The Decoder:** https://the-decoder.com/anthropic-launches-skills-so-claude-can-automatically-pick-prompts-for-specialized-tasks/
- **Anthropic Skills - The Landscape:** https://dev.to/dbolotov/anthropic-skills-the-landscape-for-new-models-and-architectures-2ld3
- **Anthropic torna Skills um padrão aberto - SiliconANGLE:** https://siliconangle.com/2025/12/18/anthropic-makes-agent-skills-open-standard/

### 2. Conceito de Harness - Agent = Model + Harness
- **Agent Harness - Wikipedia:** https://en.wikipedia.org/wiki/Agent_harness
- **AI Agent Harnesses: The Infrastructure Behind Autonomy - TechTarget:** https://www.techtarget.com/ai/tip/AI-agent-harnesses-The-infrastructure-behind-autonomy
- **Agent = Model + Harness - Towards AI:** https://pub.towardsai.net/agent-model-harness-what-a-coding-agent-harness-actually-is-3149945c26b5
- **What is an Agent Harness? - ODSC:** https://odsc.medium.com/what-is-an-agent-harness-the-architecture-behind-reliable-agentic-ai-76f4c1f243fb

### 3. Model Context Protocol (MCP)
- **Model Context Protocol - Site Oficial:** https://modelcontextprotocol.io/
- **Model Context Protocol - Wikipedia:** https://en.wikipedia.org/wiki/Model_Context_Protocol
- **Documentação e Spec do MCP - GitHub:** https://github.com/barefootford/anthropic-mcp-docs

### 4. Function Calling / Tool Use
- **Tool Calling - OpenAI Platform Docs:** https://platform.openai.com/docs/guides/function-calling
- **Tool Calling - vLLM Docs:** https://docs.vllm.ai/en/v0.7.1/features/tool_calling.html
- **Claude Skill Registry - llm-function-calling:** https://github.com/majiayu000/claude-skill-registry/blob/HEAD/skills/other/other/llm-function-calling/SKILL.md

### 5. Geração de arquivos
- **python-docx - Docs:** https://python-docx.readthedocs.io/
- **ReportLab - Docs:** https://www.reportlab.com/docs/reportlab-userguide.pdf
- **pandas to_excel:** https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.to_excel.html

---

## 6. Como usar este repositório

O diretório `/mnt/data/meta-ai-skills-showcase/` contém as 8 skills prontas:

```
meta-ai-skills-showcase/
├── html-artifact/
│   ├── manifest.yaml
│   ├── SKILL.md
│   └── handlers/main.py
├── docx-generator/
├── pdf-generator/
├── png-transparent/
├── md-report/
├── xlsx-report/
├── py-script/
└── bash-task/
```

Para testar, peça: "gere uma planilha xlsx", "gere um pdf", "gere um png transparente".

---
Created August 2026
