# Especificação - Document Management System

## 1. Objetivo

Entregar uma aplicação web simples para enviar, listar e baixar documentos,
mantendo metadados em memória e armazenando os arquivos localmente no
servidor.

## 2. Escopo

### Dentro do escopo

- Upload de um documento por uma requisição HTTP;
- Validação básica da entrada e do arquivo recebido;
- Listagem dos documentos enviados;
- Download de um documento pelo seu identificador;
- Associação dos documentos a um identificador lógico de usuário;
- Interface web básica para upload, listagem e download;
- Tratamento consistente de erros de entrada e de recursos não encontrados;
- Testes automatizados básicos do backend.

### Fora do escopo

- Armazenamento externo, em nuvem ou em provedores de terceiros;
- Banco de dados persistente;
- Versionamento ou histórico de versões;
- Exclusão ou substituição de documentos;
- Busca por conteúdo ou metadados avançados;
- Paginação, ordenação ou filtros complexos;
- Autenticação, autorização ou gestão de contas;
- Compartilhamento entre usuários;
- Antivírus, preview, OCR ou conversão de documentos;
- Escalabilidade multi-instância com compartilhamento de arquivos.

## 3. Requisitos funcionais

| ID | Requisito |
| --- | --- |
| RF-01 | O sistema deve aceitar o envio de um arquivo por `multipart/form-data`. |
| RF-02 | O sistema deve gravar o conteúdo recebido no filesystem local da aplicação. |
| RF-03 | O sistema deve criar metadados para cada documento enviado e atribuir um identificador único. |
| RF-04 | O sistema deve registrar o nome original, o tamanho, a data/hora do upload e o dono do documento. |
| RF-05 | O sistema deve rejeitar requisições de upload que não incluam um campo de arquivo válido. |
| RF-06 | O sistema deve listar os metadados dos documentos disponíveis. |
| RF-07 | A listagem deve ser limitada aos documentos pertencentes ao dono identificado na requisição. |
| RF-08 | O sistema deve permitir baixar o conteúdo pelo identificador do documento. |
| RF-09 | O download deve usar o nome original do arquivo no cabeçalho de disposição da resposta. |
| RF-10 | O sistema deve informar que o documento não existe ou não pertence ao dono solicitado. |
| RF-11 | O frontend deve disponibilizar um campo de seleção de arquivo e uma ação de envio. |
| RF-12 | O frontend deve atualizar a lista após um upload concluído com sucesso. |
| RF-13 | O frontend deve exibir erros de upload e download em português de forma compreensível. |
| RF-14 | O frontend deve disponibilizar um link ou botão de download para cada documento listado. |

## 4. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | O backend deve ser implementado em Node.js com Express e JavaScript CommonJS. |
| RNF-02 | O frontend deve ser implementado com React, Vite e JavaScript ESM. |
| RNF-03 | Arquivos devem ser gravados exclusivamente em `backend/storage`, por padrão, usando `multer.diskStorage`. |
| RNF-04 | Metadados devem permanecer em memória durante a execução da aplicação. |
| RNF-05 | Os metadados não precisam sobreviver a reinicializações do processo. |
| RNF-06 | A configuração de porta, diretório de armazenamento e limite de tamanho deve ser feita por variáveis de ambiente. |
| RNF-07 | O diretório de armazenamento deve ser criado automaticamente quando não existir. |
| RNF-08 | Toda entrada HTTP deve ser validada em um controller ou middleware na borda da aplicação. |
| RNF-09 | Erros internos não devem expor caminhos físicos, pilhas ou detalhes sensíveis ao cliente. |
| RNF-10 | Os nomes armazenados no filesystem não podem usar o nome original, evitando colisões e caminhos inválidos. |
| RNF-11 | O backend deve usar a arquitetura Clean Architecture simples: `routes -> controllers -> services -> repositories`. |
| RNF-12 | Rotas não devem acessar diretamente repositórios ou o filesystem. |
| RNF-13 | A persistência em memória deve ser encapsulada em repository para permitir substituição futura por outro mecanismo. |
| RNF-14 | Respostas JSON devem usar `application/json; charset=utf-8`. |
| RNF-15 | Datas devem ser representadas como strings ISO 8601 em UTC. |
| RNF-16 | Testes backend devem usar o runner nativo `node:test`. |

## 5. Modelo de dados

### Documento

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `id` | string | Sim | Identificador único gerado pelo sistema. |
| `originalName` | string | Sim | Nome enviado pelo usuário no multipart, sem remoção indevida da extensão. |
| `storedName` | string | Sim | Nome técnico usado em `backend/storage`; não deve ser recebido nem exposto pelo cliente. |
| `mimeType` | string | Não | Tipo MIME informado pelo navegador para o upload. |
| `size` | number | Sim | Tamanho do arquivo em bytes. |
| `uploadedAt` | string | Sim | Data/hora do upload no formato ISO 8601 UTC. |
| `owner` | string | Sim | Identificador lógico do usuário dono. |

### Estrutura JSON pública

Os campos técnicos de armazenamento não fazem parte das respostas públicas:

