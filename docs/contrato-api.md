# Contrato da API — OrbitBoard

Base URL (dev): `http://localhost:5200`
Base URL (Docker Compose): `http://localhost:5200`
Documentação interativa: `/swagger`

Todas as respostas são em `application/json`. Erros seguem o formato `ProblemDetails`
(veja a seção "Formato de erro" no final deste documento).

---

## Dashboard

### `GET /api/dashboard`

Retorna métricas gerais e as tarefas mais recentes.

**Resposta `200 OK`:**

```json
{
  "totalProjects": 4,
  "activeProjects": 2,
  "totalTasks": 10,
  "completedTasks": 3,
  "overdueTasks": 1,
  "recentTasks": [ /* WorkItemResponse[] */ ],
  "tasksByStatus": { "Backlog": 4, "InProgress": 3, "Review": 1, "Done": 2 }
}
```

---

## Projetos

### `GET /api/projects`

Lista todos os projetos, ordenados por prazo (`dueDate`).

**Resposta `200 OK`:** `ProjectResponse[]`

### `GET /api/projects/{id}`

Consulta um projeto específico.

- `200 OK` → `ProjectResponse`
- `404 Not Found` → projeto não existe

### `POST /api/projects`

Cria um novo projeto.

**Corpo da requisição (`CreateProjectRequest`):**

```json
{
  "name": "string (3–80 caracteres)",
  "description": "string (10–500 caracteres)",
  "status": "Planning | Active | OnHold | Completed",
  "startDate": "2026-07-28",
  "dueDate": "2026-09-01",
  "ownerId": "guid de um membro da equipe existente"
}
```

- `201 Created` → `ProjectResponse`
- `400 Bad Request` → dados inválidos, ou `startDate` posterior ao `dueDate`, ou `ownerId`
  não corresponde a um membro existente
- `409 Conflict` → já existe um projeto com esse nome

### `PUT /api/projects/{id}`

Atualiza um projeto existente. Mesmo corpo de `POST`, mesmas validações (`UpdateProjectRequest`).

- `200 OK` → `ProjectResponse`

### `DELETE /api/projects/{id}`

Remove um projeto.

- `204 No Content` → removido com sucesso
- `409 Conflict` → o projeto ainda possui tarefas associadas e não pode ser excluído

---

## Tarefas

### `GET /api/tasks`

Lista tarefas, com filtros opcionais via query string:

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `projectId` | guid | Filtra por projeto |
| `status` | `Backlog \| InProgress \| Review \| Done` | Filtra por status |
| `priority` | `Low \| Medium \| High \| Critical` | Filtra por prioridade |
| `assigneeId` | guid | Filtra por responsável |
| `search` | string | Busca por título ou descrição (case-insensitive) |

Resultado ordenado por status, depois por prioridade (decrescente), depois por prazo.

**Resposta `200 OK`:** `WorkItemResponse[]`

### `GET /api/tasks/{id}`

- `200 OK` → `WorkItemResponse`
- `404 Not Found` → tarefa não existe

### `POST /api/tasks`

**Corpo da requisição (`CreateWorkItemRequest`):**

```json
{
  "projectId": "guid de um projeto existente",
  "title": "string (3–120 caracteres)",
  "description": "string (5–800 caracteres)",
  "status": "Backlog | InProgress | Review | Done",
  "priority": "Low | Medium | High | Critical",
  "assigneeId": "guid de um membro (opcional)",
  "dueDate": "2026-08-15 (opcional)",
  "estimatedHours": "inteiro entre 1 e 200"
}
```

- `201 Created` → `WorkItemResponse`
- `400 Bad Request` → dados inválidos, projeto ou responsável inexistentes

### `PUT /api/tasks/{id}`

Atualiza uma tarefa existente. Mesmo corpo de `POST` (`UpdateWorkItemRequest`).

- `200 OK` → `WorkItemResponse`

### `PATCH /api/tasks/{id}/status`

Altera somente o status da tarefa.

**Corpo da requisição:**

```json
{ "status": "InProgress" }
```

- `200 OK` → `WorkItemResponse`

### `DELETE /api/tasks/{id}`

- `204 No Content` → removida com sucesso

---

## Equipe

### `GET /api/team-members`

Lista todos os membros da equipe cadastrados (usados como `ownerId`/`assigneeId` em projetos e
tarefas).

**Resposta `200 OK`:**

```json
[
  { "id": "guid", "name": "string", "role": "string", "email": "string", "initials": "string" }
]
```

---

## Formato de erro (`ProblemDetails`)

Toda resposta de erro segue este formato padronizado:

```json
{
  "status": 409,
  "title": "Conflito de regra",
  "detail": "Já existe um projeto com esse nome.",
  "instance": "/api/projects",
  "traceId": "0HN..."
}
```

| Status | Título | Quando ocorre |
|--------|--------|----------------|
| 400 | Dados inválidos | Falha de validação (ex: datas inconsistentes, integrante inexistente) |
| 404 | Recurso não encontrado | Projeto ou tarefa não encontrados |
| 409 | Conflito de regra | Nome de projeto duplicado; exclusão de projeto com tarefas |
| 500 | Erro interno | Falha não tratada pela API |
