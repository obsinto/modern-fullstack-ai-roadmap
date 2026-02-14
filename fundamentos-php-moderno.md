# Fundamentos do PHP Moderno (8.x)

## Introdução

Este guia cobre os recursos essenciais do PHP 8.x que todo desenvolvedor Laravel precisa dominar. O PHP evoluiu significativamente, e conhecer esses fundamentos é crucial para escrever código limpo, seguro e performático.

---

## Strict Types

### Por que usar?

```php
// Sem strict types - PHP faz coerção automática
function soma($a, $b) {
    return $a + $b;
}
soma("5", "3"); // Retorna 8 (strings convertidas)

// Com strict types - erro se tipos não coincidem
declare(strict_types=1);

function soma(int $a, int $b): int {
    return $a + $b;
}
soma("5", "3"); // TypeError!
```

### Quando Usar

```
SEMPRE use strict_types=1 em arquivos novos

┌─────────────────────────────────────────────────────────┐
│  declare(strict_types=1);  ← Primeira linha do arquivo  │
│                                                          │
│  namespace App\Services;                                 │
│                                                          │
│  class OrderService { ... }                              │
└─────────────────────────────────────────────────────────┘
```

---

## Sistema de Tipos

### Tipos Escalares

```php
declare(strict_types=1);

class UserService
{
    public function process(
        string $name,      // texto
        int $age,          // inteiro
        float $balance,    // decimal
        bool $active       // booleano
    ): void {
        // ...
    }
}
```

### Tipos Compostos

```php
// Arrays
public function setTags(array $tags): void { }

// Callable
public function onComplete(callable $callback): void { }

// Iterable (array ou Traversable)
public function processItems(iterable $items): void { }

// Object
public function serialize(object $data): string { }
```

### Union Types (PHP 8.0+)

```php
class PaymentProcessor
{
    // Aceita múltiplos tipos
    public function process(int|float $amount): void
    {
        // $amount pode ser int ou float
    }

    // Retorna múltiplos tipos possíveis
    public function find(int $id): User|null
    {
        return User::find($id);
    }

    // Combinações úteis
    public function setValue(string|int|float $value): void { }
}
```

### Intersection Types (PHP 8.1+)

```php
// Objeto deve implementar AMBAS interfaces
interface Loggable
{
    public function log(): void;
}

interface Serializable
{
    public function serialize(): string;
}

class AuditService
{
    // $entity DEVE implementar Loggable E Serializable
    public function audit(Loggable&Serializable $entity): void
    {
        $entity->log();
        $data = $entity->serialize();
    }
}
```

### Nullable Types

```php
class UserRepository
{
    // Três formas equivalentes de nullable
    public function findById(int $id): ?User
    {
        return User::find($id); // User ou null
    }

    public function findByEmail(string $email): User|null
    {
        return User::where('email', $email)->first();
    }

    // Parâmetro nullable com default
    public function search(?string $query = null): Collection
    {
        if ($query === null) {
            return User::all();
        }
        return User::where('name', 'like', "%{$query}%")->get();
    }
}
```

### Mixed Type

```php
// Aceita qualquer tipo (use com cautela)
public function cache(string $key, mixed $value): void
{
    Cache::put($key, $value);
}

// Quando usar mixed:
// - Cache genérico
// - Serialização
// - Wrappers genéricos
// - Evite em domínio específico
```

### Never Type (PHP 8.1+)

```php
class ErrorHandler
{
    // Indica que o método NUNCA retorna normalmente
    public function abort(string $message): never
    {
        Log::error($message);
        throw new RuntimeException($message);
    }

    public function redirect(string $url): never
    {
        header("Location: {$url}");
        exit;
    }
}
```

---

## Readonly Properties (PHP 8.1+)

### Propriedades Imutáveis

