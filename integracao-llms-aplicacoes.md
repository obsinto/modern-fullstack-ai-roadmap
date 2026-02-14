# Integração de LLMs em Aplicações

## Visão Geral

Este guia cobre padrões práticos para integrar Large Language Models (LLMs) em aplicações Laravel/PHP de produção.

---

## Arquitetura de Integração

### Camadas de Abstração

```
┌─────────────────────────────────────────────┐
│              Application Layer               │
│         (Controllers, Commands, Jobs)        │
└─────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│              Service Layer                   │
│    (AIAssistant, ContentGenerator, etc.)     │
└─────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│             Abstraction Layer                │
│         (LLMClient Interface)                │
└─────────────────────────────────────────────┘
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
     ┌─────────┐ ┌─────────┐ ┌─────────┐
     │ OpenAI  │ │ Claude  │ │ Gemini  │
     │ Adapter │ │ Adapter │ │ Adapter │
     └─────────┘ └─────────┘ └─────────┘
```

### Interface do Cliente LLM

```php
<?php

declare(strict_types=1);

namespace App\Contracts;

interface LLMClientInterface
{
    /**
     * Envia mensagens e recebe resposta
     */
    public function chat(array $messages, array $options = []): LLMResponse;

    /**
     * Stream de resposta para UI em tempo real
     */
    public function stream(array $messages, array $options = []): Generator;

    /**
     * Gera embedding para texto
     */
    public function embed(string $text): array;

    /**
     * Retorna informações sobre o modelo
     */
    public function getModelInfo(): ModelInfo;
}
```

### Value Objects para Respostas

```php
<?php

declare(strict_types=1);

namespace App\ValueObjects;

final readonly class LLMResponse
{
    public function __construct(
        public string $content,
        public int $inputTokens,
        public int $outputTokens,
        public string $model,
        public ?string $finishReason = null,
        public array $metadata = [],
    ) {}

    public function totalTokens(): int
    {
        return $this->inputTokens + $this->outputTokens;
    }

    public function estimatedCost(array $pricing): float
    {
        return ($this->inputTokens / 1000 * $pricing['input']) +
               ($this->outputTokens / 1000 * $pricing['output']);
    }
}

final readonly class ModelInfo
{
    public function __construct(
        public string $id,
        public string $provider,
        public int $maxContextTokens,
        public int $maxOutputTokens,
        public array $capabilities = [],
    ) {}

    public function supportsVision(): bool
    {
        return in_array('vision', $this->capabilities, true);
    }

    public function supportsFunctionCalling(): bool
    {
        return in_array('function_calling', $this->capabilities, true);
    }
}
```

---

## Implementação de Adapters

### OpenAI Adapter

```php
<?php

declare(strict_types=1);

namespace App\LLM\Adapters;

use App\Contracts\LLMClientInterface;
use App\ValueObjects\LLMResponse;
use App\ValueObjects\ModelInfo;
use OpenAI\Client;
use Generator;

final class OpenAIAdapter implements LLMClientInterface
{
    private const DEFAULT_MODEL = 'gpt-4-turbo-preview';

    public function __construct(
        private Client $client,
        private string $model = self::DEFAULT_MODEL,
    ) {}

    public function chat(array $messages, array $options = []): LLMResponse
    {
        $response = $this->client->chat()->create([
            'model' => $options['model'] ?? $this->model,
            'messages' => $this->formatMessages($messages),
            'temperature' => $options['temperature'] ?? 0.7,
            'max_tokens' => $options['max_tokens'] ?? 4096,
        ]);

        $choice = $response->choices[0];

        return new LLMResponse(
            content: $choice->message->content ?? '',
            inputTokens: $response->usage->promptTokens,
            outputTokens: $response->usage->completionTokens,
            model: $response->model,
            finishReason: $choice->finishReason,
        );
    }

    public function stream(array $messages, array $options = []): Generator
    {
        $stream = $this->client->chat()->createStreamed([
            'model' => $options['model'] ?? $this->model,
            'messages' => $this->formatMessages($messages),
            'temperature' => $options['temperature'] ?? 0.7,
            'max_tokens' => $options['max_tokens'] ?? 4096,
            'stream' => true,
        ]);

        foreach ($stream as $response) {
            $delta = $response->choices[0]->delta->content ?? '';
            if ($delta !== '') {
                yield $delta;
            }
        }
    }

    public function embed(string $text): array
    {
        $response = $this->client->embeddings()->create([
            'model' => 'text-embedding-3-small',
            'input' => $text,
        ]);

        return $response->embeddings[0]->embedding;
    }

    public function getModelInfo(): ModelInfo
    {
        return new ModelInfo(
            id: $this->model,
            provider: 'openai',
            maxContextTokens: 128000,
            maxOutputTokens: 4096,
            capabilities: ['vision', 'function_calling', 'json_mode'],
        );
    }

    private function formatMessages(array $messages): array
    {
        return array_map(
            fn(array $msg) => [
                'role' => $msg['role'],
                'content' => $msg['content'],
            ],
            $messages
        );
    }
}
```

