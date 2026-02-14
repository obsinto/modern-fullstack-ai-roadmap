# Engenharia de Prompts

## O que é Engenharia de Prompts?

Engenharia de Prompts é a arte e ciência de projetar instruções efetivas para obter os melhores resultados de modelos de linguagem (LLMs).

```
Prompt ruim: "escreve um código"
Prompt bom: "Escreva uma função PHP que valide CPF, retornando true/false, com testes unitários"
```

---

## Anatomia de um Prompt Efetivo

### Estrutura CRISP

```
C - Contexto: Quem você é, qual o cenário
R - Role: Papel que o modelo deve assumir
I - Instrução: O que fazer
S - Specifics: Detalhes, formato, restrições
P - Proof/Examples: Exemplos do esperado
```

### Exemplo Completo

```php
$systemPrompt = <<<PROMPT
# Contexto
Você é um assistente de code review em uma empresa que usa Laravel 10 com PHP 8.2.

# Role
Atue como um desenvolvedor senior focado em qualidade de código.

# Instrução
Analise o código fornecido e identifique problemas.

# Specifics
- Foco em: segurança, performance, legibilidade, padrões Laravel
- Formato de resposta: lista de issues com severidade (alta/média/baixa)
- Sugira correções com código

# Exemplo de Output
## Issues Encontradas

### [ALTA] SQL Injection em UserController:42
**Problema**: Query concatenada com input do usuário
**Correção**:
```php
// Antes
\$users = DB::select("SELECT * FROM users WHERE name = '\$name'");

// Depois
\$users = DB::select("SELECT * FROM users WHERE name = ?", [\$name]);
```
PROMPT;
```

---

## Técnicas Fundamentais

### 1. Zero-Shot Prompting

Instrução direta sem exemplos:

```
Classifique o sentimento do texto como positivo, negativo ou neutro:

"O produto chegou antes do prazo e funciona perfeitamente!"

Sentimento:
```

**Quando usar**: Tarefas simples e bem definidas.

### 2. Few-Shot Prompting

Fornecer exemplos antes da tarefa:

```
Extraia entidades do texto no formato JSON.

Texto: "João Silva comprou 3 iPhones na loja de São Paulo"
JSON: {"pessoa": "João Silva", "quantidade": 3, "produto": "iPhone", "local": "São Paulo"}

Texto: "Maria Santos vendeu 10 notebooks em Curitiba"
JSON: {"pessoa": "Maria Santos", "quantidade": 10, "produto": "notebook", "local": "Curitiba"}

Texto: "Pedro Oliveira devolveu 2 TVs no Rio de Janeiro"
JSON:
```

**Quando usar**: Tarefas com formato específico ou domínio especializado.

### 3. Chain of Thought (CoT)

Instrua o modelo a "pensar passo a passo":

```
Resolva o problema mostrando seu raciocínio passo a passo:

Um e-commerce tem 1000 usuários. 30% fizeram compras no último mês.
Desses que compraram, 20% usaram cupom de desconto.
Quantos usuários usaram cupom?

Pensando passo a passo:
1. Total de usuários: 1000
2. Usuários que compraram (30%): 1000 × 0.30 = 300
3. Desses, usaram cupom (20%): 300 × 0.20 = 60

Resposta: 60 usuários usaram cupom.
```

**Quando usar**: Problemas matemáticos, lógica complexa, debugging.

### 4. Self-Consistency

Execute múltiplas vezes e escolha a resposta mais consistente:

```php
class SelfConsistencyResolver
{
    public function solve(string $prompt, int $samples = 5): string
    {
        $responses = [];

        for ($i = 0; $i < $samples; $i++) {
            $response = $this->llm->complete($prompt, temperature: 0.7);
            $answer = $this->extractAnswer($response);
            $responses[$answer] = ($responses[$answer] ?? 0) + 1;
        }

        // Retorna resposta mais frequente
        arsort($responses);
        return array_key_first($responses);
    }
}
```

