---
name: joey-gate-tecnico
description: Gate técnico de material de uma marca a partir de um envio em briefings. Use em toda subtarefa de gate de material atribuída pela Anna. Diz o que ler no Directus, como medir cada arquivo com as ferramentas do JOEY MCP Server, o que gravar sem aprovação, como decidir cada asset, como registrar o seu passo na jornada e como passar o bastão com abrir_validacao.
---

# Gate técnico de material

As regras duras estão nas suas instruções. Aqui está o **como**. Se esta skill e as instruções divergirem, as instruções valem e você reporta a divergência à Anna.

Esta skill pertence à Joey do Multica. Ela prepara e entrega o gate humano; não executa a conversa da Joey do n8n. O contrato de `abrir_validacao` é o contrato de entrada dessa entrega. Não o confunda com o objeto de quatro campos que a Joey do n8n devolve durante a conversa.

## Ferramentas

| Ferramenta | Para quê |
|---|---|
| MCP do Directus: `schema`, `items`, `files` | ler e atualizar registros; `schema` antes de gravar, nunca adivinhe campo |
| `autoteste_inspecao` (JOEY MCP Server) | uma vez por marca, antes do primeiro número medido |
| `inspecionar_arquivo` (JOEY MCP Server) | medir **todo** arquivo: veredicto de transparência, formato real, dimensões |
| `medir_contraste` (JOEY MCP Server) | contraste WCAG 7:1 de hex **já confirmado** em `brands.colors` |
| `abrir_validacao` (JOEY MCP Server) | passar o bastão para a Joey do n8n — uma vez por issue, no fim |

Nunca use a ferramenta `assets` do MCP do Directus: ela decide por tipo cadastrado, não por conteúdo.

Nunca use `trigger-flow`. Permissão negada (403) é informação: registre e reporte à Anna.

## 1. Leitura

Use o briefing e a marca que a issue indicar.

1. `briefings` pelo código da issue: `id`, `code`, `type`, `company`, `received_at`, `structured`.
2. `companies` da empresa do envio: `id`, `code`, `trade_name`, `document` (CNPJ).
3. `brand_assets` com `briefing` = esse envio: `id`, `brand`, `kind`, `status`, `source_kind`, `file`,
   `mime_type`, `width`, `height`, `has_transparency`, `usable`, `usage_rights`, `confirmed_by`.
4. `assets` com `briefing` = esse envio.
5. `brands` da marca indicada: **`id`**, **`code`**, `colors`, `fonts`, `tone`, `kit_status`,
   `confirmed_by`, `notes`, `logo_safe_area`, `logo_min_width_px`, `has_negative_version`,
   `colors_by_owner`.
   **O `id` é obrigatório nesta leitura**: é ele que vai no `brand_id` da `abrir_validacao`, no passo 6.4.
   Sem ele você não consegue passar o bastão.
6. Itens já confirmados da marca: `brand_assets` com `brand` = a marca e `confirmed_by` diferente de
   `sem_confirmacao`.
7. `fonts` com `status = approved` e `suits_channel` contendo `tv_indoor`.
8. `journeys` com `rise_project_id` = o da issue: `id`, `code`, `phase`, `status`. Sem
   `rise_project_id` na issue ou sem jornada encontrada: não bloqueie; registre a ausência no relatório
   para a Anna e siga.

Bloqueie se: o envio não existir, os arquivos do envio apontarem para outra marca, ou a marca da issue não for da empresa do envio.

## 2. Medição

1. Rode `autoteste_inspecao`. `integro = false`: nenhuma medição vale; bloqueie.
2. Rode `inspecionar_arquivo` com o `file` de cada `brand_asset` e de cada `asset` do envio.
3. Grave a medição **só em item com `confirmed_by = sem_confirmacao`**:
   - `width`, `height` e `mime_type` do formato real medido (não o cadastrado);
   - `has_transparency = true` somente quando `usavel_sobre_fundo_colorido = true` (veredicto
     `transparente`). Qualquer outro veredicto grava `false`;
   - o veredicto literal, `toca_a_borda`, `moldura_chapada`, `action_version` e `engine_version` vão para
     o bloco em `brands.notes`.
4. Item já confirmado cuja medição diverge do registrado: **não grave**. Vira reconfirmação.

## 3. Cor e fonte

**Cor é gate manual, 100%.** Você não extrai nem propõe hex.

- O que o cliente declarou está em `brands.colors_by_owner` (texto literal em `declared_as`), gravado pela
  ingestão. Leve exatamente isso para a lista de itens. Se o campo estiver vazio, use
  `briefings.structured.cores_declaradas` e registre a falha da ingestão no relatório.
- A cor proibida declarada está em `briefings.structured.cor_proibida_declarada`. Leve junto.
- Papel e hex de cada cor são definidos pelo Marcelo na conversa com a Joey do n8n, depois que você passa
  o bastão. Para uma cor nova, o fluxo do n8n gera a escala, os fundos, mede o contraste e grava após
  receber a decisão estruturada. A Joey do n8n não mede contraste por conta própria.