### Claude Adapter

```php
<?php

declare(strict_types=1);

namespace App\LLM\Adapters;

use App\Contracts\LLMClientInterface;
use App\ValueObjects\LLMResponse;
use App\ValueObjects\ModelInfo;
use Anthropic\Client;
use Generator;

final class ClaudeAdapter implements LLMClientInterface
{
    private const DEFAULT_MODEL = 'claude-3-sonnet-20240229';

    public function __construct(
        private Client $client,
        private string $model = self::DEFAULT_MODEL,
    ) {}

    public function chat(array $messages, array $options = []): LLMResponse
    {
        $systemMessage = $this->extractSystemMessage($messages);
        $userMessages = $this->filterUserMessages($messages);

        $response = $this->client->messages()->create([
            'model' => $options['model'] ?? $this->model,
            'max_tokens' => $options['max_tokens'] ?? 4096,
            'system' => $systemMessage,
            'messages' => $userMessages,
        ]);

        return new LLMResponse(
            content: $response->content[0]->text,
            inputTokens: $response->usage->inputTokens,
            outputTokens: $response->usage->outputTokens,
            model: $response->model,
            finishReason: $response->stopReason,
        );
    }

    public function stream(array $messages, array $options = []): Generator
    {
        $systemMessage = $this->extractSystemMessage($messages);
        $userMessages = $this->filterUserMessages($messages);

        $stream = $this->client->messages()->createStreamed([
            'model' => $options['model'] ?? $this->model,
            'max_tokens' => $options['max_tokens'] ?? 4096,
            'system' => $systemMessage,
            'messages' => $userMessages,
        ]);

        foreach ($stream as $event) {
            if ($event->type === 'content_block_delta') {
                yield $event->delta->text ?? '';
            }
        }
    }

    public function embed(string $text): array
    {
        // Claude não tem API de embeddings nativa
        // Use Voyage AI ou outra solução
        throw new \RuntimeException('Claude does not support embeddings directly');
    }

    public function getModelInfo(): ModelInfo
    {
        return new ModelInfo(
            id: $this->model,
            provider: 'anthropic',
            maxContextTokens: 200000,
            maxOutputTokens: 4096,
            capabilities: ['vision', 'function_calling'],
        );
    }

    private function extractSystemMessage(array $messages): string
    {
        foreach ($messages as $message) {
            if ($message['role'] === 'system') {
                return $message['content'];
            }
        }
        return '';
    }

    private function filterUserMessages(array $messages): array
    {
        return array_values(array_filter(
            $messages,
            fn($msg) => $msg['role'] !== 'system'
        ));
    }
}
```

---

## Service Provider e Factory

### LLM Service Provider

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use App\Contracts\LLMClientInterface;
use App\LLM\Adapters\OpenAIAdapter;
use App\LLM\Adapters\ClaudeAdapter;
use App\LLM\LLMFactory;
use Illuminate\Support\ServiceProvider;

class LLMServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(LLMFactory::class, function ($app) {
            return new LLMFactory(config('llm'));
        });

        $this->app->bind(LLMClientInterface::class, function ($app) {
            return $app->make(LLMFactory::class)->create(
                config('llm.default')
            );
        });
    }

    public function boot(): void
    {
        $this->publishes([
            __DIR__.'/../config/llm.php' => config_path('llm.php'),
        ], 'llm-config');
    }
}
```

### Factory Pattern

```php
<?php

