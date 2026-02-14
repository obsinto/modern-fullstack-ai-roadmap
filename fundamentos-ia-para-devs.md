# Fundamentos de IA para Desenvolvedores

## O que é IA e Machine Learning?

### Hierarquia de Conceitos

```
Inteligência Artificial (IA)
└── Machine Learning (ML)
    └── Deep Learning
        └── Large Language Models (LLMs)
            └── GPT, Claude, Gemini, etc.
```

**Inteligência Artificial**: Campo amplo que busca criar sistemas capazes de realizar tarefas que normalmente requerem inteligência humana.

**Machine Learning**: Subconjunto de IA onde sistemas aprendem padrões a partir de dados, sem serem explicitamente programados.

**Deep Learning**: ML usando redes neurais com múltiplas camadas (deep neural networks).

**LLMs**: Modelos de linguagem treinados em grandes volumes de texto, capazes de gerar e compreender linguagem natural.

---

## Como LLMs Funcionam

### Arquitetura Transformer

LLMs modernos são baseados na arquitetura **Transformer** (2017, Google):

```
Input: "O gato sentou no"
    ↓
[Tokenização] → [42, 891, 2847, 19]
    ↓
[Embedding] → Vetores de alta dimensão
    ↓
[Self-Attention] → Entende contexto e relações
    ↓
[Feed-Forward Layers] → Processa informação
    ↓
[Output Layer] → Probabilidades para próximo token
    ↓
Output: "sofá" (maior probabilidade)
```

### Conceitos Fundamentais

#### 1. Tokens

Tokens são as unidades básicas de processamento. Não são palavras exatas:

```
"Olá, mundo!" → ["Ol", "á", ",", " mundo", "!"]
"desenvolvimento" → ["des", "env", "olv", "imento"]
"API" → ["API"]
```

**Regras práticas**:
- 1 token ≈ 4 caracteres em inglês
- 1 token ≈ 3 caracteres em português
- 100 tokens ≈ 75 palavras

```php
// Estimativa aproximada de tokens
function estimateTokens(string $text): int
{
    // Para português, ~3 chars por token
    return (int) ceil(mb_strlen($text) / 3);
}
```

#### 2. Context Window (Janela de Contexto)

É o limite máximo de tokens que o modelo pode processar de uma vez (input + output):

| Modelo | Context Window |
|--------|----------------|
| GPT-3.5 | 4K - 16K tokens |
| GPT-4 | 8K - 128K tokens |
| Claude 3 | 200K tokens |
| Claude 3.5 | 200K tokens |
| Gemini 1.5 | 1M - 2M tokens |

```php
class ContextManager
{
    private int $maxTokens;
    private array $messages = [];

    public function __construct(int $maxTokens = 4096)
    {
        $this->maxTokens = $maxTokens;
    }

    public function addMessage(string $role, string $content): void
    {
        $this->messages[] = [
            'role' => $role,
            'content' => $content,
            'tokens' => $this->estimateTokens($content),
        ];

        $this->trimIfNeeded();
    }

    private function trimIfNeeded(): void
    {
        while ($this->getTotalTokens() > $this->maxTokens * 0.8) {
            // Remove mensagens antigas, mantendo system prompt
            array_splice($this->messages, 1, 1);
        }
    }

    private function getTotalTokens(): int
    {
        return array_sum(array_column($this->messages, 'tokens'));
    }
}
```

#### 3. Temperature e Sampling

**Temperature** controla a aleatoriedade das respostas:

| Temperature | Comportamento | Uso |
|-------------|---------------|-----|
| 0.0 | Determinístico, sempre mesma resposta | Código, dados estruturados |
| 0.3-0.5 | Balanceado | Tarefas gerais |
| 0.7-0.9 | Criativo, variado | Escrita criativa |
| 1.0+ | Muito aleatório | Brainstorming |

```php
// Configuração para diferentes casos de uso
$codeGeneration = [
    'temperature' => 0.0,
    'top_p' => 1.0,
];

$creativeWriting = [
    'temperature' => 0.8,
    'top_p' => 0.9,
];

$dataExtraction = [
    'temperature' => 0.0,
    'top_p' => 1.0,
];
```

---

## Tipos de Modelos e Suas Aplicações

### 1. Modelos de Linguagem (LLMs)

**Casos de uso**:
- Geração de texto
- Análise e resumo
- Tradução
- Código
- Conversação

```php
// Exemplo: Resumo de texto
$response = $client->chat()->create([
    'model' => 'gpt-4',
    'messages' => [
        ['role' => 'system', 'content' => 'Você é um assistente que resume textos.'],
        ['role' => 'user', 'content' => "Resuma em 3 pontos:\n\n{$article}"],
    ],
    'temperature' => 0.3,
]);
```

### 2. Modelos de Embedding

