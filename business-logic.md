# Business Logic (Lógica de Negócio)

## Índice

1. [Introdução](#introdução)
2. [Por que isso importa?](#por-que-isso-importa)
3. [Conceitos Fundamentais](#conceitos-fundamentais)
4. [Onde a Lógica deve viver?](#onde-a-lógica-deve-viver)
5. [Service Layer Pattern](#service-layer-pattern)
6. [Action Pattern](#action-pattern)
7. [Domain Driven Design](#domain-driven-design)
8. [Repository Pattern](#repository-pattern)
9. [Value Objects](#value-objects)
10. [Domain Events](#domain-events)
11. [Anti-patterns Comuns](#anti-patterns-comuns)
12. [Boas Práticas](#boas-práticas)
13. [Exercícios Práticos](#exercícios-práticos)
14. [Recursos Adicionais](#recursos-adicionais)

---

## Introdução

**Lógica de negócio** são as regras que definem como o seu sistema funciona — não do ponto de vista técnico, mas do ponto de vista do domínio que ele serve. São as regras que existiriam mesmo que o software não existisse.

Alguns exemplos concretos: "um pedido só pode ser cancelado se ainda não foi despachado", "desconto de 10% para compras acima de R$ 500", "funcionário só pode aprovar licitação se tiver certificação válida", "nota fiscal deve ser emitida em até 24h após confirmação de pagamento".

Essas regras são o coração do sistema. Todo o resto — controllers, rotas, views, banco de dados — existe para **servir** a essas regras. E é exatamente por isso que a forma como você organiza a lógica de negócio determina se o sistema será sustentável ou se vai virar um pesadelo de manutenção em poucos meses.

O padrão mais comum em projetos Laravel que crescem sem disciplina é o **controller gordo**: um único arquivo de 300+ linhas que valida input, aplica regras de negócio, faz queries, envia emails, gera PDFs e retorna a resposta — tudo no mesmo método. Funciona no começo. Depois, cada mudança exige ler e entender centenas de linhas para não quebrar algo inesperado.

A alternativa é separar responsabilidades em camadas com propósitos claros:

```
Controller (recebe requisição, delega, retorna resposta)
    ↓
Action / Service (orquestra lógica de negócio)
    ↓
Repository (abstrai acesso a dados)
    ↓
Model (representa a entidade e seus comportamentos intrínsecos)
```

Cada camada tem um papel bem definido. O controller não calcula. O service não faz query. O model não envia email. Quando essas fronteiras são respeitadas, o código se torna previsível, testável e fácil de modificar.

---

## Por que isso importa?

### O custo de lógica mal organizada

**Controllers gigantes.** Arquivos de 500+ linhas onde validação, cálculos, integrações e side effects estão todos juntos. Para entender o que o método `store` faz, você precisa ler tudo — não há como pular para a parte relevante.

**Duplicação silenciosa.** A regra "pedido não pode ser cancelado depois de despachado" aparece no controller web, no controller da API, no Job de cancelamento automático e no comando Artisan de manutenção. Quando a regra muda (agora pedidos despachados podem ser cancelados em até 2h), alguém atualiza três dos quatro lugares e o sistema fica inconsistente.

**Testes impossíveis.** Para testar se o cálculo de desconto funciona, você precisa montar uma requisição HTTP completa, autenticar um usuário, preparar o banco de dados e parsear a resposta. O que deveria ser um teste unitário de 5 linhas vira um teste de integração de 50.

**Mudanças com efeito cascata.** Alterar a fórmula de cálculo de imposto quebra o endpoint de criação de pedido, porque a lógica de imposto está embutida no controller junto com 15 outras responsabilidades. Ninguém percebe até que o QA encontra o bug — ou pior, o cliente encontra.

**Reuso zero.** A lógica de criação de requerimento vive dentro do controller web. Quando surge a necessidade de criar requerimentos via API, via comando Artisan ou via Job agendado, a única opção é copiar o código ou fazer gambiarras para chamar o controller de dentro de outro contexto.

### O que uma boa organização oferece

**Testabilidade real.** A regra de negócio vive em uma Action que recebe dados tipados e retorna um resultado. Testá-la requer apenas instanciar a classe e chamar o método — sem HTTP, sem sessão, sem banco (ou com banco, se for teste de integração, mas por escolha, não por obrigação).

**Reutilização natural.** A mesma Action funciona quando chamada pelo controller web, pelo controller da API, por um Job, por um comando Artisan ou por outro Service. O contexto de invocação muda; a lógica permanece a mesma.

**Manutenção localizada.** Quando a regra de cancelamento muda, a alteração é feita em um único lugar — a Action ou o método do Model que encapsula essa regra. Todos os consumidores herdam a mudança automaticamente.

**Legibilidade.** O controller diz `$this->createOrder->execute($data)`. Para entender o que acontece, você abre a Action e lê um fluxo linear de passos. Não precisa filtrar mentalmente código de validação HTTP, formatação de resposta e tratamento de exceções para encontrar a lógica.

**Colaboração em equipe.** Com responsabilidades bem separadas, dois desenvolvedores podem trabalhar em paralelo: um no controller/Resource (como os dados entram e saem) e outro na Action (o que acontece com esses dados). Sem conflitos de merge e sem pisar no trabalho do outro.

---

## Conceitos Fundamentais

### 1. Separação de Responsabilidades

Cada classe tem um único propósito. Quando você lê o nome da classe, já sabe o que ela faz — e, mais importante, o que ela **não** faz.

```php
// Controller: recebe requisição, delega, retorna resposta
class OrderController extends Controller
{
    public function store(
        StoreOrderRequest $request,
        CreateOrderAction $createOrder,
    ) {
        $order = $createOrder->execute($request->validated());

        return new OrderResource($order);
    }
}

// Action: contém a lógica de negócio
class CreateOrderAction
{
    public function execute(array $data): Order
    {
        // Toda a orquestração acontece aqui
    }
}

// Repository: acessa o banco de dados
class OrderRepository
{
    public function create(array $data): Order
    {
        return Order::create($data);
    }
}

// Model: representa a entidade e seus comportamentos intrínsecos
class Order extends Model
{
    public function canBeCancelled(): bool
    {
        return ! in_array($this->status, [
            OrderStatus::SHIPPED,
            OrderStatus::DELIVERED,
            OrderStatus::CANCELLED,
        ]);
    }
}
```

O controller tem 4 linhas de lógica. Se amanhã a criação de pedido precisar de uma nova etapa (verificar crédito do cliente, por exemplo), a mudança acontece na Action — o controller não é tocado.

### 2. Models: Anêmico vs Gordo vs Equilibrado

A discussão sobre quanto comportamento colocar no Model é uma das mais recorrentes em projetos Laravel. As três abordagens existem em um espectro:

**Modelo Anêmico** — apenas propriedades e relacionamentos, sem nenhum comportamento. Toda lógica vive em Services externos.

```php
// ❌ Anêmico demais — o Model é apenas um saco de dados
class Order extends Model
{
    protected $fillable = ['total', 'status'];
}

// Lógica espalhada pelo controller ou service
if ($order->status !== 'shipped' && $order->status !== 'delivered') {
    $order->status = 'cancelled';
    $order->save();
}
```

O problema: a regra de "quando um pedido pode ser cancelado" está exposta e duplicável. Qualquer lugar do código que precisar cancelar um pedido terá que reimplementar a verificação.

**Modelo Gordo** — tudo vive no Model, incluindo integrações externas e side effects pesados.

```php
// ❌ Gordo demais — o Model faz coisas que não são da sua responsabilidade
class Order extends Model
{
    public function cancel(): void
    {
        $this->update(['status' => 'cancelled']);
        $this->processRefund();           // Integração com gateway de pagamento
        $this->restoreInventory();         // Lógica de estoque
        $this->generateCancellationPDF();  // Geração de documento
        Mail::to($this->customer)->send(); // Envio de email
    }
}
```

O problema: o Model acumula responsabilidades demais. Testar o método `cancel` exige mock de gateway de pagamento, serviço de estoque, gerador de PDF e mailer.

**Modelo Equilibrado** — comportamentos que são **intrínsecos** à entidade vivem no Model. Orquestração complexa que envolve múltiplas entidades e serviços externos vive em Actions/Services.

```php
// ✅ Equilibrado — o Model conhece suas próprias regras
class Order extends Model
{
    public function canBeCancelled(): bool
    {
        return ! in_array($this->status, [
            OrderStatus::SHIPPED,
            OrderStatus::DELIVERED,
            OrderStatus::CANCELLED,
        ]);
    }

    public function markAsCancelled(): void
    {
        if (! $this->canBeCancelled()) {
            throw new OrderCannotBeCancelledException($this);
        }

        $this->update([
            'status' => OrderStatus::CANCELLED,
            'cancelled_at' => now(),
        ]);
    }

    public function total(): float
    {
        return $this->items->sum(fn ($item) => $item->price * $item->quantity);
    }
}

// Action: orquestra o processo completo de cancelamento
class CancelOrderAction
{
    public function __construct(
        private RefundService $refunds,
        private InventoryService $inventory,
    ) {}

    public function execute(Order $order): void
    {
        $order->markAsCancelled();
        $this->inventory->release($order);
        $this->refunds->process($order);
        event(new OrderCancelled($order));
    }
}
```

A pergunta para decidir onde colocar o comportamento é: **essa lógica faz sentido sem conhecer nenhuma outra entidade ou serviço externo?** Se sim, pertence ao Model. Se não, pertence à Action/Service.

### 3. Tell, Don't Ask

Em vez de interrogar um objeto sobre seu estado interno para tomar uma decisão fora dele, diga ao objeto o que fazer e deixe que ele decida internamente se é possível.

```php
// ❌ Ask — lê o estado, decide fora, modifica de fora
$order = Order::find($id);
if ($order->status === 'pending' && $order->payment_confirmed) {
    $order->status = 'confirmed';
    $order->save();
    Mail::to($order->customer)->send(new OrderConfirmed($order));
}

// ✅ Tell — pede para o objeto agir
$order->confirm();
```

```php
class Order extends Model
{
    public function confirm(): void
    {
        if (! $this->isPending() || ! $this->payment_confirmed) {
            throw new InvalidOrderStateException(
                "Pedido {$this->id} não pode ser confirmado no estado atual"
            );
        }

        $this->update(['status' => OrderStatus::CONFIRMED]);

        event(new OrderConfirmed($this));
    }
}
```

A vantagem não é apenas estética. No padrão "Ask", a regra de quando um pedido pode ser confirmado está exposta — qualquer desenvolvedor que esqueça de verificar `payment_confirmed` vai introduzir um bug. No padrão "Tell", a regra está encapsulada: é impossível confirmar um pedido sem pagamento, porque o próprio Model impede.

---

## Onde a Lógica deve viver?

### 1. Controller (Camada de Apresentação)

O controller é o ponto de entrada HTTP. Sua responsabilidade é exclusivamente: receber a requisição, delegar para a camada de negócio, e retornar a resposta. Nada mais.

```php
class OrderController extends Controller
{
    public function __construct(
        private CreateOrderAction $createOrder,
        private CancelOrderAction $cancelOrder,
    ) {}

    public function store(StoreOrderRequest $request)
    {
        $order = $this->createOrder->execute($request->validated());

        return new OrderResource($order);
    }

    public function cancel(Order $order)
    {
        $this->cancelOrder->execute($order);

        return response()->json(['message' => 'Pedido cancelado']);
    }
}
```

O que **não** deve estar no controller: cálculos de negócio, queries ao banco (além do Route Model Binding), regras condicionais complexas, envio de emails, integrações com APIs externas. Se o método do controller tem mais de 10 linhas, provavelmente está fazendo trabalho demais.

### 2. Action / Service (Camada de Aplicação)

É aqui que a lógica de negócio vive. A Action recebe dados já validados, aplica as regras, orquestra as operações necessárias e retorna o resultado.

```php
class CreateOrderAction
{
    public function __construct(
        private OrderRepository $orders,
        private ProductRepository $products,
        private InventoryService $inventory,
        private PaymentGateway $payment,
    ) {}

    public function execute(array $data): Order
    {
        return DB::transaction(function () use ($data) {
            // 1. Verificar estoque de todos os itens
            foreach ($data['items'] as $item) {
                $product = $this->products->findOrFail($item['product_id']);

                if (! $this->inventory->hasStock($product, $item['quantity'])) {
                    throw new InsufficientStockException($product, $item['quantity']);
                }
            }

            // 2. Criar o pedido
            $order = $this->orders->create([
                'customer_id' => $data['customer_id'],
                'total' => $this->calculateTotal($data['items']),
                'status' => OrderStatus::PENDING,
            ]);

            // 3. Adicionar itens
            foreach ($data['items'] as $item) {
                $order->items()->create($item);
            }

            // 4. Reservar estoque
            $this->inventory->reserve($order);

            // 5. Processar pagamento
            $this->payment->charge($order);

            // 6. Disparar evento (fora da transaction, idealmente)
            event(new OrderCreated($order));

            return $order;
        });
    }

    private function calculateTotal(array $items): float
    {
        return collect($items)->sum(fn ($item) => $item['price'] * $item['quantity']);
    }
}
```

A Action lê como uma receita: verificar estoque, criar pedido, adicionar itens, reservar estoque, cobrar, notificar. Cada passo é delegado para o serviço responsável. Se amanhã a regra de estoque mudar, a alteração é feita no `InventoryService` — a Action não precisa ser modificada.

### 3. Repository (Camada de Dados)

O Repository abstrai o acesso ao banco de dados. Ele sabe *como* buscar e persistir dados, mas não sabe *por que* esses dados estão sendo buscados.

```php
class OrderRepository
{
    public function create(array $data): Order
    {
        return Order::create($data);
    }

    public function findOrFail(int $id): Order
    {
        return Order::with(['items', 'customer'])->findOrFail($id);
    }

    public function findPendingOlderThan(Carbon $date): Collection
    {
        return Order::query()
            ->where('status', OrderStatus::PENDING)
            ->where('created_at', '<', $date)
            ->with(['items', 'customer'])
            ->get();
    }

    public function findByCustomer(int $customerId): Collection
    {
        return Order::query()
            ->where('customer_id', $customerId)
            ->with(['items', 'customer'])
            ->latest()
            ->get();
    }
}
```

O que **não** deve estar no Repository: regras de negócio, cálculos, envio de emails, integrações externas. Se o Repository tem um `if` que verifica status ou calcula valor, essa lógica pertence a outra camada.

### 4. Model (Camada de Domínio)

O Model representa a entidade e seus comportamentos intrínsecos — coisas que são verdadeiras sobre a entidade independente do contexto em que ela é usada.

```php
class Order extends Model
{
    protected $casts = [
        'status' => OrderStatus::class,
    ];

    // Scopes — encapsulam queries recorrentes
    public function scopePending(Builder $query): Builder
    {
        return $query->where('status', OrderStatus::PENDING);
    }

    // Accessors — derivam dados a partir do estado
    protected function totalFormatted(): Attribute
    {
        return Attribute::get(
            fn () => 'R$ ' . number_format($this->total, 2, ',', '.')
        );
    }

    // Comportamento intrínseco — regras que pertencem à entidade
    public function canBeShipped(): bool
    {
        return $this->status === OrderStatus::CONFIRMED
            && $this->payment_confirmed;
    }

    public function ship(): void
    {
        if (! $this->canBeShipped()) {
            throw new OrderCannotBeShippedException($this);
        }

        $this->update([
            'status' => OrderStatus::SHIPPED,
            'shipped_at' => now(),
        ]);

        event(new OrderShipped($this));
    }

    // Relationships
    public function items(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }

    public function customer(): BelongsTo
    {
        return $this->belongsTo(Customer::class);
    }
}
```

---

## Service Layer Pattern

Services são classes que encapsulam lógica de negócio que envolve múltiplas entidades ou coordena operações complexas. Diferente de Actions (que fazem uma coisa), Services podem ter vários métodos relacionados a um mesmo contexto.

### Service focado em um contexto

```php
namespace App\Services;

class InvoiceService
{
    public function __construct(
        private InvoiceRepository $invoices,
        private TaxCalculator $taxCalculator,
    ) {}

    public function generate(Order $order): Invoice
    {
        $subtotal = $order->items->sum('subtotal');
        $tax = $this->taxCalculator->calculate($subtotal, $order->customer->state);

        return $this->invoices->create([
            'order_id' => $order->id,
            'subtotal' => $subtotal,
            'tax' => $tax,
            'total' => $subtotal + $tax,
            'due_date' => now()->addDays(7),
        ]);
    }

    public function cancel(Invoice $invoice): void
    {
        if ($invoice->isPaid()) {
            throw new InvoiceAlreadyPaidException($invoice);
        }

        $invoice->markAsCancelled();

        event(new InvoiceCancelled($invoice));
    }
}
```

Perceba que `generate` e `cancel` são operações diferentes sobre a mesma entidade (Invoice). Faz sentido agrupá-las em um Service quando elas compartilham dependências e contexto.

### Service para processo complexo de domínio

```php
class ProcessoLicitacaoService
{
    public function __construct(
        private ProcessoRepository $processos,
        private DocumentoService $documentos,
        private NotificationService $notifications,
        private AuditoriaService $auditoria,
    ) {}

    public function publicar(Processo $processo): void
    {
        // 1. Validar pré-condições
        $this->validarDocumentacaoObrigatoria($processo);

        // 2. Gerar identificação formal
        $processo->numero = $this->gerarNumeroProcesso();

        // 3. Transicionar estado
        $processo->status = ProcessoStatus::PUBLICADO;
        $processo->data_publicacao = now();
        $processo->save();

        // 4. Publicar no diário oficial (integração externa)
        $this->publicarDiarioOficial($processo);

        // 5. Notificar interessados
        $this->notifications->notificarFornecedores($processo);

        // 6. Registrar para auditoria
        $this->auditoria->registrar('processo_publicado', $processo);
    }

    private function validarDocumentacaoObrigatoria(Processo $processo): void
    {
        $obrigatorios = ['termo_referencia', 'edital', 'orcamento'];
        $presentes = $processo->documentos->pluck('tipo')->toArray();
        $faltantes = array_diff($obrigatorios, $presentes);

        if (! empty($faltantes)) {
            throw new DocumentacaoIncompleta(
                'Documentos faltantes: ' . implode(', ', $faltantes)
            );
        }
    }

    private function gerarNumeroProcesso(): string
    {
        $ano = now()->year;
        $sequencia = Processo::whereYear('created_at', $ano)->count() + 1;

        return sprintf('%04d/%d', $sequencia, $ano);
    }

    private function publicarDiarioOficial(Processo $processo): void
    {
        Http::timeout(30)->post(config('services.diario_oficial.url'), [
            'tipo' => 'licitacao',
            'numero_processo' => $processo->numero,
            'conteudo' => $this->documentos->gerarPublicacao($processo),
        ]);
    }
}
```

O Service lê como a descrição de um processo de negócio: para publicar uma licitação, primeiro valide os documentos, gere o número, atualize o status, publique no diário oficial, notifique os fornecedores e registre na auditoria. Cada etapa é um método privado ou uma chamada a um serviço especializado.

---

## Action Pattern

Actions são classes com um único método público (`execute` ou `handle`). Cada Action faz **uma** coisa. Enquanto Services agrupam operações relacionadas, Actions são mais granulares e focadas.

### Action simples

```php
namespace App\Actions\Orders;

class CancelOrderAction
{
    public function __construct(
        private RefundService $refundService,
        private InventoryService $inventoryService,
    ) {}

    public function execute(Order $order): void
    {
        // O Model cuida da validação de estado
        $order->markAsCancelled();

        DB::transaction(function () use ($order) {
            // Devolver estoque
            $this->inventoryService->release($order);

            // Processar reembolso se houve pagamento
            if ($order->payment_confirmed) {
                $this->refundService->process($order);
            }
        });

        // Eventos disparados fora da transaction
        event(new OrderCancelled($order));
    }
}
```

### Action com DTO

Para Actions que recebem muitos parâmetros, um DTO formaliza o contrato de entrada:

```php
final readonly class CreateUserData
{
    public function __construct(
        public string $name,
        public string $email,
        public string $password,
        public ?int $departmentId = null,
    ) {}

    public static function fromRequest(StoreUserRequest $request): self
    {
        return new self(
            name: $request->string('name'),
            email: $request->string('email'),
            password: $request->string('password'),
            departmentId: $request->integer('department_id') ?: null,
        );
    }
}
```

```php
class CreateUserAction
{
    public function __construct(
        private UserRepository $users,
        private DepartmentRepository $departments,
    ) {}

    public function execute(CreateUserData $data): User
    {
        // Validação de regra de negócio
        if ($this->users->emailExists($data->email)) {
            throw new EmailAlreadyTakenException($data->email);
        }

        if ($data->departmentId) {
            $department = $this->departments->findOrFail($data->departmentId);

            if (! $department->acceptingNewMembers()) {
                throw new DepartmentFullException($department);
            }
        }

        $user = $this->users->create([
            'name' => $data->name,
            'email' => $data->email,
            'password' => Hash::make($data->password),
            'department_id' => $data->departmentId,
        ]);

        event(new UserCreated($user));

        return $user;
    }
}
```

```php
// Controller — apenas cola as peças
public function store(StoreUserRequest $request, CreateUserAction $action)
{
    $user = $action->execute(CreateUserData::fromRequest($request));

    return new UserResource($user);
}
```

A grande vantagem das Actions sobre Services para operações como criação de usuário: a Action pode ser reutilizada em qualquer contexto (controller web, API, seeder, comando Artisan) sem modificação. O DTO garante que ela sempre recebe os dados no formato esperado, independente de onde vêm.

### Quando usar Service vs Action?

A decisão é pragmática. Use **Action** quando a operação é autocontida e focada: criar pedido, cancelar pedido, publicar processo. Use **Service** quando várias operações compartilham dependências e contexto: geração e cancelamento de faturas, cálculos tributários, operações sobre um mesmo agregado. Em projetos menores, Actions são suficientes para quase tudo. À medida que o sistema cresce e padrões emergem, alguns grupos de Actions podem ser consolidados em Services.

---

## Domain Driven Design

DDD propõe organizar o código por **domínio de negócio** ao invés de por **tipo técnico**. Em vez de ter uma pasta `Controllers/` com 40 controllers de contextos diferentes, você agrupa tudo que pertence a um mesmo domínio (Vendas, Licitações, RH) em um único lugar.

### Estrutura convencional (por tipo técnico)

```
app/
├── Http/Controllers/
│   ├── OrderController.php
│   ├── ProductController.php
│   ├── ProcessoController.php
│   └── FuncionarioController.php
├── Models/
│   ├── Order.php
│   ├── Product.php
│   ├── Processo.php
│   └── Funcionario.php
└── Services/
    ├── OrderService.php
    ├── ProductService.php
    └── ProcessoService.php
```

O problema aparece quando o sistema cresce: a pasta `Models/` tem 50 arquivos de domínios diferentes, e para entender o módulo de Licitações você precisa navegar entre 5 pastas diferentes, filtrando mentalmente o que pertence ao seu contexto.

### Estrutura DDD (por domínio)

```
app/
├── Domain/
│   ├── Sales/
│   │   ├── Models/
│   │   │   ├── Order.php
│   │   │   └── OrderItem.php
│   │   ├── Actions/
│   │   │   ├── CreateOrderAction.php
│   │   │   └── CancelOrderAction.php
│   │   ├── Repositories/
│   │   │   └── OrderRepository.php
│   │   ├── Events/
│   │   │   ├── OrderCreated.php
│   │   │   └── OrderCancelled.php
│   │   └── Exceptions/
│   │       └── InsufficientStockException.php
│   │
│   ├── Catalog/
│   │   ├── Models/
│   │   │   ├── Product.php
│   │   │   └── Category.php
│   │   ├── Actions/
│   │   │   └── CreateProductAction.php
│   │   └── Repositories/
│   │       └── ProductRepository.php
│   │
│   └── Licitacao/
│       ├── Models/
│       │   ├── Processo.php
│       │   └── Lance.php
│       ├── Actions/
│       │   └── PublicarProcessoAction.php
│       ├── Services/
│       │   └── ProcessoLicitacaoService.php
│       └── Enums/
│           └── ProcessoStatus.php
│
└── Http/
    └── Controllers/
        ├── Sales/
        │   └── OrderController.php
        └── Licitacao/
            └── ProcessoController.php
```

A vantagem é de navegabilidade: para entender o módulo de Licitações, basta olhar `app/Domain/Licitacao/`. Tudo que pertence àquele contexto está ali — Models, Actions, Repositories, Events, Exceptions.

### Exemplo de domínio completo

```php
// Domain/Licitacao/Models/Processo.php
namespace App\Domain\Licitacao\Models;

class Processo extends Model
{
    protected $casts = [
        'status' => ProcessoStatus::class,
    ];

    public function podeSerPublicado(): bool
    {
        return $this->status === ProcessoStatus::RASCUNHO
            && $this->documentacaoCompleta();
    }

    public function documentacaoCompleta(): bool
    {
        $obrigatorios = ['termo_referencia', 'edital', 'orcamento'];
        $presentes = $this->documentos->pluck('tipo')->toArray();

        return empty(array_diff($obrigatorios, $presentes));
    }

    public function documentos(): HasMany
    {
        return $this->hasMany(Documento::class);
    }
}
```

```php
// Domain/Licitacao/Actions/PublicarProcessoAction.php
namespace App\Domain\Licitacao\Actions;

class PublicarProcessoAction
{
    public function __construct(
        private DiarioOficialService $diarioOficial,
        private NotificationService $notifications,
    ) {}

    public function execute(Processo $processo): void
    {
        if (! $processo->podeSerPublicado()) {
            throw new ProcessoNaoPodeSerPublicadoException($processo);
        }

        DB::transaction(function () use ($processo) {
            $processo->update([
                'status' => ProcessoStatus::PUBLICADO,
                'data_publicacao' => now(),
                'numero' => $this->gerarNumero(),
            ]);

            $this->diarioOficial->publicar($processo);
        });

        $this->notifications->notificarFornecedores($processo);

        event(new ProcessoPublicado($processo));
    }

    private function gerarNumero(): string
    {
        $ano = now()->year;
        $seq = Processo::whereYear('created_at', $ano)->count() + 1;

        return sprintf('%04d/%d', $seq, $ano);
    }
}
```

```php
// Http/Controllers/Licitacao/ProcessoController.php
namespace App\Http\Controllers\Licitacao;

class ProcessoController extends Controller
{
    public function publicar(Processo $processo, PublicarProcessoAction $action)
    {
        $action->execute($processo);

        return redirect()
            ->route('licitacoes.show', $processo)
            ->with('success', 'Processo publicado com sucesso!');
    }
}
```

O controller tem 5 linhas. A validação de pré-condições está no Model (`podeSerPublicado`). A orquestração está na Action. A integração externa está em um Service dedicado. Cada classe tem uma responsabilidade clara.

---

## Repository Pattern

O Repository cria uma abstração entre a lógica de negócio e o mecanismo de persistência. Sua Action não sabe (e não deve saber) se os dados vêm do Eloquent, de uma API externa ou de um arquivo CSV.

### Interface como contrato

```php
namespace App\Contracts;

interface OrderRepositoryInterface
{
    public function find(int $id): ?Order;
    public function findOrFail(int $id): Order;
    public function create(array $data): Order;
    public function update(Order $order, array $data): Order;
    public function delete(Order $order): void;
    public function findByCustomer(int $customerId): Collection;
    public function findPending(): Collection;
}
```

### Implementação com Eloquent

```php
namespace App\Repositories;

class EloquentOrderRepository implements OrderRepositoryInterface
{
    public function find(int $id): ?Order
    {
        return Order::with(['items', 'customer'])->find($id);
    }

    public function findOrFail(int $id): Order
    {
        return Order::with(['items', 'customer'])->findOrFail($id);
    }

    public function create(array $data): Order
    {
        return Order::create($data);
    }

    public function update(Order $order, array $data): Order
    {
        $order->update($data);
        return $order->fresh();
    }

    public function delete(Order $order): void
    {
        $order->delete();
    }

    public function findByCustomer(int $customerId): Collection
    {
        return Order::query()
            ->where('customer_id', $customerId)
            ->with(['items', 'customer'])
            ->latest()
            ->get();
    }

    public function findPending(): Collection
    {
        return Order::pending()->with(['items', 'customer'])->get();
    }
}
```

### Binding no Service Provider

```php
namespace App\Providers;

class RepositoryServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(
            OrderRepositoryInterface::class,
            EloquentOrderRepository::class
        );
    }
}
```

### Consumo via interface

```php
class OrderService
{
    public function __construct(
        private OrderRepositoryInterface $orders, // Interface, não implementação
    ) {}

    public function getCustomerOrders(int $customerId): Collection
    {
        return $this->orders->findByCustomer($customerId);
    }
}
```

Os benefícios concretos: a implementação pode ser trocada sem alterar nenhum consumidor (Eloquent → Query Builder → API HTTP). Em testes, você pode injetar um fake que retorna dados pré-definidos sem tocar no banco. Queries complexas ficam centralizadas em um único lugar — se o índice do banco mudar e a query precisar ser otimizada, a alteração acontece no Repository, não nos 12 lugares que faziam a mesma consulta.

Uma nota pragmática: em projetos menores, o overhead de criar interface + implementação + binding para cada entidade pode não compensar. Comece usando o Eloquent diretamente nas Actions e extraia Repositories quando a necessidade de abstração surgir — quando precisar trocar a implementação, quando queries complexas se repetirem, ou quando os testes ficarem difíceis de escrever.

---

## Value Objects

Value Objects são objetos imutáveis que representam **valores** — não entidades. A diferença: duas entidades com os mesmos dados são coisas diferentes (dois pedidos com o mesmo total são pedidos distintos). Dois Value Objects com os mesmos dados são **iguais** (R$ 100 é R$ 100, independente de qual instância você está segurando).

### Money

O Value Object mais útil e mais negligenciado. Representar dinheiro como `float` é um bug esperando para acontecer.

```php
namespace App\ValueObjects;

final readonly class Money
{
    public function __construct(
        public int $amount,       // Sempre em centavos
        public string $currency,
    ) {
        if ($amount < 0) {
            throw new InvalidArgumentException('Valor não pode ser negativo');
        }
    }

    public static function fromFloat(float $value, string $currency = 'BRL'): self
    {
        return new self((int) round($value * 100), $currency);
    }

    public function add(self $other): self
    {
        $this->assertSameCurrency($other);

        return new self($this->amount + $other->amount, $this->currency);
    }

    public function subtract(self $other): self
    {
        $this->assertSameCurrency($other);

        return new self($this->amount - $other->amount, $this->currency);
    }

    public function multiply(int $factor): self
    {
        return new self($this->amount * $factor, $this->currency);
    }

    public function formatted(): string
    {
        $value = $this->amount / 100;

        return match ($this->currency) {
            'BRL' => 'R$ ' . number_format($value, 2, ',', '.'),
            'USD' => '$' . number_format($value, 2, '.', ','),
            default => $this->currency . ' ' . number_format($value, 2),
        };
    }

    public function equals(self $other): bool
    {
        return $this->amount === $other->amount
            && $this->currency === $other->currency;
    }

    private function assertSameCurrency(self $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new InvalidArgumentException(
                "Não é possível operar {$this->currency} com {$other->currency}"
            );
        }
    }
}

// Uso
$preco = Money::fromFloat(99.90);
$total = $preco->multiply(3);
echo $total->formatted(); // "R$ 299,70"
```

Por que centavos? Porque `0.1 + 0.2` em float dá `0.30000000000000004`. Com inteiros em centavos, `10 + 20 = 30` — sem surpresas. A regra geral: **nunca use float para dinheiro**.

### CPF

Um Value Object que encapsula validação, formatação e sanitização. Depois de construído, você sabe que o CPF é válido — não precisa verificar novamente.

```php
final readonly class CPF
{
    public readonly string $value;

    public function __construct(string $input)
    {
        $this->value = preg_replace('/\D/', '', $input);

        if (! $this->isValid()) {
            throw new InvalidCPFException($input);
        }
    }

    private function isValid(): bool
    {
        if (strlen($this->value) !== 11) {
            return false;
        }

        // Rejeitar sequências repetidas (111.111.111-11, etc.)
        if (preg_match('/^(\d)\1{10}$/', $this->value)) {
            return false;
        }

        // Validar dígitos verificadores
        for ($t = 9; $t < 11; $t++) {
            $sum = 0;
            for ($c = 0; $c < $t; $c++) {
                $sum += $this->value[$c] * (($t + 1) - $c);
            }

            $digit = ((10 * $sum) % 11) % 10;

            if ((int) $this->value[$t] !== $digit) {
                return false;
            }
        }

        return true;
    }

    public function formatted(): string
    {
        return preg_replace(
            '/(\d{3})(\d{3})(\d{3})(\d{2})/',
            '$1.$2.$3-$4',
            $this->value
        );
    }

    public function __toString(): string
    {
        return $this->formatted();
    }
}
```

### Email

```php
final readonly class Email
{
    public function __construct(public string $value)
    {
        if (! filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidEmailException($value);
        }
    }

    public function domain(): string
    {
        return substr($this->value, strpos($this->value, '@') + 1);
    }

    public function isGovernment(): bool
    {
        return str_ends_with($this->domain(), '.gov.br');
    }

    public function __toString(): string
    {
        return $this->value;
    }
}

$email = new Email('contato@itagi.ba.gov.br');
$email->isGovernment(); // true
```

O padrão é consistente: o construtor valida, e a existência do objeto garante a validade do valor. Se você recebe um `CPF` como parâmetro de método, não precisa validar novamente — o tipo já é a garantia.

---

## Domain Events

Eventos representam **fatos** que aconteceram no domínio: "um pedido foi criado", "um processo foi publicado", "uma fatura foi cancelada". Não são comandos (que pedem para algo acontecer) — são notificações de que algo **já** aconteceu.

### O evento como fato

```php
namespace App\Domain\Sales\Events;

class OrderPlaced
{
    public function __construct(
        public readonly Order $order,
    ) {}
}
```

### Listeners como reações independentes

Cada listener é uma reação ao evento, independente das demais. Se um novo requisito surgir ("quando um pedido for criado, notificar o Slack da equipe de logística"), basta adicionar um listener — sem tocar em nenhum código existente.

```php
class SendOrderConfirmation
{
    public function handle(OrderPlaced $event): void
    {
        Mail::to($event->order->customer)
            ->send(new OrderConfirmationMail($event->order));
    }
}

class UpdateInventory
{
    public function handle(OrderPlaced $event): void
    {
        foreach ($event->order->items as $item) {
            $item->product->decreaseStock($item->quantity);
        }
    }
}

class GenerateInvoice
{
    public function handle(OrderPlaced $event): void
    {
        Invoice::create([
            'order_id' => $event->order->id,
            'amount' => $event->order->total,
            'due_date' => now()->addDays(7),
        ]);
    }
}
```

### Registro dos listeners

```php
// EventServiceProvider
protected $listen = [
    OrderPlaced::class => [
        SendOrderConfirmation::class,
        UpdateInventory::class,
        GenerateInvoice::class,
    ],
];
```

### Disparando o evento

```php
class CreateOrderAction
{
    public function execute(array $data): Order
    {
        $order = DB::transaction(function () use ($data) {
            $order = Order::create($data);
            // ... lógica de criação ...
            return $order;
        });

        // Evento disparado DEPOIS da transaction confirmar.
        // Se a transaction falhar, o evento não é disparado.
        event(new OrderPlaced($order));

        return $order;
    }
}
```

Note que o evento é disparado **fora** da `DB::transaction`. Se estivesse dentro e um listener falhasse (ex: timeout ao enviar email), toda a transaction faria rollback — o pedido seria perdido por causa de um email que não foi. Side effects assíncronos (emails, notificações, integrações) devem ser tolerantes a falhas e, idealmente, processados via fila.

---

## Anti-patterns Comuns

### 1. God Service

Um Service que cresce indefinidamente até ter dezenas de métodos e centenas de linhas. Geralmente começa como `OrderService` e termina como "tudo que remotamente envolve pedidos".

```php
// ❌ Um Service para governar todos
class OrderService
{
    public function create() {}
    public function update() {}
    public function delete() {}
    public function cancel() {}
    public function ship() {}
    public function refund() {}
    public function sendEmail() {}
    public function generatePDF() {}
    public function exportCSV() {}
    // ... 50 métodos e 2000 linhas
}

// ✅ Cada operação é uma classe focada
class CreateOrderAction {}
class CancelOrderAction {}
class ShipOrderAction {}
class OrderPdfGenerator {}
class OrderCsvExporter {}
```

A regra: se o Service está sendo injetado em muitos contextos diferentes e cada contexto usa apenas 1-2 métodos, ele está grande demais. Quebre em Actions ou Services menores.

### 2. Feature Envy

Quando um Service manipula intensamente o estado interno de um Model ao invés de pedir para o Model fazer o trabalho:

```php
// ❌ O Service "inveja" os dados do Model
class OrderService
{
    public function calculateTotal(Order $order): float
    {
        $total = 0;
        foreach ($order->items as $item) {
            $total += $item->price * $item->quantity;
        }
        return $total;
    }
}

// ✅ O cálculo pertence ao Model — é ele quem conhece seus itens
class Order extends Model
{
    public function total(): float
    {
        return $this->items->sum(fn ($item) => $item->price * $item->quantity);
    }
}
```

A heurística: se um método acessa mais dados de outro objeto do que do próprio, ele provavelmente pertence àquele outro objeto.

### 3. Acoplamento a implementações concretas

```php
// ❌ Acoplado — impossível testar sem banco e servidor SMTP
class OrderService
{
    public function create(array $data): Order
    {
        $order = Order::create($data);                   // Eloquent direto
        Mail::to('admin@example.com')->send(new Mailable()); // Mail direto

        return $order;
    }
}

// ✅ Desacoplado — dependências injetadas via interface
class CreateOrderAction
{
    public function __construct(
        private OrderRepositoryInterface $orders,
        private NotificationServiceInterface $notifications,
    ) {}

    public function execute(array $data): Order
    {
        $order = $this->orders->create($data);
        $this->notifications->orderCreated($order);

        return $order;
    }
}
```

Em testes, você pode injetar um `FakeOrderRepository` que opera em memória e um `FakeNotificationService` que apenas registra quais notificações foram disparadas — sem banco, sem SMTP, sem latência.

### 4. Lógica de negócio no Blade ou no Vue

```php
// ❌ Regra de negócio no template
@if($order->status === 'pending' && $order->created_at->diffInHours(now()) < 24)
    <button>Cancelar</button>
@endif

// ✅ Regra encapsulada no Model
// Model
public function canBeCancelled(): bool
{
    return $this->status === OrderStatus::PENDING
        && $this->created_at->diffInHours(now()) < 24;
}

// Template
@if($order->canBeCancelled())
    <button>Cancelar</button>
@endif
```

Se amanhã o prazo mudar de 24h para 48h, a alteração é feita no Model — um lugar — ao invés de em cada template que verifica essa condição.

---

## Boas Práticas

### 1. Single Responsibility Principle

Cada classe faz uma coisa e faz bem. Se o nome da classe não descreve completamente sua responsabilidade, ela provavelmente faz coisas demais.

```php
// Cada classe tem um propósito claro e limitado
class CalculateOrderTax
{
    public function calculate(Order $order, string $state): Money
    {
        // Só calcula imposto — não aplica, não persiste, não notifica
    }
}

class ApplyOrderDiscount
{
    public function apply(Order $order, Coupon $coupon): void
    {
        // Só aplica desconto — não valida o cupom, não envia email
    }
}
```

### 2. Dependency Injection — sempre

Nunca instancie dependências manualmente dentro de um método. Injeção de dependência torna as relações entre classes visíveis e testáveis.

```php
// ❌ Dependência escondida — impossível substituir em testes
class OrderService
{
    public function create(): void
    {
        $mailer = new SmtpMailer(); // Quem lê a assinatura não sabe que isso existe
        $mailer->send(/* ... */);
    }
}

// ✅ Dependência explícita — visível na assinatura do construtor
class CreateOrderAction
{
    public function __construct(
        private MailerInterface $mailer,
    ) {}

    public function execute(): void
    {
        $this->mailer->send(/* ... */);
    }
}
```

### 3. Validação em camadas

Cada camada valida o que é da sua responsabilidade. Validações não são redundantes — são complementares.

```php
// Camada HTTP: formato e existência de campos
class StoreOrderRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'customer_id' => 'required|exists:customers,id',
            'items' => 'required|array|min:1',
            'items.*.product_id' => 'required|exists:products,id',
            'items.*.quantity' => 'required|integer|min:1',
        ];
    }
}

// Camada de aplicação: regras de negócio
class CreateOrderAction
{
    public function execute(array $data): Order
    {
        foreach ($data['items'] as $item) {
            $product = $this->products->findOrFail($item['product_id']);

            if (! $this->inventory->hasStock($product, $item['quantity'])) {
                throw new InsufficientStockException($product, $item['quantity']);
            }
        }

        // ... criação do pedido
    }
}

// Camada de domínio: invariantes da entidade
class Order extends Model
{
    public function addItem(Product $product, int $quantity): void
    {
        if ($quantity <= 0) {
            throw new InvalidArgumentException('Quantidade deve ser positiva');
        }

        // ...
    }
}
```

O FormRequest verifica se o campo existe e se o formato está correto. A Action verifica se há estoque disponível. O Model verifica se a quantidade faz sentido para a operação. Cada validação protege contra um tipo diferente de erro.

### 4. Transactions para operações compostas

Quando uma operação envolve múltiplas escritas no banco que precisam ser atômicas (ou todas acontecem ou nenhuma acontece), use transactions:

```php
public function execute(array $data): Order
{
    return DB::transaction(function () use ($data) {
        $order = $this->orders->create($data);

        foreach ($data['items'] as $item) {
            $order->items()->create($item);
            $this->inventory->decrease($item['product_id'], $item['quantity']);
        }

        $this->payment->process($order);

        return $order;
    });

    // Side effects (emails, eventos) ficam FORA da transaction
}
```

Se o pagamento falhar, o estoque volta ao normal e o pedido não é criado. Sem a transaction, você teria um pedido sem pagamento e estoque já descontado — um estado inconsistente que exige correção manual.

---

## Exercícios Práticos

### Exercício 1: Refatorar um controller gordo

Pegue este controller e separe as responsabilidades em camadas:

```php
// ANTES — tudo em um lugar só
class RequerimentoController extends Controller
{
    public function store(Request $request)
    {
        $validated = $request->validate([
            'tipo_id' => 'required|exists:tipos,id',
            'descricao' => 'required|min:10',
        ]);

        $protocolo = date('Y') . '-' . str_pad(
            Requerimento::count() + 1, 6, '0', STR_PAD_LEFT
        );

        $requerimento = Requerimento::create([
            'cidadao_id' => auth()->id(),
            'tipo_id' => $validated['tipo_id'],
            'descricao' => $validated['descricao'],
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

        Mail::to('atendimento@prefeitura.gov.br')
            ->send(new NovoRequerimento($requerimento));

        return redirect()->route('requerimentos.show', $requerimento);
    }
}
```

O exercício consiste em criar: um `StoreRequerimentoRequest` com as regras de validação e `prepareForValidation`, um DTO `CreateRequerimentoData`, uma Action `CreateRequerimentoAction` que orquestra a criação, um evento `RequerimentoCriado` com listener para o envio de email, e um controller enxuto que apenas delega.

### Exercício 2: Criar Value Objects para o domínio brasileiro

Implemente Value Objects para CNPJ, CEP e Telefone, seguindo o padrão de CPF e Email mostrados neste documento. Cada Value Object deve validar no construtor, sanitizar a entrada, oferecer formatação, e implementar `__toString`.

Para CNPJ: validar os dois dígitos verificadores, aceitar entrada com ou sem máscara, formatar como `XX.XXX.XXX/XXXX-XX`.

Para CEP: validar formato de 8 dígitos, formatar como `XXXXX-XXX`, e opcionalmente integrar com a API dos Correios via método `lookup()`.

Para Telefone: aceitar celular (11 dígitos) e fixo (10 dígitos), validar DDD, formatar com parênteses e hífen.

### Exercício 3: Implementar Domain Events para Licitações

Para o sistema de licitações, crie o evento `ProcessoPublicado` e três listeners independentes:

- `PublicarDiarioOficial` — integra com a API do diário oficial para publicação formal.
- `NotificarFornecedores` — envia email/notificação para fornecedores cadastrados na modalidade do processo.
- `RegistrarAuditoria` — grava registro de auditoria com dados do usuário, IP e timestamp.

Registre tudo no EventServiceProvider e garanta que os listeners são tolerantes a falhas (se o diário oficial estiver fora do ar, o processo ainda deve ser publicado — a publicação no diário pode ser retentada via fila).

---

## Recursos Adicionais

### Livros

- **Domain-Driven Design** — Eric Evans. O livro fundacional sobre como modelar software a partir do domínio de negócio. Denso, mas essencial.
- **Clean Code** — Robert C. Martin. Princípios de escrita de código legível e manutenível, com foco em nomes, funções pequenas e responsabilidade única.
- **Refactoring** — Martin Fowler. Catálogo de técnicas para melhorar código existente sem alterar comportamento. Referência para quando você precisa refatorar um controller gordo.

### Cursos

- Laracasts: "SOLID Principles in PHP" — Jeffrey Way explicando os cinco princípios com exemplos em PHP/Laravel.
- Laracasts: "Object-Oriented Principles in PHP" — base sólida para entender encapsulamento, herança e polimorfismo no contexto do PHP.

### Artigos e referências

- [Service Layer Pattern](https://martinfowler.com/eaaCatalog/serviceLayer.html) — Martin Fowler definindo o padrão de camada de serviço.
- [Value Objects](https://martinfowler.com/bliki/ValueObject.html) — Martin Fowler sobre quando e por que usar Value Objects.
- [Actions in Laravel](https://stitcher.io/blog/laravel-beyond-crud-03-actions) — Brent Roose sobre o padrão Action no contexto do Laravel.

---

## Conclusão

Lógica de negócio bem organizada é o que separa um sistema que escala de um que paralisa. Os princípios são poucos, mas exigem disciplina:

**Controller fino.** O controller recebe, delega e retorna. Se ele tem mais de 10 linhas de lógica, algo está no lugar errado.

**Lógica em Actions e Services.** Regras de negócio vivem em classes dedicadas, com dependências injetadas e responsabilidades claras. Uma Action por operação; um Service quando várias operações compartilham contexto.

**Model com comportamento intrínseco.** O Model sabe suas próprias regras: quando pode ser cancelado, como calcular seu total, quais transições de estado são válidas. Orquestração complexa fica fora dele.

**Repository para abstração de dados.** O código de negócio não sabe se os dados vêm do Eloquent, de uma API ou de um CSV. O Repository é a fronteira entre lógica e persistência.

**Value Objects para valores com regras.** CPF, CNPJ, Money, Email — se um valor tem validação, formatação ou operações específicas, ele merece uma classe própria.

**Events para desacoplamento.** Side effects (emails, notificações, integrações) são reações a fatos do domínio, não passos obrigatórios dentro da lógica principal.

Comece pelo exercício mais impactante: pegue o controller mais gordo do seu sistema e extraia a lógica para uma Action. Depois, identifique valores que aparecem como strings ou floats e promova-os a Value Objects. Cada passo melhora a testabilidade, a legibilidade e a confiança nas mudanças.

**Próximo passo:** combinar esses conceitos com **Architecture Decisions** (como organizar módulos e pastas) e **Preventing Messes** (como testes e análise estática protegem a arquitetura ao longo do tempo).