declare(strict_types=1);

namespace App\LLM;

use App\Contracts\LLMClientInterface;
use App\LLM\Adapters\OpenAIAdapter;
use App\LLM\Adapters\ClaudeAdapter;
use OpenAI;
use Anthropic;
use InvalidArgumentException;

final class LLMFactory
{
    public function __construct(
        private array $config,
    ) {}

    public function create(string $provider): LLMClientInterface
    {
        return match ($provider) {
            'openai' => $this->createOpenAI(),
            'anthropic', 'claude' => $this->createClaude(),
            default => throw new InvalidArgumentException(
                "Unknown LLM provider: {$provider}"
            ),
        };
    }

    private function createOpenAI(): OpenAIAdapter
    {
        $client = OpenAI::client($this->config['openai']['api_key']);

        return new OpenAIAdapter(
            client: $client,
            model: $this->config['openai']['model'] ?? 'gpt-4-turbo-preview',
        );
    }

    private function createClaude(): ClaudeAdapter
    {
        $client = Anthropic::client($this->config['anthropic']['api_key']);

        return new ClaudeAdapter(
            client: $client,
            model: $this->config['anthropic']['model'] ?? 'claude-3-sonnet-20240229',
        );
    }
}
```

### Configuração

```php
<?php

// config/llm.php

return [
    'default' => env('LLM_PROVIDER', 'openai'),

    'openai' => [
        'api_key' => env('OPENAI_API_KEY'),
        'model' => env('OPENAI_MODEL', 'gpt-4-turbo-preview'),
        'organization' => env('OPENAI_ORGANIZATION'),
    ],

    'anthropic' => [
        'api_key' => env('ANTHROPIC_API_KEY'),
        'model' => env('ANTHROPIC_MODEL', 'claude-3-sonnet-20240229'),
    ],

    'cache' => [
        'enabled' => env('LLM_CACHE_ENABLED', true),
        'ttl' => env('LLM_CACHE_TTL', 3600),
        'store' => env('LLM_CACHE_STORE', 'redis'),
    ],

    'rate_limit' => [
        'enabled' => env('LLM_RATE_LIMIT_ENABLED', true),
        'max_requests_per_minute' => env('LLM_RATE_LIMIT_RPM', 60),
    ],
];
```

---

## Padrões de Uso

### 1. Service Layer para Funcionalidades

```php
<?php

declare(strict_types=1);

namespace App\Services\AI;

use App\Contracts\LLMClientInterface;
use App\DTOs\ContentSummaryDTO;

final class ContentSummarizer
{
    private const SYSTEM_PROMPT = <<<'PROMPT'
    Você é um assistente especializado em resumir conteúdo.

    Regras:
    - Mantenha os pontos principais
    - Use linguagem clara e objetiva
    - Responda em português brasileiro
    PROMPT;

    public function __construct(
        private LLMClientInterface $llm,
    ) {}

    public function summarize(
        string $content,
        int $maxWords = 100,
    ): ContentSummaryDTO {
        $response = $this->llm->chat([
            ['role' => 'system', 'content' => self::SYSTEM_PROMPT],
            ['role' => 'user', 'content' => $this->buildPrompt($content, $maxWords)],
        ], [
            'temperature' => 0.3,
            'max_tokens' => 500,
        ]);

        return new ContentSummaryDTO(
            summary: $response->content,
            originalLength: str_word_count($content),
            summaryLength: str_word_count($response->content),
            tokensUsed: $response->totalTokens(),
        );
    }

    private function buildPrompt(string $content, int $maxWords): string
    {
        return <<<PROMPT
        Resuma o seguinte conteúdo em no máximo {$maxWords} palavras:

        ---
        {$content}
        ---

        Resumo:
        PROMPT;
    }
}
```

### 2. Chat com Contexto (Conversational)

```php
<?php

declare(strict_types=1);

namespace App\Services\AI;

use App\Contracts\LLMClientInterface;
use App\Models\Conversation;
use App\Models\Message;
use Illuminate\Support\Collection;