```php
class User
{
    // Pode ser setada apenas uma vez (construtor ou declaração)
    public readonly int $id;
    public readonly string $email;
    public readonly DateTimeImmutable $createdAt;

    public function __construct(int $id, string $email)
    {
        $this->id = $id;
        $this->email = $email;
        $this->createdAt = new DateTimeImmutable();
    }
}

$user = new User(1, 'test@example.com');
$user->email = 'other@example.com'; // Error! Cannot modify readonly
```

### Readonly Classes (PHP 8.2+)

```php
// Todas as propriedades são automaticamente readonly
readonly class OrderDTO
{
    public function __construct(
        public int $orderId,
        public string $customerName,
        public float $total,
        public array $items,
    ) {}
}

// Equivalente a:
class OrderDTO
{
    public readonly int $orderId;
    public readonly string $customerName;
    public readonly float $total;
    public readonly array $items;

    public function __construct(/* ... */) { /* ... */ }
}
```

### Uso com DTOs

```php
readonly class CreateUserDTO
{
    public function __construct(
        public string $name,
        public string $email,
        public ?string $phone = null,
    ) {}

    public static function fromRequest(Request $request): self
    {
        return new self(
            name: $request->validated('name'),
            email: $request->validated('email'),
            phone: $request->validated('phone'),
        );
    }
}
```

---

## Constructor Property Promotion

### Sintaxe Tradicional vs Moderna

```php
// PHP 7.x - Verbose
class Product
{
    private string $name;
    private float $price;
    private int $stock;

    public function __construct(string $name, float $price, int $stock)
    {
        $this->name = $name;
        $this->price = $price;
        $this->stock = $stock;
    }
}

// PHP 8.x - Property Promotion
class Product
{
    public function __construct(
        private string $name,
        private float $price,
        private int $stock,
    ) {}
}
```

### Combinando com Readonly

```php
class Invoice
{
    public function __construct(
        public readonly int $id,
        public readonly string $number,
        private readonly float $amount,
        private readonly DateTimeImmutable $issuedAt,
        private ?DateTimeImmutable $paidAt = null, // Não readonly, pode mudar
    ) {}

    public function markAsPaid(): void
    {
        $this->paidAt = new DateTimeImmutable();
    }
}
```

### Misturando Promoção com Lógica

```php
class Order
{
    private string $status;

    public function __construct(
        public readonly int $id,
        public readonly string $customerEmail,
        private array $items = [],
    ) {
        // Lógica adicional no construtor
        $this->status = 'pending';
        $this->validateEmail($customerEmail);
    }

    private function validateEmail(string $email): void
    {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException('Invalid email');
        }
    }
}
```

---

## Named Arguments

### Sintaxe e Benefícios

```php
class ReportGenerator
{
    public function generate(
        string $type,
        DateTimeInterface $startDate,
        DateTimeInterface $endDate,
        bool $includeCharts = false,
        bool $includeSummary = true,
        ?string $format = 'pdf',
    ): Report {
        // ...
    }
}

$generator = new ReportGenerator();

// Sem named arguments - difícil ler
$generator->generate('sales', $start, $end, true, false, 'xlsx');

// Com named arguments - claro e legível
$generator->generate(
    type: 'sales',
    startDate: $start,
    endDate: $end,
    includeCharts: true,
    includeSummary: false,
    format: 'xlsx',
);

// Pular argumentos opcionais
$generator->generate(
    type: 'sales',
    startDate: $start,
    endDate: $end,
    format: 'xlsx', // Pula includeCharts e includeSummary
);
```

### Com Constructors

```php
readonly class SearchFilters
{
    public function __construct(
        public ?string $query = null,
        public ?string $category = null,
        public ?float $minPrice = null,
        public ?float $maxPrice = null,
        public string $sortBy = 'created_at',
        public string $sortDir = 'desc',
        public int $perPage = 15,
    ) {}
}

// Uso com named arguments
$filters = new SearchFilters(
    category: 'electronics',
    minPrice: 100.00,
    sortBy: 'price',
);
```

---

## Match Expression

### Switch vs Match

