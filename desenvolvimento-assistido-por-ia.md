# Desenvolvimento Assistido por IA

## Introdução

Este guia apresenta práticas e fluxos de trabalho para utilizar IA de forma efetiva no dia a dia do desenvolvimento de software.

---

## Mindset para Trabalhar com IA

### IA como Par de Programação

```
Desenvolvedor + IA = Pair Programming Assíncrono

┌──────────────────┐    ┌──────────────────┐
│   Desenvolvedor  │    │        IA        │
├──────────────────┤    ├──────────────────┤
│ - Decisões       │◄──►│ - Sugestões      │
│ - Contexto       │    │ - Geração        │
│ - Validação      │    │ - Análise        │
│ - Responsabilidade│   │ - Alternativas   │
└──────────────────┘    └──────────────────┘
```

### Princípios Fundamentais

1. **Você é o Piloto, IA é o Copiloto**
   - Decisões finais são suas
   - IA sugere, você valida
   - Responsabilidade pelo código é sua

2. **Confiança com Verificação**
   - Revise todo código gerado
   - Teste antes de usar
   - Entenda o que foi gerado

3. **Iteração > Perfeição Inicial**
   - Comece com algo funcional
   - Refine progressivamente
   - Peça melhorias específicas

4. **Contexto é Rei**
   - Quanto mais contexto, melhor resultado
   - Inclua restrições e padrões
   - Especifique stack e versões

---

## Fluxos de Trabalho Diários

### 1. Começando uma Nova Feature

```
┌─────────────────────────────────────────────────────────────┐
│                    FLUXO: NOVA FEATURE                       │
└─────────────────────────────────────────────────────────────┘

1. PLANEJAMENTO
   ├── Descreva a feature para IA
   ├── Peça análise de impacto
   ├── Solicite breakdown em tarefas
   └── Revise e ajuste o plano

2. DESIGN
   ├── Discuta arquitetura com IA
   ├── Peça alternativas de design
   ├── Valide com padrões existentes
   └── Documente decisões

3. IMPLEMENTAÇÃO
   ├── Gere estrutura inicial
   ├── Implemente por partes
   ├── Revise cada parte
   └── Integre e teste

4. REVISÃO
   ├── Peça code review à IA
   ├── Aplique correções
   ├── Gere testes
   └── Documente
```

#### Exemplo Prático

```markdown
**Prompt para Planejamento:**

Preciso implementar um sistema de notificações em tempo real no Laravel.

Contexto:
- Laravel 10, PHP 8.2
- Vue 3 no frontend
- PostgreSQL
- Já temos Redis configurado

Requisitos:
- Notificações in-app (badge, dropdown)
- Push notifications (opcional)
- Marcar como lido
- Preferências do usuário

Por favor:
1. Analise o impacto dessa feature
2. Sugira a arquitetura (Events, Listeners, Broadcasting)
3. Quebre em tarefas menores
4. Identifique riscos e dependências
```

### 2. Debugging com IA

```
┌─────────────────────────────────────────────────────────────┐
│                    FLUXO: DEBUGGING                          │
└─────────────────────────────────────────────────────────────┘

1. COLETA DE INFORMAÇÕES
   ├── Erro exato (mensagem, stack trace)
   ├── Código relevante
   ├── Passos para reproduzir
   └── O que já tentou

2. ANÁLISE
   ├── Compartilhe com IA
   ├── Peça hipóteses ordenadas
   ├── Solicite formas de verificar cada uma
   └── Discuta findings

3. CORREÇÃO
   ├── Implemente fix sugerido
   ├── Teste localmente
   ├── Peça review do fix
   └── Adicione teste de regressão

4. PREVENÇÃO
   ├── Pergunte como evitar no futuro
   ├── Documente learnings
   └── Atualize padrões se necessário
```

#### Template de Debug