final class ChatService
{
    public function __construct(
        private LLMClientInterface $llm,
        private ConversationRepository $conversations,
    ) {}

    public function sendMessage(
        Conversation $conversation,
        string $userMessage,
    ): Message {
        // Salvar mensagem do usuário
        $userMsg = $conversation->messages()->create([
            'role' => 'user',
            'content' => $userMessage,
        ]);

        // Construir histórico
        $messages = $this->buildMessageHistory($conversation);

        // Chamar LLM
        $response = $this->llm->chat($messages, [
            'temperature' => 0.7,
        ]);

        // Salvar resposta
        $assistantMsg = $conversation->messages()->create([
            'role' => 'assistant',
            'content' => $response->content,
            'tokens_used' => $response->totalTokens(),
        ]);

        return $assistantMsg;
    }

    private function buildMessageHistory(Conversation $conversation): array
    {
        $messages = [
            ['role' => 'system', 'content' => $conversation->system_prompt],
        ];

        // Últimas N mensagens para manter contexto
        $history = $conversation->messages()
            ->orderBy('created_at', 'desc')
            ->take(20)
            ->get()
            ->reverse();

        foreach ($history as $msg) {
            $messages[] = [
                'role' => $msg->role,
                'content' => $msg->content,
            ];
        }

        return $messages;
    }
}
```

### 3. Streaming Response para UI

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Services\AI\ChatService;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\StreamedResponse;

class ChatController extends Controller
{
    public function __construct(
        private ChatService $chat,
    ) {}

    public function stream(Request $request): StreamedResponse
    {
        $validated = $request->validate([
            'conversation_id' => 'required|exists:conversations,id',
            'message' => 'required|string|max:10000',
        ]);

        return response()->stream(function () use ($validated) {
            $conversation = Conversation::find($validated['conversation_id']);

            foreach ($this->chat->streamMessage($conversation, $validated['message']) as $chunk) {
                echo "data: " . json_encode(['content' => $chunk]) . "\n\n";
                ob_flush();
                flush();
            }

            echo "data: [DONE]\n\n";
            ob_flush();
            flush();
        }, 200, [
            'Content-Type' => 'text/event-stream',
            'Cache-Control' => 'no-cache',
            'X-Accel-Buffering' => 'no',
        ]);
    }
}
```

---

## Function Calling / Tool Use

### Definindo Tools

```php
<?php

declare(strict_types=1);

namespace App\LLM\Tools;

final class ToolDefinition
{
    public static function weatherTool(): array
    {
        return [
            'type' => 'function',
            'function' => [
                'name' => 'get_weather',
                'description' => 'Obtém informações do clima para uma cidade',
                'parameters' => [
                    'type' => 'object',
                    'properties' => [
                        'city' => [
                            'type' => 'string',
                            'description' => 'Nome da cidade',
                        ],
                        'unit' => [
                            'type' => 'string',
                            'enum' => ['celsius', 'fahrenheit'],
                            'description' => 'Unidade de temperatura',
                        ],
                    ],
                    'required' => ['city'],
                ],
            ],
        ];
    }

    public static function searchTool(): array
    {
        return [
            'type' => 'function',
            'function' => [
                'name' => 'search_database',
                'description' => 'Busca informações no banco de dados da empresa',
                'parameters' => [
                    'type' => 'object',
                    'properties' => [
                        'query' => [
                            'type' => 'string',
                            'description' => 'Termo de busca',
                        ],
                        'table' => [
                            'type' => 'string',
                            'enum' => ['products', 'orders', 'customers'],
                            'description' => 'Tabela onde buscar',
                        ],
                        'limit' => [
                            'type' => 'integer',
                            'description' => 'Máximo de resultados',
                        ],
                    ],
                    'required' => ['query', 'table'],
                ],
            ],
        ];
    }
}
```

### Tool Executor

