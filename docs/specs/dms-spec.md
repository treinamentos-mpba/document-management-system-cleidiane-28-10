# Especificacao - Document Management System

## 1. Objetivo

Entregar uma aplicacao web que permita enviar, listar e baixar documentos, armazenando os arquivos no filesystem local e mantendo seus metadados em memoria nesta fase.

## 2. Escopo

### Dentro do escopo

- Enviar um documento por requisicao multipart.
- Listar os documentos conhecidos pelo processo atual.
- Baixar um documento por identificador.
- Registrar um campo `owner` informativo, sem autenticacao ou autorizacao nesta fase.
- Exibir no frontend o formulario de envio, a lista e a acao de download.
- Tratar validacoes, erros de armazenamento e identificadores inexistentes de forma previsivel.

### Fora do escopo

- Autenticacao, autorizacao e isolamento de documentos por usuario.
- Armazenamento externo ou em nuvem.
- Persistencia duravel dos metadados, recuperacao apos reinicio ou sincronizacao entre instancias.
- Versionamento, edicao, exclusao, busca, compartilhamento e pre-visualizacao de documentos.
- Restricao por extensao ou tipo MIME.

### Estado atual

O backend atualmente expoe apenas `GET /health`. As rotas de documentos, a logica de upload/listagem/download e a interface funcional ainda nao estao implementadas. Esta especificacao descreve o comportamento a construir, nao uma funcionalidade existente.

## 3. Requisitos funcionais

| ID | Requisito | Criterio de aceite |
| --- | --- | --- |
| RF-01 | O usuario pode enviar um documento usando `multipart/form-data`. | Uma requisicao valida cria um arquivo local e retorna os metadados com status `201`. |
| RF-02 | O upload aceita o campo multipart `file` e, opcionalmente, `owner`. | Sem `file`, a API retorna `400`; sem `owner`, registra `anonymous`. |
| RF-03 | O sistema atribui um identificador unico ao documento. | O identificador nao depende do nome fornecido e identifica o arquivo salvo. |
| RF-04 | O sistema limita o tamanho do arquivo. | Arquivos acima do limite configurado sao rejeitados com `413`, sem registro de metadados. |
| RF-05 | O usuario pode listar os metadados dos documentos conhecidos pelo processo atual. | A resposta inclui todos os documentos registrados, do mais recente ao mais antigo; uma lista vazia e valida. |
| RF-06 | O usuario pode baixar um documento pelo identificador. | Um identificador existente retorna o conteudo binario como anexo com o nome original. |
| RF-07 | O sistema informa quando um documento nao existe ou nao esta disponivel no disco. | A API retorna `404` com erro JSON, sem expor caminhos internos. |
| RF-08 | O frontend permite upload, lista os documentos retornados e solicita o download por identificador. | Erros da API sao apresentados sem apagar a lista ou indicar sucesso indevidamente. |
| RF-09 | O sistema mantem `owner` apenas como dado informativo nesta fase. | O campo nao autentica o solicitante nem restringe listagem ou download. |

## 4. Requisitos nao funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | Os arquivos sao gravados exclusivamente no filesystem local em `backend/storage`, usando Multer com `diskStorage`. |
| RNF-02 | Metadados ficam em memoria no processo da aplicacao; nao se adiciona banco de dados nesta fase. |
| RNF-03 | O limite padrao de upload e 10 MiB (10.485.760 bytes) e pode ser configurado por `UPLOAD_MAX_SIZE_BYTES`. |
| RNF-04 | A porta HTTP e configurada por `PORT`, mantendo o valor padrao existente de `3000`. |
| RNF-05 | Nenhuma resposta publica expoe caminhos absolutos ou nomes internos usados no filesystem. |
| RNF-06 | Erros de entrada, limites do Multer e falhas de disco sao convertidos para respostas HTTP/JSON consistentes. |
| RNF-07 | O backend permanece em Node.js/Express/CommonJS; o frontend permanece em React/Vite/ESM e usa `fetch`. |
| RNF-08 | A arquitetura do backend segue `routes -> controllers -> services -> repositories`; dependencias apontam para camadas internas. |

