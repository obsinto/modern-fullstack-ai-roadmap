# Preventing Long-Term Messes (Evitando o Caos a Longo Prazo)

## Índice

1. [Introdução](#introdução)
2. [Por que isso importa?](#por-que-isso-importa)
3. [Testes Automatizados](#testes-automatizados)
4. [Análise Estática](#análise-estática)
5. [Code Quality Tools](#code-quality-tools)
6. [SOLID Principles](#solid-principles)
7. [Design Patterns](#design-patterns)
8. [Refactoring](#refactoring)
9. [Technical Debt](#technical-debt)
10. [Documentation](#documentation)
11. [Code Review](#code-review)
12. [Anti-patterns Comuns](#anti-patterns-comuns)
13. [Boas Práticas](#boas-práticas)
14. [Exercícios Práticos](#exercícios-práticos)
15. [Recursos Adicionais](#recursos-adicionais)

---

## Introdução

Os quatro documentos anteriores desta série tratam de **como** organizar código: onde colocar estado, como desenhar fronteiras, como isolar lógica de negócio, como tomar decisões de arquitetura. Este documento trata de uma questão diferente e igualmente importante: **como impedir que tudo isso se deteriore com o tempo**.

Código novo é limpo por definição — não acumulou complexidade, gambiarras ou mudanças de requisito. O problema não é escrever código bom; é mantê-lo bom enquanto dezenas de funcionalidades são adicionadas, bugs são corrigidos sob pressão, e novos desenvolvedores entram e saem do projeto. Sem salvaguardas ativas, a entropia vence: o que era organizado vira confuso, o que era testado deixa de ser, e o que era documentado fica desatualizado.

A solução não é disciplina individual (humanos falham sob pressão) — é automação. Ferramentas que detectam problemas antes que virem catástrofes, testes que verificam o comportamento esperado a cada commit, análise estática que encontra bugs sem executar código, e pipelines de CI que bloqueiam deploys quando os padrões não são atendidos.

Este documento cobre o arsenal completo: testes automatizados, análise estática, ferramentas de qualidade, princípios SOLID, design patterns úteis no dia a dia, técnicas de refatoração, gestão de dívida técnica, documentação eficaz e code review produtivo.

---

## Por que isso importa?

### A deterioração é gradual

Nenhum sistema passa de "limpo" para "caótico" em um dia. A deterioração acontece commit a commit: um atalho aqui porque o prazo apertou, um teste que ninguém escreveu porque "era simples demais", uma regra de negócio colocada no controller porque "depois eu movo". Cada decisão individual parece inofensiva. Acumuladas ao longo de meses, criam um sistema onde ninguém confia no próprio código.

**Fear of Change** é o sintoma mais revelador. Quando o time evita refatorar, evita mexer em funcionalidades existentes, e trata cada deploy como uma operação de risco, o sistema já está doente. A causa raiz quase sempre é a mesma: ausência de testes automatizados que dariam confiança de que mudanças não quebraram nada.

**Bug Whack-a-Mole** é outro sintoma clássico. Corrigir um bug introduz outro, porque as dependências entre partes do código são invisíveis e não-verificadas. Sem testes que exercitem os caminhos críticos, cada correção é um palpite — funciona aqui, quebra ali.

**Onboarding Hell** aparece quando um novo desenvolvedor leva semanas para ser produtivo. Não porque o domínio é complexo, mas porque o código não tem padrões consistentes, não tem testes que sirvam como documentação executável, e não tem fronteiras claras entre módulos.

### O que salvaguardas proporcionam

**Confiança para mudar.** Com cobertura de testes adequada e análise estática configurada, refatorar deixa de ser arriscado. Se os testes passam e o PHPStan não reclama, a mudança provavelmente está correta.

**Velocidade sustentável.** Sem salvaguardas, a velocidade inicial é alta mas decai exponencialmente à medida que o código acumula complexidade acidental. Com salvaguardas, a velocidade inicial é um pouco menor (custa escrever testes e configurar ferramentas), mas se mantém estável ao longo de meses e anos.

**Qualidade em produção.** Bugs que passariam despercebidos são capturados por testes automatizados no CI antes de chegar ao deploy. O custo de corrigir um bug no desenvolvimento é ordens de magnitude menor do que corrigir em produção.

**Onboarding eficiente.** Testes servem como especificação executável: um novo desenvolvedor pode ler os testes de feature para entender o que o sistema faz, sem depender de documentação que pode estar desatualizada.

---

## Testes Automatizados

### A pirâmide de testes

A distribuição ideal de testes segue uma pirâmide: muitos testes unitários (rápidos, isolados), um número menor de testes de integração (verificam a interação entre componentes), e poucos testes end-to-end (lentos, frágeis, mas abrangentes).

```
        ╱╲
       ╱  ╲         E2E Tests (poucos)
      ╱    ╲        Lentos, frágeis, abrangentes
     ╱──────╲
    ╱        ╲      Integration Tests (alguns)
   ╱          ╲     Banco de dados, APIs, filas
  ╱────────────╲
 ╱              ╲   Unit Tests (muitos)
╱                ╲  Rápidos, isolados, específicos
╱──────────────────╲
```

Na prática com Laravel, a maioria dos testes serão **Feature tests** (que o Laravel chama assim, mas são integration tests: usam banco, fazem requisições HTTP, verificam o fluxo completo) e **Unit tests** (para Value Objects, cálculos, e lógica pura).

### Unit Tests

Testam uma unidade isolada — uma classe, um método, uma função pura. São rápidos porque não dependem de banco, filesystem ou rede.

```php
// tests/Unit/ValueObjects/MoneyTest.php
use App\ValueObjects\Money;
use PHPUnit\Framework\TestCase;

class MoneyTest extends TestCase
{
    /** @test */
    public function soma_valores_da_mesma_moeda(): void
    {
        $a = Money::fromFloat(10.50);
        $b = Money::fromFloat(5.25);

        $resultado = $a->add($b);

        $this->assertEquals(1575, $resultado->amount);
        $this->assertEquals('BRL', $resultado->currency);
    }

    /** @test */
    public function rejeita_soma_de_moedas_diferentes(): void
    {
        $this->expectException(InvalidArgumentException::class);

        $brl = new Money(1000, 'BRL');
        $usd = new Money(1000, 'USD');

        $brl->add($usd);
    }

    /** @test */
    public function formata_corretamente_em_reais(): void
    {
        $money = Money::fromFloat(1234.56);

        $this->assertEquals('R$ 1.234,56', $money->formatted());
    }

    /** @test */
    public function nao_aceita_valor_negativo(): void
    {
        $this->expectException(InvalidArgumentException::class);

        new Money(-100, 'BRL');
    }
}
```

Note que `MoneyTest` estende `PHPUnit\Framework\TestCase`, não `Tests\TestCase` do Laravel. Isso garante que nenhuma dependência de framework é carregada — o teste roda em milissegundos.

### Feature Tests

Testam o fluxo completo: da requisição HTTP à resposta, passando por validação, lógica de negócio e persistência.

```php
// tests/Feature/Orders/CreateOrderTest.php
use App\Models\Order;
use App\Models\Product;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

class CreateOrderTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function usuario_autenticado_cria_pedido_com_sucesso(): void
    {
        $user = User::factory()->create();
        $product = Product::factory()->create([
            'price' => 99.90,
            'stock' => 10,
        ]);

        $response = $this->actingAs($user)->postJson('/api/v1/orders', [
            'items' => [
                ['product_id' => $product->id, 'quantity' => 2],
            ],
        ]);

        $response
            ->assertCreated()
            ->assertJsonStructure([
                'data' => ['id', 'status', 'total', 'items'],
            ]);

        $this->assertDatabaseHas('orders', [
            'user_id' => $user->id,
            'status' => 'pending',
        ]);
    }

    /** @test */
    public function visitante_nao_pode_criar_pedido(): void
    {
        $response = $this->postJson('/api/v1/orders', [
            'items' => [['product_id' => 1, 'quantity' => 1]],
        ]);

        $response->assertUnauthorized();
    }

    /** @test */
    public function rejeita_pedido_quando_estoque_insuficiente(): void
    {
        $user = User::factory()->create();
        $product = Product::factory()->create(['stock' => 5]);

        $response = $this->actingAs($user)->postJson('/api/v1/orders', [
            'items' => [
                ['product_id' => $product->id, 'quantity' => 10],
            ],
        ]);

        $response->assertStatus(409);
        $this->assertDatabaseCount('orders', 0);
    }

    /** @test */
    public function valida_campos_obrigatorios(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)->postJson('/api/v1/orders', []);

        $response
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['items']);
    }
}
```

Cada teste verifica um cenário específico: o caminho feliz, a autenticação, uma regra de negócio, e a validação de input. Juntos, eles formam uma especificação executável do endpoint.

### Test Driven Development (TDD)

O ciclo é curto: escreva um teste que falha (Red), implemente o mínimo para que passe (Green), refatore mantendo os testes verdes (Refactor).

```php
// 1. RED — escrever teste que falha
/** @test */
public function pedido_pendente_pode_ser_cancelado(): void
{
    $order = Order::factory()->create(['status' => 'pending']);

    $order->cancel();

    $this->assertEquals('cancelled', $order->fresh()->status);
}

// 2. GREEN — implementar o mínimo
class Order extends Model
{
    public function cancel(): void
    {
        $this->update(['status' => 'cancelled']);
    }
}

// 3. REFACTOR — adicionar validação e evento
class Order extends Model
{
    public function cancel(): void
    {
        if (! $this->canBeCancelled()) {
            throw new OrderCannotBeCancelledException($this);
        }

        $this->update([
            'status' => OrderStatus::CANCELLED,
            'cancelled_at' => now(),
        ]);

        event(new OrderCancelled($this));
    }

    public function canBeCancelled(): bool
    {
        return $this->status === OrderStatus::PENDING;
    }
}

// 4. Adicionar teste para o caso negativo
/** @test */
public function pedido_enviado_nao_pode_ser_cancelado(): void
{
    $this->expectException(OrderCannotBeCancelledException::class);

    $order = Order::factory()->create(['status' => 'shipped']);

    $order->cancel();
}
```

TDD não é sobre testar — é sobre design. Escrever o teste primeiro força você a pensar na interface antes da implementação: como o consumidor vai chamar esse código? Que dados precisa? Que erros pode gerar?

### Pest como alternativa ao PHPUnit

Pest oferece uma sintaxe mais expressiva e menos verbosa. Se você está começando um projeto novo, vale considerar:

```php
// tests/Pest.php — configuração global
uses(RefreshDatabase::class)->in('Feature');

// tests/Feature/OrderTest.php
test('usuario autenticado cria pedido', function () {
    $user = User::factory()->create();
    $product = Product::factory()->create();

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

Pest brilha com **datasets** — quando o mesmo teste precisa rodar com múltiplas combinações de entrada:

```php
test('valida transicoes de status', function (string $from, string $to, bool $valido) {
    $order = Order::factory()->create(['status' => $from]);

    if ($valido) {
        expect(fn () => $order->transitionTo(OrderStatus::from($to)))
            ->not->toThrow(Exception::class);
    } else {
        expect(fn () => $order->transitionTo(OrderStatus::from($to)))
            ->toThrow(InvalidStatusTransition::class);
    }
})->with([
    'pending → confirmed'  => ['pending', 'confirmed', true],
    'pending → shipped'    => ['pending', 'shipped', false],
    'confirmed → shipped'  => ['confirmed', 'shipped', true],
    'shipped → pending'    => ['shipped', 'pending', false],
    'delivered → cancelled' => ['delivered', 'cancelled', false],
]);
```

Um único teste com dataset substitui cinco testes individuais, e a saída mostra exatamente qual combinação falhou.

### Cobertura de código

Cobertura mede quais linhas de código são exercitadas pelos testes. Não é uma métrica de qualidade perfeita (código coberto pode ter testes ruins), mas é um bom indicador de riscos: código sem cobertura é código que pode quebrar sem que ninguém perceba.

```bash
# Gerar relatório de cobertura
./vendor/bin/pest --coverage

# Falhar se a cobertura for menor que 80%
./vendor/bin/pest --coverage --min=80
```

A meta de cobertura depende do contexto. 80% é um bom alvo geral: cobre os caminhos críticos sem exigir testes para getters triviais. Para código financeiro ou de saúde, 95%+ pode ser justificável.

---

## Análise Estática

Análise estática examina o código sem executá-lo, encontrando bugs que só apareceriam em runtime: acesso a propriedades inexistentes, tipos incompatíveis, variáveis que podem ser null quando o código assume que não são.

### PHPStan / Larastan

Larastan é o PHPStan com suporte específico para Laravel (entende Eloquent, facades, relationships).

```bash
# Instalar
composer require --dev "nunomaduro/larastan:^3.0"
```

```yaml
# phpstan.neon
includes:
    - vendor/nunomaduro/larastan/extension.neon

parameters:
    paths:
        - app
    level: 5
```

```bash
# Executar
./vendor/bin/phpstan analyse
```

#### O que o PHPStan encontra

```php
// ❌ Typo em nome de propriedade — runtime error que o PHPStan captura
class OrderService
{
    public function getTotal(Order $order): float
    {
        return $order->totla; // PHPStan: Property 'totla' not found
    }
}

// ❌ Tipo de retorno incorreto — find() pode retornar null
class ProductRepository
{
    public function find(int $id): Product
    {
        return Product::find($id);
        // PHPStan: Method find() returns Product|null, but Product expected
    }
}

// ✅ Corrigido — tipo de retorno reflete a realidade
class ProductRepository
{
    public function find(int $id): ?Product
    {
        return Product::find($id);
    }

    public function findOrFail(int $id): Product
    {
        return Product::findOrFail($id);
    }
}
```

#### Níveis de strictness

O PHPStan tem 10 níveis (0-9). Cada nível adiciona verificações mais rigorosas. A estratégia recomendada é começar no nível mais baixo que funciona, corrigir todos os erros, e subir gradualmente.

```php
// Level 0-2: básico — tipos de retorno faltando, variáveis undefined
class User
{
    public function getName()  // Sem return type
    {
        return $this->name;
    }
}

// Level 3-5: médio (recomendado como meta) — nullability, dead code
class User
{
    public function getName(): string
    {
        return $this->name;
    }
}

// Level 6-9: estrito — union types, generics, never-returning functions
class User
{
    /** @return non-empty-string */
    public function getName(): string
    {
        if ($this->name === '') {
            throw new RuntimeException('Nome vazio');
        }
        return $this->name;
    }
}
```

Level 5 é um bom alvo para projetos Laravel. Acima disso, o esforço de satisfazer o PHPStan cresce desproporcionalmente ao benefício, especialmente por causa de limitações na tipagem do Eloquent.

---

## Code Quality Tools

### Laravel Pint

Pint formata o código automaticamente segundo um padrão definido. Elimina discussões sobre estilo (tabs vs espaços, posição de chaves, espaçamento) — o Pint decide, e todo o time segue.

```bash
# Já vem incluído no Laravel
./vendor/bin/pint

# Verificar sem modificar (útil no CI)
./vendor/bin/pint --test
```

```json
// pint.json — customizar regras
{
    "preset": "laravel",
    "rules": {
        "simplified_null_return": true,
        "ordered_imports": {
            "sort_algorithm": "alpha"
        }
    }
}
```

No CI, use `--test` para falhar o pipeline se o código não estiver formatado. Localmente, configure seu editor para rodar Pint ao salvar — assim o código sempre está formatado e o CI nunca reclama.

### PHP Insights

Analisa qualidade de código em quatro dimensões: complexidade, arquitetura, estilo e segurança. Útil como "check-up geral" periódico.

```bash
composer require --dev nunomaduro/phpinsights

php artisan insights
```

O output mostra uma pontuação de 0-100 em cada dimensão e lista os problemas encontrados com sugestões de correção. Não é para rodar no CI a cada commit (é lento), mas vale rodar semanalmente ou antes de releases para identificar tendências de deterioração.

### Rector

Rector faz refatorações automatizadas: atualiza sintaxe para versões mais recentes do PHP, aplica padrões de código, e remove código morto.

```bash
composer require --dev rector/rector
```

```php
// rector.php
use Rector\Config\RectorConfig;
use Rector\Set\ValueObject\LevelSetList;

return RectorConfig::configure()
    ->withPaths([__DIR__ . '/app'])
    ->withSets([
        LevelSetList::UP_TO_PHP_83,  // Atualizar para sintaxe PHP 8.3
    ]);
```

```bash
# Ver o que mudaria (dry run)
vendor/bin/rector process --dry-run

# Aplicar as mudanças
vendor/bin/rector process
```

Rector é especialmente útil quando você atualiza a versão do PHP: ele converte automaticamente a codebase para usar as novas features (readonly properties, enums, match expressions, etc.).

---

## SOLID Principles

SOLID não é um conjunto de regras rígidas — é um vocabulário para discutir qualidade de design. Os cinco princípios se complementam e, aplicados com pragmatismo, produzem código que é fácil de entender, testar e modificar.

### S — Single Responsibility Principle

Uma classe deve ter apenas um motivo para mudar. Se você precisa alterá-la quando a regra de desconto muda *e* quando o formato do email muda, ela tem responsabilidades demais.

```php
// ❌ Quatro motivos para mudar: cálculo, persistência, notificação, geração de PDF
class Order extends Model
{
    public function calculateTotal(): float { /* ... */ }
    public function save(): bool { /* ... */ }
    public function sendConfirmationEmail(): void { /* ... */ }
    public function generatePDF(): string { /* ... */ }
}

// ✅ Cada classe tem um único motivo para existir
class Order extends Model
{
    public function total(): float { /* ... */ }
}

class OrderRepository
{
    public function save(Order $order): void { /* ... */ }
}

class OrderNotifier
{
    public function sendConfirmation(Order $order): void { /* ... */ }
}

class OrderPdfGenerator
{
    public function generate(Order $order): string { /* ... */ }
}
```

### O — Open/Closed Principle

O sistema deve ser extensível sem precisar modificar código existente. Na prática, isso se traduz em usar polimorfismo ao invés de condicionais crescentes.

```php
// ❌ Adicionar novo método de pagamento exige modificar esta classe
class PaymentProcessor
{
    public function process(string $type, float $amount): Payment
    {
        if ($type === 'credit_card') {
            // 20 linhas
        } elseif ($type === 'pix') {
            // 15 linhas
        } elseif ($type === 'boleto') {
            // 25 linhas
        }
        // Novo tipo = mais um elseif aqui
    }
}

// ✅ Novo método de pagamento = nova classe, sem tocar nas existentes
interface PaymentMethodInterface
{
    public function process(float $amount): Payment;
}

class CreditCardPayment implements PaymentMethodInterface
{
    public function process(float $amount): Payment { /* ... */ }
}

class PixPayment implements PaymentMethodInterface
{
    public function process(float $amount): Payment { /* ... */ }
}

class PaymentProcessor
{
    public function process(PaymentMethodInterface $method, float $amount): Payment
    {
        return $method->process($amount);
    }
}
```

### L — Liskov Substitution Principle

Se uma classe B herda de A, substituir A por B em qualquer lugar do código não deve quebrar o comportamento. Na prática, o exemplo mais útil é garantir que implementações de interfaces respeitam o contrato completo:

```php
interface CacheInterface
{
    /** @return mixed|null Retorna o valor ou null se não existir */
    public function get(string $key): mixed;
    public function set(string $key, mixed $value, int $ttl): void;
}

// ✅ Ambas as implementações respeitam o contrato
class RedisCache implements CacheInterface
{
    public function get(string $key): mixed
    {
        return Redis::get($key);  // Retorna valor ou null
    }

    public function set(string $key, mixed $value, int $ttl): void
    {
        Redis::setex($key, $ttl, serialize($value));
    }
}

class ArrayCache implements CacheInterface
{
    private array $store = [];

    public function get(string $key): mixed
    {
        return $this->store[$key] ?? null;  // Mesmo contrato
    }

    public function set(string $key, mixed $value, int $ttl): void
    {
        $this->store[$key] = $value;
    }
}

// Substituir RedisCache por ArrayCache (em testes, por exemplo)
// não quebra nenhum consumidor, porque o contrato é o mesmo.
```

### I — Interface Segregation Principle

Interfaces devem ser específicas. Um consumidor não deveria ser forçado a depender de métodos que não usa.

```php
// ❌ Interface gorda — Robot não come nem dorme
interface WorkerInterface
{
    public function work(): void;
    public function eat(): void;
    public function sleep(): void;
}

class RobotWorker implements WorkerInterface
{
    public function work(): void { /* ok */ }
    public function eat(): void { /* ??? */ }   // Robô não come
    public function sleep(): void { /* ??? */ }  // Robô não dorme
}

// ✅ Interfaces segregadas — cada consumidor implementa apenas o que faz sentido
interface Workable
{
    public function work(): void;
}

interface NeedsRest
{
    public function eat(): void;
    public function sleep(): void;
}

class HumanWorker implements Workable, NeedsRest
{
    public function work(): void { /* ... */ }
    public function eat(): void { /* ... */ }
    public function sleep(): void { /* ... */ }
}

class RobotWorker implements Workable
{
    public function work(): void { /* ... */ }
}
```

### D — Dependency Inversion Principle

Já discutido em profundidade nos documentos anteriores (Architecture Decisions, Business Logic). O resumo: dependa de abstrações, não de implementações concretas. O Service Provider do Laravel é o mecanismo que conecta interfaces às implementações.

```php
// Domínio define O QUE precisa (interface)
interface OrderRepositoryInterface
{
    public function save(Order $order): void;
}

// Infraestrutura implementa COMO (implementação concreta)
class EloquentOrderRepository implements OrderRepositoryInterface
{
    public function save(Order $order): void
    {
        $order->save();
    }
}

// Service Provider conecta
$this->app->bind(OrderRepositoryInterface::class, EloquentOrderRepository::class);
```

---

## Design Patterns

Patterns não são receitas para copiar — são soluções nomeadas para problemas recorrentes. Conhecê-los permite reconhecer quando um problema já foi resolvido antes e comunicar a solução usando vocabulário compartilhado.

### Factory

Encapsula a lógica de criação de objetos. Útil quando a decisão de qual classe instanciar depende de um parâmetro.

```php
class PaymentFactory
{
    public static function create(string $type): PaymentMethodInterface
    {
        return match ($type) {
            'credit_card' => app(CreditCardPayment::class),
            'pix' => app(PixPayment::class),
            'boleto' => app(BoletoPayment::class),
            default => throw new InvalidPaymentTypeException($type),
        };
    }
}

// Uso
$payment = PaymentFactory::create($request->input('payment_type'));
$result = $payment->process($order->total);
```

Adicionar um novo tipo de pagamento requer apenas adicionar uma linha no `match` e criar a nova classe. Nenhum código existente é modificado (Open/Closed Principle).

### Strategy

Permite trocar algoritmos em runtime. Útil quando diferentes variações de um comportamento precisam coexistir.

```php
interface ShippingStrategy
{
    public function calculate(Order $order): Money;
}

class StandardShipping implements ShippingStrategy
{
    public function calculate(Order $order): Money
    {
        return Money::fromFloat(15.00);
    }
}

class ExpressShipping implements ShippingStrategy
{
    public function calculate(Order $order): Money
    {
        $base = 25.00;
        $perKg = $order->totalWeight() * 2.50;

        return Money::fromFloat($base + $perKg);
    }
}

class FreeShipping implements ShippingStrategy
{
    public function calculate(Order $order): Money
    {
        return new Money(0, 'BRL');
    }
}
```

```php
class ShippingCalculator
{
    public function calculate(Order $order): Money
    {
        $strategy = $this->resolveStrategy($order);

        return $strategy->calculate($order);
    }

    private function resolveStrategy(Order $order): ShippingStrategy
    {
        if ($order->total()->amount >= 20000) {  // Acima de R$ 200
            return new FreeShipping();
        }

        return $order->isExpress()
            ? new ExpressShipping()
            : new StandardShipping();
    }
}
```

A lógica de **qual** estratégia usar está separada da lógica de **como** calcular o frete. Cada responsabilidade pode mudar independentemente.

### Observer

O sistema de eventos do Laravel é uma implementação do Observer pattern. Os documentos anteriores já cobriram isso em detalhes (ver Business Logic → Domain Events). O ponto-chave: o emissor do evento não conhece os observadores, e novos observadores podem ser adicionados sem modificar o emissor.

### Decorator

Adiciona comportamento a um objeto existente sem modificar sua classe. Útil para funcionalidades transversais como cache, logging ou retry.

```php
interface ProductRepositoryInterface
{
    public function find(int $id): ?Product;
    public function findAll(): Collection;
}

class EloquentProductRepository implements ProductRepositoryInterface
{
    public function find(int $id): ?Product
    {
        return Product::find($id);
    }

    public function findAll(): Collection
    {
        return Product::all();
    }
}

// Decorator que adiciona cache sem modificar a classe original
class CachedProductRepository implements ProductRepositoryInterface
{
    public function __construct(
        private ProductRepositoryInterface $inner,
        private CacheInterface $cache,
    ) {}

    public function find(int $id): ?Product
    {
        return $this->cache->remember(
            "product.{$id}",
            3600,
            fn () => $this->inner->find($id)
        );
    }

    public function findAll(): Collection
    {
        return $this->cache->remember(
            'products.all',
            3600,
            fn () => $this->inner->findAll()
        );
    }
}
```

```php
// Service Provider — a composição define o comportamento
$this->app->bind(ProductRepositoryInterface::class, function ($app) {
    return new CachedProductRepository(
        inner: new EloquentProductRepository(),
        cache: $app->make(CacheInterface::class),
    );
});
```

O `EloquentProductRepository` não sabe que está sendo cacheado. O `CachedProductRepository` não sabe nada sobre Eloquent. Cada classe tem uma responsabilidade. Se amanhã você quiser adicionar logging, crie um `LoggedProductRepository` e empilhe mais um decorator.

---

## Refactoring

Refatorar é melhorar a estrutura interna do código sem mudar seu comportamento externo. A pré-condição é ter testes: sem eles, você não pode garantir que a refatoração não quebrou nada.

### Extract Method

O refactoring mais comum e mais impactante. Quando um método faz várias coisas, extraia cada etapa para um método com nome descritivo.

```php
// ❌ Método longo que faz tudo
public function process(Order $order): void
{
    if ($order->items->isEmpty()) {
        throw new EmptyOrderException();
    }

    $total = 0;
    foreach ($order->items as $item) {
        $total += $item->price * $item->quantity;
    }
    $order->total = $total;

    if ($order->customer->isPremium()) {
        $order->total *= 0.9;
    }

    $order->save();
}

// ✅ Cada etapa tem nome e propósito claro
public function process(Order $order): void
{
    $this->ensureNotEmpty($order);
    $this->calculateTotal($order);
    $this->applyPremiumDiscount($order);
    $order->save();
}

private function ensureNotEmpty(Order $order): void
{
    if ($order->items->isEmpty()) {
        throw new EmptyOrderException();
    }
}

private function calculateTotal(Order $order): void
{
    $order->total = $order->items->sum(
        fn ($item) => $item->price * $item->quantity
    );
}

private function applyPremiumDiscount(Order $order): void
{
    if ($order->customer->isPremium()) {
        $order->total *= 0.9;
    }
}
```

O método principal agora lê como uma receita: garantir que não está vazio, calcular total, aplicar desconto, salvar. Os detalhes de *como* cada etapa funciona estão nos métodos privados.

### Replace Conditional with Polymorphism

Quando um `if/elseif/switch` cresce ao longo do tempo (cada novo tipo de pagamento adiciona um novo bloco), substitua por polimorfismo. Já demonstrado na seção de SOLID (Open/Closed Principle).

### Introduce Parameter Object

Quando um método recebe muitos parâmetros, agrupe-os em um DTO:

```php
// ❌ 7 parâmetros — difícil de ler, fácil de errar a ordem
public function createUser(
    string $name,
    string $email,
    string $password,
    ?string $phone,
    ?string $address,
    ?string $city,
    ?string $state,
): User {
    // ...
}

// ✅ DTO agrupa os dados com nomes claros
final readonly class CreateUserData
{
    public function __construct(
        public string $name,
        public string $email,
        public string $password,
        public ?string $phone = null,
        public ?string $address = null,
        public ?string $city = null,
        public ?string $state = null,
    ) {}
}

public function createUser(CreateUserData $data): User
{
    // Acessa $data->name, $data->email, etc.
}
```

Named arguments no PHP 8+ tornam a criação do DTO expressiva: `new CreateUserData(name: 'Maria', email: 'maria@email.com', password: 'senha')`. Parâmetros opcionais com default não precisam ser passados.

---

## Technical Debt

Dívida técnica é a diferença entre como o código *está* e como ele *deveria estar*. Como dívida financeira, ela acumula juros: quanto mais tempo passa sem ser paga, mais caro fica corrigi-la.

### Reconhecer os sinais

TODOs e HACKs acumulados são o sinal mais visível, mas não o único:

```php
// Sinal 1: TODOs que nunca são resolvidos
class OrderService
{
    public function process(): void
    {
        // TODO: Adicionar validação (há 8 meses)
        // TODO: Implementar retry (há 6 meses)
        // TODO: Tratar erros (há 3 meses)
    }
}

// Sinal 2: Código comentado que ninguém remove
class PaymentService
{
    public function charge(): Payment
    {
        // $legacyResult = $this->oldGateway->charge($amount);
        // if ($legacyResult->failed()) {
        //     Log::error('Legacy payment failed');
        //     return $legacyResult;
        // }

        return $this->newGateway->charge($amount);
    }
}

// Sinal 3: Workarounds com prazo de validade que nunca expira
class UserController
{
    public function store(): void
    {
        // HACK: sleep temporário até resolver race condition (2023-03-15)
        sleep(1);
    }
}
```

Sinais menos visíveis: testes que foram desativados "temporariamente", classes que cresceram além de 500 linhas, dependências circulares que "funcionam por enquanto", e regras de negócio duplicadas em múltiplos lugares.

### Registrar dívida técnica com ADRs

Nem toda dívida técnica precisa ser paga imediatamente. Às vezes o atalho é a decisão correta dado o prazo. O importante é **registrar** a decisão e o custo estimado de corrigi-la depois:

```markdown
# ADR-007: Usar cache manual ao invés de cache tag

## Status: Aceito (dívida técnica)

## Contexto
O driver de cache atual (file) não suporta tags.
Precisamos invalidar cache de produtos quando preço muda.

## Decisão
Invalidar manualmente cada chave de cache afetada.

## Dívida
Quando migrarmos para Redis (planejado para Q2), substituir
por cache tags que invalidam automaticamente.

## Custo estimado de correção
2-4 horas de trabalho quando Redis estiver configurado.
```

### Pagar dívida: Boy Scout Rule

"Deixe o código melhor do que encontrou." A cada mudança em um arquivo, aplique uma melhoria pequena: renomeie uma variável confusa, extraia um método, remova código comentado, adicione um type hint. Melhorias incrementais são mais sustentáveis e menos arriscadas do que refatorações massivas.

```php
// Antes de adicionar a nova feature, refatore o mínimo necessário
// ANTES
class OrderController extends Controller
{
    public function store(Request $request)
    {
        // 150 linhas de lógica misturada
    }
}

// DEPOIS (refatoração + feature nova)
class OrderController extends Controller
{
    public function store(
        StoreOrderRequest $request,
        CreateOrderAction $action,
    ) {
        $order = $action->execute($request->validated());

        return new OrderResource($order);
    }
}
```

A feature nova é adicionada na Action (que é o lugar correto), e o controller gordinho é curado no processo. Duas melhorias pelo preço de uma.

---

## Documentation

A melhor documentação é aquela que não pode ficar desatualizada. Testes são documentação que o CI verifica a cada commit. Types e PHPDoc são documentação que o IDE usa em tempo real. ADRs são documentação de decisões que raramente mudam.

### PHPDoc para contratos de API

Use PHPDoc quando o sistema de tipos do PHP não é expressivo o suficiente para comunicar a intenção:

```php
/**
 * Cria um novo pedido a partir dos dados validados.
 *
 * @param  array{
 *     customer_id: int,
 *     items: array<int, array{product_id: int, quantity: int}>
 * } $data
 *
 * @throws InsufficientStockException Quando um produto não tem estoque suficiente
 * @throws CustomerSuspendedException Quando o cliente está bloqueado
 */
public function createOrder(array $data): Order
{
    // ...
}
```

Documente **exceções** que o método pode lançar — isso é informação que o sistema de tipos não captura. Documente **formatos de array** quando receber arrays associativos — PHPStan usa essas anotações para validação estática.

Não documente o óbvio. Se o método se chama `calculateTotal` e retorna `float`, um `@return float Retorna o total` não adiciona informação — é ruído.

### README.md

O README é a porta de entrada do projeto. Deve responder três perguntas em 30 segundos: o que é o projeto, como instalar, como rodar.

```markdown
# Sistema de Gestão Municipal

Sistema para gerenciamento de processos administrativos
(licitações, requerimentos, protocolos) da Prefeitura de Itagi.

## Stack

PHP 8.3, Laravel 11, Vue.js 3, Inertia.js, PostgreSQL 16, Tailwind CSS

## Setup

    composer install
    npm install
    cp .env.example .env
    php artisan key:generate
    php artisan migrate --seed
    npm run dev

## Testes

    php artisan test                    # Todos os testes
    php artisan test --filter=OrderTest # Teste específico
    ./vendor/bin/phpstan analyse        # Análise estática

## Estrutura

    app/
    ├── Domain/         Regras de negócio (Models, Value Objects, Enums)
    ├── Application/    Casos de uso (Actions, Services, DTOs)
    ├── Infrastructure/ Implementações técnicas (Repositories, APIs)
    └── Http/           Interface HTTP (Controllers, Requests, Resources)
```

Não documente o que muda frequentemente (lista de features, estado atual do backlog). Documente o que é estável: instalação, estrutura, convenções de código.

### Código autoexplicativo

A documentação mais eficaz é código que não precisa de documentação. Nomes descritivos, funções pequenas e tipos explícitos comunicam a intenção sem comentários:

```php
// ❌ Comentário compensa nome ruim
/** Verifica se o pedido pode ser cancelado */
public function check(): bool { /* ... */ }

// ✅ O nome já diz tudo
public function canBeCancelled(): bool { /* ... */ }

// ❌ Comentário repete o código
// Aplica desconto de 10% para clientes premium
$total = $subtotal * 0.9;

// ✅ Constante com nome descritivo
$total = $subtotal * (1 - self::PREMIUM_DISCOUNT_RATE);
```

---

## Code Review

Code review não é só encontrar bugs — é uma ferramenta de qualidade, compartilhamento de conhecimento e mentoria. Um bom review verifica se o código está correto, legível, testado e alinhado com a arquitetura do projeto.

### O que verificar

**Funcionalidade.** O código faz o que deveria? Os edge cases estão tratados? O comportamento está coberto por testes?

**Design.** As responsabilidades estão bem distribuídas? A lógica está na camada certa? As dependências fazem sentido? O código segue os padrões do projeto?

**Segurança.** O input está validado? Dados sensíveis estão protegidos? Queries são parametrizadas? Endpoints têm autenticação e autorização?

**Performance.** Há queries N+1? Cache está sendo usado onde faz sentido? Loops desnecessários? Eager loading onde necessário?

**Legibilidade.** Um desenvolvedor que não escreveu o código consegue entendê-lo? Nomes são descritivos? A complexidade é proporcional ao problema?

### Dar feedback eficaz

Feedback de code review deve ser específico, construtivo e focado no código — não na pessoa.

```
❌ "Isso está errado."
✅ "Esse método pode retornar null quando o produto não existe.
    Considere usar findOrFail() ou tratar o caso null."

❌ "Não faça assim."
✅ "A validação de estoque está no controller. No nosso padrão,
    regras de negócio vivem na Action — o controller apenas delega.
    Sugestão: mover para CreateOrderAction."

❌ "Falta teste."
✅ "O cenário de estoque insuficiente não está coberto por teste.
    Seria bom adicionar um test case para garantir que a exceção
    InsufficientStockException é lançada corretamente."
```

Diferencie entre bloqueadores (precisam ser corrigidos antes do merge) e sugestões (melhorias opcionais que podem ser feitas depois). Evite nitpicking sobre estilo — o Pint cuida disso automaticamente.

---

## Anti-patterns Comuns

### 1. Copy-Paste Programming

Copiar e colar é a forma mais rápida de criar duplicação. Quando a regra muda, é necessário encontrar e atualizar todas as cópias — e esquecer uma delas é o cenário mais comum.

```php
// ❌ Mesma lógica duplicada em dois métodos
public function cancelOrder(int $orderId): void
{
    $order = Order::findOrFail($orderId);
    $order->status = 'cancelled';
    $order->cancelled_at = now();
    $order->save();
    Mail::to($order->customer)->send(new CancellationMail($order));
}

public function cancelSubscription(int $subscriptionId): void
{
    $subscription = Subscription::findOrFail($subscriptionId);
    $subscription->status = 'cancelled';
    $subscription->cancelled_at = now();
    $subscription->save();
    Mail::to($subscription->customer)->send(new CancellationMail($subscription));
}

// ✅ Comportamento compartilhado via trait ou método comum
trait Cancellable
{
    public function cancel(): void
    {
        $this->update([
            'status' => 'cancelled',
            'cancelled_at' => now(),
        ]);

        event(new ResourceCancelled($this));
    }
}
```

### 2. Magic Numbers e Strings

Valores literais no código sem explicação do que representam.

```php
// ❌ O que significa 18? O que significa 0.9?
if ($user->age > 18) { /* ... */ }
$total = $subtotal * 0.9;

// ✅ Constantes com nomes descritivos
class AgeRequirement
{
    public const MINIMUM_ADULT = 18;
}

class Discount
{
    public const PREMIUM_RATE = 0.10;
}

if ($user->age > AgeRequirement::MINIMUM_ADULT) { /* ... */ }
$total = $subtotal * (1 - Discount::PREMIUM_RATE);
```

### 3. Shotgun Surgery

Quando uma mudança conceitual (ex: "mudar a fórmula de desconto") exige alterar dezenas de arquivos espalhados pelo sistema. É o resultado direto de lógica de negócio duplicada ou distribuída em camadas erradas.

A cura: centralizar cada regra de negócio em um único lugar (Action, Service, ou Value Object). Se mudar a fórmula de desconto exige tocar em mais de um arquivo, a regra está fragmentada.

### 4. Lava Flow

Código que ninguém entende, ninguém usa, mas ninguém tem coragem de remover — porque "talvez seja importante". Classes, métodos, rotas e migrations que sobrevivem indefinidamente por medo de quebrar algo.

A cura: cobertura de testes que dá confiança para deletar. Se os testes passam sem aquele código, ele é seguro de remover. Ferramentas como o PHPStan em níveis mais altos também detectam código morto.

---

## Boas Práticas

### 1. Automatize a verificação de qualidade

Não dependa de disciplina humana. Configure um pipeline de CI que verifica tudo a cada push:

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          coverage: xdebug

      - name: Install dependencies
        run: composer install --no-interaction

      - name: Code style
        run: ./vendor/bin/pint --test

      - name: Static analysis
        run: ./vendor/bin/phpstan analyse

      - name: Tests
        run: php artisan test --coverage --min=80
```

Se qualquer etapa falhar, o pipeline bloqueia o merge. Código que não está formatado, que tem erros de tipo, ou que quebra testes existentes simplesmente não entra no main.

### 2. Pre-commit hooks como rede de segurança local

Antes mesmo de chegar ao CI, hooks locais capturam problemas no momento do commit:

```bash
#!/bin/bash
# .git/hooks/pre-commit

echo "Running Pint..."
./vendor/bin/pint --test --quiet || {
    echo "Code style failed. Run: ./vendor/bin/pint"
    exit 1
}

echo "Running PHPStan..."
./vendor/bin/phpstan analyse --no-progress --quiet || {
    echo "Static analysis failed."
    exit 1
}

echo "Running tests..."
php artisan test --parallel --no-interaction || {
    echo "Tests failed."
    exit 1
}

echo "All checks passed."
```

### 3. Keep It Simple

Overengineering é tão prejudicial quanto código sem estrutura. Se a solução mais simples resolve o problema, use-a:

```php
// ❌ Abstração prematura — Factory → Builder → Strategy → Validator → Chain
class OrderFactory
{
    public function create(OrderDTO $dto, OrderSpecification $spec): Order
    {
        return $this->builder
            ->withStrategy($this->strategyFactory->create($spec))
            ->withValidator($this->validatorChain->build())
            ->build($dto);
    }
}

// ✅ Se o CRUD é simples, a implementação deve ser simples
class CreateOrderAction
{
    public function execute(array $data): Order
    {
        return DB::transaction(function () use ($data) {
            $order = Order::create($data);

            foreach ($data['items'] as $item) {
                $order->items()->create($item);
            }

            event(new OrderCreated($order));

            return $order;
        });
    }
}
```

Adicione abstrações quando a complexidade real justificar — não como prevenção especulativa de problemas que podem nunca surgir.

### 4. Melhoria contínua, não revolução

Reescritas totais raramente dão certo. Melhorias incrementais — um controller refatorado por sprint, um módulo extraído por mês, cobertura de testes subindo 5% por semana — são mais sustentáveis e menos arriscadas.

Mantenha um backlog de dívida técnica priorizado junto com o backlog de features. Em cada sprint, reserve tempo para pagar pelo menos um item de dívida técnica — mesmo que pequeno.

---

## Exercícios Práticos

### Exercício 1: Adicionar testes a um módulo existente

Escolha o módulo mais crítico do seu sistema (ex: criação de requerimentos, processamento de licitações) e escreva testes que cubram: o caminho feliz (criação bem-sucedida), validação de campos obrigatórios, autenticação e autorização, pelo menos dois edge cases específicos do domínio (estoque insuficiente, documento faltante, status inválido).

Use `assertDatabaseHas`, `assertJsonStructure` e verificação de exceções para garantir que o comportamento está correto.

### Exercício 2: PHPStan Level Up

```bash
# Comece no nível 0 e suba gradualmente
./vendor/bin/phpstan analyse --level=0

# Corrija todos os erros do nível atual antes de subir
# Repita até atingir o nível 5

# Meta: nível 5 sem erros, rodando no CI
```

Dica: os primeiros níveis são rápidos de corrigir (tipos de retorno faltando, variáveis undefined). A partir do nível 4, os erros ficam mais sutis (nullability, dead code). Não pule níveis — cada um constrói sobre o anterior.

### Exercício 3: Refatorar uma classe grande

Identifique a classe mais longa do seu projeto (geralmente um controller ou service com mais de 200 linhas) e aplique:

1. **Extract Method** — quebre métodos longos em métodos privados com nomes descritivos.
2. **Extract Class** — se os métodos extraídos formam grupos coesos, mova-os para classes dedicadas (Action, Service, Value Object).
3. **Adicione testes** — antes de refatorar, escreva testes para o comportamento atual. Depois, verifique que os testes continuam passando.

### Exercício 4: Configurar pipeline de CI

Se seu projeto ainda não tem CI, configure um pipeline mínimo que execute: `pint --test` (formatação), `phpstan analyse` (análise estática), e `php artisan test` (testes). Faça o pipeline rodar a cada push e bloquear merges que falham.

---

## Recursos Adicionais

### Livros

- **Refactoring** — Martin Fowler. O catálogo definitivo de técnicas de refatoração, com motivação, mecânica e exemplos para cada uma. Se você for ler um livro desta lista, que seja este.
- **Clean Code** — Robert C. Martin. Princípios de código legível: nomes descritivos, funções pequenas, formatação consistente. Alguns exemplos estão datados, mas os princípios permanecem válidos.
- **Working Effectively with Legacy Code** — Michael Feathers. Para quando você herda um sistema sem testes e precisa modificá-lo com segurança. Ensina técnicas para adicionar testes a código que não foi projetado para ser testável.

### Ferramentas

- **PHPStan / Larastan** — análise estática com suporte a Laravel.
- **Pest / PHPUnit** — frameworks de teste (Pest para sintaxe moderna, PHPUnit para compatibilidade universal).
- **Laravel Pint** — formatação automática de código.
- **Rector** — refatoração automatizada e atualização de sintaxe.
- **Deptrac** — verificação de dependências entre camadas (previne violações arquiteturais).

### Cursos

- Laracasts: "Pest Driven Laravel" — testes com Pest aplicados a projetos Laravel reais.
- Laracasts: "SOLID Principles in PHP" — os cinco princípios com exemplos práticos.

---

## Conclusão

Prevenir o caos é mais barato do que remediá-lo. Cada hora investida em testes, análise estática e refatoração incremental economiza dias de debugging, correções emergenciais e reescritas futuras.

Os pilares são cinco:

**Testes automatizados.** Sua rede de segurança. Sem testes, toda mudança é um risco. Com testes, refatorar é rotina.

**Análise estática.** Encontra bugs que testes não encontram — typos, tipos incompatíveis, variáveis que podem ser null. Roda em segundos e pega erros antes do runtime.

**Automação no CI.** Verificações de qualidade não podem depender de alguém lembrar de rodar o PHPStan. O pipeline roda tudo automaticamente e bloqueia o que não passa.

**SOLID e refatoração contínua.** Princípios de design que mantêm o código flexível, e a prática de melhorar um pouco a cada mudança (Boy Scout Rule) ao invés de acumular dívida.

**Documentação que não envelhece.** Testes como especificação executável, types como contratos verificáveis, ADRs como registro de decisões. Evite documentação que precisa de manutenção manual.

---

**Com isso, os cinco pilares estão completos:**

1. **State & Data Flow** — como o estado nasce, se transforma e persiste
2. **APIs and System Boundaries** — como sistemas se comunicam através de contratos estáveis
3. **Business Logic** — como isolar e organizar regras de negócio
4. **Architecture Decisions** — como estruturar o sistema para que evolua
5. **Preventing Long-Term Messes** — como garantir que tudo acima se mantenha ao longo do tempo

Os cinco documentos formam um ciclo: a arquitetura define a estrutura, a lógica de negócio habita essa estrutura, as APIs definem as fronteiras, o estado flui entre elas, e as salvaguardas garantem que nada se deteriore. Comece pelo pilar que causa mais dor no seu projeto atual, aplique os conceitos gradualmente, e observe a diferença.