```php
<?php

declare(strict_types=1);

namespace App\LLM\Tools;

use App\Services\WeatherService;
use App\Repositories\SearchRepository;

final class ToolExecutor
{
    public function __construct(
        private WeatherService $weather,
        private SearchRepository $search,
    ) {}

    public function execute(string $toolName, array $arguments): mixed
    {
        return match ($toolName) {
            'get_weather' => $this->executeWeather($arguments),
            'search_database' => $this->executeSearch($arguments),
            default => throw new \InvalidArgumentException("Unknown tool: {$toolName}"),
        };
    }

    private function executeWeather(array $args): array
    {
        $weather = $this->weather->get(
            city: $args['city'],
            unit: $args['unit'] ?? 'celsius',
        );

        return [
            'temperature' => $weather->temperature,
            'condition' => $weather->condition,
            'humidity' => $weather->humidity,
        ];
    }

    private function executeSearch(array $args): array
    {
        return $this->search->search(
            query: $args['query'],
            table: $args['table'],
            limit: $args['limit'] ?? 10,
        );
    }
}
```

### Agent com Tool Use

```php
<?php

declare(strict_types=1);

namespace App\Services\AI;

use App\Contracts\LLMClientInterface;
use App\LLM\Tools\ToolDefinition;
use App\LLM\Tools\ToolExecutor;

final class AgentService
{
    private const MAX_ITERATIONS = 5;

    public function __construct(
        private LLMClientInterface $llm,
        private ToolExecutor $toolExecutor,
    ) {}

    public function run(string $userQuery): string
    {
        $messages = [
            ['role' => 'system', 'content' => $this->getSystemPrompt()],
            ['role' => 'user', 'content' => $userQuery],
        ];

        $tools = [
            ToolDefinition::weatherTool(),
            ToolDefinition::searchTool(),
        ];

        for ($i = 0; $i < self::MAX_ITERATIONS; $i++) {
            $response = $this->llm->chatWithTools($messages, $tools);

            // Se não há tool calls, retornar resposta final
            if (empty($response->toolCalls)) {
                return $response->content;
            }

            // Adicionar resposta do assistant
            $messages[] = [
                'role' => 'assistant',
                'content' => $response->content,
                'tool_calls' => $response->toolCalls,
            ];

            // Executar cada tool call
            foreach ($response->toolCalls as $toolCall) {
                $result = $this->toolExecutor->execute(
                    $toolCall['name'],
                    $toolCall['arguments'],
                );

                $messages[] = [
                    'role' => 'tool',
                    'tool_call_id' => $toolCall['id'],
                    'content' => json_encode($result),
                ];
            }
        }

        throw new \RuntimeException('Max iterations reached without final response');
    }

    private function getSystemPrompt(): string
    {
        return <<<'PROMPT'
        Você é um assistente útil que pode buscar informações e responder perguntas.
        Use as ferramentas disponíveis quando necessário para obter dados atualizados.
        Sempre responda em português brasileiro.
        PROMPT;
    }
}
```

---

## Caching e Otimização

### Cache de Respostas

```php
<?php

declare(strict_types=1);

namespace App\LLM;

use App\Contracts\LLMClientInterface;
use App\ValueObjects\LLMResponse;
use Illuminate\Contracts\Cache\Repository as CacheRepository;

final class CachedLLMClient implements LLMClientInterface
{
    public function __construct(
        private LLMClientInterface $client,
        private CacheRepository $cache,
        private int $ttl = 3600,
    ) {}

    public function chat(array $messages, array $options = []): LLMResponse
    {
        // Não cachear se temperatura > 0
        if (($options['temperature'] ?? 0.7) > 0) {
            return $this->client->chat($messages, $options);
        }

        $cacheKey = $this->buildCacheKey($messages, $options);

        return $this->cache->remember($cacheKey, $this->ttl, function () use ($messages, $options) {
            return $this->client->chat($messages, $options);
        });
    }

    private function buildCacheKey(array $messages, array $options): string
    {
        $hash = md5(json_encode([
            'messages' => $messages,
            'options' => $options,
        ]));

        return "llm:response:{$hash}";
    }

    // ... outros métodos delegam para o client
}
```

### Rate Limiting

