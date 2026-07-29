# Arquitetura — OrbitBoard

## Visão geral

O OrbitBoard é uma aplicação full stack de acompanhamento de projetos e tarefas, dividida em
três camadas: front-end (React), back-end/API (.NET) e dados em memória — sem banco de dados
externo.

```
┌─────────────────────┐        HTTP/JSON        ┌──────────────────────────┐
│   Front-end (React)  │ ───────────────────────▶│  Back-end/API (.NET 8)   │
│   Vite + Nginx        │◀───────────────────────│  ASP.NET Core            │
└─────────────────────┘                           └──────────────────────────┘
                                                              │
                                                              ▼
                                                   ┌──────────────────────────┐
                                                   │  WorkspaceService         │
                                                   │  (dados em memória)       │
                                                   └──────────────────────────┘
```

## Front-end

- **Tecnologia:** React 18 + Vite, com `react-router-dom` para navegação entre páginas.
- **Páginas:** Dashboard, Projetos, Tarefas e Equipe (`src/pages/`).
- **Comunicação com a API:** centralizada em `src/api/client.js`, que lê a URL base da API a
  partir da variável de ambiente `VITE_API_URL`.
- **Tratamento de estado:** cada página trata os três estados de uma chamada HTTP — carregando,
  sucesso e erro — exibindo mensagens amigáveis quando a API retorna um erro (ex: nome de
  projeto duplicado).
- **Execução em produção:** o build gerado pelo Vite é servido por um container Nginx
  (`frontend/nginx.conf`), e não pelo próprio Vite.

## Back-end / API

- **Tecnologia:** .NET 8 / ASP.NET Core, com documentação automática via Swagger (Swashbuckle).

- **Organização em camadas:**

  - `Controllers/` — expõem os endpoints HTTP (`DashboardController`, `ProjectsController`,
    `TasksController`, `TeamMembersController`), sem lógica de negócio própria.

  - `Services/WorkspaceService` (via a interface `IWorkspaceService`) — concentra todas as
    regras de negócio e o acesso aos dados.

  - `DTOs/` — objetos de requisição (`CreateProjectRequest`, `UpdateWorkItemRequest` etc.) e de
    resposta (`ProjectResponse`, `WorkItemResponse`, `DashboardResponse`), evitando expor os
    modelos internos diretamente na API.

  - `Models/` — entidades de domínio (`Project`, `WorkItem`, `TeamMember`) e enums
    (`ProjectStatus`, `WorkItemStatus`, `WorkItemPriority`).

  - `Middleware/ExceptionHandlingMiddleware` — captura qualquer exceção lançada pelos serviços e
    converte em uma resposta JSON padronizada (`ProblemDetails`).

  - `Exceptions/` — exceções de negócio específicas (`NotFoundException`, `ConflictException`,
    `ValidationException`), usadas para sinalizar o tipo de erro ao middleware.

## Dados

Os dados (projetos, tarefas e membros da equipe) são mantidos inteiramente em memória, dentro
de `WorkspaceService`, registrado como *singleton* no `Program.cs`. Não há banco de dados —
os dados de exemplo são recriados sempre que a API é reiniciada. O acesso é protegido por um
lock (`_sync`), já que o singleton pode ser acessado por múltiplas requisições simultâneas.

## Tratamento de erros

Toda regra de negócio violada gera uma exceção específica, capturada pelo
`ExceptionHandlingMiddleware` e convertida automaticamente em uma resposta `ProblemDetails`
(formato padronizado de erro em JSON, RFC 7807):

| Exceção | Status HTTP | Quando ocorre |
|---------|-------------|----------------|
| `ValidationException` | 400 Bad Request | Dados inválidos (ex: data final anterior à inicial, integrante inexistente) |
| `NotFoundException` | 404 Not Found | Projeto ou tarefa não encontrados |
| `ConflictException` | 409 Conflict | Nome de projeto duplicado, ou exclusão de projeto que ainda tem tarefas |
| Qualquer outra exceção | 500 Internal Server Error | Erro não tratado |

## Infraestrutura

- **Docker:** `backend/Dockerfile` e `frontend/Dockerfile` fazem build multi-stage — o backend
  compila com o SDK do .NET e roda com a imagem de runtime do ASP.NET; o frontend builda com
  Node/Vite e roda com Nginx.

- **Docker Compose:** orquestra os dois serviços — API na porta `5200` e front-end na porta
  `8080` (mapeada para a porta `80` do Nginx dentro do container).

- **CORS:** o back-end libera dinamicamente qualquer origem `localhost`/`127.0.0.1`, cobrindo
  tanto o modo de desenvolvimento (`:5173`) quanto o front-end containerizado (`:8080`).
  
- **CI:** um workflow de GitHub Actions builda back-end e front-end a cada push/PR, garantindo
  que ambos continuam compilando.