```php
// Switch tradicional - verbose e propenso a bugs
function getStatusLabel(string $status): string
{
    switch ($status) {
        case 'pending':
            return 'Pendente';
        case 'processing':
            return 'Processando';
        case 'completed':
            return 'Concluído';
        case 'cancelled':
            return 'Cancelado';
        default:
            return 'Desconhecido';
    }
}

// Match expression - conciso e seguro
function getStatusLabel(string $status): string
{
    return match ($status) {
        'pending' => 'Pendente',
        'processing' => 'Processando',
        'completed' => 'Concluído',
        'cancelled' => 'Cancelado',
        default => 'Desconhecido',
    };
}
```

### Características do Match

```php
class PaymentProcessor
{
    public function calculateFee(string $method, float $amount): float
    {
        // Match é uma EXPRESSÃO (retorna valor)
        // Match usa comparação ESTRITA (===)
        // Match deve ser exaustivo ou ter default

        $rate = match ($method) {
            'credit_card' => 0.029,
            'debit_card' => 0.015,
            'pix' => 0.0,
            'boleto' => 0.019,
            default => throw new InvalidArgumentException("Unknown method: {$method}"),
        };

        return $amount * $rate;
    }
}
```

### Múltiplos Valores

```php
function getCategory(string $extension): string
{
    return match ($extension) {
        'jpg', 'jpeg', 'png', 'gif', 'webp' => 'image',
        'mp4', 'avi', 'mov', 'webm' => 'video',
        'mp3', 'wav', 'ogg', 'flac' => 'audio',
        'pdf', 'doc', 'docx', 'txt' => 'document',
        default => 'other',
    };
}
```

### Match com Condições

```php
class Pricing
{
    public function getDiscount(User $user): float
    {
        return match (true) {
            $user->isPremium() && $user->ordersCount() > 100 => 0.25,
            $user->isPremium() => 0.15,
            $user->ordersCount() > 50 => 0.10,
            $user->ordersCount() > 10 => 0.05,
            default => 0.0,
        };
    }
}
```

---

## Enums (PHP 8.1+)

### Pure Enums

```php
enum OrderStatus
{
    case Pending;
    case Processing;
    case Shipped;
    case Delivered;
    case Cancelled;

    public function label(): string
    {
        return match ($this) {
            self::Pending => 'Aguardando',
            self::Processing => 'Processando',
            self::Shipped => 'Enviado',
            self::Delivered => 'Entregue',
            self::Cancelled => 'Cancelado',
        };
    }

    public function color(): string
    {
        return match ($this) {
            self::Pending => 'yellow',
            self::Processing => 'blue',
            self::Shipped => 'purple',
            self::Delivered => 'green',
            self::Cancelled => 'red',
        };
    }

    public function canCancel(): bool
    {
        return in_array($this, [self::Pending, self::Processing]);
    }
}

// Uso
$order = new Order();
$order->status = OrderStatus::Pending;

echo $order->status->label();  // "Aguardando"
echo $order->status->color();  // "yellow"

if ($order->status->canCancel()) {
    $order->cancel();
}
```

### Backed Enums

```php
// String-backed enum
enum PaymentMethod: string
{
    case CreditCard = 'credit_card';
    case DebitCard = 'debit_card';
    case Pix = 'pix';
    case Boleto = 'boleto';

    public function fee(): float
    {
        return match ($this) {
            self::CreditCard => 2.9,
            self::DebitCard => 1.5,
            self::Pix => 0.0,
            self::Boleto => 1.9,
        };
    }
}

// Int-backed enum
enum Priority: int
{
    case Low = 1;
    case Medium = 2;
    case High = 3;
    case Urgent = 4;
}

// Conversão
$method = PaymentMethod::from('pix');        // PaymentMethod::Pix
$method = PaymentMethod::tryFrom('invalid'); // null (não lança exceção)

echo PaymentMethod::Pix->value; // "pix"
```

### Enums no Laravel

