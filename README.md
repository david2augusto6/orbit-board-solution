# OrbitBoard

> Trabalho Final — Módulo 5: Integração Full Stack


O objetivo didático do trabalho é revisar e
praticar a integração entre front-end, back-end/API e infraestrutura conteinerizada
(Módulo 5 — Integração Full Stack).


## Integrantes da equipe

- Cristiano Peniche Ceccon
- David Augusto de Oliveira e Silva
- Stevão Whinter Marques De Andrade
- Nelson Enrique Villarreal Gonzalez


## Descrição da aplicação

O **OrbitBoard** é uma aplicação de acompanhamento de projetos, tarefas e equipe (estilo
kanban/gestão de projetos). Permite cadastrar projetos, criar e filtrar tarefas, mudar o status
delas e visualizar métricas em um dashboard. 


## Arquitetura

- **Front-end:** React 18 + Vite. Consome a API via `src/api/client.js`, com telas de
  Dashboard, Projetos, Tarefas e Equipe, e tratamento de estados de carregamento, sucesso e erro.

- **Back-end/API:** ASP.NET Core (.NET 8), com controllers para Dashboard, Projetos, Tarefas e
  Membros da equipe. Respostas em JSON, validação com Data Annotations, erros padronizados via
  `ProblemDetails` e Swagger/OpenAPI habilitado.

- **Dados:** mantidos em memória no back-end (`WorkspaceService`), recriados a cada reinício da
  API — sem necessidade de banco de dados.
  
- **Infraestrutura:** front-end e back-end executados via Docker Compose — back-end em imagem
  `.NET`, front-end buildado com Vite e servido via Nginx (ver seção "Via Docker Compose" abaixo).

## Tecnologias utilizadas

- **Front-end:** React 18, Vite 5, react-router-dom
- **Back-end:** .NET 8 / ASP.NET Core, Swagger (Swashbuckle)
- **Infraestrutura:** Docker, Docker Compose, Nginx (para servir o build do front-end)

## Como executar

### Localmente (sem Docker)

**Back-end** (requer .NET SDK 8):

```bash
cd backend
dotnet restore OrbitBoard.Api.sln
dotnet run --project OrbitBoard.Api
```

A API sobe em `http://localhost:5200`.

**Front-end** (requer Node.js 20+ e npm 10+):

```bash
cd frontend
npm install
cp .env.example .env   # já vem com VITE_API_URL=http://localhost:5200
npm run dev
```

O front-end sobe em `http://localhost:5173`.

### Via Docker Compose

```bash
docker compose up --build
```

Serviços definidos no `docker-compose.yml`:

| Serviço  | Container            | Porta host → container | Observações |
|----------|----------------------|--------------------------|-------------|
| backend  | `orbitboard-backend`  | `5200 → 5200`            | `ASPNETCORE_URLS=http://0.0.0.0:5200` |
| frontend | `orbitboard-frontend` | `8080 → 80`               | build feito com `VITE_API_URL=http://localhost:5200`; servido via Nginx |

- **Backend:** imagem multi-stage (`dotnet/sdk:8.0` para build, `dotnet/aspnet:8.0` para runtime).
- **Frontend:** imagem multi-stage (`node:22-alpine` para build do Vite, `nginx:1.27-alpine`
  para servir os arquivos estáticos usando o `nginx.conf` do projeto).

## URLs de acesso

| Serviço      | Local (dev)             | Via Docker Compose        |
|--------------|--------------------------|-----------------------------|
| Front-end    | http://localhost:5173    | http://localhost:8080       |
| Back-end/API | http://localhost:5200    | http://localhost:5200       |
| Swagger      | http://localhost:5200/swagger | http://localhost:5200/swagger |
| Health check (API)       | http://localhost:5200/health | http://localhost:5200/health |
| Health check (front-end) | —                        | http://localhost:8080/health *(via Nginx)* |

## Endpoints principais da API

### Dashboard

| Método | Rota | Descrição |
|--------|------|-----------|
| GET | `/api/dashboard` | Retorna métricas e tarefas recentes |


### Health

| Método | Rota | Descrição |
|--------|------|-----------|
| GET | `/health` | Verifica a saúde da aplicação |

