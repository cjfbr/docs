# ADR — Documento de Decisão de Arquitetura
### Chave — Sistema de Autoavaliação de Competências para Pessoas Idosas

| Campo | Valor |
|---|---|
| **Projeto** | Chave — Sistema de autoavaliação de competências para pessoas idosas |
| **Disciplina** | Engenharia de Software II |
| **Turma** | 31 — PUCRS / 2026-1 |
| **Professor** | Prof. José Pedro Schardosim Simão |
| **Parceira** | Letícia Sophia Rocha Machado (UFRGS) |
| **Versão / Data** | v2.0 — 21 de junho de 2026 |
| **Status** | ✔ Aceita (reflete a implementação entregue) |
| **Escopo** | Fatia ponta-a-ponta: Autenticação + Competências |

---

## 1. Contexto e Problema

O **Chave** permite que pessoas idosas avaliem suas próprias competências e acompanhem sua evolução ao longo do tempo. O domínio é modelado pelo framework **CHA (Conhecimentos, Habilidades e Atitudes)**: cada competência se desdobra em sub-competências, e cada uma possui **níveis pontuados** (faixas de score) que servem de régua para a autoavaliação.

Este ADR documenta as decisões da **fatia vertical efetivamente construída** pela equipe — não o plano genérico da turma. A fatia cobre toda a cadeia de dependências de uma funcionalidade real: o usuário se autentica, recebe um JWT, e um administrador gerencia o catálogo de competências por uma interface acessível, com a escrita protegida por papel (role).

As principais forças que moldaram a arquitetura foram:

- **Desenvolvimento paralelo e independente** entre os serviços de Autenticação e de Competências.
- **Acoplamento mínimo em tempo de execução** entre serviços de domínio e o serviço de identidade.
- **Acessibilidade** da interface destinada a pessoas idosas.
- **Custo zero de infraestrutura** durante o desenvolvimento (sem depender de uma conta AWS real).

> **Nota de honestidade arquitetural:** o template da turma previa 5 microsserviços de domínio e comunicação assíncrona via SQS/SNS. A equipe **não** implementou mensageria assíncrona porque nenhum fluxo entre serviços a exige neste recorte (ver §4.4 e §8). Este ADR registra o que **de fato** existe.

---

## 2. Decisão Arquitetural

Adotar **microsserviços com microfrontends**, compostos por unidades deployáveis independentes, integradas em tempo de execução. Cada serviço de domínio é dono do seu banco e do seu ciclo de vida; o frontend é composto dinamicamente por um *Shell* host.

> **Decisão Central:** 2 microsserviços (**Auth** + **Competências**) **poliglotas**, cada um com banco PostgreSQL dedicado; 2 microfrontends React integrados ao **Shell** via **Module Federation (Vite)**; autenticação **stateless** nos serviços de domínio via **JWT HS256 com segredo simétrico compartilhado**, sem chamada de rede ao Auth a cada requisição. Infra local: **Docker Compose + Ministack + Terraform**.

---

## 3. Componentes Implementados

| Repositório | Papel | Stack | Porta local |
|---|---|---|---|
| `chave-shell` | Host de microfrontends; roteamento, sessão, *route guards* | React 18 + Vite + `@originjs/vite-plugin-federation` | 3000 |
| `chave-mfe-auth` | MFE de login, cadastro e dashboard | React + TypeScript + MUI + Vite | 4001 |
| `chave-mfe-competency` | MFE de gestão do catálogo de competências | React 18 + TypeScript + MUI + Vite | 4002 |
| `chave-ms-auth` | Identidade: login, JWT, papéis, revogação | Node 20 + Express 5 + TypeScript + TypeORM | 3001 |
| `competency` (`chave-ms-competency`) | Domínio CHA: competências, sub-competências e níveis | Java 25 + Spring Boot 4 + Spring Data JPA + Flyway | 8080 |
| `chave-infra` | Orquestração local e provisionamento AWS emulado | Docker Compose + Ministack + Terraform | 4566 |

---

## 4. Visão da Arquitetura e Dependências

### 4.1 Composição do Frontend (Shell ↔ MFEs)