```php
// No Model
class Order extends Model
{
    protected $casts = [
        'status' => OrderStatus::class,
        'payment_method' => PaymentMethod::class,
    ];
}

// Migration
Schema::create('orders', function (Blueprint $table) {
    $table->string('status')->default(OrderStatus::Pending->value);
    $table->string('payment_method');
});

// Validation
$request->validate([
    'status' => ['required', new Enum(OrderStatus::class)],
]);

// Uso
$order = Order::find(1);
if ($order->status === OrderStatus::Delivered) {
    // ...
}
```

### Traits em Enums

```php
trait EnumHelpers
{
    public static function values(): array
    {
        return array_column(self::cases(), 'value');
    }

    public static function names(): array
    {
        return array_column(self::cases(), 'name');
    }

    public static function options(): array
    {
        return array_combine(
            array_column(self::cases(), 'value'),
            array_map(fn($case) => $case->label(), self::cases())
        );
    }
}

enum UserRole: string
{
    use EnumHelpers;

    case Admin = 'admin';
    case Manager = 'manager';
    case User = 'user';

    public function label(): string
    {
        return match ($this) {
            self::Admin => 'Administrador',
            self::Manager => 'Gerente',
            self::User => 'Usuário',
        };
    }
}

// Para selects no frontend
UserRole::options(); // ['admin' => 'Administrador', 'manager' => 'Gerente', ...]
```

---

## Attributes (Anotações)

### Conceito

```php
// Attributes são metadados estruturados
// Substituem docblocks para configuração

// Antes (docblocks)
/**
 * @Route("/users/{id}", methods={"GET"})
 * @Cache(maxAge=3600)
 */
public function show(int $id) { }

// Agora (attributes)
#[Route('/users/{id}', methods: ['GET'])]
#[Cache(maxAge: 3600)]
public function show(int $id) { }
```

### Attributes Nativos do PHP

```php
class UserController
{
    #[Deprecated(
        message: 'Use findById() instead',
        since: '2.0'
    )]
    public function find(int $id): User
    {
        return $this->findById($id);
    }

    public function findById(int $id): User
    {
        return User::findOrFail($id);
    }
}

class MathService
{
    // Indica que retorno pode ser ignorado
    #[Pure]
    public function calculate(int $a, int $b): int
    {
        return $a + $b;
    }
}
```

### Attributes no Laravel

```php
use Illuminate\Contracts\Queue\ShouldQueue;

// Route Attributes (Laravel 10+)
class UserController extends Controller
{
    #[Get('/users')]
    public function index() { }

    #[Get('/users/{user}')]
    public function show(User $user) { }

    #[Post('/users')]
    #[Middleware('auth')]
    public function store(StoreUserRequest $request) { }
}

// Validation
readonly class CreateProductDTO
{
    public function __construct(
        #[Required]
        #[StringType]
        #[Max(255)]
        public string $name,

        #[Required]
        #[Numeric]
        #[Min(0)]
        public float $price,

        #[Nullable]
        #[StringType]
        public ?string $description = null,
    ) {}
}
```

### Criando Attributes Customizados

```php
#[Attribute(Attribute::TARGET_PROPERTY)]
class Encrypted
{
    public function __construct(
        public string $algorithm = 'aes-256-cbc'
    ) {}
}

#[Attribute(Attribute::TARGET_METHOD)]
class RateLimit
{
    public function __construct(
        public int $maxAttempts = 60,
        public int $decayMinutes = 1,
    ) {}
}

// Uso
class User extends Model
{
    #[Encrypted]
    public string $ssn;
}

class ApiController
{
    #[RateLimit(maxAttempts: 10, decayMinutes: 1)]
    public function sensitiveAction() { }
}
```

---

## Null Safe Operator

### Encadeamento Seguro