- Se a marca já tiver cores **confirmadas** em `brands.colors` (reenvio), rode `medir_contraste` sobre
  elas e registre o resultado em `notes`. `blocked_no_solution` vira item para o Marcelo.
- Fundo para texto não é hex escolhido caso a caso: é degrau da escala 100–900 da marca. Não proponha hex
  de fundo. Enquanto a escala não estiver disponível entre as suas ferramentas, registre a necessidade de
  fundo alternativo como lacuna no relatório para a Anna.
- Fonte: compare a declarada com o catálogo. Fora do catálogo: proponha a substituta mais próxima, com o
  motivo.

Prepare um item separado para cada papel de cor que dependa de decisão. Se as quatro cores dependerem
de aprovação, são quatro itens, como definido no passo 5. O tópico identifica o papel em discussão;
não aprova a cor nem autoriza você a escolher ou sugerir um hex.

## 4. Decisão por asset

Classifique cada arquivo em `APPROVED`, `REJECTED`, `BLOCKED` ou `NEEDS_HUMAN_GATE`, pela tabela das suas instruções, e grave `brand_assets.status` pelo mapeamento de lá. Casos frequentes:

- `arquivo_sem_conteudo` ou arquivo ilegível: `REJECTED`.
- `kind` declarado diferente do conteúdo medido (ex.: *manual* que é uma imagem 1200×630):
  `NEEDS_HUMAN_GATE`, porque classificação é julgamento.
- Material que não é logo próprio, sem `usage_rights`: `BLOCKED`.
- Aprovação técnica sua, sem julgamento envolvido: `confirmed_by = joey_multica`.

## 5. Itens para o gate humano

Monte a lista numerada. Um item por decisão de julgamento: **uma cor por papel**, fonte substituta,
classificação, recorte, direito de uso ou legenda. Não agrupe decisões que precisem de respostas ou
registros independentes. Cada item traz:

- o que o cliente informou;
- o que você encontrou, com o id do arquivo e a página, quando houver;
- a sua proposta (exceto cor: em cor não há proposta de hex);
- a pergunta, respondível sozinha.

Reconfirmação é item próprio, com o tópico começando por *Reconfirmação —*, e nunca é fundida com um item novo. Aviso que não bloqueia entra como aviso, sem pergunta.

### Cores: uma entrada por papel

Cada item de cor da aprovação do n8n só comporta um par `cor_papel`/`valor_hex`. Por isso:

- Se a paleta inteira depender de decisão, crie **quatro entradas distintas**: `Cor primária`,
  `Cor secundária`, `Cor terciária` e `Cor acentuada`.
- Se apenas parte da paleta depender de decisão, inclua somente os papéis correspondentes. Não reabra
  aprovação de cor confirmada que não mudou.
- Uma alteração de cor confirmada vira um item próprio: `Reconfirmação — Cor primária`, por exemplo.
  Leve o valor vigente, a procedência da aprovação e a mudança a decidir, sem propor hex novo.
- Cada entrada tem ID próprio e `tipo = cor`. Não use um único item `Cores`, `Cores da marca` ou
  `Paleta` para capturar várias cores.
- Leve a declaração literal do cliente e a cor proibida para os itens a que se aplicam. Se a declaração
  não identificar papéis, não invente essa associação. Marcelo define a decisão na conversa.
- Não crie novos campos na ferramenta para explicar os papéis. Use os campos documentados de
  `abrir_validacao`; os nomes `cor_papel` e `valor_hex` acima explicam a capacidade da saída posterior
  do n8n e não autorizam incluí-los no payload de abertura.

Quando as quatro cores forem os primeiros itens pendentes, a organização é:

| `id` na lista | `topico` | `tipo` |
|---|---|---|
| 1 | Cor primária | cor |
| 2 | Cor secundária | cor |
| 3 | Cor terciária | cor |
| 4 | Cor acentuada | cor |

Os números exemplificam a ordem dessa lista. Use a numeração real da lista completa, com IDs únicos.
O comentário na issue e os itens enviados em `abrir_validacao` devem representar a mesma lista.
Depois de entregue, não renumere IDs, não reescreva tópicos nem acrescente itens por uma nova chamada
para a mesma aprovação. O n8n recebe uma lista fixa, não um rascunho que a agente possa reorganizar.

Uma lacuna técnica de contraste não é uma proposta de nova cor. Se `blocked_no_solution` exigir uma
decisão de Marcelo, formule um item próprio `tipo = informacao`, identificando a cor afetada e a
decisão possível, sem pedir um hex substituto nem autorizar ajuste manual de degrau. Se for apenas
ausência da ferramenta de fundo alternativo, registre a lacuna no comentário de relatório da issue
para Anna, como no passo 3; não crie uma pergunta que Marcelo não tenha como resolver.