### 5. ReAct (Reasoning + Acting)

Combine raciocínio com ações:

```
Pergunta: Qual a população atual do Brasil?

Pensamento: Preciso buscar dados atualizados sobre população.
Ação: search("população Brasil 2024")
Observação: Segundo IBGE, população estimada é 203 milhões.

Pensamento: Encontrei a informação atualizada.
Resposta: A população do Brasil é aproximadamente 203 milhões de habitantes.
```

---

## Padrões de Prompt para Desenvolvimento

### 1. Code Generation

```php
$prompt = <<<PROMPT
Crie uma classe PHP 8.2 com as seguintes especificações:

## Requisitos
- Nome: MoneyValueObject
- Imutável (readonly)
- Propriedades: amount (int em centavos), currency (string)
- Métodos: add, subtract, multiply, format
- Deve lançar InvalidArgumentException para operações inválidas

## Padrões
- Use strict types
- Docblocks completos
- Named arguments onde apropriado

## Formato
Retorne apenas o código PHP, sem explicações.
PROMPT;
```

### 2. Code Review

```php
$prompt = <<<PROMPT
Revise o código abaixo identificando:

1. **Bugs**: Erros que causarão falhas
2. **Segurança**: Vulnerabilidades (OWASP Top 10)
3. **Performance**: Problemas de eficiência
4. **Manutenibilidade**: Violações de SOLID, código confuso
5. **Laravel Best Practices**: Desvios dos padrões do framework

Para cada issue, forneça:
- Linha(s) afetada(s)
- Severidade (crítica/alta/média/baixa)
- Descrição do problema
- Código corrigido

```php
{$code}
```
PROMPT;
```

### 3. Bug Fixing

```php
$prompt = <<<PROMPT
## Contexto
Sistema Laravel 10, PHP 8.2, PostgreSQL 15

## Erro
{$errorMessage}

## Stack Trace
{$stackTrace}

## Código Relevante
```php
{$relevantCode}
```

## Tarefa
1. Identifique a causa raiz do erro
2. Explique por que está acontecendo
3. Forneça a correção com código
4. Sugira como prevenir erros similares
PROMPT;
```

### 4. Refactoring

```php
$prompt = <<<PROMPT
Refatore o código abaixo aplicando:

## Padrões Desejados
- Single Responsibility Principle
- Dependency Injection
- Value Objects para dados de domínio
- Early returns para reduzir aninhamento

## Restrições
- Manter compatibilidade com a interface pública
- Código deve funcionar em PHP 8.2
- Usar tipagem estrita

## Código Original
```php
{$originalCode}
```

## Output
Forneça:
1. Código refatorado
2. Lista das mudanças realizadas
3. Justificativa para cada mudança
PROMPT;
```

### 5. Test Generation

```php
$prompt = <<<PROMPT
Gere testes para a classe abaixo usando Pest PHP:

```php
{$classCode}
```

## Requisitos
- Cobertura de todos os métodos públicos
- Casos de sucesso e falha
- Edge cases relevantes
- Use datasets para testes parametrizados
- Mocks para dependências externas

## Formato
```php
<?php

uses(Tests\TestCase::class);

describe('NomeDaClasse', function () {
    // testes aqui
});
```
PROMPT;
```

### 6. Documentation

```php
$prompt = <<<PROMPT
Gere documentação para a API endpoint abaixo:

```php
{$controllerMethod}
```

## Formato: OpenAPI 3.0

Inclua:
- Summary e description
- Parameters (path, query, body)
- Request body schema com exemplos
- Responses (200, 400, 401, 404, 422, 500)
- Exemplos realistas de request/response
PROMPT;
```

---

## System Prompts Efetivos

### Estrutura Recomendada