```php
class User
{
    public ?Address $address = null;
}

class Address
{
    public ?City $city = null;
}

class City
{
    public string $name;
}

// Antes - verificações manuais
$cityName = null;
if ($user !== null && $user->address !== null && $user->address->city !== null) {
    $cityName = $user->address->city->name;
}

// Agora - null safe operator
$cityName = $user?->address?->city?->name;
// Retorna null se qualquer parte for null
```

### Casos de Uso

```php
class OrderService
{
    public function getCustomerEmail(Order $order): ?string
    {
        // Encadeamento seguro
        return $order->customer?->primaryEmail?->address;
    }

    public function getShippingCity(Order $order): ?string
    {
        return $order->shipping?->address?->city?->name;
    }

    public function notifyCustomer(Order $order): void
    {
        // Com chamada de método
        $order->customer?->notify(new OrderShipped($order));

        // Com array access
        $firstItem = $order->items?->first()?->product?->name;
    }
}
```

### Combinando com Null Coalescing

```php
class ConfigService
{
    public function get(string $key, mixed $default = null): mixed
    {
        // ?-> retorna null se objeto for null
        // ?? retorna default se resultado for null
        return $this->config?->get($key) ?? $default;
    }
}

// Exemplos práticos
$name = $user?->profile?->displayName ?? $user?->name ?? 'Guest';
$avatar = $user?->avatar?->url ?? '/images/default-avatar.png';
```

---

## Arrow Functions

### Sintaxe Curta para Closures

```php
// Closure tradicional
$doubled = array_map(function ($n) {
    return $n * 2;
}, [1, 2, 3, 4]);

// Arrow function
$doubled = array_map(fn($n) => $n * 2, [1, 2, 3, 4]);

// Acesso automático a variáveis do escopo pai
$multiplier = 3;

// Closure tradicional - precisa de use
$result = array_map(function ($n) use ($multiplier) {
    return $n * $multiplier;
}, [1, 2, 3, 4]);

// Arrow function - acesso automático
$result = array_map(fn($n) => $n * $multiplier, [1, 2, 3, 4]);
```

### Casos de Uso Comuns

```php
class ProductService
{
    public function filterActive(Collection $products): Collection
    {
        return $products->filter(fn($p) => $p->active);
    }

    public function getNames(Collection $products): array
    {
        return $products->map(fn($p) => $p->name)->toArray();
    }

    public function calculateTotal(Collection $items): float
    {
        return $items->sum(fn($item) => $item->price * $item->quantity);
    }

    public function sortByPrice(Collection $products): Collection
    {
        return $products->sortBy(fn($p) => $p->price);
    }

    public function groupByCategory(Collection $products): Collection
    {
        return $products->groupBy(fn($p) => $p->category_id);
    }
}
```

### Com Tipagem

```php
$formatter = fn(float $price): string => number_format($price, 2, ',', '.');

$validator = fn(?string $value): bool => $value !== null && strlen($value) > 0;

$transformer = fn(User $user): array => [
    'id' => $user->id,
    'name' => $user->name,
    'email' => $user->email,
];
```

---

## First-Class Callable Syntax

### Referência a Métodos

```php
class StringHelper
{
    public function uppercase(string $s): string
    {
        return strtoupper($s);
    }

    public static function lowercase(string $s): string
    {
        return strtolower($s);
    }
}

$helper = new StringHelper();

// Antes
$fn = Closure::fromCallable([$helper, 'uppercase']);
$static = Closure::fromCallable([StringHelper::class, 'lowercase']);

// PHP 8.1+ - First-class callable syntax
$fn = $helper->uppercase(...);
$static = StringHelper::lowercase(...);

// Uso
$names = ['john', 'jane'];
$upper = array_map($helper->uppercase(...), $names);
// ['JOHN', 'JANE']
```

### Uso Prático

```php
class OrderProcessor
{
    public function processAll(array $orders): void
    {
        // Passando método como callback
        array_map($this->processOne(...), $orders);

        // Em collections
        collect($orders)
            ->filter($this->isValid(...))
            ->each($this->processOne(...));
    }

    private function isValid(Order $order): bool
    {
        return $order->total > 0;
    }

    private function processOne(Order $order): void
    {
        // ...
    }
}
```