### Limites e comportamento de armazenamento

- O limite aplica-se ao tamanho do arquivo, nao ao tamanho total da requisicao multipart.
- Sao aceitos arquivos de qualquer extensao e tipo MIME, inclusive arquivos vazios. O campo `file` e obrigatorio.
- O nome fisico e gerado pelo servidor a partir do identificador; o nome enviado pelo cliente nao e usado como caminho.
- O diretorio `backend/storage` deve existir e estar gravavel. Falha de escrita resulta em erro `500` e nao cria metadado.
- Metadados desaparecem quando o processo reinicia, mas os arquivos podem permanecer no disco. Arquivos sem metadados nao sao recuperados nem removidos automaticamente nesta fase.
- Em uma falha ao registrar metadados depois de gravar o arquivo, o fluxo deve tentar remover o arquivo recem-gravado para evitar orfao adicional.
- O armazenamento local e adequado somente ao ambiente e escala desta fase; multiplas instancias nao compartilham arquivos nem metadados.

## 5. Modelo de dados

### Metadados publicos do documento

| Campo | Tipo | Obrigatorio | Descricao |
| --- | --- | --- | --- |
| `id` | string | sim | UUID gerado pelo servidor; identificador publico do documento. |
| `originalName` | string | sim | Nome original fornecido no upload, retornado apenas como metadado e nome de download. |
| `size` | number | sim | Tamanho do arquivo em bytes, informado pelo filesystem apos a gravacao. |
| `uploadedAt` | string | sim | Data/hora UTC em ISO 8601, por exemplo `2026-10-28T15:04:05.000Z`. |
| `owner` | string | sim | Identificador informativo enviado no formulario; `anonymous` quando ausente ou vazio. Nao representa identidade autenticada. |

