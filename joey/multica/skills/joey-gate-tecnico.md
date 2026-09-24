--- name: joey-gate-tecnico ## description: Gate técnico de material de uma marca a partir de um envio em briefings. Use em toda subtarefa de gate de material atribuída pela Anna. Diz o que ler no Directus, como medir cada arquivo com as ferramentas do **JOEY** **MCP** Server, o que gravar sem aprovação, como decidir cada asset, como registrar o seu passo na jornada e como passar o bastão com abrir_validacao. # Gate técnico de material

As regras duras estão nas suas instruções. Aqui está o **como**. Se esta skill e as instruções divergirem, as instruções valem e você reporta a divergência à Anna.

## Ferramentas

| Ferramenta | Para quê |
|---|---|
| MCP do Directus: `schema`, `items`, `files` | ler e atualizar registros; `schema` antes de gravar, nunca adivinhe campo |
| `autoteste_inspecao` (JOEY MCP Server) | uma vez por marca, antes do primeiro número medido |
| `inspecionar_arquivo` (JOEY MCP Server) | medir **todo** arquivo: veredicto de transparência, formato real, dimensões |
| `medir_contraste` (JOEY MCP Server) | contraste WCAG 7:1 de hex **já confirmado** em `brands.colors` |
| `abrir_validacao` (JOEY MCP Server) | passar o bastão para a Joey do n8n — uma vez por issue, no fim |

Nunca use a ferramenta `assets` do **MCP** do Directus: ela decide por tipo cadastrado, não por conteúdo.

Nunca use `trigger-flow`. Permissão negada (**403**) é informação: registre e reporte à Anna.

## 1. Leitura

Use o briefing e a marca que a issue indicar.

1. `briefings` pelo código da issue: `id`, `code`, `type`, `company`, `received_at`, `structured`.
2. `companies` da empresa do envio: `id`, `code`, `trade_name`, `document` (**CNPJ**).
3. `brand_assets` com `briefing` = esse envio: `id`, `brand`, `kind`, `status`, `source_kind`, `file`,
   `mime_type`, `width`, `height`, `has_transparency`, `usable`, `usage_rights`, `confirmed_by`.
4. `assets` com `briefing` = esse envio.
5. `brands` da marca indicada: `colors`, `fonts`, `tone`, `kit_status`, `confirmed_by`, `notes`,
   `logo_safe_area`, `logo_min_width_px`, `has_negative_version`, `colors_by_owner`.
## Itens já confirmados da marca: `brand_assets` com `brand` = a marca e `confirmed_by` diferente de
   `sem_confirmacao`.
7. `fonts` com `status = approved` e `suits_channel` contendo `tv_indoor`.
8. `journeys` com `rise_project_id` = o da issue: `id`, `code`, `phase`, `status`. Sem
    `rise_project_id` na issue ou sem jornada encontrada: não bloqueie; registre a ausência no relatório
    para a Anna e siga.

Bloqueie se: o envio não existir, os arquivos do envio apontarem para outra marca, ou a marca da issue não for da empresa do envio.

## 2. Medição

## Rode `autoteste_inspecao`. `integro = false`: nenhuma medição vale; bloqueie.

## Rode `inspecionar_arquivo` com o `file` de cada `brand_asset` e de cada `asset` do envio.

## Grave a medição **só em item com `confirmed_by = sem_confirmacao`**:
    - `width`, `height` e `mime_type` do formato real medido (não o cadastrado);
    - `has_transparency = true` somente quando `usavel_sobre_fundo_colorido = true` (veredicto
    `transparente`). Qualquer outro veredicto grava `false`;
    - o veredicto literal, `toca_a_borda`, `moldura_chapada`, `action_version` e `engine_version` vão para
    o bloco em `brands.notes`.
## Item já confirmado cuja medição diverge do registrado: **não grave**. Vira reconfirmação.

## 3. Cor e fonte

**Cor é gate manual, **100**%.** Você não extrai nem propõe hex.

- O que o cliente declarou está em `brands.colors_by_owner` (texto literal em `declared_as`), gravado pela
    ingestão. Leve exatamente isso para a lista de itens. Se o campo estiver vazio, use
    `briefings.structured.cores_declaradas` e registre a falha da ingestão no relatório.
- A cor proibida declarada está em `briefings.structured.cor_proibida_declarada`. Leve junto.
- Papel e hex de cada cor são definidos pelo Marcelo na conversa com a Joey do n8n, depois que você passa
  o bastão. O contraste do hex novo é medido lá, não aqui.