Cada item vira uma entrada do campo `itens` da `abrir_validacao`:

| Assunto do item | `tipo` |
|---|---|
| decisão de papel e hex de uma única cor, inclusive reconfirmação | `cor` |
| logo, classificação de arquivo, recorte derivado | `logo` |
| fonte, direito de uso, legenda, decisão sobre impedimento técnico de contraste, qualquer outro | `informacao` |

`id` é o número do item na lista; `topico` é o nome curto do item.

Antes de publicar a lista, confira: cada decisão de cor ocupa uma entrada; cada ID é único; cada
pergunta pode ser respondida sozinha; não há cor confirmada sem mudança reaberta por rotina. Não
preencha a decisão por Marcelo e não invente propriedades fora do contrato de `abrir_validacao`.

## 6. Registro e passagem de bastão

Nesta ordem:

1. **`brands.notes`** — anexe o bloco `[JOEY ...]` com o que você gravou sem aprovação (medições,
   aprovações técnicas `joey_multica`), no formato das suas instruções.
2. **Lista na issue** — publique a lista completa como comentário, com o título
   ***Itens para o gate humano — <marca>***. É o registro legível do que foi entregue ao Marcelo.
3. **Evento na jornada** — crie em `journey_events`: `journey` (id lido no passo 1.8), `type = gate`,
   `actor = joey_multica`, `multica_issue_id` (a sua subtarefa, nunca a pai) e `summary` (a sua passada,
   pelo roteiro das suas instruções). Sem jornada: pule e registre no relatório.
4. **`abrir_validacao`** — uma única chamada, com:

   Antes da chamada, confira a mesma lista completa publicada na issue: IDs, tópicos, tipos e uma
   entrada por papel de cor pendente. Não envie um item agregado para a Joey do n8n desmembrar depois.

   | Campo | O que é | Obrigatório |
   |---|---|---|
   | `multica_issue_id` | UUID da **sua** subtarefa. Nunca o da issue pai | sim |
   | `brand_id` | UUID da marca no Directus — o `id` que você leu no passo 1.5 | **sim** |
   | `company_id` | UUID da empresa no Directus | sim (ou `company_code`) |
   | `company_code` | código da empresa, ex. `CLI-000003` | sim (ou `company_id`) |
   | `trade_name` | nome fantasia da marca que será validada | sim |
   | `documento` | `companies.document`, só dígitos | sim |
   | `itens` | a lista do passo 5 | sim |
   | `issue_ref` | identificador legível, ex. `CLIPOOH-42` | não bloqueia |
   | `rise_project_id` | chave da jornada, quando a issue trouxer | não bloqueia |
   | `objetivo` | uma frase sobre o que o Marcelo precisa decidir | não bloqueia |

   **O `brand_id` é obrigatório e não tem substituto.** Ele identifica a marca cujo `brands.notes`
   participa da conferência de recibo no n8n. Sem esse ID, a abertura é recusada. Use o UUID lido no
   passo 1.5; não use código da marca, ID da empresa ou ID da issue em seu lugar. Não escreva recibos
   `[JOEY-N8N ...]` no Multica para antecipar ou suprir a aprovação futura: seu registro é `[JOEY ...]`,
   das medições e aprovações técnicas que você realmente executou.

   Você **não** informa o agente: a ferramenta já vai cravada como `joey`. E **não** informa contato,
   nome nem telefone: no perfil da casa o interlocutor é fixo.

5. **Código de resposta** — aja pela tabela das suas instruções. Com 200, comente na issue que a
   conversa foi aberta (com o `conversation_id`) e encerre. **Não mude o status da subtarefa**: quem
   fecha é o fluxo do n8n.

   Dois detalhes do que você pode receber:

   - **400 com `brand_id` na lista `faltando`** significa que você não mandou o id da marca. Corrija com
     o que leu no passo 1.5 e chame **uma** vez a mais.
   - **409** tem dois motivos, e eles pedem reações diferentes: `pedido_ja_aberto` é **esta mesma issue**
     que já abriu conversa, e aí nunca se chama de novo; `ocupado` é **outra marca** em aprovação neste
     momento — o corpo traz `aprovacao_ativa_marca` —, e aí você não insiste, registra na issue, grava um
     evento de bloqueio na jornada e bloqueia. O seu escopo de ocupação é global: uma aprovação da Joey
     por vez, em toda a casa.

A única situação sem `abrir_validacao` é a descrita nas suas instruções: reenvio de marca já confirmada em que nada mudou, conferido item por item. Nesse caso, pule os passos 2 e 4, registre no evento da jornada que a marca já estava completa e informe a Anna.

## 7. Relatório para a Anna

Ao fim, comente na issue o que o modelo de dados tem de errado ou de ambíguo. Exemplos: marcas duplicadas na empresa, campo que você precisou forçar, divergência entre o schema e as suas instruções, ausência de `rise_project_id` ou de jornada, lacuna de ferramenta.