### Projetos

| Método | Rota | Descrição |
|--------|------|-----------|
| GET | `/api/projects` | Lista projetos |
| GET | `/api/projects/{id}` | Consulta um projeto |
| POST | `/api/projects` | Cria um projeto |
| PUT | `/api/projects/{id}` | Atualiza um projeto |
| DELETE | `/api/projects/{id}` | Exclui um projeto (sem tarefas) |

### Tarefas

| Método | Rota | Descrição |
|--------|------|-----------|
| GET | `/api/tasks` | Lista e filtra tarefas (`projectId`, `status`, `priority`, `assigneeId`, `search`) |
| GET | `/api/tasks/{id}` | Consulta uma tarefa |
| POST | `/api/tasks` | Cria uma tarefa |
| PUT | `/api/tasks/{id}` | Atualiza uma tarefa |
| PATCH | `/api/tasks/{id}/status` | Altera somente o status |
| DELETE | `/api/tasks/{id}` | Exclui uma tarefa |

### Equipe

| Método | Rota | Descrição |
|--------|------|-----------|
| GET | `/api/team-members` | Lista os integrantes |

## Variáveis de ambiente

| Variável | Onde é usada | Descrição | Exemplo |
|----------|--------------|------------|---------|
| `VITE_API_URL` | Front-end | URL base da API consumida pelo cliente HTTP | `http://localhost:5200` |
| `ASPNETCORE_ENVIRONMENT` | Back-end | Ambiente de execução do ASP.NET Core | `Development` |

Ver `frontend/.env.example`.

## CORS

A política de CORS do back-end (`Program.cs`) libera dinamicamente qualquer origem que contenha
`localhost` ou `127.0.0.1`, em qualquer porta (`SetIsOriginAllowed`). Isso cobre tanto o modo dev
(`http://localhost:5173`) quanto o front-end containerizado (`http://localhost:8080`) sem
precisar listar portas fixas.


## Ajustes realizados pela equipe

- CORS ajustado no back-end para liberar dinamicamente qualquer origem `localhost`/`127.0.0.1`
  (em vez de uma porta fixa), garantindo que o front-end funcione tanto em modo dev (`:5173`)
  quanto containerizado (`:8080`).
- Criação do `backend/Dockerfile` (build multi-stage com .NET SDK/ASP.NET runtime) e do
  `frontend/Dockerfile` (build multi-stage com Node/Vite e runtime Nginx), já que o código-base
  não vinha com nenhum dos dois.
- Criação do `docker-compose.yml` orquestrando back-end (porta 5200) e front-end (porta 8080,
  servido via Nginx), permitindo subir a aplicação completa com `docker compose up --build`.
- Pipeline de CI simples adicionada via GitHub Actions, validando o build do back-end e do
  front-end a cada push/PR.
- Alteração do README.md

## CI/CD

O repositório inclui um workflow de GitHub Actions (`.github/workflows/ci.yml`) disparado em
`push` e `pull request` para as branches `main` e `develop` (também pode ser rodado manualmente
via `workflow_dispatch`). Ele roda dois jobs em paralelo:

| Job | O que faz |
|-----|-----------|
| **Validar Backend (.NET)** | `dotnet restore` + `dotnet build` do `OrbitBoard.Api.sln` (.NET 8) |
| **Validar Frontend (Vite)** | `npm install` + `npm run build` do front-end (Node 22) |

O pipeline garante que back-end e front-end continuam compilando a cada mudança, sem rodar
testes automatizados adicionais (não há suíte de testes no código-base).

## Documentação adicional

- [`docs/arquitetura.md`](docs/arquitetura.md) — arquitetura detalhada
- [`docs/contrato-api.md`](docs/contrato-api.md) — contrato da API
- [`docs/evidencias-testes.md`](docs/evidencias-testes.md) — evidências de testes

## Contribuição da equipe

| Integrante | Contribuição |
|------------|---------------|
| David Augusto de Oliveira e Silva | Docker e CI |
| Stevão Whinter Marques De Andrade | Documentação |
| Nelson Enrique Villarreal Gonzalez | Testes e Evidências |
| Cristiano Peniche Ceccon | Apresentação e Roteiro |

