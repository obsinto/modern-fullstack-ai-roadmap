# Architecture Decisions (Decisões de Arquitetura)

## Índice

1. [Introdução](#introdução)
2. [Por que isso importa?](#por-que-isso-importa)
3. [Conceitos Fundamentais](#conceitos-fundamentais)
4. [Arquitetura em Camadas](#arquitetura-em-camadas)
5. [Modular Monolith](#modular-monolith)
6. [Domain-Driven Design](#domain-driven-design)
7. [Hexagonal Architecture](#hexagonal-architecture)
8. [CQRS](#cqrs)
9. [Event Sourcing](#event-sourcing)
10. [Microservices vs Monolith](#microservices-vs-monolith)
11. [Organização de Pastas](#organização-de-pastas)
12. [Anti-patterns Comuns](#anti-patterns-comuns)
13. [Boas Práticas](#boas-práticas)
14. [Exercícios Práticos](#exercícios-práticos)
15. [Recursos Adicionais](#recursos-adicionais)

---

## Introdução

Decisões de arquitetura são as escolhas estruturais que determinam como o sistema é organizado, como suas partes se comunicam, como ele escala e quão fácil (ou difícil) é mudá-lo no futuro. São decisões que, uma vez tomadas, são caras para reverter — por isso precisam ser conscientes.

A analogia com construção civil é útil aqui. A arquitetura de um prédio define a fundação, a distribuição dos andares, onde passam as tubulações e a fiação elétrica. Você pode trocar a pintura e os móveis depois (equivalente a refatorar um controller ou trocar uma biblioteca), mas mover uma parede estrutural é caro e arriscado (equivalente a mudar de monolito para microserviços ou trocar o banco de dados).

O que torna decisões arquiteturais difíceis é que elas são tomadas no momento de maior incerteza — no início do projeto, quando você sabe menos sobre o domínio. A resposta pragmática para isso é: comece com a arquitetura mais simples que atende aos requisitos atuais, mas organize o código de forma que evolução seja possível sem reescrever tudo.

Para o ecossistema Laravel + Vue.js + PostgreSQL, isso geralmente significa começar com um monolito bem organizado em módulos, com separação clara de responsabilidades, e evoluir para padrões mais sofisticados (CQRS, Event Sourcing, microserviços) apenas quando a complexidade justificar.

---

## Por que isso importa?

### O que acontece sem arquitetura deliberada

**Spaghetti Code.** Tudo depende de tudo. O controller de pedidos importa diretamente o model de estoque, que referencia o service de pagamento, que acessa o repository de clientes. Para entender o impacto de uma mudança, você precisa seguir uma teia de dependências que não tem fim visível.

**Big Ball of Mud.** O sistema funciona, mas ninguém consegue explicar como. Não há fronteiras claras entre módulos, não há convenções de organização, e cada desenvolvedor que passou pelo projeto deixou seu próprio estilo. O resultado é um sistema que só funciona porque ninguém toca nas partes críticas.

**God Classes.** Uma classe `ApplicationManager` ou `SystemService` com 3000 linhas que concentra lógica de todos os domínios. Qualquer mudança nessa classe tem potencial de quebrar funcionalidades aparentemente desconectadas.

**Impossibilidade de teste.** As dependências entre classes são tão entrelaçadas que testar uma parte isoladamente exige montar o sistema inteiro. O resultado: ninguém escreve testes, e cada deploy é uma aposta.

**Paralisia de evolução.** O time sabe que a arquitetura precisa mudar, mas o custo de mudança é tão alto que ninguém se arrisca. Novas funcionalidades são empilhadas sobre fundações instáveis, e o sistema vai ficando cada vez mais frágil.

### O que uma boa arquitetura proporciona

**Manutenibilidade.** Mudanças ficam localizadas. Alterar a regra de cálculo de impostos não exige tocar no módulo de envio de emails, porque os dois vivem em contextos separados com fronteiras claras.

**Escalabilidade.** O sistema cresce de forma organizada. Novos módulos são adicionados seguindo o mesmo padrão, sem necessidade de "encaixar" código em estruturas que não foram projetadas para isso.

**Onboarding eficiente.** Um desenvolvedor novo consegue abrir a pasta `app/Domain/Licitacao/` e entender o que aquele módulo faz, quais entidades existem, quais operações são possíveis — sem precisar de um tour guiado pelo código.

**Flexibilidade tecnológica.** Trocar o gateway de pagamento, migrar de MySQL para PostgreSQL, ou substituir o serviço de email não exige reescrever lógica de negócio, porque a infraestrutura está isolada atrás de interfaces.

---

## Conceitos Fundamentais

### 1. Acoplamento (Coupling)

Acoplamento mede o grau de dependência entre módulos. Quanto mais um módulo sabe sobre os detalhes internos de outro, maior o acoplamento — e maior o risco de que mudanças em um propaguem para o outro.

```php
// ❌ Alto acoplamento — dependência de implementações concretas
class OrderController
{
    public function store()
    {
        $mailer = new SendGridMailer();
        $mailer->send(/* ... */);

        $logger = new FileLogger('/var/log/orders.log');
        $logger->log('Order created');

        // Se você quiser trocar SendGrid por SES, ou FileLogger por
        // CloudWatch, precisa alterar ESTE código.
    }
}

// ✅ Baixo acoplamento — dependência de abstrações
class CreateOrderAction
{
    public function __construct(
        private MailerInterface $mailer,
        private LoggerInterface $logger,
    ) {}

    public function execute(): void
    {
        $this->mailer->send(/* ... */);
        $this->logger->info('Order created');

        // A implementação concreta é resolvida pelo container.
        // Este código não muda quando o provedor de email muda.
    }
}
```

A diferença parece sutil em exemplos pequenos, mas em um sistema com dezenas de classes, alto acoplamento cria uma reação em cadeia: mudar A exige mudar B, que exige mudar C, que exige mudar D. Baixo acoplamento quebra essa cadeia.

### 2. Coesão (Cohesion)

Coesão mede o quanto os elementos de um módulo pertencem juntos. Uma classe com alta coesão faz uma coisa bem; uma classe com baixa coesão é uma coleção aleatória de funcionalidades que acabaram no mesmo arquivo.

```php
// ❌ Baixa coesão — coisas sem relação juntas
class Utils
{
    public function formatCurrency(float $value): string { /* ... */ }
    public function sendEmail(string $to, string $body): void { /* ... */ }
    public function generatePDF(array $data): string { /* ... */ }
    public function validateCPF(string $cpf): bool { /* ... */ }
}

// ✅ Alta coesão — cada classe agrupa funcionalidades relacionadas
class MoneyFormatter
{
    public function format(Money $money): string { /* ... */ }
    public function parse(string $formatted): Money { /* ... */ }
}

class Mailer
{
    public function send(Mailable $mail): void { /* ... */ }
    public function queue(Mailable $mail): void { /* ... */ }
}
```

A heurística é simples: se você não consegue descrever a responsabilidade da classe em uma frase sem usar "e", ela provavelmente faz coisas demais.

### 3. Separação de Responsabilidades (Separation of Concerns)

Cada camada do sistema cuida de uma preocupação específica. A camada de apresentação não calcula impostos. A camada de domínio não sabe que HTTP existe. A camada de infraestrutura não decide regras de negócio.

```php
// Presentation Layer — preocupação: HTTP
class OrderController extends Controller
{
    public function store(StoreOrderRequest $request, CreateOrderAction $action)
    {
        $order = $action->execute($request->validated());
        return new OrderResource($order);
    }
}

// Application Layer — preocupação: orquestração de use case
class CreateOrderAction
{
    public function execute(array $data): Order
    {
        // Coordena as etapas do caso de uso
    }
}

// Domain Layer — preocupação: regras de negócio
class Order extends Model
{
    public function canBeCancelled(): bool
    {
        return $this->status === OrderStatus::PENDING;
    }
}

// Infrastructure Layer — preocupação: detalhes técnicos
class EloquentOrderRepository implements OrderRepositoryInterface
{
    public function save(Order $order): void
    {
        $order->save();
    }
}
```

Cada camada pode ser modificada, testada e até substituída sem impactar as demais — desde que as interfaces entre elas sejam respeitadas.

### 4. Inversão de Dependência (Dependency Inversion)

O princípio mais contraintuitivo e mais poderoso: módulos de alto nível não devem depender de módulos de baixo nível. Ambos devem depender de abstrações.

```php
// ❌ Dependência direta — alto nível depende de baixo nível
class OrderService
{
    private MySQLOrderRepository $repository;

    public function __construct()
    {
        $this->repository = new MySQLOrderRepository();
        // OrderService está preso ao MySQL.
        // Para testar, precisa de um banco MySQL rodando.
    }
}

// ✅ Inversão de dependência — ambos dependem de abstração
interface OrderRepositoryInterface
{
    public function save(Order $order): void;
    public function findById(int $id): ?Order;
}

class MySQLOrderRepository implements OrderRepositoryInterface
{
    public function save(Order $order): void { /* ... */ }
    public function findById(int $id): ?Order { /* ... */ }
}

class OrderService
{
    public function __construct(
        private OrderRepositoryInterface $repository,
    ) {}
    // OrderService depende da INTERFACE, não do MySQL.
    // Em testes, pode receber um InMemoryOrderRepository.
}
```

```php
// O Service Provider conecta interface à implementação
$this->app->bind(
    OrderRepositoryInterface::class,
    MySQLOrderRepository::class,
);
// Trocar para MongoDB? Mude apenas esta linha.
```

A inversão acontece porque a interface (`OrderRepositoryInterface`) pertence à camada de domínio — é o domínio que define **o que** precisa. A implementação (`MySQLOrderRepository`) pertence à camada de infraestrutura — é ela que decide **como**. O fluxo de dependência se inverte: a infraestrutura depende do domínio, não o contrário.

---

## Arquitetura em Camadas

A organização mais clássica e ponto de partida para a maioria dos projetos. Cada camada tem uma responsabilidade e só pode depender das camadas abaixo dela.

### Visão geral

```
┌─────────────────────────────────────┐
│  Presentation Layer                 │  Controllers, Views, API Resources
│  (Interface com o usuário)          │
├─────────────────────────────────────┤
│  Application Layer                  │  Actions, Services, Use Cases
│  (Orquestração de casos de uso)     │
├─────────────────────────────────────┤
│  Domain Layer                       │  Models, Value Objects, Enums
│  (Regras de negócio)                │
├─────────────────────────────────────┤
│  Infrastructure Layer               │  Repositories, APIs externas, Cache
│  (Detalhes técnicos)                │
└─────────────────────────────────────┘
```

### Implementação em Laravel

```php
// Presentation Layer
// app/Http/Controllers/OrderController.php
class OrderController extends Controller
{
    public function store(StoreOrderRequest $request, CreateOrderAction $action)
    {
        $order = $action->execute($request->validated());

        return new OrderResource($order);
    }
}

// Application Layer
// app/Actions/CreateOrderAction.php
class CreateOrderAction
{
    public function __construct(
        private OrderRepositoryInterface $orders,
        private InventoryService $inventory,
        private PaymentGateway $payment,
    ) {}

    public function execute(array $data): Order
    {
        return DB::transaction(function () use ($data) {
            $order = $this->orders->create($data);
            $this->inventory->reserve($order);
            $this->payment->charge($order);

            return $order;
        });
    }
}

// Domain Layer
// app/Models/Order.php
class Order extends Model
{
    public function canBeCancelled(): bool
    {
        return $this->status === OrderStatus::PENDING;
    }

    public function total(): float
    {
        return $this->items->sum('subtotal');
    }
}

// Infrastructure Layer
// app/Repositories/EloquentOrderRepository.php
class EloquentOrderRepository implements OrderRepositoryInterface
{
    public function create(array $data): Order
    {
        return Order::create($data);
    }
}
```

### Regra de dependência

O fluxo de dependência segue uma direção:

```
Presentation → Application → Domain ← Infrastructure
```

A Presentation pode chamar a Application. A Application pode chamar o Domain. A Infrastructure **implementa** interfaces definidas pelo Domain, mas o Domain não sabe que a Infrastructure existe. Isso mantém o núcleo do sistema (as regras de negócio) isolado dos detalhes técnicos.

---

## Modular Monolith

O Modular Monolith é um monolito organizado em módulos independentes — cada módulo é autocontido com seus próprios controllers, models, services, rotas e migrations. É a arquitetura mais recomendada para a maioria dos projetos Laravel porque combina a simplicidade operacional do monolito com a organização e isolamento dos microserviços.

### Estrutura

```
app/
├── Modules/
│   ├── Sales/
│   │   ├── Controllers/
│   │   ├── Models/
│   │   ├── Actions/
│   │   ├── Repositories/
│   │   ├── Events/
│   │   ├── Listeners/
│   │   ├── Database/
│   │   │   └── Migrations/
│   │   ├── routes.php
│   │   └── SalesServiceProvider.php
│   │
│   ├── Inventory/
│   │   ├── Controllers/
│   │   ├── Models/
│   │   ├── ...
│   │   └── InventoryServiceProvider.php
│   │
│   └── Billing/
│       └── ...
│
└── Shared/
    ├── ValueObjects/
    ├── Contracts/
    └── Traits/
```

### Anatomia de um módulo

Cada módulo registra seus próprios recursos através de um ServiceProvider dedicado:

```php
// app/Modules/Sales/SalesServiceProvider.php
class SalesServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(
            OrderRepositoryInterface::class,
            EloquentOrderRepository::class
        );
    }

    public function boot(): void
    {
        $this->loadRoutesFrom(__DIR__ . '/routes.php');
        $this->loadMigrationsFrom(__DIR__ . '/Database/Migrations');
        $this->loadViewsFrom(__DIR__ . '/Resources/views', 'sales');
    }
}
```

```php
// app/Modules/Sales/routes.php
Route::prefix('sales')->name('sales.')->middleware('auth')->group(function () {
    Route::apiResource('orders', OrderController::class);
});
```

```php
// app/Modules/Sales/Controllers/OrderController.php
namespace App\Modules\Sales\Controllers;

class OrderController extends Controller
{
    public function store(StoreOrderRequest $request, CreateOrderAction $action)
    {
        $order = $action->execute($request->validated());

        return new OrderResource($order);
    }
}
```

O módulo é autocontido: tem suas rotas, controllers, models, migrations e bindings. Se amanhã o módulo Sales precisar ser extraído para um serviço separado, o trabalho é significativamente menor do que se o código estivesse espalhado por toda a aplicação.

### Comunicação entre módulos

Esta é a decisão mais importante em um Modular Monolith. Módulos não devem acessar diretamente os internos uns dos outros.

```php
// ❌ Acoplamento direto — Sales importa o Model de Inventory
namespace App\Modules\Sales\Actions;

use App\Modules\Inventory\Models\Product; // Dependência direta!

class CreateOrderAction
{
    public function execute(array $data): Order
    {
        $product = Product::find($data['product_id']); // Acessa outro módulo diretamente
        // Se Inventory mudar a estrutura de Product, Sales quebra.
    }
}
```

```php
// ✅ Via interface — Sales depende de um contrato, não de uma implementação
namespace App\Modules\Sales\Actions;

class CreateOrderAction
{
    public function __construct(
        private ProductServiceInterface $products, // Contrato definido em Shared
    ) {}

    public function execute(array $data): Order
    {
        $product = $this->products->findOrFail($data['product_id']);
        // Sales não sabe se Product vem do Eloquent, de uma API ou de um cache.
    }
}
```

```php
// ✅ Via eventos — desacoplamento máximo
// Sales dispara o evento
event(new OrderCreated($order));

// Inventory reage ao evento
class ReserveStockWhenOrderCreated
{
    public function handle(OrderCreated $event): void
    {
        foreach ($event->order->items as $item) {
            $this->inventory->reserve($item->product_id, $item->quantity);
        }
    }
}
```

Eventos são a forma preferida de comunicação entre módulos quando o módulo emissor não precisa da resposta imediata do receptor. Interfaces são preferíveis quando a comunicação é síncrona e o emissor precisa dos dados de volta.

---

## Domain-Driven Design

DDD organiza o código a partir do domínio de negócio, não da tecnologia. O conceito central é o **Bounded Context** — um limite dentro do qual um modelo de domínio é consistente e completo.

### Bounded Contexts

Cada contexto delimita um vocabulário e um conjunto de regras coerentes. A mesma palavra pode significar coisas diferentes em contextos diferentes: "Processo" no contexto de Licitações é um processo licitatório; no contexto de RH, pode ser um processo seletivo.

```
Sistema de Prefeitura
│
├── Atendimento ao Cidadão
│   ├── Requerimentos
│   ├── Protocolos
│   └── Ouvidoria
│
├── Recursos Humanos
│   ├── Funcionários
│   ├── Folha de Pagamento
│   └── Ponto Eletrônico
│
├── Licitações
│   ├── Processos
│   ├── Fornecedores
│   └── Contratos
│
└── Financeiro
    ├── Receitas
    ├── Despesas
    └── Orçamento
```

### Estrutura DDD em Laravel

```
app/
├── Domain/
│   ├── Licitacao/
│   │   ├── Models/
│   │   │   ├── Processo.php
│   │   │   ├── Lance.php
│   │   │   └── Documento.php
│   │   ├── ValueObjects/
│   │   │   ├── NumeroProcesso.php
│   │   │   └── CNPJ.php
│   │   ├── Repositories/              (interfaces)
│   │   │   └── ProcessoRepositoryInterface.php
│   │   ├── Events/
│   │   │   ├── ProcessoPublicado.php
│   │   │   └── LanceRecebido.php
│   │   ├── Enums/
│   │   │   └── ProcessoStatus.php
│   │   └── Exceptions/
│   │       └── ProcessoInvalidoException.php
│   │
│   └── RH/
│       └── ...
│
├── Application/
│   ├── Licitacao/
│   │   └── PublicarProcessoAction.php
│   └── RH/
│       └── RegistrarPontoAction.php
│
└── Infrastructure/
    ├── Repositories/
    │   └── EloquentProcessoRepository.php
    └── ExternalServices/
        └── DiarioOficialAPI.php
```

### Agregados (Aggregates)

Um agregado é um cluster de objetos tratados como uma unidade para efeitos de consistência. O Aggregate Root é o ponto de entrada — toda manipulação dos objetos internos passa por ele.

```php
// Processo é o Aggregate Root.
// Lances só podem ser adicionados ATRAVÉS do Processo.
class Processo extends Model
{
    protected $casts = [
        'status' => ProcessoStatus::class,
    ];

    public function addLance(Fornecedor $fornecedor, Money $valor): Lance
    {
        if ($this->status !== ProcessoStatus::ABERTO) {
            throw new ProcessoNaoAbertoException($this);
        }

        if ($valor->amount < $this->valorMinimo()->amount) {
            throw new LanceAbaixoMinimoException($valor, $this->valorMinimo());
        }

        if ($this->fornecedorJaLancou($fornecedor)) {
            throw new FornecedorJaPossuiLanceException($fornecedor, $this);
        }

        $lance = $this->lances()->create([
            'fornecedor_id' => $fornecedor->id,
            'valor' => $valor->amount,
        ]);

        event(new LanceRecebido($this, $lance));

        return $lance;
    }

    private function fornecedorJaLancou(Fornecedor $fornecedor): bool
    {
        return $this->lances()->where('fornecedor_id', $fornecedor->id)->exists();
    }

    public function lances(): HasMany
    {
        return $this->hasMany(Lance::class);
    }
}
```

O ponto fundamental: `Lance::create()` nunca deve ser chamado diretamente. Toda criação de lance passa por `$processo->addLance()`, que aplica as regras de negócio antes de permitir a operação. Isso garante que o agregado está sempre em um estado consistente.

### Linguagem Ubíqua (Ubiquitous Language)

O código deve usar os mesmos termos que os especialistas do domínio usam. Se a equipe da prefeitura fala em "publicar", "homologar" e "fracassar", o código deve usar esses termos — não abstrações genéricas como `setStatus` ou `updateData`.

```php
// ✅ Linguagem do domínio — um servidor público entende
class Processo extends Model
{
    public function publicar(): void { /* ... */ }
    public function homologar(): void { /* ... */ }
    public function fracassar(): void { /* ... */ }
    public function revogar(string $justificativa): void { /* ... */ }
}

// ❌ Linguagem técnica genérica — só o programador entende
class Processo extends Model
{
    public function setStatus(string $status): void { /* ... */ }
    public function update(array $data): void { /* ... */ }
    public function changeState(int $stateCode): void { /* ... */ }
}
```

Quando o código fala a linguagem do domínio, as conversas entre desenvolvedores e especialistas ficam mais produtivas: ambos usam os mesmos termos, e mal-entendidos diminuem.

---

## Hexagonal Architecture

Também conhecida como "Ports and Adapters". A ideia central é que o núcleo da aplicação (domínio + casos de uso) não conhece o mundo externo. Toda comunicação acontece através de **Ports** (interfaces) e **Adapters** (implementações concretas).

### Conceito visual

```
                    ┌─────────────────┐
         ┌─────────┤  Web Adapter    ├─────────┐
         │         │  (Controller)    │         │
         │         └─────────────────┘         │
         │                                      │
    ┌────▼─────┐                          ┌────▼─────┐
    │   Port   │                          │   Port   │
    │  (Input) │                          │  (Input) │
    └────┬─────┘                          └────┬─────┘
         │      ┌──────────────────┐          │
         └──────►  Application     ◄──────────┘
                │     Core         │
                │                  │
                │  Domain +        │
                │  Use Cases       │
                │                  │
         ┌──────┤                  ├──────────┐
         │      └──────────────────┘          │
    ┌────▼─────┐                         ┌────▼─────┐
    │   Port   │                         │   Port   │
    │ (Output) │                         │ (Output) │
    └────┬─────┘                         └────┬─────┘
         │                                     │
    ┌────▼─────────┐                 ┌────────▼──────┐
    │  MySQL        │                 │  Stripe       │
    │  Adapter      │                 │  Adapter      │
    └──────────────┘                 └───────────────┘
```

Os **Input Ports** definem o que o mundo externo pode pedir ao sistema (casos de uso). Os **Output Ports** definem o que o sistema precisa do mundo externo (persistência, pagamento, email). Os **Adapters** conectam os ports ao mundo real.

### Implementação em Laravel

```php
// Output Ports — interfaces definidas pelo domínio
interface OrderRepositoryPort
{
    public function save(Order $order): void;
    public function findById(int $id): ?Order;
}

interface PaymentGatewayPort
{
    public function charge(Order $order): PaymentResult;
    public function refund(Order $order): RefundResult;
}
```

```php
// Application Core — não conhece HTTP, não conhece MySQL, não conhece Stripe
class CreateOrderUseCase
{
    public function __construct(
        private OrderRepositoryPort $orders,
        private PaymentGatewayPort $payment,
    ) {}

    public function execute(CreateOrderCommand $command): Order
    {
        $order = new Order($command->customerId, $command->items);
        $this->payment->charge($order);
        $this->orders->save($order);

        return $order;
    }
}
```

```php
// Output Adapters — implementações concretas dos ports
class EloquentOrderAdapter implements OrderRepositoryPort
{
    public function save(Order $order): void
    {
        OrderModel::updateOrCreate(
            ['id' => $order->id],
            $order->toArray()
        );
    }

    public function findById(int $id): ?Order
    {
        $model = OrderModel::find($id);
        return $model ? Order::fromModel($model) : null;
    }
}

class StripePaymentAdapter implements PaymentGatewayPort
{
    public function charge(Order $order): PaymentResult
    {
        $charge = Stripe::charges()->create([
            'amount' => $order->total()->amount,
            'currency' => 'brl',
        ]);

        return new PaymentResult($charge->id, $charge->status);
    }

    public function refund(Order $order): RefundResult { /* ... */ }
}
```

```php
// Service Provider conecta ports aos adapters
class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(OrderRepositoryPort::class, EloquentOrderAdapter::class);
        $this->app->bind(PaymentGatewayPort::class, StripePaymentAdapter::class);
    }
}
```

A beleza desse padrão é que o `CreateOrderUseCase` é puro — não tem nenhuma dependência de framework, banco de dados ou serviço externo. Pode ser testado com mocks em milissegundos. E trocar Stripe por PagSeguro exige apenas criar um novo adapter e mudar uma linha no ServiceProvider.

Na prática, a arquitetura hexagonal completa adiciona overhead considerável. Para a maioria dos projetos Laravel, aplicar o princípio parcialmente — usando interfaces para dependências de infraestrutura críticas (pagamento, armazenamento, integrações externas) sem necessariamente criar a estrutura completa de ports/adapters — já traz a maior parte dos benefícios.

---

## CQRS

CQRS (Command Query Responsibility Segregation) é o princípio de separar operações que **mudam** estado (Commands) de operações que **lêem** estado (Queries). A ideia é que leitura e escrita têm requisitos fundamentalmente diferentes e se beneficiam de otimizações distintas.

### Por que separar?

Em um sistema típico, a leitura é muito mais frequente que a escrita. Um dashboard pode consultar dados 100x por segundo, enquanto novos pedidos são criados 2x por segundo. Se leitura e escrita usam o mesmo modelo, você não pode otimizar um sem afetar o outro.

Com CQRS, o lado de escrita pode priorizar consistência e validação. O lado de leitura pode priorizar performance, usando views materializadas, caches ou até read replicas do banco.

### Commands (Escrita)

```php
// O Command carrega a intenção e os dados necessários
final readonly class CreateOrderCommand
{
    public function __construct(
        public int $customerId,
        public array $items,
    ) {}
}

// O Handler executa a operação
class CreateOrderHandler
{
    public function __construct(
        private OrderRepositoryInterface $orders,
        private InventoryService $inventory,
    ) {}

    public function handle(CreateOrderCommand $command): void
    {
        $order = Order::create([
            'customer_id' => $command->customerId,
            'status' => OrderStatus::PENDING,
        ]);

        foreach ($command->items as $item) {
            $order->items()->create($item);
            $this->inventory->reserve($item['product_id'], $item['quantity']);
        }

        event(new OrderCreated($order));
    }
}
```

### Queries (Leitura)

```php
// A Query define o que se quer ler
final readonly class GetCustomerOrdersQuery
{
    public function __construct(
        public int $customerId,
        public int $page = 1,
        public int $perPage = 15,
    ) {}
}

// O Handler busca os dados de forma otimizada
class GetCustomerOrdersHandler
{
    public function handle(GetCustomerOrdersQuery $query): LengthAwarePaginator
    {
        // Pode usar query otimizada, view materializada, cache, read replica.
        // Não precisa passar pelo mesmo modelo que a escrita usa.
        return DB::table('orders')
            ->join('order_items', 'orders.id', '=', 'order_items.order_id')
            ->where('orders.customer_id', $query->customerId)
            ->select([
                'orders.id',
                'orders.status',
                'orders.created_at',
                DB::raw('SUM(order_items.price * order_items.quantity) as total'),
                DB::raw('COUNT(order_items.id) as items_count'),
            ])
            ->groupBy('orders.id', 'orders.status', 'orders.created_at')
            ->paginate(perPage: $query->perPage, page: $query->page);
    }
}
```

### Command Bus simples

```php
interface CommandBusInterface
{
    public function dispatch(object $command): mixed;
}

class SimpleCommandBus implements CommandBusInterface
{
    public function __construct(
        private Container $container,
    ) {}

    public function dispatch(object $command): mixed
    {
        $handlerClass = $this->resolveHandler($command);
        $handler = $this->container->make($handlerClass);

        return $handler->handle($command);
    }

    private function resolveHandler(object $command): string
    {
        // Convenção: CreateOrderCommand → CreateOrderHandler
        return str_replace('Command', 'Handler', get_class($command));
    }
}
```

CQRS completo, com modelos separados para leitura e escrita, é necessário apenas em sistemas com alta carga ou requisitos de performance muito distintos entre leitura e escrita. Para a maioria dos projetos Laravel, aplicar o princípio de forma leve — separando Actions (escrita) de queries dedicadas (leitura) — já é suficiente.

---

## Event Sourcing

Event Sourcing é uma mudança fundamental na forma de persistir dados: em vez de armazenar o **estado atual** de uma entidade, você armazena a **sequência de eventos** que levaram a esse estado. O estado atual é reconstruído "reproduzindo" os eventos.

### A diferença

```
Abordagem tradicional — armazena o estado final:
┌──────────────────────────────────────────┐
│  orders                                  │
│  id: 1, status: "delivered", total: 299  │
│  updated_at: 2024-02-10                  │
└──────────────────────────────────────────┘
Pergunta: quando o pedido foi pago? Não sabemos.

Event Sourcing — armazena o que aconteceu:
┌──────────────────────────────────────────┐
│  events                                  │
│  1. OrderCreated   (total: 299)  Feb 01  │
│  2. OrderPaid      (method: pix) Feb 02  │
│  3. OrderShipped   (tracking: X) Feb 05  │
│  4. OrderDelivered (signed: Y)   Feb 10  │
└──────────────────────────────────────────┘
O estado atual pode ser reconstruído reproduzindo os eventos.
Além disso, temos histórico completo e auditável.
```

### Implementação conceitual

```php
// Event Store — persiste eventos no banco
class EventStore
{
    public function append(string $aggregateId, DomainEvent $event): void
    {
        DB::table('event_store')->insert([
            'aggregate_id' => $aggregateId,
            'aggregate_type' => $event->aggregateType(),
            'event_type' => get_class($event),
            'payload' => json_encode($event->toArray()),
            'version' => $this->nextVersion($aggregateId),
            'created_at' => now(),
        ]);
    }

    public function getEvents(string $aggregateId): Collection
    {
        return DB::table('event_store')
            ->where('aggregate_id', $aggregateId)
            ->orderBy('version')
            ->get()
            ->map(fn ($row) => $this->deserialize($row));
    }

    private function nextVersion(string $aggregateId): int
    {
        $current = DB::table('event_store')
            ->where('aggregate_id', $aggregateId)
            ->max('version');

        return ($current ?? 0) + 1;
    }

    private function deserialize(object $row): DomainEvent
    {
        $class = $row->event_type;
        return $class::fromArray(json_decode($row->payload, true));
    }
}
```

```php
// O agregado reconstrói seu estado a partir dos eventos
class Order
{
    public OrderStatus $status;
    public int $total = 0;
    public ?Carbon $paidAt = null;

    public static function reconstituteFrom(Collection $events): self
    {
        $order = new self();

        foreach ($events as $event) {
            $order->apply($event);
        }

        return $order;
    }

    private function apply(DomainEvent $event): void
    {
        match (true) {
            $event instanceof OrderCreated => $this->applyCreated($event),
            $event instanceof OrderPaid => $this->applyPaid($event),
            $event instanceof OrderShipped => $this->applyShipped($event),
            $event instanceof OrderDelivered => $this->applyDelivered($event),
            default => null,
        };
    }

    private function applyCreated(OrderCreated $event): void
    {
        $this->total = $event->total;
        $this->status = OrderStatus::PENDING;
    }

    private function applyPaid(OrderPaid $event): void
    {
        $this->status = OrderStatus::PAID;
        $this->paidAt = $event->paidAt;
    }

    private function applyShipped(OrderShipped $event): void
    {
        $this->status = OrderStatus::SHIPPED;
    }

    private function applyDelivered(OrderDelivered $event): void
    {
        $this->status = OrderStatus::DELIVERED;
    }
}
```

Event Sourcing traz benefícios poderosos: auditoria completa, capacidade de reconstruir o estado em qualquer ponto do tempo, e a possibilidade de projetar os mesmos eventos em modelos de leitura diferentes (combinando com CQRS). Mas o custo é significativo: complexidade adicional na persistência, necessidade de snapshots para performance, e uma curva de aprendizado íngreme.

Para a maioria dos projetos Laravel, Event Sourcing completo é overengineering. A abordagem pragmática é usar o conceito seletivamente: implementar uma tabela de audit trail (como mostrado no documento de State & Data Flow) para entidades críticas, enquanto mantém a persistência tradicional para o resto.

---

## Microservices vs Monolith

Esta é provavelmente a decisão arquitetural mais discutida — e mais frequentemente tomada pelas razões erradas.

### Monolith

Todo o sistema roda em um único processo, compartilhando banco de dados, memória e deploy.

```
┌────────────────────────────────┐
│         Monolith               │
│                                │
│  ┌──────────┐ ┌──────────┐    │
│  │  Sales   │ │ Inventory│    │
│  └──────────┘ └──────────┘    │
│  ┌──────────┐ ┌──────────┐    │
│  │ Billing  │ │    RH    │    │
│  └──────────┘ └──────────┘    │
│                                │
│  ┌──────────────────────────┐  │
│  │       PostgreSQL         │  │
│  └──────────────────────────┘  │
└────────────────────────────────┘
```

Vantagens: simplicidade de desenvolvimento, deploy e operação. Transações ACID são triviais. Debugging é local. Um único deploy atualiza tudo. Ideal para times pequenos e médios.

Desvantagens: escala como um todo (se um módulo precisa de mais recursos, todos ganham). Um deploy afeta todo o sistema. Se o acoplamento entre módulos crescer descontrolado, mudanças ficam arriscadas.

### Microservices

Cada módulo é um serviço independente com seu próprio banco, deploy e ciclo de vida.

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Sales      │  │  Inventory   │  │   Billing    │
│   Service    │  │   Service    │  │   Service    │
│              │  │              │  │              │
│  PostgreSQL  │  │    Redis     │  │   MongoDB    │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┴─────────────────┘
                    API Gateway
```

Vantagens: escala independente, deploy independente, times autônomos, flexibilidade de tecnologia por serviço, falhas isoladas.

Desvantagens: complexidade operacional drasticamente maior. Transações distribuídas são difíceis (saga pattern, compensações). Debugging requer distributed tracing. Comunicação via rede adiciona latência e pontos de falha. Monitoramento requer ferramentas especializadas. O overhead inicial é alto — não compensa para times pequenos.

### A recomendação pragmática: Modular Monolith

Comece com um Modular Monolith. Se e quando a necessidade surgir, extraia módulos específicos para microserviços. A chave é que as fronteiras entre módulos estejam bem definidas desde o início:

```php
// Módulos se comunicam via interfaces e eventos — não importa se estão
// no mesmo processo ou em servidores diferentes.

// Interface que hoje resolve localmente...
class LocalInventoryService implements InventoryServiceInterface
{
    public function reserveStock(array $items): void
    {
        foreach ($items as $item) {
            Product::findOrFail($item['product_id'])
                ->decreaseStock($item['quantity']);
        }
    }
}

// ...e amanhã pode resolver via HTTP, se Inventory virar microserviço.
class HttpInventoryService implements InventoryServiceInterface
{
    public function reserveStock(array $items): void
    {
        Http::post('http://inventory-service/api/v1/reserve', [
            'items' => $items,
        ])->throw();
    }
}

// A troca é transparente para quem consome a interface.
```

A regra é: **não construa microserviços porque você acha que vai precisar**. Construa um monolito modular com fronteiras claras. Se a escala ou a organização do time exigir separação, a migração será viável porque os módulos já são independentes.

---

## Organização de Pastas

A estrutura de pastas é o reflexo físico da arquitetura. Três abordagens existem em um espectro de simplicidade para sofisticação:

### Laravel Padrão (projetos pequenos)

```
app/
├── Console/
├── Exceptions/
├── Http/
│   ├── Controllers/
│   ├── Middleware/
│   └── Requests/
├── Models/
├── Providers/
└── Services/
```

Funciona bem até ~20 models e ~30 controllers. Depois disso, a pasta `Models/` tem arquivos de domínios completamente diferentes (Order, Product, Processo, Funcionario) e navegar fica difícil.

### Modular (recomendado para projetos médios)

```
app/
├── Modules/
│   ├── Sales/
│   │   ├── Controllers/
│   │   ├── Models/
│   │   ├── Actions/
│   │   ├── Repositories/
│   │   ├── Events/
│   │   ├── Listeners/
│   │   ├── Database/Migrations/
│   │   ├── routes.php
│   │   └── SalesServiceProvider.php
│   │
│   ├── Inventory/
│   ├── Billing/
│   └── Licitacao/
│
├── Shared/
│   ├── ValueObjects/
│   │   ├── Money.php
│   │   ├── CPF.php
│   │   └── Email.php
│   ├── Contracts/
│   └── Traits/
│
└── Infrastructure/
    ├── ExternalServices/
    └── Cache/
```

Cada módulo é autocontido. A pasta `Shared/` contém código usado por múltiplos módulos (Value Objects, interfaces compartilhadas, traits utilitários). A pasta `Infrastructure/` contém implementações de serviços técnicos transversais.

### Domain-Driven (projetos grandes)

```
app/
├── Domain/                        # Núcleo do negócio (puro)
│   ├── Licitacao/
│   │   ├── Models/
│   │   ├── ValueObjects/
│   │   ├── Repositories/         # Apenas interfaces
│   │   ├── Events/
│   │   ├── Enums/
│   │   └── Exceptions/
│   │
│   └── RH/
│       └── ...
│
├── Application/                   # Casos de uso
│   ├── Licitacao/
│   │   ├── PublicarProcessoAction.php
│   │   └── ReceberLanceAction.php
│   └── RH/
│       └── RegistrarPontoAction.php
│
├── Infrastructure/                # Implementações concretas
│   ├── Persistence/
│   │   └── Eloquent/
│   │       └── EloquentProcessoRepository.php
│   ├── ExternalServices/
│   │   └── DiarioOficialAPI.php
│   └── Messaging/
│
└── Presentation/                  # Interface com o usuário
    ├── Http/
    │   ├── Controllers/
    │   ├── Requests/
    │   └── Resources/
    └── Console/
        └── Commands/
```

A separação é rígida: o Domain não importa nada de Infrastructure ou Presentation. Isso garante que as regras de negócio são puras e testáveis sem dependências de framework.

A escolha entre essas estruturas depende do tamanho do projeto e do time. Comece com a mais simples que funciona e evolua quando a dor de navegação e manutenção justificar.

---

## Anti-patterns Comuns

### 1. God Class

Uma classe que concentra responsabilidades demais. Geralmente tem nomes vagos como `Manager`, `Handler`, `Processor` ou `Utils`.

```php
// ❌ 3000 linhas, 50 métodos, importa 40 classes
class ApplicationManager
{
    public function handleOrder() { /* ... */ }
    public function processPayment() { /* ... */ }
    public function generateReport() { /* ... */ }
    public function sendNotification() { /* ... */ }
    // Uma mudança em qualquer funcionalidade toca este arquivo.
}

// ✅ Cada contexto tem sua própria classe
class CreateOrderAction { /* ... */ }
class ProcessPaymentAction { /* ... */ }
class ReportGenerator { /* ... */ }
class NotificationService { /* ... */ }
```

### 2. Big Ball of Mud

O sistema funciona mas não tem estrutura reconhecível. Arquivos estão em qualquer lugar, nomes são inconsistentes, e ninguém consegue explicar a organização.

```
❌                              ✅
app/                            app/
├── stuff.php                   ├── Domain/
├── helpers_v2.php              ├── Application/
├── OrderThing.php              ├── Infrastructure/
├── utils_final_FINAL.php       └── Presentation/
└── backup_old/
```

A correção não é reorganizar tudo de uma vez (isso é arriscado). É definir a estrutura alvo, mover arquivos gradualmente, e garantir que código novo sempre segue a nova convenção.

### 3. Dependências Circulares

Módulo A depende de B, e B depende de A. Isso cria um ciclo impossível de resolver e indica que os limites entre módulos estão errados.

```php
// ❌ Circular: OrderService ↔ PaymentService
class OrderService
{
    public function __construct(private PaymentService $payment) {}
}

class PaymentService
{
    public function __construct(private OrderService $orders) {}
    // O container não consegue resolver: quem instanciar primeiro?
}

// ✅ Quebrar o ciclo com interface ou evento
class OrderService
{
    public function __construct(
        private PaymentServiceInterface $payment,
    ) {}
}

class PaymentService implements PaymentServiceInterface
{
    // Não depende de OrderService.
    // Se precisar reagir a eventos de Order, usa um Listener.
}
```

### 4. Arquitetura Astronauta

O oposto do Big Ball of Mud: abstrações demais, camadas demais, interfaces para tudo — mesmo quando não há justificativa. Um CRUD simples passa por Command → CommandBus → Handler → Service → Repository → Model → Event → Listener.

A cura é o princípio YAGNI (You Aren't Gonna Need It): adicione complexidade apenas quando a necessidade for real, não especulativa.

---

## Boas Práticas

### 1. Start Simple, Evolve as Needed

Comece com a arquitetura mais simples que resolve o problema atual. Adicione camadas e abstrações à medida que a complexidade justifica.

```php
// Fase 1: MVP — Controller + Model (suficiente para protótipos)
class OrderController extends Controller
{
    public function store(Request $request)
    {
        $order = Order::create($request->validated());
        return redirect()->route('orders.show', $order);
    }
}

// Fase 2: Lógica cresce — extrair para Action
class OrderController extends Controller
{
    public function store(StoreOrderRequest $request, CreateOrderAction $action)
    {
        $order = $action->execute($request->validated());
        return new OrderResource($order);
    }
}

// Fase 3: Múltiplos consumidores — adicionar interface de repositório
// (quando a mesma Action é usada por web, API e CLI)

// Fase 4: Alta carga — considerar CQRS
// (quando leitura e escrita têm requisitos de performance muito diferentes)
```

Cada fase é uma evolução natural da anterior. Nenhuma exige reescrever o que já existe — apenas adicionar uma camada de abstração onde a complexidade demanda.

### 2. Screaming Architecture

Olhe para a estrutura de pastas do seu projeto. Um estranho consegue dizer o que o sistema faz apenas pela lista de diretórios?

```
// ❌ Isso não grita nada — poderia ser qualquer sistema
app/
├── Controllers/
├── Models/
├── Services/
└── Repositories/

// ✅ Isso grita "Sistema de Gestão Municipal!"
app/
├── Licitacoes/
│   ├── Processos/
│   ├── Fornecedores/
│   └── Contratos/
├── AtendimentoCidadao/
│   ├── Requerimentos/
│   └── Protocolos/
└── RecursosHumanos/
    ├── Funcionarios/
    └── FolhaPagamento/
```

O conceito vem de Robert C. Martin: a arquitetura do sistema deve comunicar sua intenção, não suas ferramentas. Um prédio de hospital tem a planta de um hospital, não a planta de "um lugar que usa tijolos e cimento".

### 3. Package by Feature, Not by Layer

Agrupe código por funcionalidade de negócio, não por tipo técnico. Quando você precisa trabalhar em "Pedidos", todos os arquivos relevantes estão juntos — não espalhados em 6 pastas diferentes.

```
// ❌ Organizado por camada técnica
// Para trabalhar em Orders, precisa navegar 4 pastas
app/
├── Controllers/
│   ├── OrderController.php         ← aqui
│   └── ProductController.php
├── Models/
│   ├── Order.php                   ← aqui
│   └── Product.php
├── Services/
│   ├── OrderService.php            ← aqui
│   └── ProductService.php
└── Repositories/
    ├── OrderRepository.php         ← aqui
    └── ProductRepository.php

// ✅ Organizado por feature
// Tudo sobre Orders está em um lugar
app/
├── Sales/
│   ├── OrderController.php
│   ├── Order.php
│   ├── CreateOrderAction.php
│   └── OrderRepository.php
└── Catalog/
    ├── ProductController.php
    ├── Product.php
    └── ProductRepository.php
```

### 4. Documente decisões arquiteturais (ADRs)

Decisões de arquitetura têm contexto que se perde com o tempo. Daqui a um ano, ninguém vai lembrar *por que* escolheu Sanctum ao invés de Passport, ou *por que* a comunicação entre módulos usa eventos e não chamadas diretas. Architecture Decision Records (ADRs) preservam esse contexto:

```markdown
# ADR-001: Usar Modular Monolith ao invés de Microservices

## Status: Aceito

## Contexto
O sistema atende ~5000 usuários e é mantido por um time de 3 devs.
A complexidade operacional de microservices não se justifica.

## Decisão
Organizar o sistema como Modular Monolith com módulos independentes
comunicando-se via interfaces e eventos.

## Consequências
- Simplicidade operacional (1 deploy, 1 banco)
- Módulos podem ser extraídos para microservices no futuro se necessário
- Risco: acoplamento entre módulos pode crescer se fronteiras não forem respeitadas
```

ADRs não precisam ser formais. Um arquivo markdown por decisão, com contexto, decisão e consequências, já é suficiente.

---

## Exercícios Práticos

### Exercício 1: Modularizar uma aplicação existente

Pegue seu projeto atual e reorganize-o em módulos:

1. Identifique os domínios principais (Licitações, Atendimento, RH, Financeiro).
2. Crie a estrutura de pastas para cada módulo com ServiceProvider dedicado.
3. Mova gradualmente os arquivos existentes para os módulos correspondentes — um módulo por vez, testando a cada migração.
4. Defina as interfaces entre módulos: o que Sales pode pedir de Inventory? Qual o contrato?
5. Substitua dependências diretas entre módulos por interfaces ou eventos.

Dica: comece pelo módulo mais isolado (menos dependências de outros módulos). Deixe o módulo mais acoplado para o final.

### Exercício 2: Extrair Bounded Contexts

Se você tem um módulo "Sistema" que concentra tudo, divida-o em contextos coesos:

```
// Antes
app/Modules/Sistema/
├── Controllers/      (40 controllers)
├── Models/           (30 models)
└── Services/         (20 services)

// Depois
app/Modules/
├── AtendimentoCidadao/
├── Licitacoes/
├── RecursosHumanos/
└── Financeiro/
```

Para cada contexto extraído, defina: quais entidades pertencem a ele, quais operações são possíveis, e como ele se comunica com os demais contextos.

### Exercício 3: Implementar fronteiras hexagonais em um módulo

Escolha um módulo existente e aplique os princípios da Hexagonal Architecture:

1. Identifique as dependências externas do módulo (banco de dados, APIs, email, storage).
2. Crie interfaces (Ports) para cada dependência.
3. Implemente os Adapters concretos.
4. Registre os bindings no ServiceProvider.
5. Escreva testes para o core usando fakes/mocks dos Adapters.

O objetivo não é implementar hexagonal "puro", mas isolar o domínio dos detalhes de infraestrutura nos pontos onde isso traz mais valor.

---

## Recursos Adicionais

### Livros

- **Clean Architecture** — Robert C. Martin. Os princípios de organização de software que transcendem frameworks e linguagens. O conceito de "Screaming Architecture" vem deste livro.
- **Building Microservices** — Sam Newman. Leitura obrigatória antes de decidir por microserviços — ironicamente, o livro é excelente em explicar quando *não* usar microserviços.
- **Domain-Driven Design** — Eric Evans. A referência para Bounded Contexts, Aggregates e Ubiquitous Language.
- **Patterns of Enterprise Application Architecture** — Martin Fowler. Catálogo de padrões arquiteturais (Repository, Unit of Work, Service Layer) com trade-offs detalhados.

### Ferramentas

- **Deptrac** — analisa dependências entre camadas e módulos, prevenindo violações arquiteturais automaticamente no CI.
- **PHPStan / Larastan** — análise estática que detecta problemas de tipo e acesso a propriedades inexistentes.
- **Laravel Modules (nwidart/laravel-modules)** — pacote que facilita a criação e gerenciamento de módulos no Laravel.

### Artigos

- [Modular Laravel](https://laravelpackage.com/) — guia prático sobre desenvolvimento modular no Laravel.
- [Architecture Decision Records](https://adr.github.io/) — padrão para documentar decisões arquiteturais.

---

## Conclusão

Arquitetura não é um diagrama bonito que você desenha no início do projeto e pendura na parede. É o conjunto de decisões estruturais que evolui com o sistema — e que determina se essa evolução é possível ou se cada mudança exige uma batalha.

Os princípios que guiam boas decisões arquiteturais são:

**Baixo acoplamento, alta coesão.** Módulos independentes com responsabilidades bem definidas. Mudar um módulo não deve exigir mudar outros.

**Separação de responsabilidades.** Cada camada cuida de uma preocupação. Lógica de negócio não sabe que HTTP existe. Persistência não decide regras de domínio.

**Inversão de dependência.** Dependa de abstrações. O domínio define o que precisa (interfaces); a infraestrutura implementa.

**Screaming Architecture.** A estrutura do projeto deve comunicar o que o sistema faz, não quais frameworks usa.

**Comece simples, evolua quando necessário.** Um monolito bem organizado é melhor que microserviços mal implementados. Adicione complexidade apenas quando a dor real justificar.

**Próximo passo:** combine esses princípios com **Preventing Messes** (testes automatizados, análise estática e CI/CD) para garantir que a arquitetura definida seja respeitada ao longo do tempo — porque arquitetura que existe apenas na documentação e não é verificada pelo tooling é arquitetura que será violada.
