--- name: briefing-cliente-directus ## description: Procedimento da Amy para consultar o Directus na fase de briefing do cliente — encontrar a empresa, a fase, o contato técnico e o formulário certo, e acionar o contato com o cliente pela ferramenta `abrir_briefing`. Use sempre que receber subtarefa de briefing da Anna, antes de acionar o contato com o cliente, e quando for pedida uma conferência manual de envio em `briefings`. Use também sempre que precisar ler `companies`, `contacts`, `forms` ou `briefings`, mesmo que a pergunta pareça simples. # Briefing do cliente no Directus

Esta skill diz **como** consultar o Directus e acionar o contato em cada passo da sua etapa. O **que** fazer e os seus limites estão nas suas instruções; em caso de dúvida, elas prevalecem.

Seu acesso é **somente leitura** em `companies`, `contacts`, `forms` e `briefings`. Você nunca cria, altera nem apaga registro. Se uma consulta pedir escrita, você está no passo errado.

Todas as consultas usam a ferramenta `items` do **MCP** do Directus, com `action: *read*`. Peça sempre só os campos que vai usar (`fields`).

## Como a etapa funciona

Você, aqui no Multica, **prepara e dispara** o briefing. A conversa com o cliente pelo WhatsApp é conduzida pelo fluxo do n8n, não por você. Depois que o disparo for aceito, o próprio fluxo:

- confirma o envio do formulário no Directus;
- deixa uma nota nesta subtarefa com o resumo da conversa;
- muda o status da subtarefa (`done` quando o envio é confirmado, `blocked` quando a conversa é
  encerrada sem envio);
- avisa a Anna quando for preciso.

Por isso você **não** acompanha a conversa, **não** fecha a subtarefa por conta própria e **não** comenta *briefing recebido* depois de um disparo aceito.

---

## Passo 1 — A empresa e a fase

A subtarefa da Anna traz a empresa. Use o identificador que ela informar, nesta ordem de preferência: id do Directus, `code` (**CLI**-**NNNNNN**) ou `rise_client_id`.

```json
{
    *action*: *read*,
    *collection*: *companies*,
    *query*: {
    *fields*: [*id*, *code*, *trade_name*, *briefing_stage*, *status*],
    *filter*: { *code*: { *_eq*: ***CLI**-**000001*** } },
    *limit*: 1
    }
}
```

- **`briefing_stage`** diz qual formulário enviar: `design_system` ou `campanha`. É o valor que vai
  no campo `stage` do `abrir_briefing`.
- Se a fase que a Anna escreveu na subtarefa for diferente de `briefing_stage`, **não escolha**:
  bloqueie e pergunte à Anna.
- Se `briefing_stage` estiver vazio, bloqueie e avise a Anna.
- Nenhuma empresa encontrada: bloqueie. Não procure por nome.

## Passo 2 — O contato técnico

Leia **todos** os contatos da empresa e escolha no seu raciocínio, não no filtro. `types` é uma lista (**JSON**) e filtrar dentro dela pelo Directus não é confiável.

```json
{
    *action*: *read*,
    *collection*: *contacts*,
    *query*: {
    *fields*: [*id*, *name*, *whatsapp*, *email*, *types*, *is_primary*, *archived*],
    *filter*: { *company*: { *_eq*: *<id da empresa>* } },
    *limit*: 50
    }
}
```

Depois da leitura:
1. descarte os contatos com `archived = true`;
2. fique com os que têm `*technical*` em `types`.

| Resultado | O que fazer |
|---|---|
| exatamente um técnico, com `whatsapp` | é o destinatário. Siga |
| exatamente um técnico, sem `whatsapp` | bloqueie: não há como contatar |
| nenhum técnico | bloqueie. **Não** use o contato principal no lugar |
| mais de um técnico | bloqueie. **Não** escolha um |

## Passo 3 — O formulário da fase

```json
{
    *action*: *read*,
    *collection*: *forms*,
    *query*: {
    *fields*: [*id*, *nome*, *url*, *stage*, *agent*],
    *filter*: {
    *_and*: [
    { *stage*: { *_eq*: *design_system* } },
    { *agent*: { *_eq*: *amy* } }
    ]
    },
    *limit*: 5
    }
}
```

Troque `design_system` pela fase do passo 1.

| Resultado | O que fazer |
|---|---|
| um formulário com `url` preenchido (https) | é o link. Siga |
| `url` vazio | formulário não publicado. Bloqueie. Nunca prometa o link |
| nenhum formulário | bloqueie e avise a Anna |
| mais de um | bloqueie e pergunte à Anna qual vale |

## Passo 4 — Acionar o contato (`abrir_briefing`)

A ferramenta `abrir_briefing` dispara uma **mensagem real de WhatsApp** para uma pessoa. Chame só com empresa, contato técnico e link em mãos, conferidos nos passos 1 a 3, e no máximo uma vez por subtarefa (as únicas exceções estão na tabela de respostas).

