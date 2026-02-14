# Automated Testing Strategy (Estratégia de Testes Automatizados)

## Índice

1. [Introdução](#introdução)
2. [Por que isso importa?](#por-que-isso-importa)
3. [Anatomia de um Teste](#anatomia-de-um-teste)
4. [Tipos de Teste no Laravel](#tipos-de-teste-no-laravel)
5. [Testes Unitários](#testes-unitários)
6. [Testes de Feature (Integração)](#testes-de-feature-integração)
7. [Testando Regras de Negócio sem Banco](#testando-regras-de-negócio-sem-banco)
8. [Testando APIs](#testando-apis)
9. [Mocks, Fakes e Stubs](#mocks-fakes-e-stubs)
10. [TDD — Test Driven Development](#tdd--test-driven-development)
11. [Pest vs PHPUnit](#pest-vs-phpunit)
12. [O que Testar (e o que não testar)](#o-que-testar-e-o-que-não-testar)
13. [Organização e Convenções](#organização-e-convenções)
14. [CI e Cobertura](#ci-e-cobertura)
15. [Anti-patterns de Teste](#anti-patterns-de-teste)
16. [Exercícios Práticos](#exercícios-práticos)
17. [Recursos Adicionais](#recursos-adicionais)

---

## Introdução

Os cinco documentos anteriores desta série construíram um sistema de ideias: como organizar estado, como desenhar fronteiras, como isolar lógica de negócio, como tomar decisões de arquitetura, e como evitar a deterioração do código ao longo do tempo. Este sexto documento trata da **fundação** que sustenta tudo isso.

Sem testes automatizados, cada conceito dos documentos anteriores existe em teoria mas é impossível de manter na prática. Você pode ter a melhor arquitetura do mundo (documento 4), mas sem testes, qualquer refatoração para "prevenir bagunça" (documento 5) é um risco enorme. Você pode separar lógica de negócio em Actions e Value Objects (documento 3), mas sem testes que verifiquem essas regras, a primeira mudança sob pressão de prazo pode introduzir bugs silenciosos. Você pode definir contratos de API estáveis (documento 2), mas sem testes de contrato, uma alteração no backend quebra o frontend sem que ninguém perceba até o deploy.

Testes automatizados não são uma prática complementar — são a **pré-condição** para que todas as outras práticas funcionem. São eles que transformam "acho que funciona" em "sei que funciona", e "não posso mexer nisso" em "posso refatorar com confiança".

Este documento cobre: a anatomia de um bom teste, a diferença prática entre testes unitários e de integração no Laravel, como testar regras de negócio isoladas do banco de dados, como testar APIs, quando e como usar mocks e fakes, o ciclo TDD, as diferenças entre Pest e PHPUnit, o que testar e o que não testar, e como integrar tudo em um pipeline de CI.

---

## Por que isso importa?

### O custo de não ter testes

**Medo de refatorar.** O controller gordo que todo mundo sabe que precisa ser refatorado continua gordo porque ninguém confia que a refatoração não vai quebrar algo. Sem testes, cada mudança é uma aposta — e sob pressão, apostar é irracional.

**Regressões silenciosas.** Corrigir o cálculo de desconto no módulo de vendas quebra a geração de nota fiscal no módulo financeiro. Ninguém percebe até que um cliente reclama três semanas depois. O custo de encontrar e corrigir o bug em produção é 10-100x maior do que teria sido capturá-lo no CI.

**Especificação implícita.** Sem testes, a única forma de saber o que o sistema *deveria* fazer é ler o código e tentar deduzir a intenção. Quando a implementação tem um bug, não há forma de distingui-lo do comportamento intencional.

**Feedback loop lento.** Sem testes automatizados, o ciclo de feedback é: escrever código → fazer deploy → testar manualmente → descobrir bugs → corrigir → repetir. Esse ciclo leva minutos ou horas. Com testes, o ciclo é: escrever código → rodar testes → ver resultado. Segundos.

### O que testes proporcionam

**Confiança para mudar.** Testes são uma rede de segurança. Se os testes passam depois de uma refatoração, a mudança provavelmente está correta. Isso transforma refatoração de "operação arriscada" em "prática rotineira" — que é exatamente o que os documentos 4 e 5 precisam para funcionar.

**Documentação executável.** Um teste de feature que diz `usuario_autenticado_cria_pedido_com_sucesso` é uma especificação que nunca fica desatualizada: se o comportamento mudar e o teste não for atualizado, ele falha. Nenhum documento em markdown oferece essa garantia.

**Design feedback.** Código difícil de testar é código mal projetado. Se para testar uma Action você precisa instanciar 15 dependências e preparar 8 tabelas no banco, a Action tem responsabilidades demais. Testes são um espelho que reflete problemas de design — ouvir o que eles dizem melhora a arquitetura.

**Velocidade sustentável.** O investimento inicial em testes é real: leva mais tempo para entregar a primeira feature. Mas a partir da segunda, terceira, décima feature, o retorno aparece: menos tempo debugando, menos tempo testando manualmente, menos tempo explicando regressões para o cliente.

---

## Anatomia de um Teste

Todo teste automatizado segue a mesma estrutura em três fases, frequentemente chamada de **Arrange-Act-Assert** (ou Given-When-Then no BDD):

```php
/** @test */
public function pedido_pendente_pode_ser_cancelado(): void
{
    // ARRANGE (Given) — preparar o cenário
    $order = Order::factory()->create([
        'status' => OrderStatus::PENDING,
    ]);

    // ACT (When) — executar a ação sendo testada
    $order->cancel();

    // ASSERT (Then) — verificar o resultado
    $this->assertEquals(OrderStatus::CANCELLED, $order->fresh()->status);
    $this->assertNotNull($order->fresh()->cancelled_at);
}
```

**Arrange** monta o estado inicial: cria objetos, prepara dados, configura mocks. É a parte que mais varia entre testes.

**Act** executa a operação que está sendo testada. Idealmente, é uma única linha — uma chamada de método, uma requisição HTTP, uma invocação de Action.

**Assert** verifica que o resultado é o esperado. Pode verificar valores de retorno, estado do banco, eventos disparados, emails enviados, ou exceções lançadas.

### Princípios de um bom teste

**Um teste, um conceito.** Cada teste verifica uma coisa. Se o teste falha, o nome já diz o que quebrou. Evite testes que verificam 10 coisas diferentes — quando falham, exigem investigação para entender qual parte deu errado.

**Independência.** Testes não devem depender uns dos outros. A ordem de execução não deve importar. O `RefreshDatabase` trait do Laravel garante isso resetando o banco entre testes.

**Determinismo.** O mesmo teste rodado 100 vezes deve dar o mesmo resultado. Evite depender de timestamps, IDs auto-incrementais, ou estado global que varia entre execuções.

**Velocidade.** Testes lentos são testes que ninguém roda. Prefira testes unitários (milissegundos) sobre testes de integração (centenas de milissegundos) sobre testes E2E (segundos).

---

## Tipos de Teste no Laravel

O Laravel organiza testes em duas pastas: `tests/Unit` e `tests/Feature`. A distinção é prática:

### Unit Tests (`tests/Unit/`)

Testam uma classe ou método isoladamente. Não usam o framework, não acessam banco, não fazem requisições HTTP. Estendem `PHPUnit\Framework\TestCase` (não `Tests\TestCase`).

Bons candidatos: Value Objects, DTOs, Enums com lógica, cálculos puros, validadores, formatadores, e qualquer classe que não depende de infraestrutura.

```php
// tests/Unit/ValueObjects/CPFTest.php
use PHPUnit\Framework\TestCase;

class CPFTest extends TestCase
{
    /** @test */
    public function aceita_cpf_valido_com_mascara(): void
    {
        $cpf = new CPF('123.456.789-09');

        $this->assertEquals('12345678909', $cpf->value);
    }

    /** @test */
    public function rejeita_cpf_com_digitos_repetidos(): void
    {
        $this->expectException(InvalidCPFException::class);

        new CPF('111.111.111-11');
    }
}
```

### Feature Tests (`tests/Feature/`)

Testam o fluxo completo: da requisição HTTP à resposta, passando por validação, lógica de negócio e persistência. Usam o framework completo, acessam banco, e estendem `Tests\TestCase`.

Bons candidatos: endpoints de API, fluxos de criação/edição, autenticação, autorização, e qualquer cenário que envolva múltiplas camadas trabalhando juntas.

```php
// tests/Feature/Orders/CreateOrderTest.php
use Illuminate\Foundation\Testing\RefreshDatabase;

class CreateOrderTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function cria_pedido_com_itens_e_calcula_total(): void
    {
        $user = User::factory()->create();
        $product = Product::factory()->create(['price' => 50.00, 'stock' => 10]);

        $response = $this->actingAs($user)->postJson('/api/v1/orders', [
            'items' => [
                ['product_id' => $product->id, 'quantity' => 3],
            ],
        ]);

        $response->assertCreated();
        $this->assertDatabaseHas('orders', [
            'user_id' => $user->id,
        ]);
    }
}
```

### Onde cada tipo se encaixa

A regra geral: teste **lógica pura** com testes unitários e **comportamento integrado** com testes de feature. Na prática com Laravel, a maioria dos projetos tem mais feature tests do que unit tests — porque muito do valor está na interação entre camadas (validação → Action → banco → resposta).

```
┌──────────────────────────────────────────────────────────┐
│  Testes de Feature                                       │
│  Controllers, endpoints, fluxos completos                │
│  → "O endpoint POST /orders funciona corretamente?"      │
├──────────────────────────────────────────────────────────┤
│  Testes Unitários                                        │
│  Value Objects, cálculos, regras puras                   │
│  → "O Money soma centavos corretamente?"                 │
│  → "O CPF rejeita dígitos repetidos?"                    │
│  → "O Order.canBeCancelled() retorna false quando        │
│     status é SHIPPED?"                                   │
└──────────────────────────────────────────────────────────┘
```

---

## Testes Unitários

Testes unitários verificam uma unidade de código isolada. No contexto da arquitetura discutida nos documentos anteriores, as unidades mais valiosas para testar são: Value Objects, métodos de domínio nos Models, cálculos em Services, e lógica de DTOs.

### Testando Value Objects

Value Objects são os candidatos perfeitos para testes unitários: são puros (sem dependências), imutáveis, e têm comportamento bem definido.

```php
// tests/Unit/ValueObjects/MoneyTest.php
use App\ValueObjects\Money;
use PHPUnit\Framework\TestCase;

class MoneyTest extends TestCase
{
    /** @test */
    public function cria_a_partir_de_float_em_centavos(): void
    {
        $money = Money::fromFloat(99.90);

        $this->assertEquals(9990, $money->amount);
        $this->assertEquals('BRL', $money->currency);
    }

    /** @test */
    public function soma_dois_valores(): void
    {
        $a = Money::fromFloat(10.50);
        $b = Money::fromFloat(5.25);

        $result = $a->add($b);

        $this->assertEquals(1575, $result->amount);
    }

    /** @test */
    public function impede_soma_de_moedas_diferentes(): void
    {
        $brl = new Money(1000, 'BRL');
        $usd = new Money(1000, 'USD');

        $this->expectException(InvalidArgumentException::class);
        $brl->add($usd);
    }

    /** @test */
    public function formata_em_reais(): void
    {
        $money = Money::fromFloat(1234.56);

        $this->assertEquals('R$ 1.234,56', $money->formatted());
    }

    /** @test */
    public function nao_aceita_valor_negativo(): void
    {
        $this->expectException(InvalidArgumentException::class);
        new Money(-1, 'BRL');
    }

    /** @test */
    public function multiplicacao_retorna_nova_instancia(): void
    {
        $original = Money::fromFloat(10.00);
        $resultado = $original->multiply(3);

        // Imutabilidade: original não foi alterado
        $this->assertEquals(1000, $original->amount);
        $this->assertEquals(3000, $resultado->amount);
    }
}
```

Esses testes rodam em milissegundos porque não tocam no banco, não carregam o framework, e não fazem I/O. Se algum quebrar, você sabe exatamente onde está o problema.

### Testando lógica de domínio no Model

Comportamentos intrínsecos do Model (como discutido no documento 3 — Business Logic) são testáveis unitariamente se não dependem do banco:

```php
// tests/Unit/Models/OrderTest.php
use App\Models\Order;
use App\Enums\OrderStatus;
use PHPUnit\Framework\TestCase;

class OrderTest extends TestCase
{
    /** @test */
    public function pedido_pendente_pode_ser_cancelado(): void
    {
        $order = new Order(['status' => OrderStatus::PENDING]);

        $this->assertTrue($order->canBeCancelled());
    }

    /** @test */
    public function pedido_enviado_nao_pode_ser_cancelado(): void
    {
        $order = new Order(['status' => OrderStatus::SHIPPED]);

        $this->assertFalse($order->canBeCancelled());
    }

    /** @test */
    public function pedido_entregue_nao_pode_ser_cancelado(): void
    {
        $order = new Order(['status' => OrderStatus::DELIVERED]);

        $this->assertFalse($order->canBeCancelled());
    }

    /** @test */
    public function pedido_ja_cancelado_nao_pode_ser_cancelado_novamente(): void
    {
        $order = new Order(['status' => OrderStatus::CANCELLED]);

        $this->assertFalse($order->canBeCancelled());
    }
}
```

Note que usamos `new Order([...])` e não `Order::factory()->create()`. Não estamos persistindo no banco — estamos testando lógica pura do Model. O teste verifica se o método `canBeCancelled()` se comporta corretamente para cada status, sem depender de infraestrutura.

### Testando Enums com lógica

```php
// app/Enums/OrderStatus.php
enum OrderStatus: string
{
    case PENDING = 'pending';
    case CONFIRMED = 'confirmed';
    case SHIPPED = 'shipped';
    case DELIVERED = 'delivered';
    case CANCELLED = 'cancelled';

    public function canTransitionTo(self $target): bool
    {
        return match ($this) {
            self::PENDING   => in_array($target, [self::CONFIRMED, self::CANCELLED]),
            self::CONFIRMED => in_array($target, [self::SHIPPED, self::CANCELLED]),
            self::SHIPPED   => in_array($target, [self::DELIVERED]),
            self::DELIVERED => false,
            self::CANCELLED => false,
        };
    }

    public function label(): string
    {
        return match ($this) {
            self::PENDING   => 'Pendente',
            self::CONFIRMED => 'Confirmado',
            self::SHIPPED   => 'Enviado',
            self::DELIVERED => 'Entregue',
            self::CANCELLED => 'Cancelado',
        };
    }
}
```

```php
// tests/Unit/Enums/OrderStatusTest.php
use App\Enums\OrderStatus;
use PHPUnit\Framework\TestCase;

class OrderStatusTest extends TestCase
{
    /** @test */
    public function pendente_pode_transicionar_para_confirmado(): void
    {
        $this->assertTrue(
            OrderStatus::PENDING->canTransitionTo(OrderStatus::CONFIRMED)
        );
    }

    /** @test */
    public function pendente_pode_ser_cancelado(): void
    {
        $this->assertTrue(
            OrderStatus::PENDING->canTransitionTo(OrderStatus::CANCELLED)
        );
    }

    /** @test */
    public function pendente_nao_pode_pular_para_enviado(): void
    {
        $this->assertFalse(
            OrderStatus::PENDING->canTransitionTo(OrderStatus::SHIPPED)
        );
    }

    /** @test */
    public function entregue_e_estado_final(): void
    {
        foreach (OrderStatus::cases() as $target) {
            $this->assertFalse(
                OrderStatus::DELIVERED->canTransitionTo($target)
            );
        }
    }

    /** @test */
    public function todos_os_status_tem_label_em_portugues(): void
    {
        foreach (OrderStatus::cases() as $status) {
            $this->assertNotEmpty($status->label());
            // Garante que nenhum case foi esquecido no match
        }
    }
}
```

---

## Testes de Feature (Integração)

Feature tests verificam que o sistema funciona como um todo: a requisição HTTP chega, passa pela validação, executa a lógica de negócio, persiste no banco, e retorna a resposta correta.

### Testando CRUD completo

```php
// tests/Feature/Products/ProductCrudTest.php
use App\Models\Product;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

class ProductCrudTest extends TestCase
{
    use RefreshDatabase;

    private User $admin;

    protected function setUp(): void
    {
        parent::setUp();
        $this->admin = User::factory()->admin()->create();
    }

    /** @test */
    public function lista_produtos_paginados(): void
    {
        Product::factory()->count(25)->create();

        $response = $this->actingAs($this->admin)
            ->getJson('/api/v1/products?per_page=10');

        $response
            ->assertOk()
            ->assertJsonCount(10, 'data')
            ->assertJsonStructure([
                'data' => [
                    '*' => ['id', 'name', 'price', 'description'],
                ],
                'meta' => ['total', 'current_page', 'last_page'],
            ]);
    }

    /** @test */
    public function cria_produto_com_dados_validos(): void
    {
        $response = $this->actingAs($this->admin)
            ->postJson('/api/v1/products', [
                'name' => 'Notebook Dell',
                'price' => 4599.90,
                'description' => 'Notebook com 16GB RAM',
                'stock' => 50,
            ]);

        $response
            ->assertCreated()
            ->assertJsonPath('data.name', 'Notebook Dell');

        $this->assertDatabaseHas('products', [
            'name' => 'Notebook Dell',
            'price' => 4599.90,
        ]);
    }

    /** @test */
    public function rejeita_produto_sem_nome(): void
    {
        $response = $this->actingAs($this->admin)
            ->postJson('/api/v1/products', [
                'price' => 100.00,
            ]);

        $response
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['name']);
    }

    /** @test */
    public function rejeita_preco_negativo(): void
    {
        $response = $this->actingAs($this->admin)
            ->postJson('/api/v1/products', [
                'name' => 'Produto',
                'price' => -10.00,
            ]);

        $response
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['price']);
    }

    /** @test */
    public function atualiza_produto_existente(): void
    {
        $product = Product::factory()->create(['name' => 'Original']);

        $response = $this->actingAs($this->admin)
            ->putJson("/api/v1/products/{$product->id}", [
                'name' => 'Atualizado',
                'price' => $product->price,
            ]);

        $response->assertOk();
        $this->assertDatabaseHas('products', [
            'id' => $product->id,
            'name' => 'Atualizado',
        ]);
    }

    /** @test */
    public function deleta_produto(): void
    {
        $product = Product::factory()->create();

        $response = $this->actingAs($this->admin)
            ->deleteJson("/api/v1/products/{$product->id}");

        $response->assertOk();
        $this->assertDatabaseMissing('products', ['id' => $product->id]);
    }

    /** @test */
    public function retorna_404_para_produto_inexistente(): void
    {
        $response = $this->actingAs($this->admin)
            ->getJson('/api/v1/products/99999');

        $response->assertNotFound();
    }
}
```

### Testando autenticação e autorização

```php
// tests/Feature/Orders/OrderAuthorizationTest.php
class OrderAuthorizationTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function visitante_nao_pode_criar_pedido(): void
    {
        $response = $this->postJson('/api/v1/orders', []);

        $response->assertUnauthorized();
    }

    /** @test */
    public function usuario_so_ve_seus_proprios_pedidos(): void
    {
        $maria = User::factory()->create();
        $joao = User::factory()->create();

        Order::factory()->for($maria)->count(3)->create();
        Order::factory()->for($joao)->count(2)->create();

        $response = $this->actingAs($maria)->getJson('/api/v1/orders');

        $response
            ->assertOk()
            ->assertJsonCount(3, 'data');
    }

    /** @test */
    public function usuario_nao_pode_ver_pedido_de_outro(): void
    {
        $maria = User::factory()->create();
        $joao = User::factory()->create();
        $orderDoJoao = Order::factory()->for($joao)->create();

        $response = $this->actingAs($maria)
            ->getJson("/api/v1/orders/{$orderDoJoao->id}");

        $response->assertForbidden();
    }

    /** @test */
    public function admin_pode_ver_qualquer_pedido(): void
    {
        $admin = User::factory()->admin()->create();
        $order = Order::factory()->create();

        $response = $this->actingAs($admin)
            ->getJson("/api/v1/orders/{$order->id}");

        $response->assertOk();
    }
}
```

### Testando regras de negócio via endpoint

```php
// tests/Feature/Orders/CancelOrderTest.php
class CancelOrderTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function cancela_pedido_pendente(): void
    {
        $user = User::factory()->create();
        $order = Order::factory()
            ->for($user)
            ->create(['status' => OrderStatus::PENDING]);

        $response = $this->actingAs($user)
            ->postJson("/api/v1/orders/{$order->id}/cancel");

        $response->assertOk();
        $this->assertEquals(
            OrderStatus::CANCELLED,
            $order->fresh()->status
        );
    }

    /** @test */
    public function nao_cancela_pedido_ja_enviado(): void
    {
        $user = User::factory()->create();
        $order = Order::factory()
            ->for($user)
            ->create(['status' => OrderStatus::SHIPPED]);

        $response = $this->actingAs($user)
            ->postJson("/api/v1/orders/{$order->id}/cancel");

        $response->assertStatus(409);
        $this->assertEquals(
            OrderStatus::SHIPPED,
            $order->fresh()->status
        );
    }

    /** @test */
    public function cancelamento_restaura_estoque(): void
    {
        $user = User::factory()->create();
        $product = Product::factory()->create(['stock' => 10]);
        $order = Order::factory()
            ->for($user)
            ->hasItems(1, ['product_id' => $product->id, 'quantity' => 3])
            ->create(['status' => OrderStatus::PENDING]);

        $this->actingAs($user)
            ->postJson("/api/v1/orders/{$order->id}/cancel");

        $this->assertEquals(13, $product->fresh()->stock);
    }
}
```

---

## Testando Regras de Negócio sem Banco

Esta é uma das habilidades mais valiosas — e mais subutilizadas — no ecossistema Laravel. Se sua arquitetura segue os princípios do documento 3 (Business Logic), com Actions que recebem dependências injetadas, você pode testar a lógica de negócio sem tocar no banco de dados.

### O problema com testes que dependem do banco

```php
// Este teste demora ~200ms por causa do banco
/** @test */
public function calcula_desconto_para_cliente_premium(): void
{
    // Arrange: precisa criar registros no banco
    $customer = Customer::factory()->premium()->create();
    $product = Product::factory()->create(['price' => 100.00]);
    $order = Order::factory()
        ->for($customer)
        ->hasItems(1, ['product_id' => $product->id, 'quantity' => 2])
        ->create();

    // Act + Assert
    $total = $order->totalWithDiscount();
    $this->assertEquals(180.00, $total); // 10% de desconto
}
```

O problema não é que esse teste é *errado* — é que ele é lento e frágil. Depende de factories, migrations, e transações de banco. Se a migration mudar, o teste quebra mesmo que a lógica de desconto esteja correta.

### A solução: extrair lógica para classes testáveis

Quando a lógica de negócio está em classes puras (Value Objects, Services puros, cálculos), ela pode ser testada sem infraestrutura:

```php
// app/Services/DiscountCalculator.php
class DiscountCalculator
{
    public function calculate(Money $subtotal, CustomerType $customerType): Money
    {
        $rate = match ($customerType) {
            CustomerType::PREMIUM => 0.10,
            CustomerType::VIP     => 0.15,
            default               => 0.00,
        };

        $discountAmount = (int) round($subtotal->amount * $rate);

        return new Money($discountAmount, $subtotal->currency);
    }
}
```

```php
// tests/Unit/Services/DiscountCalculatorTest.php
use PHPUnit\Framework\TestCase;

class DiscountCalculatorTest extends TestCase
{
    private DiscountCalculator $calculator;

    protected function setUp(): void
    {
        $this->calculator = new DiscountCalculator();
    }

    /** @test */
    public function cliente_premium_recebe_10_porcento(): void
    {
        $subtotal = Money::fromFloat(200.00);

        $discount = $this->calculator->calculate($subtotal, CustomerType::PREMIUM);

        $this->assertEquals(2000, $discount->amount); // R$ 20,00
    }

    /** @test */
    public function cliente_vip_recebe_15_porcento(): void
    {
        $subtotal = Money::fromFloat(200.00);

        $discount = $this->calculator->calculate($subtotal, CustomerType::VIP);

        $this->assertEquals(3000, $discount->amount); // R$ 30,00
    }

    /** @test */
    public function cliente_regular_nao_recebe_desconto(): void
    {
        $subtotal = Money::fromFloat(200.00);

        $discount = $this->calculator->calculate($subtotal, CustomerType::REGULAR);

        $this->assertEquals(0, $discount->amount);
    }

    /** @test */
    public function arredondamento_funciona_corretamente(): void
    {
        $subtotal = Money::fromFloat(33.33); // 3333 centavos

        $discount = $this->calculator->calculate($subtotal, CustomerType::PREMIUM);

        // 10% de 3333 = 333.3, arredondado para 333
        $this->assertEquals(333, $discount->amount);
    }
}
```

Esses testes rodam em ~5ms no total. Não precisam de banco, framework, ou migrations. Se algum falhar, você sabe exatamente que a lógica de desconto tem um bug — não precisa investigar se o problema é na factory, na migration, ou na conexão com o banco.

### Testando Actions com dependências mocadas

Quando a Action depende de interfaces (como recomendado nos documentos 3 e 4), você pode injetar fakes no lugar das implementações reais:

```php
// app/Actions/CreateOrderAction.php
class CreateOrderAction
{
    public function __construct(
        private OrderRepositoryInterface $orders,
        private InventoryServiceInterface $inventory,
        private DiscountCalculator $discountCalculator,
    ) {}

    public function execute(CreateOrderData $data): Order
    {
        // Verificar estoque
        foreach ($data->items as $item) {
            if (! $this->inventory->hasStock($item->productId, $item->quantity)) {
                throw new InsufficientStockException($item->productId, $item->quantity);
            }
        }

        // Calcular total com desconto
        $subtotal = $data->calculateSubtotal();
        $discount = $this->discountCalculator->calculate($subtotal, $data->customerType);
        $total = $subtotal->subtract($discount);

        // Persistir
        $order = $this->orders->create([
            'customer_id' => $data->customerId,
            'subtotal' => $subtotal->amount,
            'discount' => $discount->amount,
            'total' => $total->amount,
        ]);

        // Reservar estoque
        $this->inventory->reserve($order);

        return $order;
    }
}
```

```php
// tests/Unit/Actions/CreateOrderActionTest.php
use PHPUnit\Framework\TestCase;
use Mockery;

class CreateOrderActionTest extends TestCase
{
    /** @test */
    public function cria_pedido_quando_estoque_disponivel(): void
    {
        // Mocks que simulam as dependências
        $orderRepo = Mockery::mock(OrderRepositoryInterface::class);
        $inventory = Mockery::mock(InventoryServiceInterface::class);
        $calculator = new DiscountCalculator(); // Classe pura, não precisa de mock

        // Configurar comportamento dos mocks
        $inventory->shouldReceive('hasStock')->andReturn(true);
        $inventory->shouldReceive('reserve')->once();

        $fakeOrder = new Order(['id' => 1, 'total' => 9000]);
        $orderRepo->shouldReceive('create')->once()->andReturn($fakeOrder);

        // Executar
        $action = new CreateOrderAction($orderRepo, $inventory, $calculator);
        $data = new CreateOrderData(
            customerId: 1,
            customerType: CustomerType::REGULAR,
            items: [new OrderItemData(productId: 1, quantity: 2, price: Money::fromFloat(45.00))],
        );

        $order = $action->execute($data);

        // Verificar
        $this->assertNotNull($order);
    }

    /** @test */
    public function rejeita_pedido_sem_estoque(): void
    {
        $orderRepo = Mockery::mock(OrderRepositoryInterface::class);
        $inventory = Mockery::mock(InventoryServiceInterface::class);
        $calculator = new DiscountCalculator();

        $inventory->shouldReceive('hasStock')->andReturn(false);
        $orderRepo->shouldNotReceive('create'); // Nunca deve ser chamado

        $action = new CreateOrderAction($orderRepo, $inventory, $calculator);
        $data = new CreateOrderData(
            customerId: 1,
            customerType: CustomerType::REGULAR,
            items: [new OrderItemData(productId: 99, quantity: 100, price: Money::fromFloat(10.00))],
        );

        $this->expectException(InsufficientStockException::class);
        $action->execute($data);
    }

    protected function tearDown(): void
    {
        Mockery::close();
    }
}
```

O ponto fundamental: estamos testando a **lógica de orquestração** da Action sem banco de dados. Verificamos que ela checa estoque antes de criar, que rejeita pedidos sem estoque, e que o repositório não é chamado em caso de erro. Tudo em milissegundos.

---

## Testando APIs

Testes de API verificam o contrato entre seu backend e os consumidores (frontend, mobile, integrações). São a forma mais eficaz de garantir que mudanças no backend não quebram o frontend.

### Verificação de estrutura da resposta

```php
/** @test */
public function resposta_de_listagem_segue_formato_padrao(): void
{
    Product::factory()->count(3)->create();

    $response = $this->getJson('/api/v1/products');

    $response
        ->assertOk()
        ->assertJsonStructure([
            'data' => [
                '*' => [
                    'id',
                    'name',
                    'price',
                    'description',
                    'created_at',
                ],
            ],
            'meta' => [
                'total',
                'current_page',
                'last_page',
                'per_page',
            ],
            'links' => [
                'first',
                'last',
                'prev',
                'next',
            ],
        ]);
}
```

Esse teste quebra se alguém renomear um campo, remover um campo, ou mudar a estrutura da paginação. É exatamente o que queremos: detectar breaking changes antes do deploy.

### Verificação de dados sensíveis

```php
/** @test */
public function resposta_de_usuario_nao_expoe_dados_sensiveis(): void
{
    $user = User::factory()->create();

    $response = $this->actingAs($user)->getJson('/api/v1/user');

    $response->assertOk();

    // Garante que campos sensíveis NÃO estão na resposta
    $data = $response->json('data');
    $this->assertArrayNotHasKey('password', $data);
    $this->assertArrayNotHasKey('remember_token', $data);
    $this->assertArrayNotHasKey('two_factor_secret', $data);
}
```

### Verificação de filtros e paginação

```php
/** @test */
public function filtra_produtos_por_categoria(): void
{
    $eletronicos = Category::factory()->create(['name' => 'Eletrônicos']);
    $roupas = Category::factory()->create(['name' => 'Roupas']);

    Product::factory()->count(3)->for($eletronicos)->create();
    Product::factory()->count(2)->for($roupas)->create();

    $response = $this->getJson(
        "/api/v1/products?category={$eletronicos->id}"
    );

    $response
        ->assertOk()
        ->assertJsonCount(3, 'data');
}

/** @test */
public function respeita_limite_de_paginacao(): void
{
    Product::factory()->count(50)->create();

    $response = $this->getJson('/api/v1/products?per_page=5');

    $response
        ->assertOk()
        ->assertJsonCount(5, 'data')
        ->assertJsonPath('meta.total', 50)
        ->assertJsonPath('meta.last_page', 10);
}
```

---

## Mocks, Fakes e Stubs

Quando testar um componente que depende de serviços externos (email, pagamento, API de terceiros), você substitui a dependência real por uma versão controlada. O Laravel oferece excelente suporte nativo para isso.

### Fakes do Laravel

Fakes são implementações em memória que o Laravel fornece para seus próprios serviços:

```php
/** @test */
public function envia_email_de_confirmacao_ao_criar_pedido(): void
{
    Mail::fake(); // Substitui o Mailer real por um fake

    $user = User::factory()->create();
    $product = Product::factory()->create(['stock' => 10]);

    $this->actingAs($user)->postJson('/api/v1/orders', [
        'items' => [['product_id' => $product->id, 'quantity' => 1]],
    ]);

    // Verifica que o email foi "enviado" (na verdade, registrado pelo fake)
    Mail::assertSent(OrderConfirmationMail::class, function ($mail) use ($user) {
        return $mail->hasTo($user->email);
    });
}

/** @test */
public function dispara_evento_ao_cancelar_pedido(): void
{
    Event::fake([OrderCancelled::class]);

    $user = User::factory()->create();
    $order = Order::factory()
        ->for($user)
        ->create(['status' => OrderStatus::PENDING]);

    $this->actingAs($user)
        ->postJson("/api/v1/orders/{$order->id}/cancel");

    Event::assertDispatched(OrderCancelled::class, function ($event) use ($order) {
        return $event->order->id === $order->id;
    });
}

/** @test */
public function registra_job_de_geracao_de_nota_fiscal(): void
{
    Queue::fake();

    $user = User::factory()->create();
    $product = Product::factory()->create(['stock' => 10]);

    $this->actingAs($user)->postJson('/api/v1/orders', [
        'items' => [['product_id' => $product->id, 'quantity' => 1]],
    ]);

    Queue::assertPushed(GenerateInvoiceJob::class);
}

/** @test */
public function armazena_documento_no_storage(): void
{
    Storage::fake('local');

    $user = User::factory()->create();

    $this->actingAs($user)->postJson('/api/v1/documents', [
        'file' => UploadedFile::fake()->create('contrato.pdf', 1024),
    ]);

    Storage::disk('local')->assertExists('documents/contrato.pdf');
}
```

Fakes são preferíveis a mocks para serviços do Laravel: são mais simples, mais legíveis, e testam o comportamento real de integração com o framework.

### Mocks com Mockery

Use mocks quando precisar controlar o comportamento de uma dependência que não tem fake nativo:

```php
/** @test */
public function processa_pagamento_via_gateway(): void
{
    $gateway = Mockery::mock(PaymentGatewayInterface::class);
    $gateway->shouldReceive('charge')
        ->once()
        ->with(Mockery::on(fn ($order) => $order->total === 15000))
        ->andReturn(new PaymentResult(
            transactionId: 'txn_123',
            status: 'approved',
        ));

    $this->app->instance(PaymentGatewayInterface::class, $gateway);

    $user = User::factory()->create();
    $product = Product::factory()->create(['price' => 150.00, 'stock' => 5]);

    $this->actingAs($user)->postJson('/api/v1/orders', [
        'items' => [['product_id' => $product->id, 'quantity' => 1]],
    ]);
}
```

### Http::fake para APIs externas

Quando sua aplicação consome APIs externas, use `Http::fake` para simular as respostas:

```php
/** @test */
public function consulta_cep_na_api_dos_correios(): void
{
    Http::fake([
        'viacep.com.br/ws/45580000/json' => Http::response([
            'cep' => '45580-000',
            'logradouro' => '',
            'bairro' => '',
            'localidade' => 'Itagi',
            'uf' => 'BA',
        ]),
    ]);

    $service = app(CepLookupService::class);
    $result = $service->lookup('45580-000');

    $this->assertEquals('Itagi', $result->cidade);
    $this->assertEquals('BA', $result->estado);

    Http::assertSent(function ($request) {
        return str_contains($request->url(), '45580000');
    });
}

/** @test */
public function lida_com_timeout_da_api_externa(): void
{
    Http::fake([
        'viacep.com.br/*' => Http::response(null, 500),
    ]);

    $service = app(CepLookupService::class);

    $this->expectException(CepLookupFailedException::class);
    $service->lookup('45580-000');
}
```

---

## TDD — Test Driven Development

TDD inverte a ordem convencional: em vez de escrever código e depois testar, você escreve o teste primeiro, vê ele falhar, implementa o mínimo para que passe, e depois refatora.

### O ciclo Red-Green-Refactor

**Red.** Escreva um teste que descreve o comportamento desejado. Rode. Ele falha (porque o código ainda não existe).

**Green.** Implemente a quantidade mínima de código necessária para o teste passar. Não se preocupe com elegância — o objetivo é fazer o teste ficar verde.

**Refactor.** Agora que o teste está verde, melhore a implementação. Extraia métodos, renomeie variáveis, elimine duplicação. O teste garante que a refatoração não quebrou nada.

### Exemplo prático: construindo um validador de CPF com TDD

```php
// RED — teste falha porque a classe não existe
/** @test */
public function aceita_cpf_valido(): void
{
    $validator = new CpfValidator();

    $this->assertTrue($validator->isValid('52998224725'));
}
```

```php
// GREEN — implementação mínima
class CpfValidator
{
    public function isValid(string $cpf): bool
    {
        return true; // Mínimo para o teste passar
    }
}
```

Obviamente `return true` não é uma implementação real. Mas o teste passa. Agora adicionamos mais testes que forçam a implementação a evoluir:

```php
// RED — novo teste que força implementação real
/** @test */
public function rejeita_cpf_com_digitos_repetidos(): void
{
    $validator = new CpfValidator();

    $this->assertFalse($validator->isValid('11111111111'));
}

/** @test */
public function rejeita_cpf_com_tamanho_errado(): void
{
    $validator = new CpfValidator();

    $this->assertFalse($validator->isValid('123'));
}

/** @test */
public function rejeita_cpf_com_digito_verificador_invalido(): void
{
    $validator = new CpfValidator();

    $this->assertFalse($validator->isValid('52998224700'));
}
```

Cada teste novo força a implementação a ficar mais completa. No final do ciclo, você tem uma classe com cobertura de 100% e uma implementação que resolve exatamente os casos que os testes definem.

### TDD na prática — quando usar

TDD funciona muito bem para: Value Objects, cálculos, validadores, máquinas de estado, e qualquer lógica com regras claras e testáveis.

TDD é menos natural para: controllers (que são finos e delegam), views, e integrações com serviços externos (onde o feedback do teste é menos útil durante o desenvolvimento).

A recomendação pragmática: não force TDD em tudo. Use-o quando o problema é bem definido e a lógica é pura. Para o resto, escreva testes junto com o código ou logo após — o importante é que os testes existam, não a ordem em que foram escritos.

---

## Pest vs PHPUnit

Ambos fazem o mesmo trabalho. PHPUnit é o padrão da indústria com mais de 20 anos de uso. Pest é construído sobre o PHPUnit com uma API mais expressiva. No Laravel, ambos funcionam perfeitamente.

### PHPUnit — sintaxe clássica

```php
namespace Tests\Feature;

use Tests\TestCase;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

class OrderTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function usuario_autenticado_cria_pedido(): void
    {
        $user = User::factory()->create();
        $product = Product::factory()->create(['stock' => 10]);

        $response = $this->actingAs($user)->postJson('/api/v1/orders', [
            'items' => [['product_id' => $product->id, 'quantity' => 1]],
        ]);

        $response->assertCreated();
    }

    /** @test */
    public function visitante_nao_pode_criar_pedido(): void
    {
        $response = $this->postJson('/api/v1/orders', []);

        $response->assertUnauthorized();
    }
}
```

### Pest — sintaxe moderna

```php
// tests/Feature/OrderTest.php
uses(RefreshDatabase::class);

test('usuario autenticado cria pedido', function () {
    $user = User::factory()->create();
    $product = Product::factory()->create(['stock' => 10]);

    actingAs($user)
        ->postJson('/api/v1/orders', [
            'items' => [['product_id' => $product->id, 'quantity' => 1]],
        ])
        ->assertCreated();
});

test('visitante nao pode criar pedido', function () {
    postJson('/api/v1/orders', [])
        ->assertUnauthorized();
});
```

### Pest — Expectations API

Pest oferece uma API fluente para assertions que é mais legível:

```php
test('calcula total do pedido', function () {
    $order = Order::factory()
        ->hasItems(3, ['price' => 100.00, 'quantity' => 2])
        ->create();

    expect($order->total())
        ->toBe(600.00)
        ->toBeFloat()
        ->toBeGreaterThan(0);
});

test('status tem transicoes validas', function () {
    expect(OrderStatus::PENDING->canTransitionTo(OrderStatus::CONFIRMED))
        ->toBeTrue();

    expect(OrderStatus::DELIVERED->canTransitionTo(OrderStatus::PENDING))
        ->toBeFalse();
});
```

### Pest — Datasets

O recurso mais poderoso do Pest para evitar duplicação de testes:

```php
test('valida transicoes de status do pedido', function (
    OrderStatus $from,
    OrderStatus $to,
    bool $valido,
) {
    $result = $from->canTransitionTo($to);

    expect($result)->toBe($valido);
})->with([
    'pending → confirmed'   => [OrderStatus::PENDING, OrderStatus::CONFIRMED, true],
    'pending → shipped'     => [OrderStatus::PENDING, OrderStatus::SHIPPED, false],
    'confirmed → shipped'   => [OrderStatus::CONFIRMED, OrderStatus::SHIPPED, true],
    'confirmed → cancelled' => [OrderStatus::CONFIRMED, OrderStatus::CANCELLED, true],
    'shipped → delivered'   => [OrderStatus::SHIPPED, OrderStatus::DELIVERED, true],
    'shipped → pending'     => [OrderStatus::SHIPPED, OrderStatus::PENDING, false],
    'delivered → qualquer'  => [OrderStatus::DELIVERED, OrderStatus::CANCELLED, false],
    'cancelled → qualquer'  => [OrderStatus::CANCELLED, OrderStatus::PENDING, false],
]);
```

Um bloco `test()` com dataset substitui oito testes individuais. A saída mostra cada combinação separadamente, então falhas são fáceis de diagnosticar.

### Qual escolher?

Para projetos novos, Pest é a recomendação: sintaxe mais limpa, datasets poderosos, e suporte completo do Laravel. Para projetos existentes com PHPUnit, não há necessidade de migrar — ambos coexistem e Pest roda testes PHPUnit nativamente.

---

## O que Testar (e o que não testar)

### Sempre teste

**Regras de negócio.** Se o sistema tem uma regra ("pedido não pode ser cancelado após envio", "desconto de 10% para premium", "licitação só publica com documentação completa"), ela precisa de teste. Regras não testadas são regras que vão ser violadas.

**Endpoints de API.** Cada endpoint precisa de pelo menos: um teste de caminho feliz (happy path), um teste de validação (campos obrigatórios), e um teste de autenticação.

**Transições de estado.** Se uma entidade tem estados (pending → confirmed → shipped → delivered), teste todas as transições válidas e pelo menos as inválidas mais perigosas.

**Edge cases conhecidos.** Divisão por zero, lista vazia, datas no limite, valores negativos, strings com caracteres especiais. Se você sabe que pode dar errado, teste.

**Bugs corrigidos.** Quando corrigir um bug, escreva um teste que reproduz o bug antes de corrigi-lo. Isso garante que o bug nunca volta (regression test).

### Não perca tempo testando

**Getters e setters triviais.** Se o método apenas retorna `$this->name`, testar não adiciona valor.

**Framework code.** Não teste que o `Route::get` funciona ou que o `Model::create` persiste no banco. O framework já tem seus próprios testes.

**Código gerado.** Migrations, seeders, factories — exceto quando contêm lógica de negócio.

**Implementações concretas de infraestrutura.** Se o `EloquentOrderRepository` apenas encapsula `Order::find()`, o teste de feature do endpoint já cobre isso indiretamente.

### A heurística de prioridade

Quando o tempo é limitado (sempre é), priorize testes pela combinação de **risco** e **frequência de mudança**:

```
Alta prioridade:
├── Cálculos financeiros (alto risco, mudanças frequentes)
├── Autenticação/Autorização (alto risco)
├── Endpoints de criação de pedido (alto risco, alta frequência)
└── Transições de estado (bugs sutis, difíceis de detectar)

Média prioridade:
├── Endpoints de listagem com filtros
├── Validação de inputs
└── Formatação de dados

Baixa prioridade:
├── Endpoints de leitura simples
├── Helpers de formatação
└── Configurações
```

---

## Organização e Convenções

### Estrutura de pastas

A estrutura dos testes deve espelhar a estrutura do código:

```
tests/
├── Unit/
│   ├── ValueObjects/
│   │   ├── MoneyTest.php
│   │   ├── CPFTest.php
│   │   └── EmailTest.php
│   ├── Enums/
│   │   └── OrderStatusTest.php
│   ├── Models/
│   │   └── OrderTest.php
│   └── Services/
│       └── DiscountCalculatorTest.php
│
├── Feature/
│   ├── Orders/
│   │   ├── CreateOrderTest.php
│   │   ├── CancelOrderTest.php
│   │   ├── ListOrdersTest.php
│   │   └── OrderAuthorizationTest.php
│   ├── Products/
│   │   └── ProductCrudTest.php
│   ├── Auth/
│   │   ├── LoginTest.php
│   │   └── RegistrationTest.php
│   └── Licitacoes/
│       ├── PublicarProcessoTest.php
│       └── ReceberLanceTest.php
│
└── Pest.php   (ou TestCase.php para PHPUnit)
```

### Convenções de nomenclatura

Nomes de teste devem descrever o comportamento, não a implementação:

```php
// ✅ Descreve o comportamento
public function pedido_pendente_pode_ser_cancelado(): void {}
public function rejeita_preco_negativo(): void {}
public function admin_pode_ver_qualquer_pedido(): void {}

// ❌ Descreve a implementação
public function test_cancel_method(): void {}
public function test_validation(): void {}
public function test_show_with_admin_role(): void {}
```

Se o teste falhar, o nome deve dizer imediatamente o que quebrou, sem precisar abrir o arquivo.

### Factories bem estruturadas

Factories são a base de testes de feature legíveis. Invista tempo em factories expressivas:

```php
// database/factories/OrderFactory.php
class OrderFactory extends Factory
{
    public function definition(): array
    {
        return [
            'user_id' => User::factory(),
            'status' => OrderStatus::PENDING,
            'total' => $this->faker->numberBetween(1000, 100000),
        ];
    }

    public function confirmed(): static
    {
        return $this->state(fn () => [
            'status' => OrderStatus::CONFIRMED,
            'confirmed_at' => now(),
        ]);
    }

    public function shipped(): static
    {
        return $this->state(fn () => [
            'status' => OrderStatus::SHIPPED,
            'shipped_at' => now(),
        ]);
    }

    public function cancelled(): static
    {
        return $this->state(fn () => [
            'status' => OrderStatus::CANCELLED,
            'cancelled_at' => now(),
        ]);
    }

    public function withItems(int $count = 3): static
    {
        return $this->has(
            OrderItem::factory()->count($count),
            'items'
        );
    }
}
```

```php
// Uso nos testes — expressivo e legível
$pendingOrder = Order::factory()->create();
$shippedOrder = Order::factory()->shipped()->create();
$orderWithItems = Order::factory()->withItems(5)->create();
$cancelledOrder = Order::factory()->for($user)->cancelled()->create();
```

---

## CI e Cobertura

### Pipeline mínimo de CI

Testes que não rodam automaticamente são testes que serão ignorados. Configure um pipeline que roda a cada push:

```yaml
# .github/workflows/tests.yml
name: Tests

on: [push, pull_request]

jobs:
  tests:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testing
          POSTGRES_USER: testing
          POSTGRES_PASSWORD: testing
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 3

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: pdo_pgsql
          coverage: xdebug

      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist

      - name: Prepare environment
        run: |
          cp .env.testing .env
          php artisan key:generate

      - name: Run Pint
        run: ./vendor/bin/pint --test

      - name: Run PHPStan
        run: ./vendor/bin/phpstan analyse

      - name: Run tests
        run: php artisan test --parallel --coverage --min=80
        env:
          DB_CONNECTION: pgsql
          DB_HOST: localhost
          DB_PORT: 5432
          DB_DATABASE: testing
          DB_USERNAME: testing
          DB_PASSWORD: testing
```

O pipeline executa em sequência: formatação (Pint), análise estática (PHPStan), e testes com cobertura mínima. Se qualquer etapa falhar, o merge é bloqueado.

### Cobertura: métricas e metas

Cobertura mede quais linhas de código são exercitadas pelos testes. Não é uma métrica perfeita — 100% de cobertura não garante 100% de qualidade — mas é um bom indicador de risco.

```bash
# Rodar testes com relatório de cobertura
php artisan test --coverage

# Exigir cobertura mínima (falha se abaixo)
php artisan test --coverage --min=80

# Gerar relatório HTML detalhado
php artisan test --coverage-html=coverage-report
```

Metas pragmáticas:

**60%** — mínimo aceitável para projetos em produção. Cobre os caminhos críticos.

**80%** — bom alvo para a maioria dos projetos. Cobre caminhos críticos e a maioria dos edge cases.

**95%+** — justificável para código financeiro, de saúde, ou de segurança onde bugs têm consequências graves.

A meta não é atingir um número — é ter confiança de que mudanças não quebram o sistema. Se 75% de cobertura te dá essa confiança, 75% é suficiente.

### Testes paralelos

Para acelerar a execução no CI:

```bash
# Rodar testes em paralelo (Laravel 9+)
php artisan test --parallel

# Com número específico de processos
php artisan test --parallel --processes=4
```

Testes paralelos criam bancos de dados separados para cada processo, evitando conflitos. Podem reduzir o tempo de execução em 50-70% dependendo do número de testes.

---

## Anti-patterns de Teste

### 1. Teste que testa tudo de uma vez

```php
// ❌ Se falhar, qual parte está errada?
/** @test */
public function test_order_flow(): void
{
    $user = User::factory()->create();
    $product = Product::factory()->create();

    // Criação
    $response = $this->actingAs($user)->postJson('/api/v1/orders', [/*...*/]);
    $response->assertCreated();

    // Atualização
    $orderId = $response->json('data.id');
    $this->putJson("/api/v1/orders/{$orderId}", [/*...*/])->assertOk();

    // Cancelamento
    $this->postJson("/api/v1/orders/{$orderId}/cancel")->assertOk();

    // Verificações
    $this->assertDatabaseHas('orders', ['status' => 'cancelled']);
    $this->assertDatabaseHas('audit_logs', ['action' => 'cancelled']);
    Mail::assertSent(OrderCancelledMail::class);
    Event::assertDispatched(OrderCancelled::class);
}

// ✅ Cada teste verifica um cenário
/** @test */
public function cria_pedido(): void { /* ... */ }

/** @test */
public function cancela_pedido_pendente(): void { /* ... */ }

/** @test */
public function cancelamento_dispara_evento(): void { /* ... */ }

/** @test */
public function cancelamento_envia_email(): void { /* ... */ }
```

### 2. Teste frágil (Fragile Test)

Teste que quebra por motivos irrelevantes:

```php
// ❌ Frágil — quebra se adicionar campo na resposta
/** @test */
public function retorna_produto(): void
{
    $product = Product::factory()->create(['name' => 'Notebook']);

    $response = $this->getJson("/api/v1/products/{$product->id}");

    // Compara JSON inteiro — qualquer campo novo quebra o teste
    $response->assertExactJson([
        'data' => [
            'id' => $product->id,
            'name' => 'Notebook',
            'price' => $product->price,
        ],
    ]);
}

// ✅ Resiliente — verifica apenas o que importa
/** @test */
public function retorna_produto(): void
{
    $product = Product::factory()->create(['name' => 'Notebook']);

    $response = $this->getJson("/api/v1/products/{$product->id}");

    $response
        ->assertOk()
        ->assertJsonPath('data.name', 'Notebook')
        ->assertJsonStructure(['data' => ['id', 'name', 'price']]);
}
```

### 3. Teste sem assertions

```php
// ❌ O teste "passa" mas não verifica nada
/** @test */
public function cria_pedido(): void
{
    $user = User::factory()->create();

    $this->actingAs($user)->postJson('/api/v1/orders', [
        'items' => [['product_id' => 1, 'quantity' => 1]],
    ]);

    // Sem assert! Se o endpoint retornar 500, o teste ainda passa.
}

// ✅ Sempre verifique o resultado
/** @test */
public function cria_pedido(): void
{
    $user = User::factory()->create();
    $product = Product::factory()->create(['stock' => 10]);

    $response = $this->actingAs($user)->postJson('/api/v1/orders', [
        'items' => [['product_id' => $product->id, 'quantity' => 1]],
    ]);

    $response->assertCreated();
    $this->assertDatabaseHas('orders', ['user_id' => $user->id]);
}
```

### 4. Dependência entre testes

```php
// ❌ Teste B depende do estado que Teste A criou
public function test_A_creates_order(): void
{
    $this->postJson('/api/orders', [/*...*/]); // Cria order id=1
}

public function test_B_cancels_order(): void
{
    // Assume que order id=1 existe — quebra se rodar isolado ou em outra ordem
    $this->postJson('/api/orders/1/cancel');
}

// ✅ Cada teste cria seu próprio estado
public function test_cancels_order(): void
{
    $order = Order::factory()->create(['status' => 'pending']);

    $this->postJson("/api/orders/{$order->id}/cancel")->assertOk();
}
```

### 5. Mock excessivo

```php
// ❌ Tudo é mock — o teste verifica mocks, não comportamento real
public function test_create_order(): void
{
    $repo = Mockery::mock(OrderRepositoryInterface::class);
    $inventory = Mockery::mock(InventoryServiceInterface::class);
    $payment = Mockery::mock(PaymentGatewayInterface::class);
    $mailer = Mockery::mock(MailerInterface::class);
    $logger = Mockery::mock(LoggerInterface::class);

    $repo->shouldReceive('create')->once()->andReturn(new Order());
    $inventory->shouldReceive('hasStock')->andReturn(true);
    $inventory->shouldReceive('reserve')->once();
    $payment->shouldReceive('charge')->once();
    $mailer->shouldReceive('send')->once();
    $logger->shouldReceive('info')->once();

    // O teste só verifica que os mocks foram chamados
    // Se a implementação mudar a ordem das chamadas, o teste quebra
    // mesmo que o comportamento continue correto
}
```

A regra: use mocks para **fronteiras** (APIs externas, gateways de pagamento, serviços de email). Use implementações reais (ou fakes do Laravel) para tudo dentro do seu sistema. Se um teste precisa de mais de 3 mocks, provavelmente a classe sendo testada tem responsabilidades demais.

---

## Exercícios Práticos

### Exercício 1: Testar um Value Object completo

Crie o Value Object `CNPJ` (seguindo o padrão de CPF do documento 3) e escreva testes unitários que cubram: CNPJ válido com e sem máscara, rejeição de CNPJ inválido (dígito verificador errado), rejeição de dígitos repetidos (`11.111.111/1111-11`), rejeição de tamanho incorreto, formatação correta, e método `__toString`.

### Exercício 2: Testar um endpoint de criação

Para o endpoint `POST /api/v1/requerimentos`, escreva testes de feature que cubram: criação com dados válidos (verificando banco e estrutura da resposta), rejeição sem autenticação, rejeição com campos obrigatórios faltando, rejeição com tipo de requerimento inválido, e geração automática de protocolo.

### Exercício 3: Testar regra de negócio sem banco

Extraia a lógica de cálculo de impostos para uma classe `TaxCalculator` que recebe um `Money` e um estado (UF) e retorna o imposto calculado. Escreva testes unitários puros (sem banco, sem framework) que verifiquem: alíquotas diferentes por estado, arredondamento correto, e rejeição de valores negativos.

### Exercício 4: Implementar TDD

Usando o ciclo Red-Green-Refactor, implemente um `ProcessoStatusMachine` que gerencia as transições de status de um processo licitatório. Comece pelo teste mais simples (`rascunho → publicado`), faça-o passar com implementação mínima, e vá adicionando testes para cada transição válida e inválida.

### Exercício 5: Configurar CI

Configure um pipeline de CI no GitHub Actions (ou equivalente) que execute: Pint, PHPStan level 5, e testes com cobertura mínima de 70%. Faça o pipeline rodar em pull requests e bloquear merge quando falhar.

---

## Recursos Adicionais

### Livros

- **Test Driven Development: By Example** — Kent Beck. O livro que definiu TDD. Exemplos em Java, mas os princípios são universais.
- **Working Effectively with Legacy Code** — Michael Feathers. Essencial para adicionar testes a código existente que não foi projetado para ser testável.
- **Growing Object-Oriented Software, Guided by Tests** — Steve Freeman e Nat Pryce. Como usar testes para guiar o design do software.

### Documentação

- [Laravel Testing](https://laravel.com/docs/testing) — documentação oficial com exemplos para HTTP tests, database testing, mocking, e mais.
- [Pest Documentation](https://pestphp.com) — guia completo do Pest com exemplos de expectations, datasets, e plugins.
- [PHPUnit Documentation](https://docs.phpunit.de) — referência completa do PHPUnit.

### Cursos

- Laracasts: "Pest Driven Laravel" — testes com Pest aplicados a projetos Laravel reais.
- Laracasts: "Testing Laravel" — série completa sobre testes no ecossistema Laravel.
- Laracasts: "TDD by Example" — o ciclo Red-Green-Refactor aplicado em PHP.

---

## Conclusão

Testes automatizados são a fundação que sustenta todas as outras práticas de engenharia de software. Sem eles, refatoração é arriscada, lógica de negócio isolada é frágil, contratos de API são promessas vazias, e a arquitetura existe apenas na documentação.

Os princípios que tornam testes eficazes são poucos:

**Teste comportamento, não implementação.** O teste deve dizer "pedido pendente pode ser cancelado", não "o método cancel seta o status para cancelled e chama event". Se a implementação mudar mas o comportamento for preservado, o teste deve continuar passando.

**Isole o que é puro.** Regras de negócio em Value Objects, calculadores e Enums devem ser testadas com testes unitários rápidos, sem banco e sem framework. Isso dá feedback instantâneo e força design limpo.

**Use Feature tests para o resto.** Endpoints, fluxos completos e integrações entre camadas são testados com os HTTP tests do Laravel, que exercitam o sistema como o usuário real o utiliza.

**Automatize no CI.** Testes que não rodam automaticamente serão ignorados. O pipeline deve rodar formatação, análise estática e testes a cada push, bloqueando código que não atende aos padrões.

**Invista em factories.** Factories expressivas (`Order::factory()->shipped()->withItems(3)`) tornam os testes legíveis e reduzem a quantidade de código de setup.

Comece pela parte do sistema que causa mais dor: o endpoint que quebra todo mês, a regra de cálculo que ninguém confia, a transição de estado que tem bugs recorrentes. Escreva testes para esses pontos críticos primeiro. A confiança que os testes trazem se paga rapidamente — e cada teste adicional torna o próximo mais fácil de justificar.
