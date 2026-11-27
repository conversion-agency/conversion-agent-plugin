---
name: puxar-dados-qw
description: >-
  Puxa do BigQuery os dados de um QuickWin específico por cliente + termo ou
  URL, ou um pacote mensal por cliente + mês de competência. Use quando o
  analista pedir dados, detalhes ou a tabela de QuickWins de um cliente.
---

# /conversion-agent:puxar-dados-qw

Consulta somente leitura. Mostre os dados no chat e não salve nenhum arquivo.

## 1. Identificar o cliente e o modo

O pedido precisa conter um cliente e uma destas seleções:

- QuickWin específico: `term` ou `url`, usando `mode: specific`;
- pacote mensal: `competence_month`, usando `mode: package`.

Aceite o mês como `YYYY-MM`, `MM/YYYY` ou abreviação portuguesa como `nov/26`.
Não aceite termo e URL simultaneamente. URL precisa ser HTTP(S).

Chame `get_active_project` primeiro. Se o projeto ativo corresponder ao cliente
pedido, use seus `ws_slug` e `proj_slug`. Caso contrário, use
`list_workspaces_projects` para localizar o cliente em um projeto acessível ao
usuário. Se houver mais de uma correspondência, peça ao analista para escolher;
se não houver nenhuma, informe que não há projeto acessível para o cliente. Não
aceite nem peça `sourceClientId` ou qualquer ID do BigQuery.

Se o projeto escolhido ainda não estiver materializado, use
`searchhub_materialize_project`. Não é necessário trocar o projeto ativo apenas
para executar esta consulta.

## 2. Consultar

Chame uma única vez a MCP tool `pull_quickwins`.

Exemplo para QuickWin por termo:

```json
{
  "ws_slug": "<workspace>",
  "proj_slug": "<projeto-do-cliente>",
  "mode": "specific",
  "term": "<termo>"
}
```

Exemplo para QuickWin por URL:

```json
{
  "ws_slug": "<workspace>",
  "proj_slug": "<projeto-do-cliente>",
  "mode": "specific",
  "url": "<url-http-ou-https>"
}
```

Exemplo para pacote mensal:

```json
{
  "ws_slug": "<workspace>",
  "proj_slug": "<projeto-do-cliente>",
  "mode": "package",
  "competence_month": "<mês-de-competência>"
}
```

Não consulte o BigQuery diretamente. A tool deriva o cliente do projeto
autorizado e mantém a credencial no backend.

Em `not_authenticated` ou `session_expired`, execute `auth_login_start` e
`auth_login_poll`, depois repita uma vez. Em `not_in_hub` ou
`project_not_in_hub`, materialize o projeto escolhido e repita uma vez.

Esta skill só funciona nos modos `searchhub` e `parallel`. Se `pull_quickwins`
não estiver anunciada no modo `legacy`, informe que a consulta está
indisponível nesse modo. Não improvise outra fonte.

## 3. Apresentar a tabela

Mostre o modo, os filtros normalizados, `queried_at`,
`source_last_modified_at`, total e avisos. Quando a atualização da fonte for
desconhecida, deixe isso explícito.

Renderize `rows` em uma tabela com estas colunas, nesta ordem:

| Cliente | Squad | Competência | Tipo de conteúdo | Tipo de otimização | Termo | URL | Status | Implementação | Tema proposto | Objetivo da otimização | Densidade | Prompts (QW) | Tipo de produção |
|---|---|---|---|---|---|---|---|---|---|---|---:|---|---|

Use `-` para valor nulo. Em `prompts`, preserve todos os itens e separe-os por
quebra de linha dentro da célula. Não inclua a coluna “Search Hub”.

Se `rows: []`, diga “Nenhum resultado encontrado para os filtros informados” e
mostre total zero. Isso é diferente de falha. Se `truncated` for `true`, avise
que foram mostrados no máximo 200 registros de `total`.

Qualquer falha da tool deve ser apresentada como erro de consulta; falha não é
tabela vazia e nunca autoriza afirmar que não existem QuickWins.

Não salve nenhum arquivo, snapshot ou entregável ao final. A resposta desta
skill existe somente no chat.