Exemplo:

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "originalName": "relatorio.pdf",
  "size": 24576,
  "uploadedAt": "2026-10-28T15:04:05.000Z",
  "owner": "anonymous"
}
```

O caminho fisico do arquivo e detalhe interno, nao faz parte do objeto publico. O repositorio mantem a associacao entre `id`, caminho interno e metadados em memoria.

## 6. Contratos de API

### Convencoes

- As rotas do backend nao incluem `/api`; o prefixo e usado pelo frontend e removido pelo proxy do Vite.
- JSON usa UTF-8. Os erros seguem `{ "error": { "code": "...", "message": "..." } }`.
- Os nomes de codigo de erro abaixo sao estaveis para consumo pelo frontend; mensagens sao descritivas e podem variar.
- `owner` e metadado fornecido pelo cliente, nao uma credencial. A API e de uso simples/local e nao deve ser considerada protegida para acesso publico.

### `POST /upload`

**Entrada:** `multipart/form-data`.

| Campo | Tipo | Obrigatorio | Regra |
| --- | --- | --- | --- |
| `file` | arquivo | sim | Um unico arquivo; tamanho ate `UPLOAD_MAX_SIZE_BYTES` (10 MiB por padrao). |
| `owner` | texto | nao | Valor informativo; se ausente ou vazio, usar `anonymous`. |

**Sucesso `201 Created`:**

```json
{
  "document": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "originalName": "relatorio.pdf",
    "size": 24576,
    "uploadedAt": "2026-10-28T15:04:05.000Z",
    "owner": "anonymous"
  }
}
```

**Erros:**

| Status | Codigo | Situacao |
| --- | --- | --- |
| `400` | `FILE_REQUIRED` | Campo `file` ausente ou nenhum arquivo recebido. |
| `413` | `FILE_TOO_LARGE` | Arquivo excede o limite configurado. |
| `500` | `UPLOAD_FAILED` | Falha inesperada ao gravar arquivo ou registrar metadados. |

### `GET /documents`

Nao recebe parametros. Retorna todos os metadados mantidos no processo atual, ordenados por `uploadedAt` decrescente.

**Sucesso `200 OK`:**

```json
{
  "documents": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "originalName": "relatorio.pdf",
      "size": 24576,
      "uploadedAt": "2026-10-28T15:04:05.000Z",
      "owner": "anonymous"
    }
  ]
}
```

Quando nao houver documentos, retorna `200 OK` com `{"documents": []}`. Nesta fase nao ha paginacao nem filtro por `owner`.

### `GET /documents/:id/download`

**Sucesso `200 OK`:** corpo binario do arquivo, `Content-Type: application/octet-stream` e `Content-Disposition: attachment` com o nome original devidamente tratado como valor de cabecalho.

**Erros:**

| Status | Codigo | Situacao |
| --- | --- | --- |
| `404` | `DOCUMENT_NOT_FOUND` | O identificador nao consta nos metadados em memoria. |
| `404` | `FILE_NOT_FOUND` | Ha metadado, mas o arquivo nao esta disponivel no filesystem. |
| `500` | `DOWNLOAD_FAILED` | Outra falha inesperada durante a leitura ou envio. |

Os erros usam o envelope JSON definido nas convencoes. O caminho interno do arquivo nunca e retornado.

### Acesso pelo frontend

O frontend usa `fetch` com os caminhos `/api/upload`, `/api/documents` e `/api/documents/:id/download`. A configuracao atual do Vite encaminha `/api` para o backend e remove esse prefixo, chegando respectivamente a `/upload`, `/documents` e `/documents/:id/download`.

## 7. Decisoes arquiteturais

- **Routes:** declaram os caminhos e compoem middleware de upload Multer; nao implementam regra de negocio.
- **Controllers:** adaptam `req`/`res`, leem campos multipart e traduzem resultados e erros para HTTP.
- **Services:** coordenam validacao de negocio, metadados e operacoes de documento; nao conhecem Express.
- **Repositories:** mantem metadados e associacao interna dos arquivos em memoria, com consulta por identificador.
- **Multer:** usa `diskStorage` no limite HTTP para gravar em `backend/storage`, com nome fisico gerado pelo servidor e limite configuravel.
- **Frontend:** componentes React consomem a API por um servico baseado em `fetch`; upload, listagem e download permanecem separados da apresentacao da pagina.
- **Persistencia:** nao introduzir banco ou servico externo. A volatilidade dos metadados e uma limitacao aceita e explicita.
- **Testabilidade:** services e repositories devem poder ser exercitados sem iniciar o frontend; testes HTTP cobrem status, payloads, limite e fluxo de arquivo temporario/local.

## 8. Plano de execucao

As etapas abaixo sao um roteiro futuro de implementacao. Nesta tarefa, somente este documento de especificacao e criado; nenhum arquivo de aplicacao e alterado ou executado.

1. **Preparacao e erros:** definir configuracao de ambiente, limite de upload e formato comum de erros, preservando o health check.
2. **Upload local:** compor a rota de upload com Multer `diskStorage`, gerar identificador seguro, validar limite e registrar os metadados por meio das camadas previstas.
3. **Listagem e download:** expor consultas de metadados em memoria e envio binario por identificador, incluindo casos nao encontrados e falhas de disco.
4. **Testes de backend:** cobrir contratos HTTP, metadados, limites, arquivo persistido localmente, download e erros; isolar arquivos temporarios para limpeza dos testes.
5. **Frontend:** implementar o formulario, estados de carregamento/erro, listagem atualizada apos upload e download de arquivo usando o prefixo `/api`.
6. **Integracao e aceite:** validar os fluxos completos com backend e frontend, conferir configuracao documentada e confirmar que os arquivos ficam apenas em `backend/storage`.

## 9. Criterios de aceite

- Os tres endpoints seguem os contratos e codigos HTTP especificados.
- O upload grava localmente com nome fisico controlado pelo servidor, respeita 10 MiB por padrao e devolve metadados sem caminho interno.
- A listagem retorna metadados em ordem decrescente e funciona vazia.
- O download devolve o arquivo como anexo ou o erro documentado.
- A UI permite enviar, listar e baixar via proxy `/api`, exibindo falhas sem declarar sucesso incorretamente.
- A arquitetura mantem o fluxo de dependencias indicado e nao adiciona persistencia externa.
- A perda dos metadados ao reiniciar e a ausencia de autenticacao estao documentadas como limitacoes desta fase.