```php
$systemPrompt = <<<PROMPT
# Identidade
Você é [PAPEL] especializado em [DOMÍNIO].

# Conhecimento
Você tem expertise em:
- [Tecnologia 1]
- [Tecnologia 2]
- [Padrão/Framework]

# Comportamento
- [Regra de comportamento 1]
- [Regra de comportamento 2]

# Formato de Resposta
[Descreva o formato esperado]

# Restrições
- [O que NÃO fazer]
- [Limites]

# Exemplos
[Exemplo de interação ideal]
PROMPT;
```

### Exemplo: Assistente Laravel

```php
$systemPrompt = <<<PROMPT
# Identidade
Você é um assistente de desenvolvimento Laravel expert.

# Conhecimento
- Laravel 10/11 com todas as features
- PHP 8.2+ com tipagem estrita
- Pest/PHPUnit para testes
- Padrões: Repository, Service, Action, DTO
- SOLID, DRY, KISS

# Comportamento
- Sempre use código PHP 8.2+ com declare(strict_types=1)
- Prefira Eloquent, mas sugira Query Builder para performance
- Valide inputs com Form Requests
- Use Resources para transformar responses
- Aplique princípios SOLID

# Formato de Resposta
1. Explicação breve do problema/solução
2. Código com comentários onde necessário
3. Testes se aplicável
4. Considerações de segurança/performance

# Restrições
- Não use facades em classes de domínio
- Não use helpers globais (use injection)
- Não exponha dados sensíveis
- Não sugira pacotes deprecated
PROMPT;
```

---

## Técnicas Avançadas

### 1. Decomposição de Tarefas

Quebre problemas complexos em sub-tarefas:

```php
$steps = [
    "1. Analise os requisitos e liste as entidades necessárias",
    "2. Defina os relacionamentos entre entidades",
    "3. Crie as migrations",
    "4. Crie os models com relacionamentos",
    "5. Crie os form requests para validação",
    "6. Implemente os controllers",
    "7. Adicione testes",
];

foreach ($steps as $step) {
    $response = $this->llm->complete("Contexto: {$context}\n\nTarefa: {$step}");
    $context .= "\n\nResultado anterior:\n{$response}";
}
```

### 2. Persona Stacking

Combine múltiplas perspectivas:

```
Analise este código sob três perspectivas:

1. **Arquiteto de Software**: Avalie a estrutura e padrões
2. **Especialista em Segurança**: Identifique vulnerabilidades
3. **DevOps Engineer**: Considere deployability e observabilidade

Para cada perspectiva, forneça:
- Pontos positivos
- Pontos de melhoria
- Recomendações específicas
```

### 3. Constraint-Based Prompting

Defina limites claros:

```
Gere uma solução que:
✓ DEVE usar apenas bibliotecas nativas do PHP
✓ DEVE funcionar em PHP 8.1+
✓ DEVE ter complexidade O(n) ou melhor
✗ NÃO PODE usar recursão (limite de stack)
✗ NÃO PODE alocar mais de 128MB
✗ NÃO PODE fazer I/O blocking
```

### 4. Iterative Refinement

Refine progressivamente:

```php
class IterativeRefiner
{
    public function refine(string $content, array $criteria): string
    {
        $current = $content;

        foreach ($criteria as $criterion) {
            $prompt = <<<PROMPT
            Melhore o seguinte conteúdo focando em: {$criterion}

            Conteúdo atual:
            {$current}

            Retorne apenas o conteúdo melhorado.
            PROMPT;

            $current = $this->llm->complete($prompt);
        }

        return $current;
    }
}

// Uso
$refiner->refine($code, [
    'legibilidade e nomes de variáveis',
    'tratamento de erros',
    'performance',
    'documentação',
]);
```

### 5. Output Structuring

Force formatos específicos:

```php
$prompt = <<<PROMPT
Analise o erro e responda EXATAMENTE neste formato JSON:

{
    "error_type": "string - tipo do erro",
    "root_cause": "string - causa raiz",
    "affected_files": ["array de arquivos afetados"],
    "fix": {
        "file": "string - arquivo a modificar",
        "line": "number - linha",
        "before": "string - código atual",
        "after": "string - código corrigido"
    },
    "prevention": "string - como prevenir no futuro"
}

Erro para analisar:
{$error}
PROMPT;
```

