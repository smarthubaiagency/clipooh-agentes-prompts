--- name: joey-registro-directus ## description: Ler e gravar no Directus com segurança: filtros úteis, listas JSON de colors e fonts, bloco de procedência em notes com o id da mensagem, confirmed_by e confirmed_at, e registro de arquivo enviado pelo WhatsApp. Use ao consultar a marca ou registrar qualquer medição ou aprovação. # Registro no Directus

Mecânica de leitura e gravação. As regras do que pode ou não ser gravado estão nas suas instruções; aqui está o **como**.

## 1. Ferramentas

| Ferramenta | Para quê |
|---|---|
| `schema` | Conferir campos antes de ler ou gravar. Nunca adivinhe nome de campo |
| `items` | Ler e atualizar `brands`, `brand_assets`, `assets`. Ler `briefings`, `companies`, `fonts`. Criar em `brand_assets` ou `assets` **só** para arquivo que o Marcelo mandou e pediu para salvar |
| `files` | Metadados do arquivo e envio de arquivo que o Marcelo mandou |

**Nunca use `trigger-flow`.** Um flow pode rodar com permissões maiores que as suas.

Se uma chamada voltar **403**, **não procure outro caminho**: diga ao Marcelo e registre no relatório. Permissão negada é informação, não obstáculo.

## 2. Filtros que você mais usa

- Envio mais recente de uma empresa: `briefings` filtrado por `company`, ordenado
  por `-received_at`, `limit 1`.
- Arquivos de um envio: `brand_assets` e `assets` filtrados por `briefing`.
- Catálogo de fontes: `fonts` com `status = approved` e `suits_channel` contendo
  `tv_indoor`.
- Cores declaradas pelo cliente: `brands.colors_by_owner` (texto literal em
  `declared_as`) e `briefings.structured.cor_proibida_declarada`.

## 3. Listas JSON — colors, fonts e colors_by_owner

São listas **JSON** em `brands`, não tabelas. Para mudar um item:

1. leia a lista atual **imediatamente antes** de gravar;
2. altere só aquele item;
3. grave a lista inteira.

Nunca grave a partir de uma leitura antiga. A Joey do Multica pode ter gravado no intervalo — gravar por cima de uma leitura velha apaga o que ela escreveu.

Campos de cada cor: `role`, `hex`, `source`, `bg_light_hex`, `text_on_light_hex`, `contrast_light`, `bg_dark_hex`, `text_on_dark_hex`, `contrast_dark`, `confirmed_by`, `confirmed_at`. Cor definida pelo Marcelo grava `source = confirmed`. Os seis campos de fundo e contraste saem da ação `scale` — ver a skill `joey-cor-e-escala`.

`brands.color_scales` é outra lista **JSON**, com a escala **100**–**900** de cada cor. Segue a mesma regra: leia, altere só a linha daquela cor, grave a lista inteira. Campos de cada fonte: `role`, `family`, `source`, `confirmed_by`, `confirmed_at`.

Os fundos recebem degraus da escala **100**–**900** da marca, nunca um hex avulso. O `hex` da cor da marca não muda em hipótese alguma.

## 4. notes — sempre somar, nunca substituir

Leia o valor atual de `notes` e grave `valor atual + novo bloco`. Nunca substitua. Valor antigo de qualquer campo vai para `notes` antes de ser sobrescrito.

Seu bloco abre com `[**JOEY**-**N8N** ...]`. O da Joey do Multica abre com `[**JOEY** ...]`. Cada item aprovado, recusado ou reconfirmado cita o **id da mensagem** do Marcelo (`msg ...`), que vem no contexto da rodada. Sem id verificado, não grave aprovação.

``` [**JOEY**-**N8N** **2026**-09-23 14:32] Aprovado por: marcelo @joey n8n colors.primary: declarado *azul* -> *#**1E5BC6*** (definido pelo Marcelo) · msg **4812** colors.primary: fundos da escala (**100** e **800**) · contraste 19,2:1 e 11,3:1 (ferramenta 1.3.0) Arquivo recebido pelo WhatsApp: [nome] -> brand_assets.[id] (logo_negativa) · msg **4823** Sem aprovação (medição): has_transparency medido (veredicto: transparente · 1.2.0) Pendente: logo_safe_area (padrão da fábrica não definido) ```

Medição também entra no registro. Sem aprovação não significa sem registro.

O veredicto completo da inspeção vive aqui, em `notes`: `brand_assets` só tem `has_transparency`, que é booleano, e não cabe os cinco valores do veredicto nem o `toca_a_borda`. Não invente nome de campo para guardar isso.

## 5. confirmed_by e confirmed_at

Toda aprovação grava quem aprovou e quando, no próprio registro: em cada cor, cada fonte, cada logo, cada foto e na marca.

| Valor | Significa |
|---|---|
| `sem_confirmacao` | nada foi aprovado ainda |
| `marcelo_joey_n8n` | Marcelo aprovou por você, no WhatsApp |
| `joey_multica` | aprovação técnica da Joey do Multica, sem julgamento |
| `marcelo_direto` | Marcelo aprovou direto no painel |
| `cliente_marcelo` | o cliente validou, repassado por Marcelo |

`marcelo_joey` e `joey_paperclip` são histórico da instância antiga do Paperclip. Foram removidos dos campos e não são procedência válida. Registro antigo que ainda os traga conta como confirmado, e muda só por reconfirmação.

**Você grava sempre `marcelo_joey_n8n`.** Todos os valores diferentes de `sem_confirmacao` contam como confirmação válida: o item **já está aprovado** e só muda por reconfirmação.

Medição nunca muda esse campo. Arquivo com `usable = true` e `confirmed_by = sem_confirmacao` é erro: reporte.

## 6. Reconfirmação — registro da pergunta e da resposta

``` [**JOEY**-**N8N** **2026**-09-23 10:02] **RECONFIRMACAO** Item: colors.primary Aprovado antes: #**1E5BC6** - marcelo @joey n8n - 10/09/**2026** Proposto: #**1A4FA8** - informado pelo Marcelo; escala regerada só desta cor · msg **4901** Pergunta: 23/09 09:58 - Resposta: *sim* 23/09 10:01 · msg **4903** Resultado: atualizado - confirmed_by marcelo_joey_n8n - confirmed_at 23/09 10:02 ```

Recusa usa o mesmo formato, com `Resultado: mantido - nao perguntar de novo por [valor proposto]`.

## 7. Arquivo enviado pelo WhatsApp

## Confirme com o Marcelo a marca, a coleção (`brand_assets` ou `assets`) e o papel.

## Envie o arquivo para `directus_files` e crie o registro ligado à marca, com `status = raw`, `confirmed_by = sem_confirmacao` e `source_kind` adequado. ## Meça com a ferramenta de inspeção e grave a medição. ## Só depois do *sim* dele grave `confirmed_by = marcelo_joey_n8n` e `confirmed_at`.

Se não conseguir enviar o arquivo, diga ao Marcelo. Não finja que salvou.

## 8. Nunca apague

Arquivo ruim recebe `usable = false` ou `archived = true`. Nenhum registro e nenhum arquivo é excluído. Fora o caso do arquivo que o Marcelo mandou, você não cria registro: proponha no relatório.

## 9. Confirmação da marca

Quando o critério das suas instruções fechar por inteiro, grave `kit_status = confirmed` com `brands.confirmed_by` e `brands.confirmed_at`, e anexe em `notes`:

``` [**ATENCAO**] Confirmacao interna ClipOOH. Cliente nao validou. ```
