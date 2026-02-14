# State & Data Flow (Estado e Fluxo de Dados)

## Índice

1. [Introdução](#introdução)
2. [Por que isso importa?](#por-que-isso-importa)
3. [Conceitos Fundamentais](#conceitos-fundamentais)
4. [Request Lifecycle no Laravel](#request-lifecycle-no-laravel)
5. [Single Source of Truth](#single-source-of-truth)
6. [Imutabilidade e Mutabilidade](#imutabilidade-e-mutabilidade)
7. [Padrões de Fluxo de Dados](#padrões-de-fluxo-de-dados)
8. [State Management em Aplicações Modernas](#state-management-em-aplicações-modernas)
9. [Anti-patterns Comuns](#anti-patterns-comuns)
10. [Boas Práticas](#boas-práticas)
11. [Exercícios Práticos](#exercícios-práticos)
12. [Recursos Adicionais](#recursos-adicionais)

---

## Introdução

Toda aplicação precisa lidar com duas questões centrais: **o que ela sabe** e **como essa informação circula**. O primeiro chamamos de **Estado** (State) — qualquer dado que o sistema precisa reter para funcionar. O segundo chamamos de **Fluxo de Dados** (Data Flow) — o caminho que esses dados percorrem entre as camadas da aplicação.

Uma analogia útil: pense no Estado como o prontuário de um paciente e no Fluxo de Dados como o protocolo que determina quem lê, quem escreve e em que ordem. Se o prontuário estiver desatualizado ou se dois médicos anotarem informações conflitantes sem saber um do outro, o paciente está em risco. O mesmo vale para os dados da sua aplicação.

### O que conta como Estado?

Qualquer informação que, se perdida ou corrompida, altera o comportamento do sistema:

- **Identidade do usuário logado** — nome, email, permissões, token de sessão
- **Carrinho de compras** — itens selecionados, quantidades, subtotais
- **Formulário em preenchimento** — valores temporários antes do submit
- **Configurações do sistema** — tema, idioma, parâmetros de negócio
- **Cache** — dados recentemente consultados para evitar idas repetidas ao banco

---

## Por que isso importa?

### Consequências de uma gestão ruim de estado

Quando o estado não é bem gerenciado, os sintomas aparecem de formas variadas — e muitas vezes difíceis de diagnosticar:

**Dados dessincronizados.** O usuário vê um saldo de R$ 500 na tela de resumo, mas R$ 480 na tela de detalhes. Dois componentes consultaram fontes diferentes no mesmo instante, e nenhum percebeu a divergência.

**Bugs não-determinísticos.** O sistema funciona 99% do tempo, mas falha misteriosamente em cenários específicos — geralmente quando o estado passa por uma sequência rara de mutações. Reproduzir o problema exige reconstruir o caminho exato dos dados.

**Performance degradada.** Sem uma estratégia clara de cache e derivação, o sistema faz consultas redundantes ao banco. Uma tela que poderia carregar em 200ms leva 2 segundos porque recomputa dados que já estavam disponíveis.

**Código frágil.** Alterar uma parte do sistema quebra outra aparentemente desconectada, porque ambas dependem de um estado compartilhado que ninguém documentou.

**Brechas de segurança.** Dados sensíveis vazam para camadas onde não deveriam estar — um token de API que aparece no log, ou permissões do admin que ficam acessíveis no frontend.

### O que uma boa arquitetura de estado oferece

**Previsibilidade** — cada informação tem um endereço conhecido; você sempre sabe onde procurar.

**Rastreabilidade** — é possível seguir o caminho dos dados da entrada à saída, etapa por etapa.

**Testabilidade** — cada camada pode ser testada isoladamente, porque as dependências de estado são explícitas.

**Escalabilidade** — o sistema cresce sem que a complexidade saia do controle.

**Debugging eficiente** — quando algo quebra, o fluxo claro reduz o espaço de busca drasticamente.

---

## Conceitos Fundamentais

### 1. Estado Local vs Estado Global

A primeira distinção a entender é o **escopo** do estado: ele pertence a um único contexto ou é compartilhado por múltiplas partes do sistema?

#### Estado Local

Existe apenas dentro de um escopo específico — uma requisição, um método, um componente. Quando aquele escopo termina, o estado desaparece.

```php
class ProductController extends Controller
{
    public function show(Product $product)
    {
        // $relatedProducts só existe neste método.
        // Quando a response é enviada, essa variável deixa de existir.
        $relatedProducts = $product->related()->take(5)->get();

        return view('products.show', [
            'product' => $product,
            'related' => $relatedProducts,
        ]);
    }
}
```

Aqui, `$relatedProducts` é estado local: nasceu no controller, serviu para montar a view e morreu junto com a requisição. Nenhuma outra parte do sistema depende dele.

#### Estado Global

Precisa ser acessível por múltiplas partes do sistema, muitas vezes ao longo de várias requisições.

```php
class ConfigurationRepository
{
    private static ?array $cache = null;

    public static function get(string $key, mixed $default = null): mixed
    {
        if (self::$cache === null) {
            self::$cache = DB::table('configurations')
                ->pluck('value', 'key')
                ->toArray();
        }

        return self::$cache[$key] ?? $default;
    }

    public static function flush(): void
    {
        self::$cache = null;
    }
}
```

Configurações do sistema são um exemplo clássico de estado global: qualquer controller, service ou job pode precisar consultá-las. O padrão acima carrega tudo uma vez e mantém em memória estática durante o ciclo de vida do processo. O método `flush()` existe para forçar recarga quando necessário — sem ele, alterações no banco não seriam refletidas até o próximo request.

> **Regra prática:** prefira estado local sempre que possível. Estado global é uma ferramenta poderosa, mas quanto mais amplo o escopo de um dado, maior o risco de efeitos colaterais inesperados.

### 2. Estado Fonte vs Estado Derivado

Esta distinção é sutil mas fundamental: nem todo dado precisa (ou deve) ser armazenado. Alguns dados existem como **fontes primárias**, outros são **calculados** a partir dessas fontes.

#### Estado Fonte (Source of Truth)

Dados que são armazenados diretamente — geralmente no banco de dados. São a base a partir da qual outros dados podem ser computados.

```php
Schema::create('orders', function (Blueprint $table) {
    $table->id();
    $table->decimal('subtotal', 10, 2);
    $table->decimal('tax', 10, 2);
    $table->decimal('shipping', 10, 2);
    // Perceba: NÃO há coluna 'total'.
    // O total é derivado da soma dos três campos acima.
    $table->timestamps();
});
```

#### Estado Derivado (Computed)

Calculado em tempo de execução a partir do estado fonte. Não precisa (e geralmente não deve) ser armazenado separadamente.

```php
class Order extends Model
{
    protected $appends = ['total'];

    public function getTotalAttribute(): float
    {
        return $this->subtotal + $this->tax + $this->shipping;
    }
}

// Em qualquer lugar:
$order->total; // Sempre correto, pois é calculado na hora
```

A vantagem dessa abordagem é que o `total` nunca fica inconsistente com seus componentes. Se `subtotal` mudar, `total` reflete automaticamente.

### 3. Estado Temporário vs Estado Persistente

A última dimensão é a **durabilidade**: por quanto tempo o dado precisa existir?

```php
// Estado Temporário — vive na sessão do usuário
// Ideal para fluxos em andamento (wizard, carrinho antes do checkout)
session(['cart' => [
    'items' => $items,
    'expires_at' => now()->addHours(2),
]]);

// Estado Persistente — vive no banco de dados
// Ideal para dados que precisam sobreviver ao fim da sessão
DB::table('carts')->insert([
    'user_id' => auth()->id(),
    'items' => json_encode($items),
    'created_at' => now(),
]);
```

A escolha entre temporário e persistente depende do contexto: um carrinho de compras pode começar na sessão (temporário) e migrar para o banco (persistente) quando o usuário faz login ou inicia o checkout.

---

## Request Lifecycle no Laravel

Entender o ciclo de vida de uma requisição no Laravel é essencial para saber **onde** o estado vive em **cada momento** e **quem** tem autoridade para modificá-lo.

### Visão geral do fluxo

```
Cliente (Browser/App)
    │
    ▼
┌─ HTTP Request ──────────────────────────────────────┐
│                                                      │
│  1. public/index.php        (Entry Point)            │
│  2. HTTP Kernel              (Bootstrap)             │
│  3. Global Middleware        (Auth, CORS, Rate Limit) │
│  4. Router                   (Match de rota)         │
│  5. Route Middleware         (Específicos da rota)   │
│  6. Controller               (Orquestração)          │
│  7. Service / Action         (Lógica de negócio)     │
│  8. Repository / Model       (Acesso a dados)        │
│  9. Database                 (Persistência)           │
│                                                      │
│  10. Response (JSON / View / Redirect)               │
│  11. Response Middleware     (Headers, Cookies)       │
│                                                      │
└─ HTTP Response ─────────────────────────────────────┘
    │
    ▼
Cliente recebe resposta
```

Cada camada tem uma responsabilidade clara com o estado:

- **Middleware** lê e valida metadados (autenticação, CORS, rate limiting)
- **Controller** orquestra — recebe input validado e delega para a camada de negócio
- **Service/Action** processa — aplica regras, calcula, transforma
- **Repository/Model** persiste — é a interface com o banco de dados

### Exemplo completo: criando um Requerimento

Vamos acompanhar o estado desde a submissão do formulário até a resposta ao usuário. Cada arquivo está anotado com sua responsabilidade no fluxo.

**Rota — o ponto de entrada:**

```php
// routes/web.php
Route::post('/requerimentos', [RequerimentoController::class, 'store'])
    ->middleware(['auth', 'verified']);
```

**FormRequest — validação e sanitização na fronteira:**

```php
// app/Http/Requests/StoreRequerimentoRequest.php
class StoreRequerimentoRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'tipo_id'       => 'required|exists:tipos_requerimento,id',
            'descricao'     => 'required|string|min:10',
            'documentos.*'  => 'file|max:5120|mimes:pdf,jpg,png',
        ];
    }

    protected function prepareForValidation(): void
    {
        $this->merge([
            'descricao'  => trim($this->descricao),
            'cidadao_id' => auth()->id(),
        ]);
    }
}
```

O `prepareForValidation` é o lugar certo para normalizar dados antes que as regras sejam aplicadas. Aqui garantimos que a descrição não tem espaços extras e que o ID do cidadão vem da sessão autenticada — nunca do input do formulário.

**Controller — orquestração enxuta:**

```php
// app/Http/Controllers/RequerimentoController.php
class RequerimentoController extends Controller
{
    public function store(StoreRequerimentoRequest $request)
    {
        $requerimento = app(CreateRequerimentoAction::class)
            ->execute($request->validated());

        return redirect()
            ->route('requerimentos.show', $requerimento)
            ->with('success', 'Requerimento criado com sucesso!');
    }
}
```

O controller não contém lógica de negócio. Recebe dados já validados, delega para a Action e retorna a resposta. Se amanhã esse mesmo fluxo precisar ser acionado por um comando Artisan ou um job, a Action pode ser reutilizada sem alterações.

**Action — lógica de negócio isolada:**

```php
// app/Actions/Requerimentos/CreateRequerimentoAction.php
class CreateRequerimentoAction
{
    public function __construct(
        private RequerimentoRepository $repository,
        private DocumentoService $documentoService,
        private NotificationService $notificationService,
    ) {}

    public function execute(array $data): Requerimento
    {
        return DB::transaction(function () use ($data) {
            $requerimento = $this->repository->create([
                'tipo_id'     => $data['tipo_id'],
                'cidadao_id'  => $data['cidadao_id'],
                'descricao'   => $data['descricao'],
                'protocolo'   => $this->gerarProtocolo(),
                'status'      => RequerimentoStatus::PENDENTE,
            ]);

            if (isset($data['documentos'])) {
                foreach ($data['documentos'] as $arquivo) {
                    $this->documentoService->attach($requerimento, $arquivo);
                }
            }

            $this->notificationService->notifyNewRequerimento($requerimento);

            activity()
                ->performedOn($requerimento)
                ->causedBy(auth()->user())
                ->log('Requerimento criado');

            return $requerimento;
        });
    }

    private function gerarProtocolo(): string
    {
        $sequencia = Requerimento::whereYear('created_at', now()->year)->count() + 1;

        return now()->format('Y') . '-' . str_pad($sequencia, 6, '0', STR_PAD_LEFT);
    }
}
```

Toda a operação está envolvida em `DB::transaction`. Se a notificação falhar depois de criar o requerimento, o banco faz rollback — nenhum registro órfão fica para trás. Esse é o tipo de garantia de consistência que a gestão cuidadosa de estado proporciona.

**Repository — abstração de persistência:**

```php
// app/Repositories/RequerimentoRepository.php
class RequerimentoRepository
{
    public function create(array $data): Requerimento
    {
        return Requerimento::create($data);
    }

    public function findByProtocolo(string $protocolo): ?Requerimento
    {
        return Requerimento::where('protocolo', $protocolo)->first();
    }
}
```

**Model — representação do domínio:**

```php
// app/Models/Requerimento.php
class Requerimento extends Model
{
    protected $fillable = [
        'tipo_id', 'cidadao_id', 'descricao', 'protocolo', 'status',
    ];

    protected $casts = [
        'status' => RequerimentoStatus::class,
    ];

    public function cidadao(): BelongsTo
    {
        return $this->belongsTo(User::class, 'cidadao_id');
    }

    public function tipo(): BelongsTo
    {
        return $this->belongsTo(TipoRequerimento::class);
    }

    public function documentos(): HasMany
    {
        return $this->hasMany(Documento::class);
    }
}
```

### Mapa do estado em cada etapa

```
ENTRADA (Request)
├── Headers HTTP (autenticação, content-type)
├── Body (JSON ou Form Data com os campos do formulário)
├── Query Parameters (filtros, paginação)
└── Arquivos (uploads de documentos)

PROCESSAMENTO
├── Sanitização (prepareForValidation)
├── Validação (rules do FormRequest)
├── Autorização (Policies / Gates)
├── Lógica de Negócio (Action)
│   ├── Geração de protocolo
│   ├── Persistência em transação
│   └── Side effects (notificações, auditoria)
└── Persistência (Repository → Model → Database)

SAÍDA (Response)
├── Status Code (302 Redirect)
├── Headers (Location, Set-Cookie)
├── Flash Data (mensagem de sucesso na sessão)
└── Dados da nova rota (show do requerimento)
```

---

## Single Source of Truth

**Princípio central:** cada informação deve ter exatamente uma fonte autoritativa. Se o mesmo dado existir em dois lugares, eventualmente eles vão divergir — e quando isso acontecer, qual deles está certo?

### O problema: fontes duplicadas

```php
// ❌ Total armazenado como coluna separada
class Order extends Model
{
    protected $fillable = ['subtotal', 'tax', 'shipping', 'total'];
}

public function store(Request $request)
{
    Order::create([
        'subtotal' => $request->subtotal,
        'tax'      => $request->tax,
        'shipping' => $request->shipping,
        'total'    => $request->subtotal + $request->tax + $request->shipping,
    ]);

    // Semanas depois, alguém atualiza subtotal sem recalcular total.
    // O banco agora tem dados inconsistentes e ninguém percebe.
}
```

### A solução: derivar ao invés de duplicar

```php
// ✅ Total sempre calculado a partir dos componentes
class Order extends Model
{
    protected $fillable = ['subtotal', 'tax', 'shipping'];
    protected $appends = ['total'];

    public function getTotalAttribute(): float
    {
        return $this->subtotal + $this->tax + $this->shipping;
    }
}

// Impossível ficar inconsistente: total é derivado, não armazenado.
```

### Exceção pragmática: cache de performance

Em situações onde o cálculo é custoso (joins pesados, agregações complexas), pode fazer sentido armazenar um valor derivado como cache. A chave é manter a **fonte de verdade** acessível e sincronizar o cache automaticamente:

```php
class Order extends Model
{
    protected $fillable = ['subtotal', 'tax', 'shipping', 'total_cached'];

    // Fonte de verdade — sempre disponível
    public function getTotalAttribute(): float
    {
        return $this->subtotal + $this->tax + $this->shipping;
    }
}

// Observer sincroniza o cache antes de cada save
class OrderObserver
{
    public function saving(Order $order): void
    {
        $order->total_cached = $order->total;
    }
}
```

Assim, queries que precisam de performance podem usar `total_cached`, mas qualquer lógica de negócio usa `$order->total` — a versão computada que nunca mente.

---

## Imutabilidade e Mutabilidade

Imutabilidade significa que um dado, uma vez criado, não pode ser alterado. Para modificá-lo, você cria um **novo** dado. Isso elimina toda uma classe de bugs relacionados a mutações inesperadas.

### Value Objects imutáveis

```php
final readonly class Money
{
    public function __construct(
        public int $amount,      // Em centavos para evitar problemas de float
        public string $currency,
    ) {}

    public function add(Money $other): self
    {
        if ($this->currency !== $other->currency) {
            throw new InvalidArgumentException(
                "Não é possível somar {$this->currency} com {$other->currency}"
            );
        }

        // Retorna um NOVO objeto. $this permanece inalterado.
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function format(): string
    {
        return number_format($this->amount / 100, 2, ',', '.') . ' ' . $this->currency;
    }
}

$preco  = new Money(10000, 'BRL');  // R$ 100,00
$frete  = new Money(2500, 'BRL');   // R$ 25,00
$total  = $preco->add($frete);      // R$ 125,00

// $preco ainda é R$ 100,00 — nenhuma surpresa.
```

A palavra-chave `readonly` (PHP 8.2+) garante em nível de linguagem que as propriedades não podem ser reatribuídas após a construção. `final` impede que subclasses quebrem o contrato de imutabilidade.

### DTOs para transferência entre camadas

```php
final readonly class CreateUserData
{
    public function __construct(
        public string $name,
        public string $email,
        public string $password,
        public ?string $phone = null,
    ) {}

    // Para criar variações, retorne uma nova instância
    public function withPhone(string $phone): self
    {
        return new self(
            name: $this->name,
            email: $this->email,
            password: $this->password,
            phone: $phone,
        );
    }
}

$dados = new CreateUserData('Maria', 'maria@email.com', 'senha123');
$dadosComTelefone = $dados->withPhone('73999999999');
// $dados original permanece sem telefone
```

### Mutabilidade controlada no Eloquent

Models do Eloquent são mutáveis por natureza — é assim que o ORM funciona. O importante é **controlar** essa mutabilidade:

```php
class User extends Model
{
    // Whitelist explícita: só esses campos aceitam mass assignment
    protected $fillable = ['name', 'email'];

    // Blacklist: esses campos NUNCA aceitam mass assignment
    protected $guarded = ['is_admin', 'email_verified_at'];

    // Mutator: intercepta a escrita para garantir hash
    protected function password(): Attribute
    {
        return Attribute::make(
            set: fn (string $value) => Hash::make($value),
        );
    }

    // Accessor: intercepta a leitura para normalizar formato
    protected function name(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucwords(mb_strtolower($value)),
        );
    }
}
```

---

## Padrões de Fluxo de Dados

### 1. Fluxo Unidirecional (recomendado)

Dados percorrem o sistema em uma única direção, como uma linha de montagem. Cada etapa recebe, processa e passa adiante. Isso torna o sistema previsível e fácil de debugar.

```php
class CreateInvoiceAction
{
    public function execute(CreateInvoiceData $data): Invoice
    {
        // Etapa 1 → Validação de regras de negócio
        $this->validator->validate($data);

        // Etapa 2 → Construção do objeto
        $invoice = $this->buildInvoice($data);

        // Etapa 3 → Persistência
        $invoice = $this->repository->save($invoice);

        // Etapa 4 → Side effects via eventos
        event(new InvoiceCreated($invoice));

        // Etapa 5 → Retorno
        return $invoice;
    }
}

// O caminho é sempre: Input → Validação → Processamento → Persistência → Eventos → Output
// Nunca há desvios ou ciclos.
```

### 2. Fluxo Bidirecional (com cautela)

Às vezes a lógica exige que o dado "volte" para uma camada anterior. O risco é criar ciclos difíceis de rastrear.

```php
// ❌ Model com efeito colateral escondido
class Product extends Model
{
    public function updateStock(int $quantity): void
    {
        $this->stock -= $quantity;
        $this->save();

        // Surpresa: este método que "atualiza estoque"
        // também dispara notificação para o fornecedor.
        if ($this->stock < $this->minimum_stock) {
            $this->supplier->notify(new LowStockAlert($this));
        }
    }
}

// ✅ Responsabilidades separadas — fluxo claro
class Product extends Model
{
    public function decreaseStock(int $quantity): void
    {
        $this->stock -= $quantity;
        $this->save();
    }
}

class StockService
{
    public function processMovement(Product $product, int $quantity): void
    {
        $product->decreaseStock($quantity);

        if ($product->stock < $product->minimum_stock) {
            event(new StockBelowMinimum($product));
        }
    }
}
```

Na versão refatorada, o Model cuida apenas da persistência. A decisão de notificar ou não é responsabilidade do Service. Quem lê `decreaseStock` sabe exatamente o que vai acontecer — sem surpresas.

### 3. Fluxo orientado a eventos (Event-Driven)

O controller apenas registra o fato ("um pedido foi criado") e dispara um evento. Os efeitos colaterais são tratados por listeners independentes, desacoplados entre si.

```php
// O evento: um fato que aconteceu no sistema
class OrderPlaced
{
    public function __construct(public Order $order) {}
}

// Listener 1: atualiza estoque
class UpdateProductStock
{
    public function handle(OrderPlaced $event): void
    {
        foreach ($event->order->items as $item) {
            $item->product->decreaseStock($item->quantity);
        }
    }
}

// Listener 2: envia confirmação por email
class SendOrderConfirmation
{
    public function handle(OrderPlaced $event): void
    {
        Mail::to($event->order->customer)
            ->send(new OrderConfirmationMail($event->order));
    }
}

// Listener 3: gera fatura
class GenerateInvoice
{
    public function handle(OrderPlaced $event): void
    {
        Invoice::create([
            'order_id' => $event->order->id,
            'amount'   => $event->order->total,
            'due_date' => now()->addDays(7),
        ]);
    }
}

// Controller: apenas registra o fato e dispara o evento
public function store(StoreOrderRequest $request)
{
    $order = Order::create($request->validated());

    event(new OrderPlaced($order));

    return redirect()->route('orders.show', $order);
}
```

A grande vantagem: adicionar um novo efeito colateral (ex: integrar com sistema de logística) requer apenas registrar um novo listener. O controller e os listeners existentes não precisam ser tocados. Isso é o **Open/Closed Principle** aplicado ao fluxo de dados.

---

## State Management em Aplicações Modernas

Em uma stack com Laravel + Inertia.js + Vue.js, o estado vive em dois mundos que precisam se comunicar: o **backend** (PHP/Laravel) e o **frontend** (JavaScript/Vue).

### Estado do Backend para o Frontend

O controller prepara os dados e os envia como props via Inertia:

```php
class DashboardController extends Controller
{
    public function index()
    {
        return inertia('Dashboard', [
            'stats' => [
                'total_users'   => User::count(),
                'active_orders' => Order::active()->count(),
                'revenue_month' => Order::thisMonth()->sum('total'),
            ],
            'recent_orders' => OrderResource::collection(
                Order::latest()->take(10)->get()
            ),
            'filters' => request()->only(['search', 'status', 'date_from']),
        ]);
    }
}
```

### Estado local no componente Vue

O componente recebe as props (imutáveis por convenção) e gerencia seu próprio estado reativo:

```vue
<script setup>
import { ref, computed } from 'vue'
import { router } from '@inertiajs/vue3'

// Props do backend — trate como read-only
const props = defineProps({
    stats: Object,
    recent_orders: Array,
    filters: Object,
})

// Estado local do componente
const searchQuery = ref(props.filters.search || '')
const selectedStatus = ref(props.filters.status || 'all')

// Estado derivado — recomputa automaticamente quando dependências mudam
const filteredOrders = computed(() => {
    return props.recent_orders.filter(order => {
        const matchesSearch = order.customer.name
            .toLowerCase()
            .includes(searchQuery.value.toLowerCase())

        const matchesStatus = selectedStatus.value === 'all'
            || order.status === selectedStatus.value

        return matchesSearch && matchesStatus
    })
})

// Sincronizar estado local com o backend via Inertia
const updateFilters = () => {
    router.get('/dashboard', {
        search: searchQuery.value,
        status: selectedStatus.value,
    }, {
        preserveState: true,
        preserveScroll: true,
    })
}
</script>
```

O padrão aqui é claro: props descem (backend → frontend), ações sobem (frontend → backend via router). Esse fluxo unidirecional entre as camadas evita a armadilha de ter "duas verdades" sobre o mesmo dado.

### Cache como estado intermediário

```php
class ReportService
{
    public function getMonthlySales(int $year, int $month): array
    {
        $key = "sales.monthly.{$year}.{$month}";

        return Cache::remember($key, now()->addHours(6), function () use ($year, $month) {
            return Order::query()
                ->whereYear('created_at', $year)
                ->whereMonth('created_at', $month)
                ->selectRaw('DATE(created_at) as date, SUM(total) as total')
                ->groupBy('date')
                ->get()
                ->toArray();
        });
    }
}

// Invalidação automática quando o estado fonte muda
class OrderObserver
{
    public function created(Order $order): void
    {
        Cache::forget("sales.monthly.{$order->created_at->year}.{$order->created_at->month}");
    }
}
```

O cache é sempre **derivado** do banco. O Observer garante que quando um novo pedido é criado, o cache correspondente é invalidado. Na próxima consulta, o valor será recalculado e cacheado novamente.

### Sessão para fluxos multi-etapas

```php
class RegistrationWizardController extends Controller
{
    public function storeStepOne(Request $request)
    {
        session()->put('registration.step1', $request->validated());

        return redirect()->route('registration.step2');
    }

    public function storeStepTwo(Request $request)
    {
        session()->put('registration.step2', $request->validated());

        return redirect()->route('registration.step3');
    }

    public function complete()
    {
        $data = array_merge(
            session('registration.step1', []),
            session('registration.step2', []),
            session('registration.step3', []),
        );

        $user = User::create($data);

        // Limpar estado temporário após persistência
        session()->forget('registration');

        return redirect()->route('dashboard');
    }
}
```

Os dados vivem na sessão (temporários) até o momento da consolidação, quando migram para o banco (persistentes). A chamada `session()->forget()` no final é essencial — estado temporário que sobrevive além de sua utilidade é lixo que pode causar bugs.

---

## Anti-patterns Comuns

### 1. God Object (Objeto Deus)

Um único objeto concentra estado e responsabilidades demais. Qualquer mudança em qualquer parte do sistema passa por ele.

```php
// ❌ Um objeto que sabe tudo e faz tudo
class Application
{
    public User $user;
    public array $config;
    public array $routes;
    public array $cache;
    public Database $db;
    public Logger $logger;
    // ... dezenas de propriedades

    public function handleRequest() { /* centenas de linhas */ }
}

// ✅ Cada contexto tem seu próprio escopo de estado
class UserContext
{
    public function __construct(public readonly User $user) {}
}

class ConfigRepository
{
    public function get(string $key): mixed { /* ... */ }
}

class RouteRegistry
{
    public function register(string $path, callable $handler): void { /* ... */ }
}
```

### 2. Abuso de Estado Global

Variáveis globais são a forma mais primitiva de compartilhar estado — e a mais perigosa. Qualquer parte do código pode ler e modificar, sem rastreabilidade.

```php
// ❌ Estado global via $GLOBALS
$GLOBALS['current_user'] = User::find(1);

function doSomething()
{
    global $current_user;
    // Quem definiu? Quando? Pode ser null?
}

// ✅ Dependency Injection — dependências explícitas
class SomeService
{
    public function __construct(private User $user) {}

    public function doSomething(): void
    {
        // $this->user foi fornecido na construção.
        // A dependência é visível, testável e rastreável.
    }
}
```

### 3. Mutações Escondidas

Métodos que modificam objetos recebidos como parâmetro sem que o nome ou a assinatura indiquem isso.

```php
// ❌ O nome diz "calculate", mas o método SALVA no banco
class OrderCalculator
{
    public function calculate(Order $order): float
    {
        $order->total = $this->sum($order);
        $order->save();  // Efeito colateral inesperado

        return $order->total;
    }
}

// ✅ Separar computação de persistência
class OrderCalculator
{
    public function calculate(Order $order): float
    {
        return $order->items->sum(fn ($item) => $item->price * $item->qty);
    }
}

class OrderService
{
    public function __construct(private OrderCalculator $calculator) {}

    public function updateTotal(Order $order): Order
    {
        $order->total = $this->calculator->calculate($order);
        $order->save();

        return $order;
    }
}
```

### 4. Acoplamento Temporal

Quando um objeto exige que seus métodos sejam chamados em uma ordem específica para funcionar, mas nada no código impõe essa ordem.

```php
// ❌ Esquecer qualquer set causa falha silenciosa ou exceção
$service = new PaymentService();
$service->setGateway($gateway);
$service->setAmount(100);
$service->setCustomer($customer);
$service->process(); // Falha se algum set foi esquecido

// ✅ Construtor exige tudo de uma vez — impossível esquecer
$service = new PaymentService(
    gateway: $gateway,
    amount: 100,
    customer: $customer,
);
$service->process();
```

---

## Boas Práticas

### 1. Minimize estado mutável

Quanto menos coisas podem mudar, menos coisas podem dar errado.

```php
// Objetos imutáveis por padrão
final readonly class Invoice
{
    public function __construct(
        public int $id,
        public Money $amount,
        public Customer $customer,
        public Carbon $dueDate,
    ) {}
}
```

### 2. Use o sistema de tipos a seu favor

Types transformam erros de runtime (que o usuário encontra) em erros de compilação/análise estática (que o desenvolvedor encontra antes do deploy).

```php
// ❌ Array genérico — qualquer coisa pode acontecer
function processOrder(array $order): array
{
    $total = $order['total']; // E se a chave não existir?
    return ['success' => true];
}

// ✅ Types explícitos — garantias verificáveis
function processOrder(Order $order): OrderResult
{
    $total = $order->total; // PHPStan garante que existe
    return new OrderResult(success: true, orderId: $order->id);
}
```

### 3. Fail Fast — valide cedo, falhe cedo

Não permita que dados inválidos avancem pelo sistema. Quanto mais cedo a validação acontece, menor o estrago de um dado corrompido.

```php
public function createInvoice(CreateInvoiceData $data): Invoice
{
    // Tudo é verificado ANTES de qualquer operação no banco
    $customer = Customer::findOrFail($data->customerId);

    if ($data->amount->amount <= 0) {
        throw new DomainException('Valor da fatura deve ser positivo');
    }

    if ($customer->isSuspended()) {
        throw new DomainException('Cliente suspenso não pode receber faturas');
    }

    // Só agora, com todas as validações passando, persistimos
    return Invoice::create([
        'customer_id' => $customer->id,
        'amount'      => $data->amount->amount,
        'currency'    => $data->amount->currency,
    ]);
}
```

### 4. Documente transições de estado

Quando um dado tem um ciclo de vida com etapas definidas (status de pedido, fases de licitação), torne as transições válidas **explícitas no código** — não em documentação externa que ninguém lê.

```php
enum OrderStatus: string
{
    case PENDING    = 'pending';
    case CONFIRMED  = 'confirmed';
    case PROCESSING = 'processing';
    case SHIPPED    = 'shipped';
    case DELIVERED  = 'delivered';
    case CANCELLED  = 'cancelled';

    /**
     * Define quais transições são válidas a partir de cada status.
     * Isso funciona como documentação executável — se alguém tentar
     * uma transição inválida, o sistema rejeita imediatamente.
     */
    public function canTransitionTo(self $target): bool
    {
        return match ($this) {
            self::PENDING    => in_array($target, [self::CONFIRMED, self::CANCELLED]),
            self::CONFIRMED  => in_array($target, [self::PROCESSING, self::CANCELLED]),
            self::PROCESSING => in_array($target, [self::SHIPPED, self::CANCELLED]),
            self::SHIPPED    => $target === self::DELIVERED,
            self::DELIVERED,
            self::CANCELLED  => false,
        };
    }
}

class Order extends Model
{
    public function transitionTo(OrderStatus $newStatus): void
    {
        if (! $this->status->canTransitionTo($newStatus)) {
            throw new InvalidStatusTransition(
                "Transição inválida: {$this->status->value} → {$newStatus->value}"
            );
        }

        $oldStatus = $this->status;
        $this->status = $newStatus;
        $this->save();

        event(new OrderStatusChanged($this, $oldStatus, $newStatus));
    }
}
```

### 5. Use DTOs para cruzar fronteiras entre camadas

DTOs (Data Transfer Objects) formalizam o contrato entre camadas. O controller sabe exatamente o que a Action espera, e a Action sabe exatamente o que vai receber.

```php
final readonly class CreateProductData
{
    public function __construct(
        public string $name,
        public string $description,
        public Money $price,
        public int $stock,
        public array $categories,
    ) {}

    public static function fromRequest(StoreProductRequest $request): self
    {
        return new self(
            name: $request->input('name'),
            description: $request->input('description'),
            price: new Money($request->integer('price'), 'BRL'),
            stock: $request->integer('stock'),
            categories: $request->input('categories', []),
        );
    }
}

// Controller
public function store(StoreProductRequest $request)
{
    $data = CreateProductData::fromRequest($request);

    $product = $this->productService->create($data);

    return new ProductResource($product);
}
```

---

## Exercícios Práticos

### Exercício 1: Mapeie o fluxo de dados de uma funcionalidade existente

Escolha uma funcionalidade crítica do seu sistema (ex: criação de processo licitatório, geração de protocolo de atendimento) e responda:

1. Desenhe o diagrama de fluxo — da submissão do formulário até a resposta ao usuário.
2. Identifique todo o estado envolvido em cada etapa.
3. Procure duplicações — existe algum dado armazenado em dois lugares que poderia ser derivado?
4. Mapeie as mutações — onde e quando cada dado é modificado? Todas essas mutações são explícitas?

### Exercício 2: Refatore um controller gordo

Pegue um controller que acumula validação, lógica de negócio e persistência em um único método, e separe as responsabilidades:

```php
// ANTES — controller faz tudo
class RequerimentoController extends Controller
{
    public function store(Request $request)
    {
        $validated = $request->validate([/* ... */]);

        $protocolo = date('Y') . '-' . str_pad(
            Requerimento::count() + 1, 6, '0', STR_PAD_LEFT
        );

        $requerimento = Requerimento::create([
            ...$validated,
            'protocolo' => $protocolo,
            'status' => 'pendente',
        ]);

        if ($request->hasFile('documentos')) {
            foreach ($request->file('documentos') as $doc) {
                $path = $doc->store('documentos');
                $requerimento->documentos()->create([
                    'path' => $path,
                    'nome' => $doc->getClientOriginalName(),
                ]);
            }
        }

        Mail::to('admin@prefeitura.gov.br')
            ->send(new NovoRequerimento($requerimento));

        return redirect()->route('requerimentos.index');
    }
}

// DEPOIS — controller orquestra, Action executa
class RequerimentoController extends Controller
{
    public function store(
        StoreRequerimentoRequest $request,
        CreateRequerimentoAction $action,
    ) {
        $requerimento = $action->execute(
            CreateRequerimentoData::fromRequest($request)
        );

        return redirect()
            ->route('requerimentos.show', $requerimento)
            ->with('success', 'Requerimento criado!');
    }
}
```

O exercício consiste em criar o `FormRequest`, o `DTO`, e a `Action` que faltam.

### Exercício 3: Implemente uma State Machine

Crie uma máquina de estados para um processo do seu domínio (licitação, requerimento, protocolo):

```php
class ProcessoLicitacaoStateMachine
{
    public function __construct(private ProcessoLicitacao $processo) {}

    public function transition(ProcessoStatus $to): void
    {
        $from = $this->processo->status;

        if (! $from->canTransitionTo($to)) {
            throw new InvalidStateTransition(
                "Transição inválida: {$from->value} → {$to->value}"
            );
        }

        $this->processo->update(['status' => $to]);

        $this->dispatchSideEffects($to);
    }

    private function dispatchSideEffects(ProcessoStatus $status): void
    {
        match ($status) {
            ProcessoStatus::PUBLICADO  => event(new ProcessoPublicado($this->processo)),
            ProcessoStatus::HOMOLOGADO => event(new ProcessoHomologado($this->processo)),
            default => null,
        };
    }
}
```

Defina o enum `ProcessoStatus` com todas as etapas e transições válidas do seu domínio.

### Exercício 4: Implemente Audit Trail

Crie um sistema de rastreamento que registre automaticamente toda mudança de estado nas entidades críticas:

```php
// Migration
Schema::create('audits', function (Blueprint $table) {
    $table->id();
    $table->morphs('auditable');
    $table->foreignId('user_id')->nullable()->constrained();
    $table->string('event'); // created, updated, deleted
    $table->json('old_values')->nullable();
    $table->json('new_values')->nullable();
    $table->ipAddress('ip_address')->nullable();
    $table->string('user_agent')->nullable();
    $table->timestamps();
});

// Trait reutilizável
trait Auditable
{
    protected static function bootAuditable(): void
    {
        static::created(fn ($model) => $model->recordAudit('created'));
        static::updated(fn ($model) => $model->recordAudit('updated'));
        static::deleted(fn ($model) => $model->recordAudit('deleted'));
    }

    public function audits(): MorphMany
    {
        return $this->morphMany(Audit::class, 'auditable');
    }

    protected function recordAudit(string $event): void
    {
        if ($event === 'updated' && ! $this->isDirty()) {
            return;
        }

        $this->audits()->create([
            'user_id'    => auth()->id(),
            'event'      => $event,
            'old_values' => $event === 'created' ? null : $this->getOriginal(),
            'new_values' => $event === 'deleted' ? null : $this->getAttributes(),
            'ip_address' => request()->ip(),
            'user_agent' => request()->userAgent(),
        ]);
    }
}
```

Aplique a trait nos models que precisam de auditoria e verifique se o histórico completo de alterações está sendo registrado corretamente.

---

## Recursos Adicionais

### Livros

- **Domain-Driven Design** — Eric Evans. O livro que definiu a disciplina de modelagem de domínio, incluindo como organizar estado e fronteiras entre contextos.
- **Implementing Domain-Driven Design** — Vaughn Vernon. Mais prático que o anterior, com exemplos de implementação real.
- **Patterns of Enterprise Application Architecture** — Martin Fowler. Referência para padrões como Repository, Unit of Work e Identity Map.

### Documentação e artigos

- [Laravel Request Lifecycle](https://laravel.com/docs/lifecycle) — documentação oficial sobre o ciclo de vida de uma requisição.
- [Immutability in PHP](https://stitcher.io/blog/php-8-readonly-properties) — Brent Roose sobre readonly properties no PHP 8.

### Vídeos

- Laracasts: "The PHP Practitioner" — Jeffrey Way cobrindo fundamentos de PHP com clareza.
- Laracasts: "Object-Oriented Principles in PHP" — base sólida para entender encapsulamento e controle de estado.

### Ferramentas

- **Laravel Debugbar** — visualiza queries, tempo de execução e fluxo de dados em tempo real.
- **Laravel Telescope** — monitoramento detalhado de requests, jobs, eventos e queries.
- **PHPStan / Larastan** — análise estática que encontra bugs de tipo e acesso a propriedades inexistentes antes do runtime.
- **Laravel Pint** — formatação consistente de código.

---

## Conclusão

Os cinco princípios que guiam uma boa gestão de estado e fluxo de dados são:

**Single Source of Truth** — cada informação tem exatamente uma fonte autoritativa. Derive ao invés de duplicar.

**Fluxo Unidirecional** — dados percorrem o sistema em uma direção previsível. Input → Validação → Processamento → Persistência → Output.

**Imutabilidade quando possível** — objetos que não mudam eliminam toda uma classe de bugs. Use `readonly` e Value Objects.

**Explícito é melhor que implícito** — mutações devem ser óbvias no código. Se um método modifica estado, o nome deve refletir isso.

**Fail Fast** — valide na fronteira, antes de processar. Dados inválidos nunca devem avançar pelo sistema.

Esses princípios não precisam ser aplicados todos de uma vez. Comece pela funcionalidade que mais causa problemas no seu sistema atual, refatore aplicando um ou dois conceitos, e observe a diferença. Com o tempo, o padrão se torna natural.

**Próximo passo:** conectar esses conceitos com **Business Logic** (como isolar e organizar regras de negócio) e **APIs and System Boundaries** (como definir contratos claros entre sistemas).