---

## Anti-Patterns a Evitar

### 1. Prompts Vagos

```
❌ Ruim:
"Melhore este código"

✅ Bom:
"Refatore este código para:
1. Usar early returns
2. Extrair método para validação
3. Adicionar tipagem estrita"
```

### 2. Contexto Insuficiente

```
❌ Ruim:
"Por que está dando erro?"

✅ Bom:
"Laravel 10, PHP 8.2
Erro: SQLSTATE[23000] Integrity constraint violation
Código: [código]
O que está causando a violação de constraint?"
```

### 3. Múltiplas Tarefas Sem Estrutura

```
❌ Ruim:
"Crie um CRUD com validação, testes, documentação e deploy"

✅ Bom:
"Tarefa 1 de 4: Criar Model e Migration
Especificações: [detalhes]
---
Após concluir, prossiga para: Tarefa 2 - Controllers"
```

### 4. Ignorar o Formato de Saída

```
❌ Ruim:
"Gere o código"

✅ Bom:
"Gere o código no formato:
1. Primeiro o arquivo de interface
2. Depois a implementação
3. Por fim os testes
Separe cada arquivo com '---FILE: nome.php---'"
```

---

## Templates Reutilizáveis

### Template de Code Review

```php
const CODE_REVIEW_PROMPT = <<<'PROMPT'
## Code Review Request

### Contexto
- **Projeto**: {project}
- **Stack**: {stack}
- **Tipo de mudança**: {change_type}

### Código para Review
```{language}
{code}
```

### Checklist de Review
Analise considerando:
- [ ] Segurança (OWASP Top 10)
- [ ] Performance (N+1, memory leaks)
- [ ] Manutenibilidade (SOLID, DRY)
- [ ] Testabilidade
- [ ] Documentação

### Formato de Resposta
Para cada issue:
1. **[SEVERIDADE]** Título curto
   - Linha(s): X-Y
   - Problema: descrição
   - Solução: código corrigido
PROMPT;
```

### Template de Debugging

```php
const DEBUG_PROMPT = <<<'PROMPT'
## Debug Request

### Comportamento Esperado
{expected}

### Comportamento Atual
{actual}

### Passos para Reproduzir
{steps}

### Ambiente
- PHP: {php_version}
- Laravel: {laravel_version}
- Database: {database}

### Código Relevante
```php
{code}
```

### Logs/Erros
```
{logs}
```

### Sua Análise
1. Possíveis causas (ordene por probabilidade)
2. Como confirmar cada hipótese
3. Solução recomendada
PROMPT;
```

---

## Métricas de Qualidade de Prompts

### Critérios de Avaliação

| Critério | Descrição | Peso |
|----------|-----------|------|
| **Clareza** | Instrução é inequívoca? | 25% |
| **Completude** | Tem todo contexto necessário? | 25% |
| **Especificidade** | Formato de saída está definido? | 20% |
| **Exemplos** | Tem exemplos quando necessário? | 15% |
| **Restrições** | Limites estão claros? | 15% |

### Checklist Antes de Enviar

```
□ O objetivo está claro na primeira frase?
□ O contexto técnico está completo?
□ O formato de saída está especificado?
□ Existem exemplos para tarefas complexas?
□ As restrições estão explícitas?
□ O prompt está livre de ambiguidades?
□ O tamanho é apropriado (nem muito curto, nem prolixo)?
```

---

## Próximos Passos

1. **Integração com Aplicações** → Como implementar LLMs em sistemas reais
2. **Desenvolvimento Assistido** → Fluxo de trabalho com IA no dia a dia

---

*Pratique constantemente: a engenharia de prompts melhora com experiência e iteração.*
