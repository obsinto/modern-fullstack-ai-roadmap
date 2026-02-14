# Roadmap de Estudos: Desenvolvedor Full Stack + IA

Este roadmap organiza o conteúdo de estudo em uma sequência lógica para construir uma base sólida em desenvolvimento web moderno e trabalho efetivo com IA.

---

## Visão Geral

```
┌────────────────────────────────────────────────────────────────────────┐
│                        JORNADA DE APRENDIZADO                          │
└────────────────────────────────────────────────────────────────────────┘

FASE 1: FUNDAMENTOS DA STACK
├── PHP 8.x Moderno
├── JavaScript Moderno (ES6+)
├── HTML, CSS e Tailwind CSS
└── Decisões Arquiteturais

FASE 2: ARQUITETURA E BACKEND
├── Lógica de Negócio (Services/Actions)
├── Fluxo de Dados e Estado
└── Design de APIs

FASE 3: FRONTEND E INTERATIVIDADE
├── Vue 3 (Composition API)
├── Inertia.js
├── Gerenciamento de Estado (Pinia)
└── Comunicação Real-time (Reverb/Echo)

FASE 4: QUALIDADE E SEGURANÇA
├── Estratégia de Testes
├── Prevenção de Bagunça (SOLID/Patterns)
└── Segurança e Permissões

FASE 5: OPERAÇÕES E DEVOPS
├── Docker e Containerização
├── CI/CD e Deployment
└── Observabilidade e Logs

FASE 6: INTELIGÊNCIA ARTIFICIAL
├── Fundamentos de IA
├── Engenharia de Prompts
├── Integração de LLMs
└── Desenvolvimento Assistido

              ↓
        PROFICIÊNCIA
```

---

## Fase 1: Fundamentos da Stack

### 1.1 PHP Moderno (8.x)
**Arquivo:** `fundamentos-php-moderno.md`
- [ ] Strict Types e Union Types
- [ ] Enums e Readonly Properties
- [ ] Constructor Promotion
- [ ] Match Expression

### 1.2 JavaScript Moderno (ES6+)
**Arquivo:** `fundamentos-javascript-moderno.md`
- [ ] Let/Const e Arrow Functions
- [ ] Destructuring e Spread Operator
- [ ] Promises e Async/Await
- [ ] Módulos (ESM)

### 1.3 Decisões Arquiteturais
**Arquivo:** `architecture-decisions.md`
- [ ] Documentação (ADRs)
- [ ] Trade-offs e Padrões

---

## Fase 2: Arquitetura e Backend

### 2.1 Organização de Lógica de Negócio
**Arquivo:** `business-logic.md`
- [ ] Services, Actions e DTOs
- [ ] Domain-Driven Design (DDD) básico

### 2.2 Fluxo de Dados e Estado
**Arquivo:** `state-data-flow.md`
- [ ] Estado Local vs Global
- [ ] Ciclo de vida do Request

### 2.3 Design de APIs
**Arquivo:** `apis-system-boundaries.md`
- [ ] RESTful Design e Versionamento
- [ ] Rate Limiting e Documentação

---

## Fase 3: Frontend Moderno

### 3.1 Vue 3 e Inertia.js
**Arquivo:** `fundamentos-vue-inertia.md`
- [ ] Composition API (`<script setup>`)
- [ ] Protocolo Inertia e Form Helpers
- [ ] Gerenciamento de Estado com Pinia
- [ ] Persistent Layouts

---

## Fase 4: Qualidade e Segurança

### 4.1 Estratégia de Testes
**Arquivo:** `automated-testing-strategy.md`
- [ ] Pest e PHPUnit
- [ ] TDD e Pirâmide de Testes

### 4.2 Prevenção de Bagunça
**Arquivo:** `preventing-messes.md`
- [ ] SOLID e Design Patterns
- [ ] Análise Estática (Larastan)

### 4.3 Segurança e Permissões
**Arquivo:** `application-security-and-roles.md`
- [ ] Policies, Gates e RBAC
- [ ] OWASP Top 10

---

## Fase 5: Operações e DevOps

### 5.1 DevOps e Observabilidade
**Arquivo:** `devops-observabilidade.md`
- [ ] Docker (Sail e Produção)
- [ ] CI/CD com GitHub Actions
- [ ] Monitoramento (Pulse, Sentry)
- [ ] Logs Estruturados

---

## Fase 6: Inteligência Artificial

### 6.1 Fundamentos de IA
**Arquivo:** `fundamentos-ia-para-devs.md`
- [ ] LLMs, Tokens e Contexto
- [ ] Embeddings e RAG

### 6.2 Engenharia de Prompts
**Arquivo:** `engenharia-de-prompts.md`
- [ ] CRISP Framework
- [ ] Chain of Thought e ReAct

### 6.3 Integração e Desenvolvimento Assistido
**Arquivos:** `integracao-llms-aplicacoes.md` e `desenvolvimento-assistido-por-ia.md`
- [ ] Function Calling
- [ ] IA como Pair Programmer

---

## Plano de Estudo Sugerido (10 Semanas)

| Semanas | Foco | Arquivos Principais |
|---------|------|---------------------|
| 1-2 | Fundamentos | fundamentos-php, fundamentos-javascript, architecture-decisions |
| 3-4 | Backend | business-logic, state-data-flow, apis-system-boundaries |
| 5 | Frontend | fundamentos-vue-inertia |
| 6-7 | Qualidade | automated-testing-strategy, preventing-messes, application-security |
| 8 | DevOps | devops-observabilidade |
| 9-10 | IA | Todos os arquivos de IA |

---

## Tracking de Progresso

### Meu Progresso

```markdown
## Fase 1 & 2: Base e Backend
- [ ] fundamentos-php-moderno.md
- [ ] fundamentos-javascript-moderno.md
- [ ] fundamentos-html-css-tailwind.md
- [ ] architecture-decisions.md
- [ ] business-logic.md
- [ ] state-data-flow.md
- [ ] apis-system-boundaries.md

## Fase 3: Frontend e Interatividade
- [ ] fundamentos-vue-inertia.md
- [ ] comunicacao-real-time.md

## Fase 4 & 5: Qualidade e Operações
- [ ] automated-testing-strategy.md
- [ ] preventing-messes.md
- [ ] application-security-and-roles.md
- [ ] devops-observabilidade.md

## Fase 6: Inteligência Artificial
- [ ] fundamentos-ia-para-devs.md
- [ ] engenharia-de-prompts.md
- [ ] integracao-llms-aplicacoes.md
- [ ] desenvolvimento-assistido-por-ia.md
```

---

## Ordem de Leitura Recomendada

1. `fundamentos-php-moderno.md`
2. `fundamentos-javascript-moderno.md`
3. `fundamentos-html-css-tailwind.md`
4. `architecture-decisions.md`
5. `business-logic.md`
5. `state-data-flow.md`
6. `fundamentos-vue-inertia.md`
7. `apis-system-boundaries.md`
8. `automated-testing-strategy.md`
9. `preventing-messes.md`
10. `application-security-and-roles.md`
11. `devops-observabilidade.md`
12. `fundamentos-ia-para-devs.md`
13. `engenharia-de-prompts.md`
14. `integracao-llms-aplicacoes.md`
15. `desenvolvimento-assistido-por-ia.md`

---

*Boa jornada de aprendizado! Lembre-se: consistência é mais importante que intensidade.*