---

## Fibers (PHP 8.1+)

### Conceito

```php
// Fibers permitem pausar e retomar execução
// Base para async/await no PHP

$fiber = new Fiber(function (): void {
    $value = Fiber::suspend('Hello');
    echo "Received: $value\n";
});

$value = $fiber->start();
echo "Fiber said: $value\n";

$fiber->resume('World');

// Output:
// Fiber said: Hello
// Received: World
```

### Na Prática

```php
// Fibers são usados internamente por bibliotecas
// Exemplo com ReactPHP ou Amp

// Você raramente usa Fiber diretamente
// Mas entender o conceito ajuda com:
// - RevoltPHP
// - Laravel Octane
// - Async drivers
```

---

## Expressões Throw

### Throw como Expressão

```php
// Antes - throw era statement
function getValue(?array $data): string
{
    if ($data === null) {
        throw new InvalidArgumentException('Data required');
    }
    return $data['value'] ?? throw new InvalidArgumentException('Value required');
}

// Agora - throw pode ser expressão
function getValue(?array $data): string
{
    // Com null coalescing
    $data = $data ?? throw new InvalidArgumentException('Data required');

    // Com ternário
    return isset($data['value'])
        ? $data['value']
        : throw new InvalidArgumentException('Value required');
}
```

### Casos de Uso

```php
class UserService
{
    public function find(int $id): User
    {
        return User::find($id)
            ?? throw new ModelNotFoundException("User {$id} not found");
    }

    public function getEmail(User $user): string
    {
        return $user->email
            ?? throw new InvalidStateException('User has no email');
    }
}

// Com arrow functions
$getRequired = fn(array $data, string $key): mixed
    => $data[$key] ?? throw new MissingKeyException($key);
```

---

## Novidades PHP 8.2+

### Constantes em Traits

```php
trait Configurable
{
    public const DEFAULT_TIMEOUT = 30;
    private const INTERNAL_KEY = 'xyz';

    public function getTimeout(): int
    {
        return self::DEFAULT_TIMEOUT;
    }
}

class HttpClient
{
    use Configurable;

    public function request(): void
    {
        $timeout = self::DEFAULT_TIMEOUT;
    }
}
```

### Tipos DNF (Disjunctive Normal Form)

```php
// Combinação de union e intersection types
interface A {}
interface B {}
class C {}

// (A&B)|C - deve implementar A E B, OU ser C
function process((A&B)|C $entity): void
{
    // ...
}
```

### Readonly Classes

```php
// Já mostrado acima - todas propriedades são readonly
readonly class ImmutableDTO
{
    public function __construct(
        public int $id,
        public string $name,
    ) {}
}
```

### Deprecações Importantes

```php
// PHP 8.2 deprecou:
// - Dynamic properties (sem declaração)
// - utf8_encode/utf8_decode (use mb_convert_encoding)

// Para permitir propriedades dinâmicas (não recomendado):
#[AllowDynamicProperties]
class LegacyClass
{
    // ...
}

// Melhor: declare as propriedades
class ModernClass
{
    public array $dynamicData = [];
}
```

---

## Boas Práticas

### Estrutura de Classe Moderna

```php
declare(strict_types=1);

namespace App\Domain\Order;

use App\Domain\Order\Events\OrderCreated;
use DateTimeImmutable;

readonly class Order
{
    public function __construct(
        public OrderId $id,
        public CustomerId $customerId,
        public Money $total,
        public OrderStatus $status,
        public DateTimeImmutable $createdAt,
        private array $items = [],
    ) {}

    public static function create(
        CustomerId $customerId,
        array $items,
    ): self {
        $total = self::calculateTotal($items);

        $order = new self(
            id: OrderId::generate(),
            customerId: $customerId,
            total: $total,
            status: OrderStatus::Pending,
            createdAt: new DateTimeImmutable(),
            items: $items,
        );

        event(new OrderCreated($order));

        return $order;
    }

    private static function calculateTotal(array $items): Money
    {
        return array_reduce(
            $items,
            fn(Money $total, OrderItem $item) => $total->add($item->subtotal()),
            Money::zero(),
        );
    }

    public function items(): array
    {
        return $this->items;
    }
}
```