```markdown
**Contexto:**
- Laravel 10, PHP 8.2, PostgreSQL 15
- Ambiente: local/staging/production

**Erro:**
```
[Cole a mensagem de erro e stack trace aqui]
```

**Código Relevante:**
```php
[Cole o código onde o erro ocorre]
```

**Comportamento Esperado:**
[Descreva o que deveria acontecer]

**Comportamento Atual:**
[Descreva o que está acontecendo]

**O que já tentei:**
1. [Tentativa 1]
2. [Tentativa 2]

**Perguntas:**
1. Qual a causa raiz mais provável?
2. Como posso confirmar essa hipótese?
3. Qual a correção recomendada?
```

### 3. Code Review Assistido

```
┌─────────────────────────────────────────────────────────────┐
│                  FLUXO: CODE REVIEW                          │
└─────────────────────────────────────────────────────────────┘

ANTES DO PR
├── Auto-review com IA
├── Corrija issues óbvios
├── Melhore documentação
└── Verifique testes

DURANTE REVIEW
├── Cole código para análise
├── Peça foco em áreas específicas
├── Discuta trade-offs
└── Solicite sugestões de melhoria

APÓS FEEDBACK
├── Analise comentários com IA
├── Discuta soluções
├── Implemente correções
└── Valide mudanças
```

#### Prompt de Code Review

```markdown
Revise o seguinte código considerando:

**Stack:** Laravel 10, PHP 8.2
**Tipo de mudança:** Nova feature / Bug fix / Refactoring

**Foco da revisão:**
- [ ] Segurança
- [ ] Performance
- [ ] Manutenibilidade
- [ ] Testes
- [ ] Padrões do projeto

**Código:**
```php
[Cole o código aqui]
```

**Contexto adicional:**
[Explique o propósito do código]

Por favor, identifique:
1. Problemas críticos (bugs, segurança)
2. Melhorias importantes
3. Sugestões de refinamento
4. Pontos positivos
```

### 4. Refactoring Guiado

```
┌─────────────────────────────────────────────────────────────┐
│                  FLUXO: REFACTORING                          │
└─────────────────────────────────────────────────────────────┘

1. ANÁLISE
   ├── Identifique code smells
   ├── Peça análise à IA
   ├── Priorize problemas
   └── Defina escopo

2. PLANEJAMENTO
   ├── Estratégia de refactoring
   ├── Ordem de mudanças
   ├── Pontos de parada
   └── Plano de rollback

3. EXECUÇÃO
   ├── Mudanças pequenas e incrementais
   ├── Teste após cada mudança
   ├── Review intermediário
   └── Commit frequente

4. VALIDAÇÃO
   ├── Testes passando
   ├── Review final
   ├── Documentação atualizada
   └── Métricas (se aplicável)
```

---

## Casos de Uso Específicos

### Gerando Código Boilerplate

```markdown
**Prompt:**

Gere a estrutura completa para um CRUD de Products no Laravel:

**Especificações:**
- Model: Product (name, description, price, category_id, active)
- Relacionamento: belongsTo Category
- Soft deletes
- Validação via Form Request
- Resource para API
- Policy para autorização
- Testes com Pest

**Padrões:**
- Use DTOs para transferência
- Service layer para lógica
- Repository pattern

**Formato:**
Gere cada arquivo separadamente, indicando o path.
```

### Escrevendo Testes

```markdown
**Prompt:**

Gere testes abrangentes para a seguinte classe:

```php
class OrderService
{
    public function createOrder(CreateOrderDTO $dto): Order
    {
        // lógica aqui
    }

    public function cancelOrder(Order $order, string $reason): void
    {
        // lógica aqui
    }
}
```

**Requisitos:**
- Framework: Pest PHP
- Cobrir cenários de sucesso e falha
- Testar edge cases
- Usar factories e mocks apropriadamente
- Incluir testes de integração relevantes

**Cenários importantes:**
- Pedido com estoque insuficiente
- Cancelamento após envio
- Usuário sem permissão
```

### Documentando Código