Convertem texto em vetores numéricos para:
- Busca semântica
- Similaridade de textos
- Clustering
- Classificação

```php
class EmbeddingService
{
    public function embed(string $text): array
    {
        $response = $this->client->embeddings()->create([
            'model' => 'text-embedding-3-small',
            'input' => $text,
        ]);

        return $response->embeddings[0]->embedding;
    }

    public function cosineSimilarity(array $a, array $b): float
    {
        $dotProduct = 0;
        $normA = 0;
        $normB = 0;

        for ($i = 0; $i < count($a); $i++) {
            $dotProduct += $a[$i] * $b[$i];
            $normA += $a[$i] ** 2;
            $normB += $b[$i] ** 2;
        }

        return $dotProduct / (sqrt($normA) * sqrt($normB));
    }
}
```

### 3. Modelos de Imagem

**Geração** (DALL-E, Midjourney, Stable Diffusion):
```php
$response = $client->images()->create([
    'model' => 'dall-e-3',
    'prompt' => 'Um logo minimalista para uma startup de tecnologia',
    'size' => '1024x1024',
    'quality' => 'hd',
]);
```

**Análise/Vision** (GPT-4V, Claude Vision):
```php
$response = $client->chat()->create([
    'model' => 'gpt-4-vision-preview',
    'messages' => [
        [
            'role' => 'user',
            'content' => [
                ['type' => 'text', 'text' => 'O que há nesta imagem?'],
                ['type' => 'image_url', 'image_url' => ['url' => $imageUrl]],
            ],
        ],
    ],
]);
```

### 4. Modelos de Áudio

**Speech-to-Text** (Whisper):
```php
$response = $client->audio()->transcribe([
    'model' => 'whisper-1',
    'file' => fopen('audio.mp3', 'r'),
    'language' => 'pt',
]);
```

**Text-to-Speech**:
```php
$response = $client->audio()->speech([
    'model' => 'tts-1-hd',
    'voice' => 'nova',
    'input' => 'Olá, bem-vindo ao nosso sistema!',
]);
```

---

## Embeddings e Busca Semântica

### O que são Embeddings?

Embeddings são representações vetoriais de texto que capturam significado semântico:

```
"rei" - "homem" + "mulher" ≈ "rainha"
[0.2, 0.8, ...] - [0.1, 0.3, ...] + [0.15, 0.35, ...] ≈ [0.25, 0.85, ...]
```

### Implementando Busca Semântica

```php
class SemanticSearch
{
    private EmbeddingService $embeddings;
    private VectorStore $vectorStore;

    public function index(string $id, string $content): void
    {
        $embedding = $this->embeddings->embed($content);

        $this->vectorStore->upsert([
            'id' => $id,
            'values' => $embedding,
            'metadata' => [
                'content' => $content,
                'indexed_at' => now()->toIso8601String(),
            ],
        ]);
    }

    public function search(string $query, int $limit = 5): array
    {
        $queryEmbedding = $this->embeddings->embed($query);

        return $this->vectorStore->query([
            'vector' => $queryEmbedding,
            'top_k' => $limit,
            'include_metadata' => true,
        ]);
    }
}

// Uso
$search = new SemanticSearch($embeddings, $vectorStore);

// Indexar documentos
$search->index('doc1', 'Laravel é um framework PHP para aplicações web');
$search->index('doc2', 'Vue.js é uma biblioteca JavaScript reativa');
$search->index('doc3', 'PostgreSQL é um banco de dados relacional');

// Buscar
$results = $search->search('framework para construir sites');
// Retorna doc1 com alta similaridade, mesmo sem palavras exatas
```

### Vector Databases

Opções populares para armazenar embeddings:

| Database | Tipo | Características |
|----------|------|-----------------|
| Pinecone | Cloud | Managed, escalável |
| Weaviate | Self-hosted/Cloud | GraphQL, modules |
| Qdrant | Self-hosted/Cloud | Rust, rápido |
| Milvus | Self-hosted | Escalável, GPU |
| pgvector | PostgreSQL extension | Familiar, SQL |
| ChromaDB | Embedded | Simples, local |

```php
// Exemplo com pgvector no Laravel
Schema::create('documents', function (Blueprint $table) {
    $table->id();
    $table->text('content');
    $table->vector('embedding', 1536); // OpenAI dimension
    $table->timestamps();
});

// Busca por similaridade
$results = DB::select("
    SELECT id, content,
           1 - (embedding <=> ?) as similarity
    FROM documents
    ORDER BY embedding <=> ?
    LIMIT 5
", [$queryEmbedding, $queryEmbedding]);
```

---

## RAG - Retrieval Augmented Generation

### O que é RAG?

RAG combina busca de informação com geração de texto:

```
Query do usuário
      ↓
[1. Retrieval] → Busca documentos relevantes no vector store
      ↓
[2. Augmentation] → Adiciona documentos ao contexto do prompt
      ↓
[3. Generation] → LLM gera resposta baseada no contexto
```

### Por que usar RAG?

1. **Conhecimento atualizado**: LLMs têm knowledge cutoff
2. **Dados proprietários**: Seus dados não estão no treinamento
3. **Reduz alucinações**: Respostas baseadas em fontes reais
4. **Citações**: Pode referenciar documentos específicos
5. **Custo**: Mais barato que fine-tuning

### Implementação Básica

```php
class RAGService
{
    public function __construct(
        private SemanticSearch $search,
        private LLMClient $llm,
    ) {}

    public function query(string $question): RAGResponse
    {
        // 1. Retrieval - buscar documentos relevantes
        $documents = $this->search->search($question, limit: 5);

        // 2. Augmentation - construir contexto
        $context = $this->buildContext($documents);

        // 3. Generation - gerar resposta
        $response = $this->llm->chat([
            'messages' => [
                [
                    'role' => 'system',
                    'content' => $this->buildSystemPrompt($context),
                ],
                [
                    'role' => 'user',
                    'content' => $question,
                ],
            ],
        ]);

        return new RAGResponse(
            answer: $response->content,
            sources: $documents,
        );
    }

    private function buildSystemPrompt(string $context): string
    {
        return <<<PROMPT
        Você é um assistente que responde perguntas baseado no contexto fornecido.

        Regras:
        - Responda APENAS com informações do contexto
        - Se não souber, diga "Não encontrei essa informação"
        - Cite as fontes quando relevante

        Contexto:
        {$context}
        PROMPT;
    }

    private function buildContext(array $documents): string
    {
        return collect($documents)
            ->map(fn($doc) => "---\nFonte: {$doc['id']}\n{$doc['content']}\n---")
            ->join("\n\n");
    }
}
```

### Estratégias Avançadas de RAG

#### 1. Chunking (Divisão de Documentos)

```php
class DocumentChunker
{
    public function chunk(
        string $content,
        int $chunkSize = 500,
        int $overlap = 50
    ): array {
        $sentences = $this->splitIntoSentences($content);
        $chunks = [];
        $currentChunk = '';
        $currentTokens = 0;

        foreach ($sentences as $sentence) {
            $sentenceTokens = $this->estimateTokens($sentence);

            if ($currentTokens + $sentenceTokens > $chunkSize) {
                $chunks[] = trim($currentChunk);
                // Overlap: manter últimas sentenças
                $currentChunk = $this->getOverlap($currentChunk, $overlap);
                $currentTokens = $this->estimateTokens($currentChunk);
            }

            $currentChunk .= ' ' . $sentence;
            $currentTokens += $sentenceTokens;
        }

        if (trim($currentChunk)) {
            $chunks[] = trim($currentChunk);
        }

        return $chunks;
    }
}
```

#### 2. Hybrid Search (Keyword + Semantic)

```php
class HybridSearch
{
    public function search(string $query, int $limit = 10): array
    {
        // Busca semântica
        $semanticResults = $this->semanticSearch->search($query, $limit * 2);

        // Busca por keywords (BM25)
        $keywordResults = $this->keywordSearch->search($query, $limit * 2);

        // Reciprocal Rank Fusion
        return $this->fuseResults($semanticResults, $keywordResults, $limit);
    }

    private function fuseResults(array $semantic, array $keyword, int $limit): array
    {
        $scores = [];
        $k = 60; // constante RRF

        foreach ($semantic as $rank => $doc) {
            $scores[$doc['id']] = ($scores[$doc['id']] ?? 0) + 1 / ($k + $rank + 1);
        }

        foreach ($keyword as $rank => $doc) {
            $scores[$doc['id']] = ($scores[$doc['id']] ?? 0) + 1 / ($k + $rank + 1);
        }

        arsort($scores);

        return array_slice(array_keys($scores), 0, $limit);
    }
}
```

#### 3. Re-ranking

```php
class RerankerService
{
    public function rerank(string $query, array $documents): array
    {
        // Usar modelo de reranking (Cohere, cross-encoder)
        $response = $this->client->rerank([
            'query' => $query,
            'documents' => array_column($documents, 'content'),
            'top_n' => 5,
        ]);

        return collect($response->results)
            ->sortByDesc('relevance_score')
            ->map(fn($r) => $documents[$r->index])
            ->values()
            ->all();
    }
}
```

---

## Limitações e Considerações

### 1. Alucinações (Hallucinations)

LLMs podem inventar informações que parecem plausíveis:

```php
class FactChecker
{
    public function validateResponse(string $response, array $sources): ValidationResult
    {
        $claims = $this->extractClaims($response);
        $validated = [];

        foreach ($claims as $claim) {
            $validated[] = [
                'claim' => $claim,
                'supported' => $this->isSupported($claim, $sources),
            ];
        }

        return new ValidationResult($validated);
    }
}
```

**Mitigações**:
- Use RAG com fontes confiáveis
- Peça citações
- Implemente verificação de fatos
- Use temperature baixa para fatos

### 2. Context Window Limits

```php
class ContextOptimizer
{
    public function optimize(array $messages, int $maxTokens): array
    {
        $totalTokens = $this->countTokens($messages);

        if ($totalTokens <= $maxTokens) {
            return $messages;
        }

        // Estratégias de otimização
        return $this->summarizeOldMessages($messages, $maxTokens);
    }

    private function summarizeOldMessages(array $messages, int $maxTokens): array
    {
        // Manter system prompt e mensagens recentes
        $systemPrompt = $messages[0];
        $recentMessages = array_slice($messages, -4);
        $oldMessages = array_slice($messages, 1, -4);

        // Resumir mensagens antigas
        $summary = $this->llm->summarize($oldMessages);

        return [
            $systemPrompt,
            ['role' => 'system', 'content' => "Resumo da conversa anterior: {$summary}"],
            ...$recentMessages,
        ];
    }
}
```

### 3. Custos

```php
class CostTracker
{
    private array $pricing = [
        'gpt-4' => ['input' => 0.03, 'output' => 0.06], // por 1K tokens
        'gpt-3.5-turbo' => ['input' => 0.0005, 'output' => 0.0015],
        'claude-3-opus' => ['input' => 0.015, 'output' => 0.075],
        'claude-3-sonnet' => ['input' => 0.003, 'output' => 0.015],
    ];

    public function calculateCost(string $model, int $inputTokens, int $outputTokens): float
    {
        $prices = $this->pricing[$model];

        return ($inputTokens / 1000 * $prices['input']) +
               ($outputTokens / 1000 * $prices['output']);
    }
}
```

### 4. Latência

```php
class StreamingResponse
{
    public function stream(array $messages): Generator
    {
        $stream = $this->client->chat()->createStreamed([
            'model' => 'gpt-4',
            'messages' => $messages,
            'stream' => true,
        ]);

        foreach ($stream as $response) {
            $delta = $response->choices[0]->delta->content ?? '';
            if ($delta) {
                yield $delta;
            }
        }
    }
}

// No controller
public function chat(Request $request)
{
    return response()->stream(function () use ($request) {
        foreach ($this->ai->stream($request->messages) as $chunk) {
            echo "data: " . json_encode(['content' => $chunk]) . "\n\n";
            ob_flush();
            flush();
        }
    }, 200, [
        'Content-Type' => 'text/event-stream',
        'Cache-Control' => 'no-cache',
    ]);
}
```

### 5. Privacidade e Segurança

```php
class DataSanitizer
{
    private array $patterns = [
        '/\b\d{3}\.\d{3}\.\d{3}-\d{2}\b/' => '[CPF REMOVIDO]',
        '/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/' => '[EMAIL REMOVIDO]',
        '/\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b/' => '[CARTÃO REMOVIDO]',
    ];

    public function sanitize(string $content): string
    {
        foreach ($this->patterns as $pattern => $replacement) {
            $content = preg_replace($pattern, $replacement, $content);
        }

        return $content;
    }
}
```

---

## Glossário de Termos

| Termo | Definição |
|-------|-----------|
| **Token** | Unidade básica de texto processada pelo modelo |
| **Context Window** | Quantidade máxima de tokens que o modelo pode processar |
| **Temperature** | Parâmetro que controla aleatoriedade das respostas |
| **Embedding** | Representação vetorial de texto |
| **RAG** | Retrieval Augmented Generation |
| **Fine-tuning** | Treinar modelo existente com dados específicos |
| **Prompt** | Instrução/texto enviado ao modelo |
| **Completion** | Resposta gerada pelo modelo |
| **Hallucination** | Quando o modelo inventa informações falsas |
| **Grounding** | Ancorar respostas em dados reais |
| **Vector Store** | Banco de dados otimizado para embeddings |
| **Semantic Search** | Busca por significado, não palavras exatas |
| **Chunking** | Dividir documentos em pedaços menores |
| **Top-p (Nucleus)** | Método de sampling alternativo à temperature |

---

## Próximos Passos

1. **Engenharia de Prompts** → Como escrever prompts efetivos
2. **Integração com Aplicações** → Padrões práticos de implementação
3. **Desenvolvimento Assistido por IA** → Usando IA no dia a dia do dev

---

*Nota: Os exemplos de código são ilustrativos. Adapte-os para seu caso de uso específico e sempre consulte a documentação oficial das APIs.*