```json
{
  "id": "c7d13d26-3f78-4e5f-a290-3d5f41ab5ee4",
  "originalName": "contrato.pdf",
  "mimeType": "application/pdf",
  "size": 248102,
  "uploadedAt": "2026-11-23T14:03:27.097Z",
  "owner": "usuario-demo"
}
```

### Vida dos dados

- Os metadados são armazenados em um repositório em memória por instância do
  processo.
- Os arquivos são armazenados no filesystem local.
- Reinicializar o processo pode remover o vínculo entre metadados e arquivos.
  Essa situação não exige migração ou recuperação nesta fase.
- Não há garantias de consistência em uma instância com múltiplos processos.

### Identificação do dono

Para esta fase inicial, sem autenticação, o identificador de dono deve ser
obtido do cabeçalho `X-User-Id`. Quando o cabeçalho estiver ausente ou vazio,
o sistema deve usar `default-user`. O cabeçalho é um identificador lógico da
aplicação, não um mecanismo de autenticação.

## 6. Contratos de API

Todas as respostas de erro devem seguir o formato:

```json
{
  "error": {
    "code": "DOCUMENT_NOT_FOUND",
    "message": "Documento não encontrado."
  }
}
```

O campo `message` não precisa ser exibido literalmente pelo frontend, mas deve
ser utilizado como fallback quando não houver mensagem específica.

### POST /upload

#### Entrada

- `Content-Type`: `multipart/form-data`;
- campo obrigatório: `file`;
- cabeçalho opcional: `X-User-Id`;
- limite de tamanho definido pela variável de ambiente `MAX_FILE_SIZE`, em
  bytes, com valor padrão documentado na configuração da implementação.

Exemplo:

```text
POST /upload
Content-Type: multipart/form-data; boundary=...
X-User-Id: usuario-demo

arquivo no campo `file`
```

#### Respostas

**201 Created**

```json
{
  "id": "c7d13d26-3f78-4e5f-a290-3d5f41ab5ee4",
  "originalName": "contrato.pdf",
  "mimeType": "application/pdf",
  "size": 248102,
  "uploadedAt": "2026-11-23T14:03:27.097Z",
  "owner": "usuario-demo"
}
```

**400 Bad Request — arquivo ausente**

```json
{
  "error": {
    "code": "FILE_REQUIRED",
    "message": "Nenhum arquivo foi enviado."
  }
}
```

**413 Payload Too Large — arquivo excede o limite**

```json
{
  "error": {
    "code": "FILE_TOO_LARGE",
    "message": "O arquivo excede o tamanho máximo permitido."
  }
}
```

**500 Internal Server Error — falha de armazenamento**

```json
{
  "error": {
    "code": "UPLOAD_FAILED",
    "message": "Não foi possível armazenar o documento."
  }
}
```

### GET /documents

#### Entrada

- Cabeçalho opcional `X-User-Id`;
- sem corpo.

#### Resposta

**200 OK**

```json
[
  {
    "id": "c7d13d26-3f78-4e5f-a290-3d5f41ab5ee4",
    "originalName": "contrato.pdf",
    "mimeType": "application/pdf",
    "size": 248102,
    "uploadedAt": "2026-11-23T14:03:27.097Z",
    "owner": "usuario-demo"
  }
]
```

A lista não inclui documentos de outros donos. Se não houver documentos, a
resposta deve ser `[]`. A ordem inicial é mais novo primeiro; a ordenação pode
ser usada pelo frontend sem depender de ordenação interna.

### GET /documents/:id/download

#### Entrada

- parâmetro de rota `id`;
- cabeçalho opcional `X-User-Id`.

#### Resposta de sucesso

**200 OK**

- `Content-Type`: tipo MIME conhecido do documento ou
  `application/octet-stream`;
- `Content-Disposition`: `attachment; filename="<originalName>"`;
- corpo: conteúdo binário armazenado.

#### Erros

**404 Not Found**

```json
{
  "error": {
    "code": "DOCUMENT_NOT_FOUND",
    "message": "Documento não encontrado."
  }
}
```

O mesmo código deve ser usado quando o identificador não existe ou pertence a
outro usuário, evitando divulgar a existência de documentos alheios.

**500 Internal Server Error**

```json
{
  "error": {
    "code": "DOWNLOAD_FAILED",
    "message": "Não foi possível baixar o documento."
  }
}
```

### Regras comuns

- Headers desconhecidos não alteram o contrato;
- o backend não deve aceitar ou devolver o campo técnico `storedName`;
- o `X-User-Id` deve ser normalizado como texto não vazio;
- identificadores de rota inválidos devem resultar em `DOCUMENT_NOT_FOUND`;
- erros devem utilizar os códigos HTTP correspondentes;
- o prefixo `/api` no frontend deve apontar para o backend pelo proxy do Vite.

## 7. Decisões arquiteturais

### Backend

A implementação futura deve seguir a estrutura:

```text
backend/src/
  routes/
  controllers/
  services/
  repositories/
  app.js
backend/storage/
backend/test/
```