O navegador carrega o **Shell**, que consome os `remoteEntry.js` expostos por cada MFE via **Module Federation** e os carrega *lazy* (`React.lazy` + `Suspense`). `react` e `react-dom` são declarados como `shared` para evitar instâncias duplicadas. O Shell concentra o roteamento (`react-router`), o *guard* de rotas privadas (`PrivateRoute`) e a sessão (token em `localStorage`), repassando *callbacks* (`onLogin`, `onLogout`, `onHome`) aos MFEs — que permanecem desacoplados da estratégia de navegação.

### 4.2 Identidade (Auth) e Confiança entre Serviços

O `ms-auth` é a **única fonte de identidade**: valida credenciais (bcrypt), emite **JWT HS256** (validade de 4 h) com os claims `email`, `idUser` e `roles`, e oferece *rate limiting*. O logout é **stateful**: o token revogado entra numa *blocklist* (`RevokedToken`) no banco do Auth.

O `ms-competency` **confia no JWT sem chamar o Auth**: valida a assinatura e a expiração localmente. Essa decisão elimina o acoplamento temporal — o domínio de competências continua respondendo leituras mesmo se o Auth estiver fora do ar.

### 4.3 Dependência crítica: o segredo JWT compartilhado

Auth e Competências compartilham o **mesmo `JWT_SECRET`** (injetado por variável de ambiente). O Auth **assina**; o Competências **verifica**. A verificação no lado Java é feita **manualmente** (HMAC-SHA256 sobre `header.payload` usando os bytes UTF-8 crus do segredo) — ver §6. Os papéis do claim `roles` viram *authorities* (`ROLE_*`) usadas na autorização.

### 4.4 Comunicação entre serviços hoje

| De → Para | Mecanismo | Observação |
|---|---|---|
| Navegador → `ms-auth` | HTTP via **API Gateway** (Ministack) | Rotas `/auth/login`, `/auth/refresh`, `/auth/logout`, `/auth/me` provisionadas por Terraform (`HTTP_PROXY`) |
| Navegador → `ms-competency` | HTTP **direto**, liberado por **CORS** | Origens permitidas (shell + MFE) configuráveis por env |
| `ms-competency` → `ms-auth` | **Nenhuma** (validação local do JWT) | Acoplamento de runtime evitado por design |

> Não há mensageria assíncrona (SQS/SNS) porque **nenhum fluxo de negócio atual cruza serviços**. Introduzir um *broker* agora seria complexidade sem demanda (YAGNI). Quando surgir um fluxo orientado a eventos (ex.: "avaliação respondida → gerar relatório"), o ADR será revisado.

---

## 5. Modelo de Domínio (Competências)

O domínio CHA é mapeado em quatro entidades, com integridade garantida no banco (Flyway, `V1__create_competency_model.sql`):

```
Competency
 ├── SubCompetency        (1:N)
 │     └── SubCompetencyLevel  (1:N)
 └── CompetencyLevel      (1:N)
```

- **Competency / SubCompetency:** `name`, `description`.
- **CompetencyLevel / SubCompetencyLevel:** `name`, `orderIndex`, faixa de pontuação (`lower_score`, `upper_score`) e `description`.
- **Invariantes no banco:** `order_index` e `name` **únicos por pai**; exclusão **em cascata** (remover uma competência remove sub-competências e níveis).

A faixa de score (`lower`/`upper`) materializa a régua de autoavaliação: a resposta do idoso cai numa faixa que corresponde a um nível nomeado, tornando o resultado interpretável.

---

## 6. Decisão em Destaque — Verificação Manual do JWT em Java

**Problema:** o `ms-auth` (Node, `jsonwebtoken`) assina HS256 com um segredo de tamanho arbitrário. Bibliotecas Java que seguem à risca a RFC 7518 exigem **mínimo de 256 bits** também na *verificação* e **rejeitariam** o segredo compartilhado.

**Decisão:** implementar a verificação HS256 manualmente no `JwtAuthenticationFilter` — HMAC-SHA256 sobre `header.payload` com os bytes crus do segredo, comparação em tempo constante (`MessageDigest.isEqual`) e checagem de `exp`. Token ausente/inválido **não autentica** (a cadeia responde 401/403); nunca derruba a requisição.

**Justificativa:** garante **interoperabilidade total** com o emissor Node sem enfraquecer a configuração do Auth, mantendo o serviço de domínio **stateless** e independente.

---

## 7. Segurança e Autorização (Competências)

Política do `SecurityConfig` (stateless, sem sessão/cookie):