```php
<?php

declare(strict_types=1);

namespace App\LLM;

use App\Contracts\LLMClientInterface;
use App\Exceptions\RateLimitExceededException;
use Illuminate\Support\Facades\RateLimiter;

final class RateLimitedLLMClient implements LLMClientInterface
{
    public function __construct(
        private LLMClientInterface $client,
        private int $maxRequestsPerMinute = 60,
    ) {}

    public function chat(array $messages, array $options = []): LLMResponse
    {
        $key = $this->getRateLimitKey();

        if (RateLimiter::tooManyAttempts($key, $this->maxRequestsPerMinute)) {
            $seconds = RateLimiter::availableIn($key);
            throw new RateLimitExceededException(
                "Rate limit exceeded. Try again in {$seconds} seconds."
            );
        }

        RateLimiter::hit($key, 60);

        return $this->client->chat($messages, $options);
    }

    private function getRateLimitKey(): string
    {
        return 'llm:rate_limit:' . (auth()->id() ?? 'global');
    }
}
```

### Retry com Exponential Backoff

```php
<?php

declare(strict_types=1);

namespace App\LLM;

use App\Contracts\LLMClientInterface;
use App\ValueObjects\LLMResponse;

final class RetryableLLMClient implements LLMClientInterface
{
    public function __construct(
        private LLMClientInterface $client,
        private int $maxRetries = 3,
        private int $baseDelayMs = 1000,
    ) {}

    public function chat(array $messages, array $options = []): LLMResponse
    {
        $attempt = 0;
        $lastException = null;

        while ($attempt < $this->maxRetries) {
            try {
                return $this->client->chat($messages, $options);
            } catch (\Exception $e) {
                $lastException = $e;

                if (!$this->isRetryable($e)) {
                    throw $e;
                }

                $attempt++;
                $delay = $this->calculateDelay($attempt);
                usleep($delay * 1000);
            }
        }

        throw $lastException;
    }

    private function isRetryable(\Exception $e): bool
    {
        // Retry em erros de rate limit ou server errors
        return str_contains($e->getMessage(), 'rate_limit') ||
               str_contains($e->getMessage(), '500') ||
               str_contains($e->getMessage(), '503');
    }

    private function calculateDelay(int $attempt): int
    {
        // Exponential backoff com jitter
        $delay = $this->baseDelayMs * (2 ** ($attempt - 1));
        $jitter = random_int(0, (int)($delay * 0.1));
        return $delay + $jitter;
    }
}
```

---

## Processamento em Background

### Job para Processamento Assíncrono

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use App\Contracts\LLMClientInterface;
use App\Models\Document;
use App\Events\DocumentProcessed;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessDocumentWithAI implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $backoff = 60;

    public function __construct(
        public Document $document,
    ) {}

    public function handle(LLMClientInterface $llm): void
    {
        $response = $llm->chat([
            ['role' => 'system', 'content' => 'Analise o documento e extraia metadados.'],
            ['role' => 'user', 'content' => $this->document->content],
        ], [
            'temperature' => 0.0,
        ]);

        $metadata = json_decode($response->content, true);

        $this->document->update([
            'ai_metadata' => $metadata,
            'ai_processed_at' => now(),
            'ai_tokens_used' => $response->totalTokens(),
        ]);

        event(new DocumentProcessed($this->document));
    }

    public function failed(\Throwable $exception): void
    {
        $this->document->update([
            'ai_error' => $exception->getMessage(),
            'ai_failed_at' => now(),
        ]);
    }
}
```

### Batch Processing

```php
<?php

declare(strict_types=1);

namespace App\Services\AI;

use App\Jobs\ProcessDocumentWithAI;
use App\Models\Document;
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;

final class BatchProcessor
{
    public function processDocuments(array $documentIds): Batch
    {
        $jobs = Document::whereIn('id', $documentIds)
            ->whereNull('ai_processed_at')
            ->get()
            ->map(fn($doc) => new ProcessDocumentWithAI($doc))
            ->all();

        return Bus::batch($jobs)
            ->name('AI Document Processing')
            ->allowFailures()
            ->onQueue('ai-processing')
            ->dispatch();
    }
}
```

---

## Monitoramento e Observabilidade

### Logging de Requests

```php
<?php

declare(strict_types=1);

namespace App\LLM;

use App\Contracts\LLMClientInterface;
use App\ValueObjects\LLMResponse;
use Illuminate\Support\Facades\Log;