### O que enviar

| Campo | Valor |
|---|---|
| `multica_issue_id` | ID (UUID) **desta** subtarefa. Nunca o da issue pai |
| `company_id` | `id` da empresa (passo 1) |
| `company_code` | `code` da empresa (passo 1) |
| `trade_name` | `trade_name` da empresa (passo 1) |
| `stage` | `briefing_stage` da empresa (passo 1): `design_system` ou `campanha` |
| `contact_id` | `id` do contato técnico (passo 2) |
| `contact_name` | `name` do contato técnico (passo 2) |
| `contact_whatsapp` | `whatsapp` do contato técnico (passo 2), só dígitos, com 55 e DDD |
| `form_id` | `id` do formulário (passo 3) |
| `form_url` | `url` do formulário (passo 3) |
| `rise_project_id` | RISE project ID, se a issue trouxer. Vazio no onboarding sem projeto |
| `objetivo` | Uma frase, se a issue deixar claro. Senão, vazio. Não invente |

### O que fazer com cada resposta

A ferramenta devolve `statusCode` e `body`.

| Código | Significa | O que fazer |
|---|---|---|
| **200** | Conversa aberta, primeira mensagem a caminho | Comente na subtarefa o formato *Briefing iniciado* abaixo e **encerre sua execução**. Não mude o status: o retorno chega sozinho |
| **400** | Pedido recusado. Nada foi enviado | Leia `faltando` e `invalidos` no `body`. Se der para corrigir com dados dos passos 1 a 3, corrija e chame **uma** vez mais. Se não der, bloqueie informando o que faltou |
| **409** | O contato já está em outro briefing. Nada foi enviado | **Não insista.** Bloqueie citando `multica_issue_id_em_andamento` e `conversation_id` do `body`. Quem resolve: Anna |
| **502** | Falha no Chatwoot. Nada foi enviado | Chame **uma** vez mais. Se repetir, bloqueie com a mensagem recebida. Quem resolve: Marcelo |
| qualquer outro | Resposta inesperada | Não repita. Bloqueie com o código e o `body` |

Formato do comentário no **200**:

``` 📨 Briefing iniciado

Empresa: <trade_name> (<code>) Fase: <design_system | campanha> Contato técnico: <name> Conversa: <conversation_id>

A conversa com o cliente segue pelo fluxo do WhatsApp. Esta subtarefa será atualizada automaticamente quando o envio do formulário for confirmado ou a conversa for encerrada. ```

---

## Conferência manual de envio (só quando pedida)

Normalmente a confirmação do envio é feita pelo fluxo do WhatsApp. Use esta consulta só quando a Anna ou o Marcelo pedirem uma conferência nesta subtarefa.

O corte de data é o momento em que a sua subtarefa foi criada no Multica. Um envio anterior a isso é de outro momento e não fecha esta etapa.

```json
{
    *action*: *read*,
    *collection*: *briefings*,
    *query*: {
    *fields*: [*id*, *code*, *type*, *source*, *received_at*, *submission_id*],
    *filter*: {
    *_and*: [
    { *company*: { *_eq*: *<id da empresa>* } },
    { *type*: { *_eq*: *design_system* } },
    { *received_at*: { *_gte*: *<data de criação da subtarefa, **ISO** **8601**>* } }
    ]
    },
    *sort*: [*-received_at*],
    *limit*: 5
    }
}
```

Responda no comentário o que a consulta retornou: o `code` e o `received_at` do registro mais recente, ou *nenhum registro* com o filtro usado. Não mude o status da subtarefa: quem decide é quem pediu.

O formulário chega ligado à empresa pelo **CNPJ**. Se o registro não aparecer, **não procure em outras empresas**. Registre que ele não apareceu e avise a Anna.

---

## Bloqueio

Quando bloquear, comente neste formato e avise a Anna:

``` ⛔ Bloqueado — <motivo em uma linha>

O que encontrei: <resultado exato da consulta ou resposta do abrir_briefing> O que falta: <o dado ou a decisão> Quem resolve: <Anna | Marcelo | comercial> ```

Nunca bloqueie sem dizer o que a consulta ou a ferramenta retornou. *Não encontrei* sem o filtro que você usou não ajuda ninguém a destravar.

---

## Erros que não se repetem

- Escolher o contato principal quando não há técnico.
- Mandar o formulário de campanha para cliente em `design_system`, ou o contrário.
- Chamar `abrir_briefing` antes de ter empresa, contato técnico e link conferidos.
- Chamar `abrir_briefing` de novo depois de um **200**, ou insistir depois de um **409**.
- Enviar o ID da issue pai em `multica_issue_id`.
- Fechar a subtarefa ou comentar *briefing recebido* por conta própria depois de um disparo aceito.
- Aceitar um briefing anterior à abertura da subtarefa.
- Procurar o envio em outra empresa quando ele não aparece na certa.
- Tentar gravar qualquer coisa no Directus.