```markdown
**Prompt:**

Documente a seguinte classe seguindo os padrões:

1. Docblock de classe com descrição
2. Docblocks de métodos com @param, @return, @throws
3. Comentários inline onde a lógica não é óbvia
4. README de uso se for uma classe pública

```php
[Cole a classe aqui]
```

**Formato:**
Retorne a classe documentada e um exemplo de uso.
```

### Migrando Código Legacy

```markdown
**Prompt:**

Ajude a modernizar este código PHP legado:

**Código atual (PHP 5.6):**
```php
[Cole o código legado]
```

**Target:**
- PHP 8.2 com strict types
- Tipagem completa
- Padrões SOLID
- Sem globals ou funções soltas
- Tratamento de erros moderno

**Restrições:**
- Manter compatibilidade de comportamento
- Mudanças devem ser incrementais
- Priorize legibilidade
```

---

## Ferramentas e Setup

### IDEs com IA

| Ferramenta | Tipo | Melhor Para |
|------------|------|-------------|
| GitHub Copilot | Autocomplete | Código inline, sugestões |
| Cursor | IDE completa | Chat + código integrado |
| Codeium | Autocomplete | Alternativa gratuita |
| Continue | Plugin VSCode | Chat local/remoto |
| Aider | Terminal | Edição de arquivos |

### Configuração Recomendada

```
Ambiente de Desenvolvimento Ideal
├── IDE com IA (Cursor/Copilot)
├── Chat separado (Claude/ChatGPT) para discussões
├── Terminal com Aider para edições batch
└── Scripts de automação próprios
```

### Prompts Salvos (Snippets)

Crie snippets para prompts frequentes:

```json
// .vscode/prompts.code-snippets
{
    "Laravel Code Review": {
        "prefix": "!review",
        "body": [
            "Revise este código Laravel 10:",
            "",
            "```php",
            "$1",
            "```",
            "",
            "Foco: segurança, performance, padrões Laravel.",
            "Formato: lista de issues com severidade e correção."
        ]
    },
    "Generate Tests": {
        "prefix": "!tests",
        "body": [
            "Gere testes Pest para:",
            "",
            "```php",
            "$1",
            "```",
            "",
            "Inclua: sucesso, falha, edge cases, mocks."
        ]
    }
}
```

---

## Melhores Práticas

### DO (Faça)

1. **Forneça Contexto Completo**
   ```markdown
   ✅ Bom:
   "No Laravel 10, usando PostgreSQL, preciso criar uma query
   que busque usuários ativos com mais de 3 pedidos no último mês,
   ordenados por total gasto. A tabela orders tem user_id e total_amount."
   ```

2. **Seja Específico sobre Output**
   ```markdown
   ✅ Bom:
   "Retorne apenas o código PHP, sem explicações.
   Use tipagem estrita e docblocks completos."
   ```

3. **Itere e Refine**
   ```markdown
   ✅ Bom:
   "O código funciona, mas preciso que:
   1. Use cache para o resultado
   2. Adicione logging
   3. Trate o caso de tabela vazia"
   ```

4. **Valide Sempre**
   - Execute o código
   - Rode os testes
   - Revise a lógica
   - Verifique segurança

### DON'T (Evite)

1. **Prompts Vagos**
   ```markdown
   ❌ Ruim:
   "Faz um código pra mim"
   ```

2. **Aceitar sem Revisar**
   ```markdown
   ❌ Ruim:
   Copiar e colar código direto na produção
   ```

3. **Ignorar Contexto**
   ```markdown
   ❌ Ruim:
   Usar código gerado para Python num projeto PHP
   ```

4. **Dados Sensíveis**
   ```markdown
   ❌ Ruim:
   Compartilhar credenciais, dados de clientes, ou código proprietário sensível
   ```

---

## Limitações e Quando NÃO Usar IA

### Limitações Conhecidas

| Área | Limitação | Mitigação |
|------|-----------|-----------|
| **Conhecimento** | Cutoff date | Verifique docs oficiais |
| **Contexto** | Não conhece seu projeto | Forneça contexto |
| **Atualidade** | Versões desatualizadas | Especifique versões |
| **Lógica complexa** | Pode errar cálculos | Valide com testes |
| **Segurança** | Pode gerar código inseguro | Review de segurança |

### Quando Preferir Abordagem Manual

1. **Decisões Arquiteturais Críticas**
   - Use IA para explorar opções
   - Decisão final requer julgamento humano

2. **Código de Segurança Sensível**
   - Autenticação, criptografia
   - Revise com especialista

3. **Performance Crítica**
   - Benchmarks reais > sugestões teóricas
   - Profile antes de otimizar

4. **Integrações Complexas**
   - APIs mudam frequentemente
   - Consulte documentação oficial

---

## Exercícios Práticos

### Exercício 1: Debug Assistido

```markdown
**Cenário:**
Sua aplicação Laravel está lenta. As páginas demoram 3-5 segundos.

