# T2 — MS de Competências

## Visão Geral

Na segunda entrega (T2), desenvolvemos o **microsserviço de Competências** do sistema **Chave — Sistema de Autoavaliação de Competências para Pessoas Idosas**, integrado ao MS de Autenticação eleito na T1.

O serviço é o **catálogo central** do sistema: define as competências avaliáveis, suas subcompetências e as faixas de nível (por score) usadas na autoavaliação. Os demais microsserviços (Avaliação, Resultados, Perfil) consomem esse catálogo.

---

## Modelo de Domínio

| Entidade | Descrição |
|---|---|
| **Competency** | Competência avaliável (id, name, description). |
| **CompetencyLevel** | Faixa de nível da competência por score (lowerScore, upperScore, orderIndex). |
| **SubCompetency** | Subcompetência pertencente a uma competência. |
| **SubCompetencyLevel** | Faixa de nível da subcompetência por score. |

Relações:

- `Competency` **1—N** `CompetencyLevel`
- `Competency` **1—N** `SubCompetency`
- `SubCompetency` **1—N** `SubCompetencyLevel`

---

## API

O contrato completo está documentado em OpenAPI 3.0:

[:lucide-file-code: openapi.yaml](openapi.yaml){ .btn .btn-default }

Para visualizar como Swagger UI, cole o conteúdo em [editor.swagger.io](https://editor.swagger.io).

### Rotas principais

| Recurso | Rotas |
|---|---|
| Competency | `GET/POST /competencies`, `GET/PUT/DELETE /competencies/{id}` |
| CompetencyLevel | `GET/POST /competencies/{competencyId}/levels`, `GET/PUT/DELETE /competency-levels/{id}` |
| SubCompetency | `GET/POST /competencies/{competencyId}/sub-competencies`, `GET/PUT/DELETE /sub-competencies/{id}` |
| SubCompetencyLevel | `GET/POST /sub-competencies/{subCompetencyId}/levels`, `GET/PUT/DELETE /sub-competency-levels/{id}` |