- Se a marca já tiver cores **confirmadas** em `brands.colors` (reenvio), rode `medir_contraste` sobre
  elas e registre o resultado em `notes`. `blocked_no_solution` vira item para o Marcelo.
- Fundo para texto não é hex escolhido caso a caso: é degrau da escala **100**–**900** da marca. Não proponha hex
    de fundo. Enquanto a escala não estiver disponível entre as suas ferramentas, registre a necessidade de
    fundo alternativo como lacuna no relatório para a Anna.
- Fonte: compare a declarada com o catálogo. Fora do catálogo: proponha a substituta mais próxima, com o
  motivo.

## 4. Decisão por asset

Classifique cada arquivo em `**APPROVED**`, `**REJECTED**`, `**BLOCKED**` ou `NEEDS_HUMAN_GATE`, pela tabela das suas instruções, e grave `brand_assets.status` pelo mapeamento de lá. Casos frequentes:

- `arquivo_sem_conteudo` ou arquivo ilegível: `**REJECTED**`.
- `kind` declarado diferente do conteúdo medido (ex.: *manual* que é uma imagem **1200**×**630**):
  `NEEDS_HUMAN_GATE`, porque classificação é julgamento.
- Material que não é logo próprio, sem `usage_rights`: `**BLOCKED**`.
- Aprovação técnica sua, sem julgamento envolvido: `confirmed_by = joey_multica`.

## 5. Itens para o gate humano

Monte a lista numerada. Um item por decisão de julgamento: cores (com o que foi declarado e a cor proibida, para o Marcelo definir papel e hex), fonte substituta, classificação, recorte, direito de uso, legenda. Cada item traz:

- o que o cliente informou;
- o que você encontrou, com o id do arquivo e a página, quando houver;
- a sua proposta (exceto cor: em cor não há proposta de hex);
- a pergunta, respondível sozinha.

Reconfirmação é item próprio, com o tópico começando por *Reconfirmação —*, e nunca é fundida com um item novo. Aviso que não bloqueia entra como aviso, sem pergunta.

Cada item vira uma entrada do campo `itens` da `abrir_validacao`:

| Assunto do item | `tipo` |
|---|---|
| cores | `cor` |
| logo, classificação de arquivo, recorte derivado | `logo` |
| fonte, direito de uso, legenda, qualquer outro | `informacao` |

`id` é o número do item na lista; `topico` é o nome curto do item.

## 6. Registro e passagem de bastão

Nesta ordem:

1. **`brands.notes`** — anexe o bloco `[**JOEY** ...]` com o que você gravou sem aprovação (medições,
   aprovações técnicas `joey_multica`), no formato das suas instruções.
2. **Lista na issue** — publique a lista completa como comentário, com o título
   ***Itens para o gate humano — <marca>***. É o registro legível do que foi entregue ao Marcelo.
3. **Evento na jornada** — crie em `journey_events`: `journey` (id lido no passo 1.8), `type = gate`,
    `actor = joey_multica`, `multica_issue_id` (a sua subtarefa, nunca a pai) e `summary` (a sua passada,
    pelo roteiro das suas instruções). Sem jornada: pule e registre no relatório.
4. **`abrir_validacao`** — uma única chamada, com `multica_issue_id`, `issue_ref`, `rise_project_id`,
    `company_id`, `company_code`, `trade_name`, `documento` (`companies.document`, só dígitos),
    `objetivo` (uma frase) e `itens`.
5. **Código de resposta** — aja pela tabela das suas instruções. Com **200**, comente na issue que a
    conversa foi aberta (com o `conversation_id`) e encerre. **Não mude o status da subtarefa**: quem
    fecha é o fluxo do n8n.

A única situação sem `abrir_validacao` é a descrita nas suas instruções: reenvio de marca já confirmada em que nada mudou, conferido item por item. Nesse caso, pule os passos 2 e 4, registre no evento da jornada que a marca já estava completa e informe a Anna.

## 7. Relatório para a Anna

Ao fim, comente na issue o que o modelo de dados tem de errado ou de ambíguo. Exemplos: marcas duplicadas na empresa, campo que você precisou forçar, divergência entre o schema e as suas instruções, ausência de `rise_project_id` ou de jornada, lacuna de ferramenta.