- `routes`: define os endpoints e delega para os controllers;
- `controllers`: valida entrada HTTP, cabeçalhos e parâmetros, monta respostas;
- `services`: aplica regras de negócio como definição do dono e consistência
  entre metadados e arquivo;
- `repositories`: mantém o índice de metadados em memória e realiza operações
  relacionadas ao armazenamento local;
- `app.js`: configura Express, multer e o encadeamento da aplicação.

O fluxo obrigatório é `routes -> controllers -> services -> repositories`.
Nenhuma camada interna pode conhecer Express, objetos `req`/`res` ou multer.

### Upload com multer

- O arquivo deve ser recebido pelo campo `file`;
- `diskStorage` deve escrever em `backend/storage`;
- `storedName` deve ser gerado pelo servidor, sem reutilizar o nome original;
- o metadado deve ser criado somente após a gravação bem-sucedida;
- `multer` deve aplicar o limite `MAX_FILE_SIZE`;
- o código deve remover o arquivo gravado quando a criação do metadado falhar,
  evitando arquivos órfãos sempre que possível.

### Frontend

- Organização por `components/`, `pages/` e `services/`;
- componentes funcionais com React Hooks;
- comunicação HTTP via `fetch`;
- URLs da aplicação devem usar o prefixo `/api`;
- o proxy do Vite deve encaminhar `/api` ao Express sem exigir URLs absolutas;
- nomes de componentes, funções e símbolos em inglês;
- mensagens apresentadas ao usuário em português.

### Configuração

Variáveis esperadas:

| Variável | Obrigatória | Valor padrão sugerido | Uso |
| --- | --- | --- | --- |
| `PORT` | Não | `3000` | Porta HTTP do backend |
| `STORAGE_DIR` | Não | `backend/storage` | Diretório local dos arquivos |
| `MAX_FILE_SIZE` | Não | `10485760` | Limite de upload em bytes |

A configuração deve ser carregada por variáveis de ambiente, sem valores
dependentes de arquivos locais no código da aplicação.

## 8. Critérios de aceite

### Upload

- Dado um arquivo válido, o envio resulta em HTTP 201;
- a resposta contém metadados públicos, mas não `storedName`;
- o arquivo aparece no diretório de armazenamento;
- o arquivo pode ser localizado por meio do identificador retornado;
- envio sem o campo `file` resulta em HTTP 400;
- arquivo acima do limite resulta em HTTP 413;
- falha de escrita resulta em HTTP 500 sem vazamento de detalhes internos.

### Listagem

- A listagem de um usuário contém apenas documentos desse usuário;
- um usuário diferente recebe uma lista vazia ou seus próprios documentos;
- quando não existem documentos, a resposta é HTTP 200 com `[]`;
- reinicializar o processo limpa os metadados em memória.

### Download

- Documento existente e pertencente ao dono solicitado resulta em HTTP 200;
- o corpo retornado corresponde ao arquivo armazenado;
- `Content-Disposition` contém o nome original;
- documento inexistente ou de outro usuário resulta em HTTP 404;
- arquivo sem metadado correspondente não é baixado.

### Frontend

- A interface permite selecionar e enviar um arquivo;
- a lista é carregada ao abrir a página;
- a lista é atualizada após um upload bem-sucedido;
- cada item oferece download;
- erros são apresentados ao usuário em português;
- estados vazio e de carregamento são apresentados de forma compreensível.

## 9. Plano de execução

Este documento define o trabalho futuro, mas não executa a implementação dos
arquivos de backend ou frontend nesta etapa.

1. Criar a base de configuração do backend:
   validar variáveis de ambiente, definir porta, diretório de armazenamento e
   limite de upload.

2. Implementar o repositório em memória:
   criar operações para inserir, listar por proprietário e buscar por
   identificador.

3. Implementar os serviços de documentos:
   gerar identificador, definir proprietário, validar regras e coordenação
   entre metadados e filesystem.

4. Configurar multer com `diskStorage`:
   assegurar diretório existente, nome técnico seguro e limite de tamanho.

5. Implementar controllers e rotas:
   adicionar `POST /upload`, `GET /documents` e
   `GET /documents/:id/download`, incluindo códigos de erro e respostas binárias.

6. Configurar o servidor Express:
   registrar rotas, middleware de JSON, tratamento de erros do multer e inicialização.

7. Criar testes backend focados:
   cobrir upload válido, campo ausente, limite excedido, listagem por usuário,
   download válido e recurso não encontrado.

8. Implementar o frontend:
   criar componente de upload, listagem e download usando React, fetch e
   estados de carregamento, sucesso e erro.

9. Configurar o proxy do Vite:
   encaminhar chamadas `/api` para o servidor Express sem URLs absolutas no
   código das páginas.

10. Executar validação ponta a ponta:
    subir backend e frontend, testar o fluxo completo e confirmar critérios de
    aceite, build e testes automatizados.

11. Registrar a execução:
    documentar comandos de inicialização e teste sem alterar o escopo ou as
    decisões desta especificação.