final class LoggingLLMClient implements LLMClientInterface
{
    public function __construct(
        private LLMClientInterface $client,
    ) {}

    public function chat(array $messages, array $options = []): LLMResponse
    {
        $startTime = microtime(true);

        try {
            $response = $this->client->chat($messages, $options);

            $this->logSuccess($messages, $response, $startTime);

            return $response;
        } catch (\Exception $e) {
            $this->logError($messages, $e, $startTime);
            throw $e;
        }
    }

    private function logSuccess(array $messages, LLMResponse $response, float $startTime): void
    {
        Log::channel('llm')->info('LLM Request Success', [
            'model' => $response->model,
            'input_tokens' => $response->inputTokens,
            'output_tokens' => $response->outputTokens,
            'duration_ms' => (microtime(true) - $startTime) * 1000,
            'user_id' => auth()->id(),
        ]);
    }

    private function logError(array $messages, \Exception $e, float $startTime): void
    {
        Log::channel('llm')->error('LLM Request Failed', [
            'error' => $e->getMessage(),
            'duration_ms' => (microtime(true) - $startTime) * 1000,
            'user_id' => auth()->id(),
        ]);
    }
}
```

### Métricas com Prometheus/StatsD

```php
<?php

declare(strict_types=1);

namespace App\LLM;

use App\Contracts\LLMClientInterface;
use App\ValueObjects\LLMResponse;
use Prometheus\CollectorRegistry;

final class MetricsLLMClient implements LLMClientInterface
{
    public function __construct(
        private LLMClientInterface $client,
        private CollectorRegistry $registry,
    ) {}

    public function chat(array $messages, array $options = []): LLMResponse
    {
        $startTime = microtime(true);

        try {
            $response = $this->client->chat($messages, $options);

            $this->recordMetrics($response, microtime(true) - $startTime);

            return $response;
        } catch (\Exception $e) {
            $this->registry
                ->getOrRegisterCounter('llm', 'requests_failed_total', 'Total failed requests', ['provider'])
                ->incBy(1, [$this->getProvider()]);
            throw $e;
        }
    }

    private function recordMetrics(LLMResponse $response, float $duration): void
    {
        $provider = $this->getProvider();

        // Contador de requests
        $this->registry
            ->getOrRegisterCounter('llm', 'requests_total', 'Total requests', ['provider', 'model'])
            ->incBy(1, [$provider, $response->model]);

        // Tokens usados
        $this->registry
            ->getOrRegisterCounter('llm', 'tokens_total', 'Total tokens', ['provider', 'type'])
            ->incBy($response->inputTokens, [$provider, 'input']);
        $this->registry
            ->getOrRegisterCounter('llm', 'tokens_total', 'Total tokens', ['provider', 'type'])
            ->incBy($response->outputTokens, [$provider, 'output']);

        // Latência
        $this->registry
            ->getOrRegisterHistogram('llm', 'request_duration_seconds', 'Request duration', ['provider'])
            ->observe($duration, [$provider]);
    }

    private function getProvider(): string
    {
        return $this->client->getModelInfo()->provider;
    }
}
```

---

## Segurança

### Sanitização de Input

```php
<?php

declare(strict_types=1);

namespace App\LLM\Security;

final class InputSanitizer
{
    private array $piiPatterns = [
        '/\b\d{3}\.\d{3}\.\d{3}-\d{2}\b/' => '[CPF]',
        '/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/' => '[EMAIL]',
        '/\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b/' => '[CARD]',
        '/\b\d{5}-?\d{3}\b/' => '[CEP]',
    ];

    private array $injectionPatterns = [
        '/ignore previous instructions/i',
        '/disregard all prior/i',
        '/you are now/i',
        '/forget everything/i',
    ];

    public function sanitize(string $input): string
    {
        // Remove PII
        foreach ($this->piiPatterns as $pattern => $replacement) {
            $input = preg_replace($pattern, $replacement, $input);
        }

        return $input;
    }

    public function detectInjection(string $input): bool
    {
        foreach ($this->injectionPatterns as $pattern) {
            if (preg_match($pattern, $input)) {
                return true;
            }
        }
        return false;
    }
}
```

### Content Moderation

```php
<?php