**Tarefa:**
1. Use IA para criar um plano de investigação
2. Peça queries para identificar N+1
3. Solicite otimizações específicas
4. Implemente e meça resultados

**Dica de Prompt:**
"Minha aplicação Laravel está lenta. Quero investigar problemas de N+1.
Me ajude a: 1) Listar lugares para verificar 2) Query para encontrar
N+1 com Laravel Debugbar/Telescope 3) Padrões de correção."
```

### Exercício 2: Refactoring de Controller

```markdown
**Cenário:**
Controller com 500+ linhas, múltiplas responsabilidades.

**Tarefa:**
1. Peça análise do código
2. Solicite estratégia de refactoring
3. Implemente passo a passo
4. Mantenha testes passando

**Dica de Prompt:**
"Este controller tem muitas responsabilidades. Ajude-me a:
1) Identificar responsabilidades distintas
2) Sugerir extração para Services/Actions
3) Ordem de refactoring (menor risco primeiro)
4) Garantir que não quebre funcionalidade"
```

### Exercício 3: Geração de Testes

```markdown
**Cenário:**
Classe de domínio sem testes.

**Tarefa:**
1. Analise a classe com IA
2. Peça identificação de casos de teste
3. Gere testes com Pest
4. Execute e refine

**Dica de Prompt:**
"Analise esta classe e identifique TODOS os cenários testáveis:
- Happy paths
- Edge cases
- Error conditions
- Boundary values
Depois gere os testes usando Pest PHP."
```

---

## Métricas de Efetividade

### O que Medir

| Métrica | Como Medir | Meta |
|---------|------------|------|
| **Tempo de desenvolvimento** | Tracking de tarefas | -20-30% |
| **Qualidade do código** | Code review, bugs | Manter ou melhorar |
| **Cobertura de testes** | Coverage reports | +10-20% |
| **Satisfação do dev** | Self-assessment | Positivo |
| **Bugs introduzidos** | Bug tracking | Não aumentar |

### Auto-avaliação Semanal

```markdown
## Retrospectiva Semanal - Uso de IA

### O que funcionou bem?
- [Liste usos efetivos de IA]

### O que não funcionou?
- [Liste tentativas falhas]

### Aprendizados
- [O que aprendi sobre prompts efetivos?]

### Próxima semana
- [O que vou tentar diferente?]
```

---

## Conclusão

### Checklist de Desenvolvimento com IA

```
Antes de usar IA:
□ Tenho contexto claro para fornecer?
□ Sei o que quero como resultado?
□ É uma tarefa apropriada para IA?

Durante o uso:
□ Forneci contexto suficiente?
□ Especifiquei formato de output?
□ Estou iterando e refinando?

Após usar:
□ Revisei o código gerado?
□ Testei localmente?
□ Entendo o que foi gerado?
□ Está alinhado com padrões do projeto?
```

### Evolução Contínua

1. **Mantenha um log** de prompts efetivos
2. **Compartilhe** aprendizados com o time
3. **Experimente** novas ferramentas
4. **Revise** seu workflow regularmente
5. **Adapte** conforme a tecnologia evolui

---

*A IA é uma ferramenta poderosa, mas você continua sendo o engenheiro responsável pelo software.*
