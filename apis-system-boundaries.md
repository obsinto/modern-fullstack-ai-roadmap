# APIs and System Boundaries (APIs e Fronteiras de Sistema)

## Índice

1. [Introdução](#introdução)
2. [Por que isso importa?](#por-que-isso-importa)
3. [Conceitos Fundamentais](#conceitos-fundamentais)
4. [Tipos de Fronteiras](#tipos-de-fronteiras)
5. [Design de API RESTful](#design-de-api-restful)
6. [API Resources no Laravel](#api-resources-no-laravel)
7. [Versionamento de APIs](#versionamento-de-apis)
8. [Autenticação e Autorização](#autenticação-e-autorização)
9. [Contratos e Documentação](#contratos-e-documentação)
10. [Tratamento de Erros](#tratamento-de-erros)
11. [Rate Limiting e Throttling](#rate-limiting-e-throttling)
12. [Anti-patterns Comuns](#anti-patterns-comuns)
13. [Boas Práticas](#boas-práticas)
14. [Exercícios Práticos](#exercícios-práticos)
15. [Recursos Adicionais](#recursos-adicionais)

---

## Introdução

Uma **API (Application Programming Interface)** é um contrato que define como duas partes de software se comunicam. Não é apenas uma coleção de endpoints HTTP — é uma promessa formal sobre o que um sistema aceita como entrada, o que devolve como saída e quais garantias oferece no processo.

**Fronteiras de Sistema (System Boundaries)** são os limites que separam contextos distintos: seu backend do frontend, um módulo do outro, seu sistema de um serviço externo. Toda comunicação que cruza uma fronteira precisa de um protocolo — e esse protocolo é a API.

Uma analogia que funciona bem: pense em uma alfândega. Dentro do país (seu sistema), as coisas funcionam segundo regras internas que você controla. Na fronteira, existe um posto de controle que inspeciona o que entra, valida documentos, aplica regras e registra a passagem. O posto pode ser modernizado internamente sem que os viajantes percebam — desde que o protocolo na fronteira (passaporte, declaração, formulários) permaneça estável. A API é esse posto de controle.

O que torna fronteiras difíceis não é a implementação técnica — é a disciplina de manter o contrato estável enquanto o sistema por trás dele evolui.

---

## Por que isso importa?

### Quando as fronteiras são mal definidas

**Acoplamento entre camadas.** O frontend depende diretamente da estrutura do banco de dados. Renomear uma coluna quebra a aplicação mobile que um parceiro desenvolveu há seis meses — e você nem sabia que ela existia.

**Vazamento de dados.** Um endpoint retorna o objeto completo do Eloquent, incluindo `password_hash`, `remember_token` e aquele campo `internal_notes` que ninguém deveria ver. Alguém descobre fazendo uma requisição simples no DevTools do navegador.

**Overfetching crônico.** A tela de listagem precisa de nome e status, mas o endpoint devolve 47 campos incluindo relacionamentos aninhados. O tempo de resposta sobe, o consumo de banda cresce, e a experiência do usuário degrada — tudo por dados que ninguém usa.

**Segurança por acidente.** Endpoints surgem sem autenticação porque "é só para uso interno". Meses depois, alguém descobre a URL e faz requisições que nunca deveriam ser possíveis.

**Manutenção paralisante.** Qualquer mudança no backend exige coordenação com todos os consumidores da API. Ninguém sabe exatamente quem consome o quê, então ninguém muda nada — e o sistema apodrece.

### Quando as fronteiras são bem definidas

**Independência real.** Frontend e backend evoluem em ritmos diferentes. O time mobile pode trabalhar com a API v1 enquanto o time web já migra para a v2.

**Reutilização natural.** A mesma API serve a aplicação web, o app mobile, o dashboard administrativo e a integração com parceiros — sem código duplicado.

**Segurança por design.** Cada fronteira tem controles explícitos: autenticação, autorização, validação de input, rate limiting. Nada passa sem inspeção.

**Testabilidade.** Cada lado da fronteira pode ser testado isoladamente. Os testes de integração verificam o contrato; os testes unitários verificam a lógica interna.

---

## Conceitos Fundamentais

### 1. Interface vs Implementação

O conceito mais importante em design de APIs é a separação entre **o que** o consumidor vê e **como** o sistema funciona internamente. O consumidor interage com a interface; a implementação é problema seu.

```php
// ❌ Implementação exposta — o cliente vê a estrutura do banco
Route::get('/users/{id}', function ($id) {
    return DB::table('users')
        ->join('roles', 'users.role_id', '=', 'roles.id')
        ->select('users.*', 'roles.name as role_name')
        ->where('users.id', $id)
        ->first();
});
// Se você renomear a tabela ou mudar o join, a API quebra.

// ✅ Interface estável — implementação escondida
Route::get('/users/{user}', function (User $user) {
    return new UserResource($user);
});
// Você pode migrar de MySQL para PostgreSQL, trocar o ORM,
// redesenhar o schema — a API continua devolvendo o mesmo JSON.
```

Essa separação não é luxo. É o que permite que o sistema evolua sem quebrar contratos com quem consome.

### 2. Contratos Explícitos

Um contrato de API define formalmente: quais endpoints existem, o que cada um aceita, o que devolve, e quais erros são possíveis. O formato mais comum é o OpenAPI (Swagger):

```php
/**
 * @OA\Get(
 *     path="/api/v1/products/{id}",
 *     summary="Buscar produto por ID",
 *     @OA\Parameter(
 *         name="id",
 *         in="path",
 *         required=true,
 *         @OA\Schema(type="integer")
 *     ),
 *     @OA\Response(
 *         response=200,
 *         description="Produto encontrado",
 *         @OA\JsonContent(ref="#/components/schemas/Product")
 *     ),
 *     @OA\Response(response=404, description="Produto não encontrado")
 * )
 */
```

Contratos explícitos servem a dois propósitos: documentar a API para consumidores e validar que a implementação respeita a especificação. Sem contrato explícito, o comportamento da API é definido por "o que acontece quando eu faço uma requisição" — o que é frágil e não-verificável.

### 3. Compatibilidade Retroativa (Backward Compatibility)

Quando consumidores dependem da sua API, você não pode simplesmente mudar o formato de resposta. A regra é: **adicione, nunca remova ou renomeie**.

```php
// V1 — formato original
class ProductResourceV1 extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'price' => $this->price, // número simples
        ];
    }
}

// V2 — evolução compatível
class ProductResourceV2 extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'price' => [                    // estrutura enriquecida
                'amount' => $this->price,
                'currency' => 'BRL',
                'formatted' => 'R$ ' . number_format($this->price, 2, ',', '.'),
            ],
            'stock' => $this->stock,         // campo novo — clientes antigos ignoram
        ];
    }
}
```

Perceba que `price` mudou de número para objeto. Isso é uma **breaking change** — por isso precisa de uma nova versão. Adicionar `stock` como campo novo, por outro lado, é seguro: consumidores que não conhecem o campo simplesmente o ignoram.

### 4. Abstração em Camadas

Cada camada do sistema tem uma responsabilidade e uma fronteira com a camada adjacente:

```
┌──────────────────────────────────┐
│     Cliente (Web / Mobile)       │
└───────────────┬──────────────────┘
                │  HTTP / JSON
┌───────────────▼──────────────────┐
│         API Layer                │
│  Autenticação, Rate Limiting,    │
│  Validação de Input, Resources   │
└───────────────┬──────────────────┘
                │
┌───────────────▼──────────────────┐
│      Application Layer           │
│  Actions, Services, DTOs         │
│  Orquestração de lógica          │
└───────────────┬──────────────────┘
                │
┌───────────────▼──────────────────┐
│        Domain Layer              │
│  Models, Repositories,           │
│  Regras de Negócio, Enums        │
└───────────────┬──────────────────┘
                │
┌───────────────▼──────────────────┐
│       Infrastructure             │
│  Database, Cache, Filesystem,    │
│  APIs Externas, Queue            │
└──────────────────────────────────┘
```

A regra fundamental: cada camada só conhece a camada imediatamente abaixo dela, e se comunica através de interfaces (contratos), não implementações concretas. O Controller não faz queries SQL; o Service não sabe se o storage é local ou S3.

---

## Tipos de Fronteiras

### 1. Fronteira Externa (Public API)

A interface que o mundo externo enxerga. É onde autenticação, autorização e rate limiting são obrigatórios — aqui você não controla quem está do outro lado.

```php
// routes/api.php
Route::prefix('v1')->group(function () {

    // Endpoints públicos — sem autenticação
    Route::get('/products', [ProductController::class, 'index']);
    Route::get('/products/{product}', [ProductController::class, 'show']);

    // Endpoints autenticados — requerem token Sanctum
    Route::middleware('auth:sanctum')->group(function () {
        Route::post('/orders', [OrderController::class, 'store']);
        Route::get('/orders/{order}', [OrderController::class, 'show']);
    });

    // Endpoints administrativos — autenticação + verificação de role
    Route::middleware(['auth:sanctum', 'admin'])->group(function () {
        Route::post('/products', [ProductController::class, 'store']);
        Route::delete('/products/{product}', [ProductController::class, 'destroy']);
    });
});
```

A organização por nível de acesso não é apenas convenção — é uma decisão de segurança. Endpoints públicos, autenticados e administrativos têm requisitos diferentes de throttling, logging e monitoramento.

### 2. Fronteira Interna (Module Boundaries)

Dentro do mesmo sistema, módulos diferentes devem se comunicar através de interfaces, não acessando diretamente o banco de dados ou os models um do outro. Isso é especialmente importante em sistemas maiores onde módulos como Licitações, RH e Financeiro coexistem.

```php
// O módulo de Licitações precisa consultar dados de funcionários.
// Ele NÃO acessa o banco de RH diretamente — usa uma interface.

namespace App\Modules\Licitacao\Services;

class ProcessoLicitacaoService
{
    public function __construct(
        private FuncionarioRepositoryInterface $funcionarioRepo,
    ) {}

    public function designarResponsavel(Processo $processo, int $funcionarioId): void
    {
        $funcionario = $this->funcionarioRepo->findOrFail($funcionarioId);

        if (! $funcionario->podeSerResponsavelLicitacao()) {
            throw new FuncionarioNaoAutorizadoException();
        }

        $processo->responsavel_id = $funcionario->id;
        $processo->save();
    }
}
```

```php
// A implementação concreta fica no módulo de RH
namespace App\Modules\RH\Repositories;

class EloquentFuncionarioRepository implements FuncionarioRepositoryInterface
{
    public function findOrFail(int $id): Funcionario
    {
        return Funcionario::findOrFail($id);
    }
}
```

A vantagem é clara: se amanhã o módulo de RH migrar para um microserviço separado, basta trocar a implementação por uma que faz chamadas HTTP — o módulo de Licitações não precisa ser tocado.

### 3. Fronteira de Infraestrutura

Isola a lógica de negócio dos detalhes técnicos de infraestrutura. Seu Service não deveria saber (nem se importar) se os arquivos estão no disco local, no S3 ou no MinIO.

```php
// A interface define O QUE o sistema precisa
interface FileStorageInterface
{
    public function store(string $path, mixed $content): string;
    public function get(string $path): string;
    public function delete(string $path): void;
}

// Implementação para disco local
class LocalFileStorage implements FileStorageInterface
{
    public function store(string $path, mixed $content): string
    {
        Storage::disk('local')->put($path, $content);
        return $path;
    }

    public function get(string $path): string
    {
        return Storage::disk('local')->get($path);
    }

    public function delete(string $path): void
    {
        Storage::disk('local')->delete($path);
    }
}

// Implementação para S3 — mesma interface, comportamento diferente
class S3FileStorage implements FileStorageInterface
{
    public function store(string $path, mixed $content): string
    {
        Storage::disk('s3')->put($path, $content);
        return Storage::disk('s3')->url($path);
    }

    public function get(string $path): string
    {
        return Storage::disk('s3')->get($path);
    }

    public function delete(string $path): void
    {
        Storage::disk('s3')->delete($path);
    }
}
```

```php
// O Service Provider decide qual implementação usar em runtime
class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(
            FileStorageInterface::class,
            config('filesystems.default') === 's3'
                ? S3FileStorage::class
                : LocalFileStorage::class
        );
    }
}
```

A troca de disco local para S3 é uma mudança de uma linha na configuração. Nenhum Service, Controller ou Action precisa saber que isso aconteceu.

---

## Design de API RESTful

### Princípios

REST não é uma tecnologia — é um conjunto de constraints arquiteturais. Os quatro mais relevantes para o dia a dia:

**Recursos como substantivos.** URLs representam "coisas" (produtos, pedidos, usuários), não "ações". A ação vem do verbo HTTP.

**Verbos HTTP como operações.** GET lê, POST cria, PUT/PATCH atualiza, DELETE remove. Cada verbo tem semântica definida pela especificação HTTP.

**Stateless.** Cada requisição carrega toda a informação necessária para ser processada. O servidor não mantém "sessão" entre requisições da API.

**Representações consistentes.** O mesmo recurso sempre retorna no mesmo formato, independente de quem pediu.

### Estrutura de URLs

```php
// Operações CRUD padrão — seguem a convenção RESTful
GET    /api/v1/products              // Listar produtos
GET    /api/v1/products/{id}         // Buscar produto específico
POST   /api/v1/products              // Criar produto
PUT    /api/v1/products/{id}         // Substituir produto inteiro
PATCH  /api/v1/products/{id}         // Atualizar campos específicos
DELETE /api/v1/products/{id}         // Remover produto

// Relacionamentos aninhados
GET    /api/v1/products/{id}/reviews // Listar reviews de um produto
POST   /api/v1/products/{id}/reviews // Adicionar review a um produto

// Ações que não se encaixam em CRUD
// Quando a operação é uma transição de estado ou comando,
// use POST com um verbo na URL — é uma concessão pragmática.
POST   /api/v1/products/{id}/publish // Publicar produto
POST   /api/v1/orders/{id}/cancel    // Cancelar pedido
```

```php
// ❌ Padrões a evitar
GET    /api/getProducts              // Verbo na URL — redundante com GET
POST   /api/createProduct            // Verbo na URL — redundante com POST
GET    /api/deleteProduct?id=1       // GET não deve ter efeito colateral
POST   /api/products/list            // POST para leitura
```

### Filtros, Paginação e Ordenação

Esses três mecanismos aparecem em praticamente todo endpoint de listagem. Vale a pena estabelecer um padrão desde o início:

```php
class ProductController extends Controller
{
    public function index(Request $request)
    {
        $query = Product::query();

        // Filtros — cada parâmetro de query aplica um escopo
        when($request->has('category'), fn () =>
            $query->where('category_id', $request->integer('category'))
        );

        when($request->has('search'), fn () =>
            $query->where('name', 'like', '%' . $request->string('search') . '%')
        );

        when($request->has('min_price'), fn () =>
            $query->where('price', '>=', $request->float('min_price'))
        );

        when($request->has('max_price'), fn () =>
            $query->where('price', '<=', $request->float('max_price'))
        );

        // Ordenação — com whitelist para evitar SQL injection
        $allowedSorts = ['name', 'price', 'created_at'];
        $sortBy = in_array($request->input('sort_by'), $allowedSorts)
            ? $request->input('sort_by')
            : 'created_at';
        $sortOrder = $request->input('sort_order') === 'asc' ? 'asc' : 'desc';

        $query->orderBy($sortBy, $sortOrder);

        // Paginação — com limite máximo para proteção
        $perPage = min($request->integer('per_page', 15), 100);

        return ProductResource::collection($query->paginate($perPage));
    }
}
```

Exemplo de chamada:
```
GET /api/v1/products?category=3&min_price=100&sort_by=price&sort_order=asc&per_page=20
```

Note a whitelist em `$allowedSorts`. Sem ela, um atacante poderia passar `sort_by=password` ou usar subqueries. Pequenos detalhes como esse são a diferença entre uma API segura e uma vulnerável.

### Formato de Resposta Padronizado

Defina um formato e mantenha-o consistente em toda a API. Aqui está um padrão pragmático:

```php
// Sucesso com dados (201 Created)
{
    "data": {
        "id": 42,
        "name": "Notebook Dell",
        "price": { "amount": 4599.00, "currency": "BRL" }
    }
}

// Sucesso com coleção paginada (200 OK)
{
    "data": [ ... ],
    "meta": {
        "total": 150,
        "current_page": 1,
        "last_page": 10,
        "per_page": 15
    },
    "links": {
        "first": "/api/v1/products?page=1",
        "last": "/api/v1/products?page=10",
        "prev": null,
        "next": "/api/v1/products?page=2"
    }
}

// Erro de validação (422 Unprocessable Entity)
{
    "message": "Validation failed",
    "errors": {
        "name": ["The name field is required."],
        "price": ["The price must be greater than 0."]
    }
}

// Erro genérico (404 Not Found)
{
    "message": "Product not found"
}
```

O formato é previsível: `data` para o payload, `meta`/`links` para metadados de paginação, `message`/`errors` para problemas. Consumidores podem escrever código genérico para lidar com qualquer resposta da sua API.

---

## API Resources no Laravel

Resources são a camada de transformação entre seus Models (domínio interno) e o JSON que a API devolve (contrato externo). Nunca retorne um Model diretamente — sempre passe por um Resource.

### Resource básico

```php
namespace App\Http\Resources;

class UserResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'created_at' => $this->created_at->toIso8601String(),

            // Campo visível apenas para quem tem permissão
            'phone' => $this->when(
                $request->user()?->can('view-phone'),
                $this->phone
            ),

            // Relacionamentos — só incluídos se foram carregados via eager loading
            'role' => new RoleResource($this->whenLoaded('role')),
            'posts' => PostResource::collection($this->whenLoaded('posts')),
        ];
    }
}
```

O método `whenLoaded` é essencial: ele evita queries N+1 acidentais. Se o controller não fez `->with('role')`, o campo simplesmente não aparece na resposta — sem erro, sem query extra.

### Resource Collection com metadados

```php
class UserCollection extends ResourceCollection
{
    public function toArray($request): array
    {
        return [
            'data' => $this->collection,
            'meta' => [
                'total' => $this->total(),
                'current_page' => $this->currentPage(),
                'last_page' => $this->lastPage(),
            ],
            'links' => [
                'first' => $this->url(1),
                'last' => $this->url($this->lastPage()),
                'prev' => $this->previousPageUrl(),
                'next' => $this->nextPageUrl(),
            ],
        ];
    }
}
```

### Resources aninhados

```php
class OrderResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'status' => $this->status,
            'total' => $this->total,

            'customer' => [
                'id' => $this->customer->id,
                'name' => $this->customer->name,
                'email' => $this->customer->email,
            ],

            'items' => $this->items->map(fn ($item) => [
                'product_id' => $item->product_id,
                'product_name' => $item->product->name,
                'quantity' => $item->quantity,
                'price' => $item->price,
                'subtotal' => $item->quantity * $item->price,
            ]),

            'created_at' => $this->created_at->toIso8601String(),
        ];
    }
}
```

### Atributos condicionais

Resources permitem controlar a visibilidade de campos com granularidade fina, sem lógica condicional espalhada pelo controller:

```php
class ProductResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'description' => $this->description,
            'price' => $this->price,

            // Custo visível apenas para administradores
            'cost' => $this->when(
                $request->user()?->isAdmin(),
                $this->cost
            ),

            // Alerta aparece apenas quando estoque está baixo
            'low_stock_alert' => $this->when(
                $this->stock < 10,
                'Estoque baixo!'
            ),

            // Bloco inteiro condicional
            $this->mergeWhen($request->user()?->isAdmin(), [
                'internal_notes' => $this->internal_notes,
                'supplier_id' => $this->supplier_id,
            ]),
        ];
    }
}
```

### Configuração do wrapper

```php
// Remover wrapping global (útil quando você controla manualmente)
JsonResource::withoutWrapping();

// Ou definir wrapper customizado por Resource
class ProductResource extends JsonResource
{
    public static $wrap = 'product';
    // Resposta: { "product": { ... } }
}

class ProductCollection extends ResourceCollection
{
    public static $wrap = 'products';
    // Resposta: { "products": [ ... ] }
}
```

---

## Versionamento de APIs

### Por URL (abordagem mais comum e recomendada)

A versão faz parte da URL. É visível, fácil de entender, e funciona bem com ferramentas de monitoramento e logging.

```php
// routes/api.php
Route::prefix('v1')->name('api.v1.')->group(function () {
    Route::apiResource('products', App\Http\Controllers\V1\ProductController::class);
});

Route::prefix('v2')->name('api.v2.')->group(function () {
    Route::apiResource('products', App\Http\Controllers\V2\ProductController::class);
});
```

### Por Header (alternativa quando a URL deve ser estável)

```php
class ApiVersionMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $version = $request->header('Accept-Version', 'v1');

        $request->attributes->set('api_version', $version);

        return $next($request);
    }
}

class ProductController extends Controller
{
    public function index(Request $request)
    {
        $version = $request->attributes->get('api_version');

        return match ($version) {
            'v1' => ProductResourceV1::collection(Product::paginate()),
            'v2' => ProductResourceV2::collection(Product::paginate()),
            default => response()->json(['message' => 'Versão não suportada'], 400),
        };
    }
}
```

### Estratégia de deprecação

Quando uma versão precisa ser aposentada, não basta desligá-la. Dê tempo para os consumidores migrarem, usando headers que comunicam a situação:

```php
class ProductControllerV1 extends Controller
{
    public function index()
    {
        return response()
            ->json(ProductResourceV1::collection(Product::paginate()))
            ->header('Warning', '299 - "API v1 está deprecated. Migre para v2."')
            ->header('Sunset', 'Sat, 31 Dec 2026 23:59:59 GMT');
    }
}
```

O header `Sunset` (RFC 8594) comunica formalmente a data em que a versão será desligada. Clientes bem escritos podem monitorar esse header e alertar seus desenvolvedores automaticamente.

---

## Autenticação e Autorização

Autenticação responde "quem é você?". Autorização responde "o que você pode fazer?". São preocupações distintas e devem ser implementadas em camadas separadas.

### Autenticação com Laravel Sanctum

Sanctum é a escolha recomendada para a maioria dos casos: SPAs com Inertia, apps mobile, e APIs para terceiros.

```php
// User Model — habilitar tokens
class User extends Authenticatable
{
    use HasApiTokens;
}
```

```php
class AuthController extends Controller
{
    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        if (! Auth::attempt($credentials)) {
            return response()->json([
                'message' => 'Credenciais inválidas',
            ], 401);
        }

        $user = User::where('email', $request->email)->first();
        $token = $user->createToken('api-token')->plainTextToken;

        return response()->json([
            'user' => new UserResource($user),
            'token' => $token,
        ]);
    }

    public function logout(Request $request)
    {
        $request->user()->currentAccessToken()->delete();

        return response()->json([
            'message' => 'Logout realizado com sucesso',
        ]);
    }
}

// Rotas protegidas
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/user', fn (Request $request) => new UserResource($request->user()));
});
```

### Tokens com habilidades (Abilities)

Para integrações com terceiros, é comum criar tokens com permissões restritas — um token de um app mobile não precisa ter acesso administrativo:

```php
// Criar token com escopo limitado
$token = $user->createToken('mobile-app', [
    'order:create',
    'order:view',
    'product:view',
])->plainTextToken;

// Verificar habilidade no endpoint
Route::middleware('auth:sanctum')->post('/orders', function (Request $request) {
    if (! $request->user()->tokenCan('order:create')) {
        return response()->json(['message' => 'Ação não permitida para este token'], 403);
    }

    // Processar criação do pedido...
});
```

### Autorização com Policies

Enquanto o Sanctum cuida da autenticação, Policies definem as regras de autorização — quem pode fazer o quê com qual recurso:

```php
class OrderPolicy
{
    public function view(User $user, Order $order): bool
    {
        return $user->id === $order->user_id || $user->isAdmin();
    }

    public function update(User $user, Order $order): bool
    {
        return $user->id === $order->user_id
            && $order->status === OrderStatus::PENDING;
    }

    public function delete(User $user, Order $order): bool
    {
        return $user->isAdmin();
    }
}
```

```php
class OrderController extends Controller
{
    public function show(Order $order)
    {
        $this->authorize('view', $order);

        return new OrderResource($order);
    }

    public function update(UpdateOrderRequest $request, Order $order)
    {
        $this->authorize('update', $order);

        $order->update($request->validated());

        return new OrderResource($order);
    }
}
```

A Policy centraliza a lógica de autorização. Se amanhã a regra mudar (ex: supervisores também podem editar pedidos), a mudança é feita em um único lugar.

---

## Contratos e Documentação

### OpenAPI / Swagger

A documentação gerada a partir de anotações no código tem a vantagem de evoluir junto com a implementação:

```php
// Instalação
// composer require darkaonline/l5-swagger

/**
 * @OA\Get(
 *     path="/api/v1/products",
 *     summary="Listar produtos",
 *     tags={"Products"},
 *     @OA\Parameter(
 *         name="category",
 *         in="query",
 *         description="Filtrar por categoria",
 *         required=false,
 *         @OA\Schema(type="integer")
 *     ),
 *     @OA\Parameter(
 *         name="per_page",
 *         in="query",
 *         description="Itens por página",
 *         required=false,
 *         @OA\Schema(type="integer", default=15)
 *     ),
 *     @OA\Response(
 *         response=200,
 *         description="Lista de produtos",
 *         @OA\JsonContent(
 *             type="object",
 *             @OA\Property(
 *                 property="data",
 *                 type="array",
 *                 @OA\Items(ref="#/components/schemas/Product")
 *             ),
 *             @OA\Property(property="meta", type="object")
 *         )
 *     ),
 *     security={{"sanctum": {}}}
 * )
 */
```

```php
/**
 * @OA\Schema(
 *     schema="Product",
 *     type="object",
 *     required={"id", "name", "price"},
 *     @OA\Property(property="id", type="integer", example=1),
 *     @OA\Property(property="name", type="string", example="Notebook Dell"),
 *     @OA\Property(property="price", type="number", format="float", example=4599.99),
 *     @OA\Property(property="description", type="string", nullable=true)
 * )
 */
```

### Testes como documentação viva

Anotações Swagger podem ficar desatualizadas. Testes de feature, por outro lado, **falham** quando a API muda — são a forma mais confiável de documentar o contrato:

```php
class ProductApiTest extends TestCase
{
    /** @test */
    public function lista_produtos_com_estrutura_correta(): void
    {
        Product::factory()->count(5)->create();

        $response = $this->getJson('/api/v1/products');

        $response
            ->assertOk()
            ->assertJsonStructure([
                'data' => [
                    '*' => ['id', 'name', 'price', 'description'],
                ],
                'meta' => ['total', 'current_page'],
            ])
            ->assertJsonCount(5, 'data');
    }

    /** @test */
    public function filtra_produtos_por_categoria(): void
    {
        $categoria = Category::factory()->create();
        Product::factory()->count(3)->create(['category_id' => $categoria->id]);
        Product::factory()->count(2)->create(); // outras categorias

        $response = $this->getJson("/api/v1/products?category={$categoria->id}");

        $response
            ->assertOk()
            ->assertJsonCount(3, 'data');
    }

    /** @test */
    public function requer_autenticacao_para_criar_produto(): void
    {
        $response = $this->postJson('/api/v1/products', [
            'name' => 'Produto Teste',
            'price' => 99.90,
        ]);

        $response->assertUnauthorized();
    }

    /** @test */
    public function retorna_erro_de_validacao_com_formato_padrao(): void
    {
        $user = User::factory()->admin()->create();

        $response = $this->actingAs($user)->postJson('/api/v1/products', []);

        $response
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['name', 'price']);
    }
}
```

Esses testes servem como especificação executável: qualquer desenvolvedor que queira entender o contrato da API pode ler os testes e ver exatamente o que cada endpoint aceita e devolve.

---

## Tratamento de Erros

### Exception Handler para APIs

No Laravel 11+, o tratamento de exceções é configurado em `bootstrap/app.php`. Para APIs, o objetivo é traduzir exceções internas em respostas JSON padronizadas:

```php
// bootstrap/app.php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->render(function (Throwable $e, Request $request) {
        if (! $request->expectsJson()) {
            return null; // Deixar o handler padrão cuidar de requisições web
        }

        return match (true) {
            $e instanceof ModelNotFoundException => response()->json([
                'message' => 'Recurso não encontrado',
            ], 404),

            $e instanceof ValidationException => response()->json([
                'message' => 'Falha na validação',
                'errors' => $e->errors(),
            ], 422),

            $e instanceof AuthenticationException => response()->json([
                'message' => 'Não autenticado',
            ], 401),

            $e instanceof AuthorizationException => response()->json([
                'message' => 'Ação não autorizada',
            ], 403),

            default => response()->json([
                'message' => config('app.debug')
                    ? $e->getMessage()
                    : 'Erro interno do servidor',
            ], $e instanceof HttpException ? $e->getStatusCode() : 500),
        };
    });
})
```

Em produção, nunca exponha stack traces ou mensagens internas de exceção. A flag `app.debug` controla isso: em desenvolvimento mostra tudo, em produção mostra apenas uma mensagem genérica.

### Exceções de domínio com renderização própria

Para erros de negócio que precisam devolver dados específicos ao cliente, crie exceções que sabem se renderizar:

```php
namespace App\Exceptions;

class InsufficientStockException extends \DomainException
{
    public function __construct(
        public readonly Product $product,
        public readonly int $requested,
        public readonly int $available,
    ) {
        parent::__construct(
            "Estoque insuficiente para {$product->name}. "
            . "Solicitado: {$requested}, Disponível: {$available}"
        );
    }

    public function render(): JsonResponse
    {
        return response()->json([
            'message' => $this->getMessage(),
            'details' => [
                'product_id' => $this->product->id,
                'requested' => $this->requested,
                'available' => $this->available,
            ],
        ], 409); // 409 Conflict
    }
}
```

```php
// Uso — a exceção carrega contexto suficiente para o cliente reagir
public function addToOrder(Product $product, int $quantity): void
{
    if ($product->stock < $quantity) {
        throw new InsufficientStockException($product, $quantity, $product->stock);
    }

    // Processar...
}
```

O status 409 (Conflict) é mais semântico que 400 para esse caso: indica que a requisição é válida sintaticamente, mas conflita com o estado atual do recurso.

---

## Rate Limiting e Throttling

Rate limiting protege sua API contra abuso, seja acidental (um bug em um cliente que entra em loop) ou intencional (ataques de força bruta, scraping).

### Limites por perfil de acesso

```php
// app/Providers/AppServiceProvider.php (Laravel 11+)
// ou RouteServiceProvider (Laravel 10)

// Leitura — mais permissivo
RateLimiter::for('api-reads', function (Request $request) {
    return $request->user()
        ? Limit::perMinute(120)->by($request->user()->id)
        : Limit::perMinute(30)->by($request->ip());
});

// Escrita — mais restritivo
RateLimiter::for('api-writes', function (Request $request) {
    return $request->user()
        ? Limit::perMinute(20)->by($request->user()->id)
        : Limit::perMinute(5)->by($request->ip());
});

// Admin — sem limite
RateLimiter::for('api-admin', function (Request $request) {
    return $request->user()?->isAdmin()
        ? Limit::none()
        : Limit::perMinute(60)->by($request->user()?->id ?? $request->ip());
});
```

```php
// Uso nas rotas
Route::middleware('throttle:api-reads')->get('/products', ...);
Route::middleware('throttle:api-writes')->post('/products', ...);
```

### Limites baseados em plano de assinatura

```php
RateLimiter::for('api', function (Request $request) {
    $user = $request->user();

    if (! $user) {
        return Limit::perMinute(10)->by($request->ip());
    }

    return match ($user->subscription_plan) {
        'free'       => Limit::perMinute(60)->by($user->id),
        'pro'        => Limit::perMinute(300)->by($user->id),
        'enterprise' => Limit::perMinute(1000)->by($user->id),
        default      => Limit::perMinute(60)->by($user->id),
    };
});
```

### Headers informativos

Bons clientes de API monitoram os headers de rate limit para ajustar seu comportamento:

```php
class RateLimitHeadersMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request);

        // O Laravel já adiciona X-RateLimit-Limit e X-RateLimit-Remaining
        // automaticamente quando o middleware throttle é usado.
        // Este middleware existe para cenários onde você precisa de controle adicional.

        return $response;
    }
}
```

Os headers `X-RateLimit-Limit`, `X-RateLimit-Remaining` e `Retry-After` (quando o limite é excedido) permitem que clientes implementem backoff inteligente ao invés de continuar martelando um endpoint que vai rejeitar tudo.

---

## Anti-patterns Comuns

### 1. Retornar Models diretamente

```php
// ❌ O Eloquent serializa TODOS os campos, incluindo sensíveis
Route::get('/users/{user}', fn (User $user) => $user);
// Resposta inclui password, remember_token, etc.

// ✅ Resource controla exatamente o que sai
Route::get('/users/{user}', fn (User $user) => new UserResource($user));
```

Mesmo que o Model tenha `$hidden`, depender disso é frágil. Um desenvolvedor pode remover um campo de `$hidden` sem perceber que ele é serializado em uma resposta de API. O Resource torna a decisão explícita.

### 2. Queries N+1

```php
// ❌ 1 query para orders + N queries para customer + N queries para items
class OrderController extends Controller
{
    public function index()
    {
        $orders = Order::all();

        return $orders->map(fn ($order) => [
            'id' => $order->id,
            'customer' => $order->customer->name,     // N queries
            'items_count' => $order->items->count(),   // N queries
        ]);
    }
}

// ✅ 3 queries no total, independente de quantos orders existam
class OrderController extends Controller
{
    public function index()
    {
        $orders = Order::with(['customer', 'items'])->paginate();

        return OrderResource::collection($orders);
    }
}
```

Com 100 pedidos, a primeira versão faz ~201 queries. A segunda faz 3. Em produção, isso é a diferença entre 200ms e 5 segundos de resposta.

### 3. Inconsistência de formato

```php
// ❌ Cada endpoint retorna em formato diferente
GET  /products/1    →  { id: 1, name: "..." }
POST /products      →  { product: { id: 2, name: "..." } }
GET  /products      →  [{ id: 1 }, { id: 2 }]

// ✅ Formato consistente — sempre com wrapper 'data'
GET  /products/1    →  { "data": { id: 1, name: "..." } }
POST /products      →  { "data": { id: 2, name: "..." } }
GET  /products      →  { "data": [{ id: 1 }, { id: 2 }], "meta": { ... } }
```

Consumidores não deveriam precisar memorizar qual formato cada endpoint usa. Consistência reduz bugs no cliente e tempo de integração.

### 4. GET com efeitos colaterais

```php
// ❌ GET não deve modificar estado — viola a especificação HTTP
Route::get('/orders/{order}/cancel', function (Order $order) {
    $order->update(['status' => 'cancelled']);
    return redirect()->back();
});

// ✅ POST para operações que modificam estado
Route::post('/orders/{order}/cancel', function (Order $order) {
    $order->update(['status' => 'cancelled']);
    return response()->json(['message' => 'Pedido cancelado']);
});
```

GET deve ser idempotente e seguro (sem side effects). Crawlers, proxies e caches assumem isso. Um crawler que acessa `/orders/123/cancel` vai cancelar o pedido — e isso é culpa do design da API, não do crawler.

---

## Boas Práticas

### 1. Form Requests para validação na fronteira

Toda validação de input acontece antes de chegar ao controller. Isso é a primeira linha de defesa da fronteira:

```php
class StoreProductRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create-products');
    }

    public function rules(): array
    {
        return [
            'name' => 'required|string|max:255',
            'price' => 'required|numeric|min:0',
            'category_id' => 'required|exists:categories,id',
        ];
    }

    public function messages(): array
    {
        return [
            'price.min' => 'O preço não pode ser negativo.',
        ];
    }
}
```

### 2. Versione desde o início

Mesmo que você tenha apenas uma versão, estruture as rotas para facilitar a adição de v2 no futuro:

```php
// Pouco esforço agora, grande economia depois
Route::prefix('v1')->name('api.v1.')->group(function () {
    Route::apiResource('products', ProductController::class);
});
```

### 3. Configure CORS corretamente

Em uma stack com SPA (Vue.js + Inertia), o CORS precisa permitir que o frontend faça requisições ao backend:

```php
// config/cors.php
return [
    'paths' => ['api/*'],
    'allowed_methods' => ['*'],
    'allowed_origins' => [
        config('app.frontend_url'), // Nunca use '*' em produção
    ],
    'allowed_headers' => ['*'],
    'exposed_headers' => ['X-RateLimit-Limit', 'X-RateLimit-Remaining'],
    'max_age' => 3600,
    'supports_credentials' => true,
];
```

### 4. Implemente idempotência para operações críticas

Para endpoints de criação que envolvem pagamento ou recursos não-reversíveis, use chaves de idempotência para evitar duplicação por retry:

```php
class CreateOrderController extends Controller
{
    public function store(StoreOrderRequest $request)
    {
        $idempotencyKey = $request->header('Idempotency-Key');

        if ($idempotencyKey) {
            $cached = Cache::get("idempotency:{$idempotencyKey}");

            if ($cached) {
                // Requisição duplicada — retorna o resultado anterior
                return response()->json($cached, 200);
            }
        }

        $order = $this->orderService->create($request->validated());

        $response = new OrderResource($order);

        if ($idempotencyKey) {
            Cache::put(
                "idempotency:{$idempotencyKey}",
                $response,
                now()->addHours(24)
            );
        }

        return response()->json($response, 201);
    }
}
```

O cliente envia um UUID único no header `Idempotency-Key`. Se a mesma chave for enviada novamente (por retry após timeout, por exemplo), o servidor retorna o resultado original ao invés de criar um pedido duplicado.

---

## Exercícios Práticos

### Exercício 1: Criar uma API RESTful completa

Implemente uma API para gerenciar Processos de Licitação com os seguintes requisitos:

- CRUD completo de processos com Resources dedicados.
- Filtros por status, modalidade e intervalo de datas.
- Relacionamentos: documentos e lances carregados via eager loading.
- Autenticação via Sanctum com tokens que possuem abilities específicas.
- Policies para autorização: apenas o criador ou admin pode editar.
- Versionamento por URL (prefixo `v1`).
- Rate limiting diferenciado para leitura e escrita.
- Pelo menos 5 testes de feature cobrindo o contrato da API.

### Exercício 2: Refatorar um endpoint existente

Pegue um endpoint como este e aplique todas as boas práticas discutidas:

```php
// ANTES — tudo errado de uma vez
Route::get('/requerimentos', function () {
    return Requerimento::all();
});
```

Sua versão refatorada deve incluir: Resource para formatar a resposta, paginação com limite máximo, filtros por status, cidadão e data, eager loading de relacionamentos, cache de 5 minutos com invalidação, autenticação obrigatória, e um teste de feature que valida a estrutura da resposta.

### Exercício 3: Implementar sistema de Webhooks

Crie um sistema que notifica endpoints externos quando eventos acontecem na sua aplicação:

```php
class WebhookService
{
    public function dispatch(string $event, array $payload): void
    {
        $subscribers = Webhook::query()
            ->where('event', $event)
            ->where('active', true)
            ->get();

        foreach ($subscribers as $webhook) {
            // Dispatch assíncrono para não bloquear o request
            SendWebhookJob::dispatch($webhook, [
                'event' => $event,
                'data' => $payload,
                'timestamp' => now()->toIso8601String(),
                'signature' => $this->sign($webhook->secret, $payload),
            ]);
        }
    }

    private function sign(string $secret, array $payload): string
    {
        return hash_hmac('sha256', json_encode($payload), $secret);
    }
}
```

O exercício consiste em: criar a migration para a tabela `webhooks`, implementar o Job que faz a requisição HTTP com retry e timeout, adicionar verificação de assinatura no lado receptor, e criar um endpoint para que consumidores registrem e gerenciem seus webhooks.

---

## Recursos Adicionais

### Ferramentas

- **Postman / Insomnia** — clientes HTTP para testar e documentar APIs interativamente.
- **Swagger UI** — interface web gerada automaticamente a partir das anotações OpenAPI.
- **Laravel Sanctum** — autenticação leve para SPAs, mobile e tokens de API.
- **Laravel Telescope** — monitora requisições, queries, jobs e eventos em tempo de desenvolvimento.

### Leitura

- [Laravel Eloquent Resources](https://laravel.com/docs/eloquent-resources) — documentação oficial sobre API Resources.
- [RESTful API Design Best Practices](https://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api) — guia pragmático e completo sobre design de APIs REST.
- [API Security Checklist](https://github.com/shieldfy/API-Security-Checklist) — lista de verificação de segurança para APIs.

### Especificações e padrões

- **OpenAPI 3.0** — o padrão de mercado para documentação de APIs REST.
- **JSON:API** — especificação que define um formato consistente para respostas JSON.
- **RFC 8594 (Sunset Header)** — padrão para comunicar deprecação de endpoints.

---

## Conclusão

APIs e fronteiras bem definidas são o que permite que um sistema cresça sem que cada mudança exija coordenação com todos os consumidores. Os princípios fundamentais são:

**Separe interface de implementação.** O que o consumidor vê (o contrato) deve ser estável. O que está por trás (a implementação) pode mudar livremente.

**Trate a API como um produto.** Ela tem consumidores, precisa de documentação, evolui com versionamento, e não pode quebrar sem aviso.

**Valide nas fronteiras.** Toda informação que cruza uma fronteira deve ser inspecionada: validada, sanitizada, autorizada. Nunca confie em dados que vêm de fora.

**Seja consistente.** Formato de resposta, tratamento de erros, convenções de URL — tudo deve seguir um padrão previsível em toda a API.

**Planeje para a evolução.** Versione desde o início, documente os contratos, comunique deprecações. Fronteiras que não evoluem eventualmente impedem que o sistema evolua.

Comece pelo básico: extraia Resources para todo endpoint que retorna dados, adicione autenticação onde falta, e escreva testes que validam o contrato. Cada passo torna o sistema mais robusto e mais fácil de manter.

**Próximo passo:** conectar esses conceitos com **Business Logic** (para garantir que as regras de negócio estejam isoladas atrás de fronteiras claras) e **State Management** (para controlar o que entra e sai através da API).