declare(strict_types=1);

namespace App\LLM\Security;

use App\Contracts\LLMClientInterface;

final class ContentModerator
{
    public function __construct(
        private LLMClientInterface $llm,
    ) {}

    public function moderate(string $content): ModerationResult
    {
        $response = $this->llm->chat([
            [
                'role' => 'system',
                'content' => 'Você é um moderador de conteúdo. Analise o texto e identifique conteúdo inapropriado.',
            ],
            [
                'role' => 'user',
                'content' => <<<PROMPT
                Analise o seguinte conteúdo e responda em JSON:
                {
                    "safe": boolean,
                    "categories": ["lista de categorias problemáticas"],
                    "severity": "low|medium|high|critical",
                    "reason": "explicação breve"
                }

                Conteúdo: {$content}
                PROMPT,
            ],
        ], ['temperature' => 0.0]);

        $result = json_decode($response->content, true);

        return new ModerationResult(
            safe: $result['safe'],
            categories: $result['categories'] ?? [],
            severity: $result['severity'] ?? 'low',
            reason: $result['reason'] ?? '',
        );
    }
}
```

---

## Testes

### Mocking do LLM Client

```php
<?php

declare(strict_types=1);

namespace Tests\Unit\Services;

use App\Contracts\LLMClientInterface;
use App\Services\AI\ContentSummarizer;
use App\ValueObjects\LLMResponse;
use Mockery;
use Tests\TestCase;

class ContentSummarizerTest extends TestCase
{
    public function test_summarizes_content(): void
    {
        // Arrange
        $mockLLM = Mockery::mock(LLMClientInterface::class);
        $mockLLM->shouldReceive('chat')
            ->once()
            ->andReturn(new LLMResponse(
                content: 'Este é um resumo do conteúdo.',
                inputTokens: 100,
                outputTokens: 20,
                model: 'gpt-4',
            ));

        $summarizer = new ContentSummarizer($mockLLM);

        // Act
        $result = $summarizer->summarize('Conteúdo longo aqui...');

        // Assert
        $this->assertEquals('Este é um resumo do conteúdo.', $result->summary);
        $this->assertEquals(120, $result->tokensUsed);
    }
}
```

### Fake Client para Testes

```php
<?php

declare(strict_types=1);

namespace Tests\Fakes;

use App\Contracts\LLMClientInterface;
use App\ValueObjects\LLMResponse;
use App\ValueObjects\ModelInfo;
use Generator;

final class FakeLLMClient implements LLMClientInterface
{
    private array $responses = [];
    private int $callCount = 0;

    public function addResponse(string $content, int $inputTokens = 100, int $outputTokens = 50): self
    {
        $this->responses[] = new LLMResponse(
            content: $content,
            inputTokens: $inputTokens,
            outputTokens: $outputTokens,
            model: 'fake-model',
        );
        return $this;
    }

    public function chat(array $messages, array $options = []): LLMResponse
    {
        if (empty($this->responses)) {
            return new LLMResponse(
                content: 'Default fake response',
                inputTokens: 10,
                outputTokens: 5,
                model: 'fake-model',
            );
        }

        return $this->responses[$this->callCount++ % count($this->responses)];
    }

    public function stream(array $messages, array $options = []): Generator
    {
        $response = $this->chat($messages, $options);
        foreach (str_split($response->content, 10) as $chunk) {
            yield $chunk;
        }
    }

    public function embed(string $text): array
    {
        return array_fill(0, 1536, 0.1);
    }

    public function getModelInfo(): ModelInfo
    {
        return new ModelInfo(
            id: 'fake-model',
            provider: 'fake',
            maxContextTokens: 4096,
            maxOutputTokens: 1024,
        );
    }

    public function getCallCount(): int
    {
        return $this->callCount;
    }
}
```

---

## Próximos Passos

1. **Desenvolvimento Assistido por IA** → Fluxo de trabalho diário com IA
2. **ROADMAP** → Guia completo de estudos

---

*Sempre considere custos, latência e limites de rate ao projetar integrações com LLMs.*