### Checklist de Código Moderno

```
Arquivo PHP Moderno:
□ declare(strict_types=1) no topo
□ Namespace correto
□ Tipagem completa (params e return)
□ Readonly onde aplicável
□ Constructor promotion
□ Named arguments para clareza
□ Match ao invés de switch
□ Enums ao invés de constantes
□ Null safe operator (?->)
□ Arrow functions para callbacks curtos
```

---

## Exercícios Práticos

### Exercício 1: Refatorar para PHP 8.x

```php
// Código PHP 7.x
class User
{
    private $id;
    private $name;
    private $email;
    private $status;

    const STATUS_ACTIVE = 'active';
    const STATUS_INACTIVE = 'inactive';
    const STATUS_SUSPENDED = 'suspended';

    public function __construct($id, $name, $email, $status = 'active')
    {
        $this->id = $id;
        $this->name = $name;
        $this->email = $email;
        $this->status = $status;
    }

    public function getStatusLabel()
    {
        switch ($this->status) {
            case 'active':
                return 'Ativo';
            case 'inactive':
                return 'Inativo';
            case 'suspended':
                return 'Suspenso';
            default:
                return 'Desconhecido';
        }
    }
}

// Refatore usando:
// - Strict types
// - Constructor promotion
// - Readonly
// - Enum para status
// - Match expression
```

### Exercício 2: Criar Value Object

```php
// Crie um Value Object Money usando:
// - Readonly class
// - Tipagem estrita
// - Métodos imutáveis (add, subtract, multiply)
// - Named arguments no constructor
// - Match para formatação por locale
```

### Exercício 3: Enum com Comportamento

```php
// Crie um Enum PaymentStatus que:
// - Tenha casos: Pending, Processing, Completed, Failed, Refunded
// - Seja backed por string
// - Tenha método label() em português
// - Tenha método color() retornando cor CSS
// - Tenha método canTransitionTo(self $status): bool
// - Tenha método transitions(): array
```

---

## 7. PHP 8.4+: O Limite da Modernidade

### 7.1 Property Hooks
Os Property Hooks (PHP 8.4) permitem definir lógica de acesso e modificação diretamente na propriedade, eliminando o boilerplate de getters e setters tradicionais.

```php
class User
{
    // Hook 'set' para garantir trim e 'get' para formatação
    public string $name {
        set => trim($value);
        get => ucfirst($this->name);
    }

    // Propriedade virtual (sem armazenamento em memória)
    public string $displayName {
        get => "User: {$this->name}";
    }
}
```

### 7.2 Asymmetric Visibility (Visibilidade Assimétrica)
Resolve o problema de querer uma propriedade pública para leitura, mas restrita para escrita, sem precisar de métodos adicionais.

```php
class Order
{
    // Público para leitura, mas só pode ser alterado internamente
    public private(set) float $total;

    public function addItem(float $price): void
    {
        $this->total += $price;
    }
}
```

---

## Conclusão

O PHP moderno (8.x) oferece ferramentas poderosas para escrever código mais seguro, legível e manutenível. Dominar esses fundamentos é essencial antes de avançar para padrões arquiteturais mais complexos.

### Próximos Passos

1. **Pratique** os exercícios acima
2. **Refatore** código legado usando essas features
3. **Configure** PHPStan para validar tipagem
4. **Avance** para os padrões arquiteturais (Services, Actions, DTOs)

---

*O PHP evoluiu significativamente. Código moderno PHP é tão expressivo quanto muitas linguagens consideradas "modernas".*