| Recurso | Regra |
|---|---|
| Swagger / OpenAPI e *preflight* CORS (`OPTIONS`) | Público |
| Leitura (`GET`) | Qualquer usuário **autenticado** |
| Escrita (`POST/PUT/PATCH/DELETE`) | Apenas **roles de administrador** (configuráveis via `APP_SECURITY_ADMIN_ROLES`) |

Respostas de erro padronizadas: **401** (token ausente/inválido/expirado) e **403** (autenticado, mas sem privilégio). A escrita restrita a administradores reflete o domínio: o **catálogo de competências é curado**, enquanto a leitura é ampla para alimentar a autoavaliação.

---

## 8. Alternativas Consideradas

| Alternativa | Prós | Contras / Motivo da Rejeição |
|---|---|---|
| Validar o JWT chamando o Auth (introspecção) a cada request | Revogação imediata e centralizada | Acoplamento temporal: Competências cairia junto com o Auth; latência por request. **Rejeitada** em favor de validação local stateless |
| Biblioteca JWT Java "padrão" (jjwt/Nimbus) | Menos código próprio | Rejeita o segredo curto do Node (RFC 7518) → incompatível com o emissor. **Rejeitada** (ver §6) |
| Stack única (só Node ou só Java) | Menos diversidade de ferramentas | Perde o aprendizado poliglota e a autonomia de equipe; **rejeitada** — cada serviço usa a stack mais adequada |
| Banco único compartilhado | Sem duplicação de infra | Quebra o isolamento e o deploy independente. **Rejeitada** em favor de *database-per-service* |
| MFE via iframes | Isolamento total | UX degradada, sem compartilhar `react`/estado. **Rejeitada** em favor de Module Federation |
| Mensageria SQS/SNS desde já | Pronto para eventos futuros | Complexidade sem caso de uso atual (YAGNI). **Adiada** até existir fluxo cross-service |
| Auth externo (Auth0/Keycloak) | Pronto para produção | Não cumpre o objetivo pedagógico de implementar o Auth. **Rejeitada** |

---

## 9. Consequências

### 9.1 Positivas

- **Independência de runtime:** Competências serve leituras mesmo com o Auth indisponível.
- **Autonomia de equipe:** backend poliglota (Node + Java) e *database-per-service* permitem evolução paralela.
- **Custo zero em dev:** Ministack + containers PostgreSQL reais, sem conta AWS.
- **Interoperabilidade verificada:** o contrato de identidade (claims + segredo HS256) é honrado por emissor Node e verificador Java.
- **Acessibilidade isolável:** o MFE de competências evolui sem impactar outros serviços.

### 9.2 Negativas / Riscos

- **Janela de revogação no domínio:** Competências não consulta a *blocklist*; um token revogado continua válido até `exp` (≤ 4 h). Aceitável para dados de catálogo curado e mitigado pela vida curta do token.
- **Segredo simétrico compartilhado:** vazar o `JWT_SECRET` compromete a confiança entre serviços; exige gestão cuidadosa de segredos.
- **Acesso direto via CORS:** Competências é exposto fora do API Gateway; consolidar tudo atrás do gateway é trabalho futuro.
- **Estratégias de schema divergentes:** Flyway (Competências) vs. `synchronize` do TypeORM (Auth) — padronizar para migrações versionadas é desejável.
- **Sem observabilidade central:** ainda não há tracing/log agregado entre serviços.

---

## 10. Infraestrutura Local

`chave-infra` orquestra tudo via Docker Compose: **Ministack** (apenas `s3`, `apigateway`, `sts`), **dois containers PostgreSQL** (`chave_auth` e `chave_auth_competencia` — *database-per-service*), um **provisionador Terraform** (`infra-provisioner`) que cria o bucket S3 e as rotas do API Gateway antes de liberar os serviços, e os cinco serviços da fatia. `make setup` sobe a stack completa; `make reset` recria do zero.

---

## 11. Histórico de Revisões

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | Abr/2026 | Prof. | Template genérico da turma (5 MSs + SQS/SNS) |
| v2.0 | 21/06/2026 | Equipe (Turma 31) | ADR reescrito para refletir a implementação real: Auth + Competências, backend poliglota, JWT HS256 com segredo compartilhado e verificação manual em Java, *database-per-service*, Module Federation (Vite); mensageria assíncrona adiada |

---

*Decisões arquiteturais que desviem deste documento devem ser registradas como nova versão do ADR, com justificativa, data e autores.*
