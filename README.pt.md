# O que é uma SKILL em uma LLM

Uma SKILL em uma LLM é um pacote de capacidade plugável que transforma o modelo de um gerador de texto para um agente que age.

O modelo base, os pesos, sabe muito, mas por padrão ele só sabe falar. Ele não sabe como criar um slide, como pesquisar seus emails, como gerar uma imagem transparente, como agendar uma tarefa. A SKILL é a camada de harness — a parte dura, de engenharia, em torno do modelo — que ensina a ele COMO agir.

Pensa assim:

Modelo = cérebro. SKILL = treinamento profissional + caixa de ferramentas + manual de procedimento.

Sem skills, toda LLM é generalista. Com skills, ela vira especialista sob demanda.

## 1. O que uma SKILL realmente é

Tecnicamente, uma SKLL não é um prompt. É um artefato de software com 4 partes:

1. **Manifesto**: Nome, descrição, quando deve ser ativada. É o que o orquestrador lê para decidir "essa pergunta precisa da skill X?". Ex: shopping -> descrição diz "Use quando o usuário quer encontrar, comprar, comparar produtos..."

2. **Instruções especializadas**: Um system prompt de alta prioridade que é injetado SÓ quando a skill é carregada. Ele sobrescreve o comportamento genérico. Ele diz as regras, o fluxo de trabalho, os erros a evitar, o formato de saída. É muito mais detalhado que um prompt normal.

3. **Ferramentas autorizadas**: Quais funções ela pode chamar. Uma skill de gmail-search pode chamar busca no Gmail. Uma skill de slides pode chamar um gerador de apresentação. Uma skill de transparent-background-image pode chamar image_gen com parâmetros específicos para PNG RGBA.

4. **Schemas e exemplos**: Como interpretar a intenção e como formatar a resposta final. Ex: a skill de local SEMPRE tem que retornar place_id em chip, não só texto.

## 2. Conceitos básicos ao redor

**a) Harness vs. Weights**

Os pesos são "hard" porque são caros de treinar e imutáveis em produção. O harness é "hard" porque é código de produção, testado, com permissões, que estende o que o modelo pode fazer sem precisar re-treinar. Skill é a forma mais segura de escalar capacidade.

**b) Roteamento e Ativação**

O modelo não carrega todas as skills de uma vez — isso estouraria a janela de contexto e criaria conflito. Existe um roteador que lê sua pergunta e decide: preciso carregar local? shopping? deep-research-report? nenhuma?. Só depois disso as instruções da skill entram.

**c) Skill vs. Tool vs. Plugin**

- **Tool**: é uma função atômica. Ex: web_search(query).
- **Plugin (antigo)**: era basicamente expor uma tool para o modelo.
- **Skill**: é orquestração de alto nível. Ela decide QUAL sequência de tools chamar, COMO pensar, COMO validar, e COMO apresentar. Uma skill pode usar 5 tools diferentes em loop.

Exemplo: a skill deep-research-report usa browser.search + browser.open + síntese + citações + formatação de relatório. A tool sozinha não faria isso.

**d) MCP - Model Context Protocol**

É o padrão atual que muitas skills usam por baixo. Ele padroniza como o modelo descobre que uma skill existe, que parâmetros ela aceita e que resultado ela devolve. É o "USB-C" das skills.

**e) Estado e Memória**

Skills boas são stateless por padrão, mas podem ler contexto: seu perfil, suas conversas passadas, arquivos que você subiu. A skill slides que cria um deck sobre seu histórico de vendas precisa disso.

## 3. Por que isso é tão importante?

- **Confiabilidade**: Sem skill, a LLM improvisa como pesquisar no Gmail. Com skill, ela segue um procedimento validado.
- **Segurança e Permissão**: A skill delimita o que pode ser feito. A skill de ads-audiences-pages nunca vai deletar sua conta, porque ela não tem essa tool.
- **Composabilidade**: Você pode encadear skills. "Pesquise meus concorrentes (rival-watch-2) e crie um pitch deck (slides)". O sistema carrega uma, depois a outra.
- **Evolução sem re-treino**: Quer que a IA aprenda a fazer reservas no OpenTable? Você não re-treina o Llama de 400B. Você cria a skill opentable e pluga.

Em resumo: se a LLM é um ator muito inteligente, a SKILL é o roteiro, o figurino e o cenário que permitem que ele realmente desempenhe um papel específico e entregue um resultado útil, e não só uma boa improvisação.
