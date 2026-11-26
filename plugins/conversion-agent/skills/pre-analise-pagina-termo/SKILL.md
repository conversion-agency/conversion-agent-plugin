---
name: pre-analise-pagina-termo
description: >-
  Verifica no histórico de QuickWins se uma keyword ou combinação de keyword e
  URL já foi trabalhada para o projeto ativo, antes de criar ou otimizar uma
  página. Use quando o analista pedir pré-análise, checagem de canibalização ou
  validação de termo e página.
---

# /conversion-agent:pre-analise-pagina-termo <keyword> [url]

Consulta somente leitura do histórico de QuickWins do cliente. Esta skill é
independente e não chama automaticamente `/conversion-agent:otimizar-pagina`.

## 1. Resolver o projeto e validar a entrada

1. Chame `get_active_project` antes de perguntar pelo destino. Se `active` for
   `null`, pare e peça ao analista para ativar um projeto com `/projeto`.
2. Use `ws_slug` e `proj_slug` do projeto ativo. Não aceite nem peça um ID de
   cliente do BigQuery.
3. A keyword é obrigatória e não pode ser vazia depois de aparar espaços.
4. A URL é opcional. Se vier preenchida, deve ser HTTP(S) e não pode ser uma
   string vazia. Sem URL, a consulta opera no modo `criar`; com URL, no modo
   `otimizar`.

Esta skill só funciona nos modos `searchhub` e `parallel`. Se
`pre_analyze_page_term` não estiver disponível no modo `legacy`, informe que a
pré-análise está indisponível nesse modo. Nunca substitua a ausência da tool por
um resultado `livre`.

## 2. Consultar o histórico

Chame uma única vez:

```json
{
  "ws_slug": "<workspace-ativo>",
  "proj_slug": "<projeto-ativo>",
  "keyword": "<keyword-informada>",
  "url": "<url-http-ou-https-ou-null>"
}
```

Use a MCP tool `pre_analyze_page_term`. Não consulte o BigQuery diretamente e
não exponha credenciais, JWT ou identificadores internos do cliente.

Em `not_authenticated` ou `session_expired`, execute `auth_login_start` e
`auth_login_poll`, depois repita a consulta. Em `not_in_hub` ou
`project_not_in_hub`, execute `searchhub_materialize_project` para o mesmo
workspace/projeto e repita. Para qualquer outra falha, apresente o erro como
`inconclusivo`; falha nunca equivale a ausência de histórico.

## 3. Apresentar o resultado

Mostre primeiro o veredito retornado, sem reinterpretá-lo:

- `conflito`: há ao menos um registro implementado;
- `atencao`: há registro ativo ou de status desconhecido, ainda não implementado;
- `livre`: a consulta foi bem-sucedida e não encontrou evidência impeditiva;
- `inconclusivo`: não foi possível validar; nunca descreva como `livre`.

Inclua modo, keyword e URL normalizadas, horário da consulta, atualização mais
recente conhecida da fonte, contagens, indicação de truncamento e todos os
avisos. Quando `source_last_modified_at` for `null`, diga que a atualização da
fonte é desconhecida, sem transformar uma consulta bem-sucedida em falha.

Para cada match, mostre uma tabela com, no mínimo: tipo do match, severidade,
keyword, URL, status e datas de criação, atualização e implementação. Não
oculte registros informativos. Se `truncated` for `true`, avise que a tabela
mostra no máximo 200 registros.

Depois de apresentar o resultado, pergunte: “Quer salvar esta pré-análise no
projeto?”. Não grave nada nessa primeira etapa. A própria invocação da skill,
mesmo que o pedido inicial mencione salvar, não conta como confirmação: o
analista precisa confirmar depois de ver o veredito e as evidências.

## 4. Salvar somente após confirmação explícita

Após a confirmação, preserve exatamente a resposta já apresentada; não rode
uma nova consulta. Calcule:

- `RUN_DATE`: data corrente em `YYYY-MM-DD`;
- `KEYWORD_SLUG`: keyword normalizada em kebab-case, limitada a 48 caracteres;
- `HASH8`: primeiros 8 caracteres do SHA-256 de
  `<normalized_keyword>\n<normalized_url-ou-string-vazia>` em UTF-8.

Em outras palavras, concatene `normalized_keyword` e `normalized_url` nessa
ordem e use os primeiros 8 caracteres do digest.

O caminho é determinístico:

```text
pesquisas/pre-analises/<RUN_DATE>-<KEYWORD_SLUG>-<HASH8>.md
```

Em termos do contrato público, o formato é
`pesquisas/pre-analises/YYYY-MM-DD-<keyword>-<hash8>.md`.

Uma nova gravação no mesmo dia e para a mesma entrada usa o mesmo caminho e
atualiza o arquivo; as versões anteriores permanecem no histórico de commits.

O Markdown deve ter este frontmatter. No modo `criar`, omita `url` e
`normalized_url`. Use `null` quando a fonte não trouxer
`source_last_modified_at`:

```yaml
---
type: pre_analise_pagina_termo
mode: <criar-ou-otimizar>
verdict: <livre-atencao-conflito-ou-inconclusivo>
keyword: <keyword-original>
url: <url-original-apenas-no-modo-otimizar>
normalized_keyword: <keyword-normalizada>
normalized_url: <url-normalizada-apenas-no-modo-otimizar>
queried_at: <ISO-8601-retornado-pela-tool>
source_last_modified_at: <ISO-8601-ou-null>
source: bigquery:iconic-rampart-279113.SearchHub.quickwins
truncated: <true-ou-false>
counts:
  total: <inteiro>
  conflito: <inteiro>
  atencao: <inteiro>
  informativo: <inteiro>
---
```

No corpo, registre o resumo, os avisos e a mesma tabela de evidências exibida
ao analista. Grave o único arquivo com `searchhub_project_save_batch`, passando
o `ws_slug` e `proj_slug` ativos, uma entrada em `files` e uma mensagem curta
de commit. Nunca escreva diretamente no filesystem do projeto.

Se a gravação retornar `conflict`, execute `searchhub_materialize_project` e
repita uma vez a gravação no mesmo caminho com o mesmo payload. Não refaça a
consulta nem altere o veredito. Se ainda falhar, preserve o conteúdo no chat e
informe que nada foi salvo.

Ao salvar com sucesso, mostre o caminho, a URL e o commit retornados pela tool.
