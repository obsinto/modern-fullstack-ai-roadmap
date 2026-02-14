# Application Security & Roles (Segurança da Aplicação e Controle de Acesso)

## Índice

1. [Introdução](#introdução)
2. [Por que isso importa?](#por-que-isso-importa)
3. [Autenticação vs Autorização](#autenticação-vs-autorização)
4. [Autenticação no Laravel](#autenticação-no-laravel)
5. [Sistema de Papéis e Permissões](#sistema-de-papéis-e-permissões)
6. [Policies e Gates](#policies-e-gates)
7. [Proteção contra IDOR](#proteção-contra-idor)
8. [Sanitização e Validação de Dados](#sanitização-e-validação-de-dados)
9. [Proteção de Dados Sensíveis na API](#proteção-de-dados-sensíveis-na-api)
10. [CSRF, XSS e SQL Injection](#csrf-xss-e-sql-injection)
11. [Rate Limiting e Proteção contra Abuso](#rate-limiting-e-proteção-contra-abuso)
12. [Auditoria e Logging de Segurança](#auditoria-e-logging-de-segurança)
13. [LGPD e Dados de Cidadãos](#lgpd-e-dados-de-cidadãos)
14. [Anti-patterns de Segurança](#anti-patterns-de-segurança)
15. [Checklist de Segurança](#checklist-de-segurança)
16. [Exercícios Práticos](#exercícios-práticos)
17. [Recursos Adicionais](#recursos-adicionais)

---

## Introdução

Os seis documentos anteriores construíram um sistema de práticas: estado e fluxo de dados, fronteiras de API, lógica de negócio, arquitetura, prevenção de deterioração e testes automatizados. Este sétimo documento trata do tema que permeia — e condiciona — todos os outros: **segurança**.

O documento 2 (APIs and System Boundaries) abordou segurança no contexto de fronteiras: autenticação via Sanctum, rate limiting, validação na borda. Mas segurança vai além de proteger endpoints. Em um sistema que manipula dados de prefeitura — CPFs de cidadãos, processos licitatórios, informações financeiras, dados de saúde via e-SUS — uma falha de segurança não é apenas um bug técnico. É uma violação legal (LGPD), um risco político e um dano real a pessoas reais.

Segurança não é uma feature que se adiciona depois. É uma propriedade que emerge de decisões tomadas em cada camada: como o controller valida input, como a Policy verifica permissões, como o Resource filtra campos sensíveis, como o Repository delimita o escopo de queries, como o Model protege atributos. Cada decisão de arquitetura discutida nos documentos anteriores tem uma dimensão de segurança.

Este documento cobre: a diferença prática entre autenticação e autorização, como implementar um sistema de papéis e permissões robusto, como usar Policies para controle granular de acesso, como proteger contra IDOR (a vulnerabilidade mais comum e mais subestimada em APIs Laravel), como sanitizar dados, como evitar vazamento de informações sensíveis, e como atender aos requisitos da LGPD em sistemas governamentais.

---

## Por que isso importa?

### O contexto de sistemas municipais

Sistemas de prefeitura não são e-commerces. O impacto de uma falha de segurança é fundamentalmente diferente:

**Dados pessoais de cidadãos.** CPFs, endereços, dados de saúde, informações financeiras. A LGPD (Lei 13.709/2018) impõe obrigações legais sobre o tratamento desses dados, com multas que podem chegar a 2% do faturamento ou R$ 50 milhões por infração.

**Processos licitatórios.** Informações sobre valores estimados, propostas de fornecedores, pareceres técnicos. Acesso indevido a essas informações pode viciar o processo inteiro e resultar em impugnações judiciais.

**Dados financeiros públicos.** Receitas, despesas, folha de pagamento. Embora muitos sejam públicos por lei (transparência), os detalhes operacionais (contas bancárias, dados de servidores) são sensíveis.

**Dados do e-SUS.** Informações de saúde de cidadãos que são protegidas tanto pela LGPD quanto pelo sigilo profissional médico.

### O que acontece quando segurança falha

**IDOR (Insecure Direct Object Reference).** Um usuário do setor de RH descobre que mudando o ID na URL de `/api/funcionarios/42` para `/api/funcionarios/43` consegue ver os dados de qualquer servidor público — incluindo salários, CPFs e dados bancários.

**Escalação de privilégios.** Um atendente de balcão com acesso de leitura descobre que a API aceita requisições de escrita porque a autorização é verificada apenas no frontend (no Vue.js), não no backend.

**Vazamento de dados na API.** O endpoint `/api/requerimentos/123` retorna o objeto completo do Model, incluindo campos internos como `ip_origem`, `observacao_interna` e o CPF completo do cidadão — que deveria ser parcialmente mascarado para a maioria dos usuários.

**SQL Injection.** Um campo de busca não sanitizado permite que alguém execute queries arbitrárias, extraindo toda a base de dados de cidadãos.

Cada um desses cenários é prevenível com práticas que este documento detalha.

---

## Autenticação vs Autorização

Esses dois conceitos são frequentemente confundidos, mas resolvem problemas completamente diferentes:

**Autenticação** responde: *quem é você?* É o processo de verificar a identidade. O usuário fornece credenciais (email + senha, token, certificado), e o sistema confirma ou rejeita.

**Autorização** responde: *o que você pode fazer?* Dado que sabemos quem você é, quais recursos pode acessar e quais operações pode executar?

```
Autenticação                          Autorização
┌─────────────────────┐               ┌─────────────────────┐
│ "Eu sou Maria,      │               │ "Maria é do setor   │
│  servidora lotada    │    ───────►   │  de RH. Pode ver    │
│  no RH, matrícula   │               │  funcionários, mas  │
│  2847"               │               │  não pode alterar   │
│                      │               │  processos de       │
│ Verificado via       │               │  licitação."        │
│ email + senha        │               │                     │
└─────────────────────┘               └─────────────────────┘
```

Um sistema pode ter autenticação perfeita (sabe exatamente quem é o usuário) e autorização quebrada (não verifica se aquele usuário tem permissão para a operação solicitada). Na prática, a maioria das vulnerabilidades de segurança em aplicações Laravel está na **autorização**, não na autenticação.

---

## Autenticação no Laravel

### Sanctum para APIs (SPA + Mobile)

Para aplicações Vue.js + Inertia.js (SPA no mesmo domínio), Sanctum usa autenticação baseada em sessão — sem tokens. Para APIs consumidas por mobile ou terceiros, Sanctum emite tokens com abilities (permissões por token).

```php
// Autenticação SPA (sessão) — usada pelo frontend Vue.js/Inertia
// config/sanctum.php
'stateful' => explode(',', env(
    'SANCTUM_STATEFUL_DOMAINS',
    'localhost,localhost:3000,127.0.0.1'
)),

// routes/api.php — rotas protegidas por sessão
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/user', fn (Request $request) => $request->user());
    Route::apiResource('orders', OrderController::class);
});
```

```php
// Autenticação por token — para integrações externas e mobile
class AuthController extends Controller
{
    public function createToken(Request $request): JsonResponse
    {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required',
            'device_name' => 'required|string',
        ]);

        $user = User::where('email', $request->email)->first();

        if (! $user || ! Hash::check($request->password, $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['Credenciais inválidas.'],
            ]);
        }

        // Token com abilities — limita o que este token pode fazer
        $token = $user->createToken(
            name: $request->device_name,
            abilities: $this->resolveAbilities($user),
        );

        return response()->json([
            'token' => $token->plainTextToken,
            'expires_at' => now()->addDays(30)->toIso8601String(),
        ]);
    }

    private function resolveAbilities(User $user): array
    {
        // Abilities baseadas no papel do usuário
        return match ($user->role) {
            UserRole::ADMIN => ['*'],
            UserRole::SERVIDOR => [
                'orders:read',
                'orders:create',
                'requerimentos:read',
                'requerimentos:create',
            ],
            UserRole::CIDADAO => [
                'requerimentos:read-own',
                'requerimentos:create',
            ],
            default => [],
        };
    }
}
```

```php
// Verificar ability do token no middleware ou controller
Route::middleware(['auth:sanctum', 'ability:orders:read'])->group(function () {
    Route::get('/orders', [OrderController::class, 'index']);
});

// Ou verificar manualmente
public function index(Request $request)
{
    if ($request->user()->tokenCan('orders:read-all')) {
        return OrderResource::collection(Order::paginate());
    }

    // Só os próprios pedidos
    return OrderResource::collection(
        $request->user()->orders()->paginate()
    );
}
```

### Protegendo o login contra ataques

```php
// app/Http/Controllers/Auth/LoginController.php
class LoginController extends Controller
{
    public function login(Request $request): JsonResponse
    {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        // Rate limiting por IP + email — previne brute force
        $throttleKey = Str::lower($request->input('email')) . '|' . $request->ip();

        if (RateLimiter::tooManyAttempts($throttleKey, maxAttempts: 5)) {
            $seconds = RateLimiter::availableIn($throttleKey);

            throw ValidationException::withMessages([
                'email' => ["Muitas tentativas. Tente novamente em {$seconds} segundos."],
            ]);
        }

        if (! Auth::attempt($request->only('email', 'password'))) {
            RateLimiter::hit($throttleKey, decaySeconds: 300);

            throw ValidationException::withMessages([
                'email' => ['Credenciais inválidas.'],
            ]);
        }

        RateLimiter::clear($throttleKey);

        $request->session()->regenerate();

        return response()->json([
            'user' => new UserResource(Auth::user()),
        ]);
    }
}
```

O `session()->regenerate()` é crítico: previne session fixation attacks. Sem ele, um atacante que conhece o ID de sessão antes do login pode sequestrar a sessão autenticada.

---

## Sistema de Papéis e Permissões

### Modelagem de dados

```php
// Migration: users (campo role simples para sistemas menores)
Schema::table('users', function (Blueprint $table) {
    $table->string('role')->default(UserRole::CIDADAO->value);
});

// Enum tipado
enum UserRole: string
{
    case ADMIN = 'admin';
    case GESTOR = 'gestor';
    case SERVIDOR = 'servidor';
    case CIDADAO = 'cidadao';

    public function label(): string
    {
        return match ($this) {
            self::ADMIN => 'Administrador',
            self::GESTOR => 'Gestor',
            self::SERVIDOR => 'Servidor',
            self::CIDADAO => 'Cidadão',
        };
    }

    public function isAtLeast(self $minimumRole): bool
    {
        $hierarchy = [
            self::CIDADAO->value => 0,
            self::SERVIDOR->value => 1,
            self::GESTOR->value => 2,
            self::ADMIN->value => 3,
        ];

        return $hierarchy[$this->value] >= $hierarchy[$minimumRole->value];
    }
}
```

### Sistema granular com tabela de permissões

Para sistemas maiores (como gestão municipal), papéis fixos não bastam. Um servidor do RH tem permissões diferentes de um servidor da Fazenda. A solução é um sistema de permissões granulares:

```php
// Migration: tabela de permissões
Schema::create('permissions', function (Blueprint $table) {
    $table->id();
    $table->string('name')->unique();         // 'licitacoes.publicar'
    $table->string('description')->nullable(); // 'Publicar processos licitatórios'
    $table->string('group');                   // 'licitacoes'
    $table->timestamps();
});

Schema::create('roles', function (Blueprint $table) {
    $table->id();
    $table->string('name')->unique();          // 'gestor_licitacoes'
    $table->string('display_name');            // 'Gestor de Licitações'
    $table->text('description')->nullable();
    $table->timestamps();
});

Schema::create('role_permission', function (Blueprint $table) {
    $table->foreignId('role_id')->constrained()->cascadeOnDelete();
    $table->foreignId('permission_id')->constrained()->cascadeOnDelete();
    $table->primary(['role_id', 'permission_id']);
});

Schema::create('user_role', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->foreignId('role_id')->constrained()->cascadeOnDelete();
    $table->string('scope')->nullable();  // 'setor:rh', 'setor:fazenda'
    $table->primary(['user_id', 'role_id']);
});
```

```php
// app/Models/User.php
class User extends Authenticatable
{
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class, 'user_role')
            ->withPivot('scope');
    }

    public function hasPermission(string $permission): bool
    {
        return $this->roles
            ->flatMap(fn (Role $role) => $role->permissions)
            ->contains('name', $permission);
    }

    public function hasPermissionWithScope(string $permission, ?string $scope = null): bool
    {
        return $this->roles
            ->filter(function (Role $role) use ($scope) {
                if ($scope === null) {
                    return true;
                }
                return $role->pivot->scope === null  // Permissão global
                    || $role->pivot->scope === $scope;  // Ou no escopo correto
            })
            ->flatMap(fn (Role $role) => $role->permissions)
            ->contains('name', $permission);
    }

    public function hasRole(string $roleName): bool
    {
        return $this->roles->contains('name', $roleName);
    }

    public function isAdmin(): bool
    {
        return $this->hasRole('admin');
    }
}
```

```php
// app/Models/Role.php
class Role extends Model
{
    public function permissions(): BelongsToMany
    {
        return $this->belongsToMany(Permission::class, 'role_permission');
    }

    public function users(): BelongsToMany
    {
        return $this->belongsToMany(User::class, 'user_role');
    }
}
```

### Seeder para permissões iniciais

```php
// database/seeders/PermissionSeeder.php
class PermissionSeeder extends Seeder
{
    public function run(): void
    {
        $permissions = [
            // Licitações
            ['name' => 'licitacoes.visualizar', 'group' => 'licitacoes', 'description' => 'Ver processos licitatórios'],
            ['name' => 'licitacoes.criar', 'group' => 'licitacoes', 'description' => 'Criar novos processos'],
            ['name' => 'licitacoes.publicar', 'group' => 'licitacoes', 'description' => 'Publicar processos no diário oficial'],
            ['name' => 'licitacoes.homologar', 'group' => 'licitacoes', 'description' => 'Homologar resultado'],
            ['name' => 'licitacoes.cancelar', 'group' => 'licitacoes', 'description' => 'Cancelar ou revogar processo'],

            // Requerimentos
            ['name' => 'requerimentos.visualizar', 'group' => 'requerimentos', 'description' => 'Ver requerimentos'],
            ['name' => 'requerimentos.criar', 'group' => 'requerimentos', 'description' => 'Criar requerimentos'],
            ['name' => 'requerimentos.despachar', 'group' => 'requerimentos', 'description' => 'Despachar requerimentos'],
            ['name' => 'requerimentos.arquivar', 'group' => 'requerimentos', 'description' => 'Arquivar requerimentos'],

            // RH
            ['name' => 'rh.funcionarios.visualizar', 'group' => 'rh', 'description' => 'Ver dados de funcionários'],
            ['name' => 'rh.funcionarios.editar', 'group' => 'rh', 'description' => 'Editar dados de funcionários'],
            ['name' => 'rh.folha.visualizar', 'group' => 'rh', 'description' => 'Ver folha de pagamento'],
            ['name' => 'rh.folha.processar', 'group' => 'rh', 'description' => 'Processar folha de pagamento'],

            // Administração
            ['name' => 'admin.usuarios.gerenciar', 'group' => 'admin', 'description' => 'Gerenciar usuários do sistema'],
            ['name' => 'admin.logs.visualizar', 'group' => 'admin', 'description' => 'Ver logs de auditoria'],
        ];

        foreach ($permissions as $permission) {
            Permission::updateOrCreate(
                ['name' => $permission['name']],
                $permission
            );
        }

        // Criar papéis com permissões
        $this->createRole('admin', 'Administrador', Permission::all());

        $this->createRole('gestor_licitacoes', 'Gestor de Licitações', Permission::query()
            ->where('group', 'licitacoes')
            ->get()
        );

        $this->createRole('atendente', 'Atendente', Permission::query()
            ->whereIn('name', [
                'requerimentos.visualizar',
                'requerimentos.criar',
                'requerimentos.despachar',
            ])
            ->get()
        );
    }

    private function createRole(string $name, string $displayName, $permissions): void
    {
        $role = Role::updateOrCreate(
            ['name' => $name],
            ['display_name' => $displayName]
        );

        $role->permissions()->sync($permissions->pluck('id'));
    }
}
```

---

## Policies e Gates

Policies são a forma Laravel de implementar autorização granular. Cada Policy encapsula as regras de acesso para um Model específico.

### Policy completa para Processos de Licitação

```php
// app/Policies/ProcessoPolicy.php
namespace App\Policies;

use App\Models\Processo;
use App\Models\User;
use App\Enums\ProcessoStatus;

class ProcessoPolicy
{
    /**
     * Administradores podem tudo.
     * Retornar null delega para os métodos específicos.
     */
    public function before(User $user, string $ability): ?bool
    {
        if ($user->isAdmin()) {
            return true;
        }

        return null;
    }

    public function viewAny(User $user): bool
    {
        return $user->hasPermission('licitacoes.visualizar');
    }

    public function view(User $user, Processo $processo): bool
    {
        // Processos publicados são visíveis para quem tem permissão básica
        if ($processo->status === ProcessoStatus::PUBLICADO) {
            return $user->hasPermission('licitacoes.visualizar');
        }

        // Rascunhos só são visíveis para quem pode criar
        return $user->hasPermission('licitacoes.criar');
    }

    public function create(User $user): bool
    {
        return $user->hasPermission('licitacoes.criar');
    }

    public function update(User $user, Processo $processo): bool
    {
        // Só pode editar processos em rascunho
        if ($processo->status !== ProcessoStatus::RASCUNHO) {
            return false;
        }

        return $user->hasPermission('licitacoes.criar');
    }

    public function publish(User $user, Processo $processo): bool
    {
        if (! $user->hasPermission('licitacoes.publicar')) {
            return false;
        }

        // Não pode publicar se a documentação está incompleta
        return $processo->podeSerPublicado();
    }

    public function approve(User $user, Processo $processo): bool
    {
        if (! $user->hasPermission('licitacoes.homologar')) {
            return false;
        }

        // Quem criou o processo não pode homologá-lo (segregação de funções)
        return $user->id !== $processo->criado_por_id;
    }

    public function cancel(User $user, Processo $processo): bool
    {
        if (! $user->hasPermission('licitacoes.cancelar')) {
            return false;
        }

        // Processos já homologados precisam de justificativa formal
        // (verificado na Action, não na Policy)
        return $processo->podeSerCancelado();
    }

    public function delete(User $user, Processo $processo): bool
    {
        // Ninguém pode excluir processos — apenas arquivar
        // Registros públicos devem ser preservados
        return false;
    }
}
```

Note a regra `approve`: quem criou o processo não pode homologá-lo. Isso é **segregação de funções** — um princípio de controle interno que impede que uma única pessoa controle todo o ciclo de uma licitação. A Policy é o lugar certo para implementar isso.

### Registrando Policies

```php
// app/Providers/AuthServiceProvider.php (Laravel 10)
// ou bootstrap/app.php (Laravel 11)

// Laravel 11 — auto-discovery resolve automaticamente se Policy
// segue a convenção de nome. Para casos explícitos:
use Illuminate\Support\Facades\Gate;

Gate::policy(Processo::class, ProcessoPolicy::class);
Gate::policy(Requerimento::class, RequerimentoPolicy::class);
Gate::policy(Funcionario::class, FuncionarioPolicy::class);
```

### Usando Policies no Controller

```php
class ProcessoController extends Controller
{
    public function index(Request $request)
    {
        // Verifica se pode listar processos
        $this->authorize('viewAny', Processo::class);

        return ProcessoResource::collection(
            Processo::query()
                ->visibleTo($request->user()) // Scope que filtra por permissão
                ->paginate()
        );
    }

    public function show(Processo $processo)
    {
        // Verifica se pode ver ESTE processo específico
        $this->authorize('view', $processo);

        return new ProcessoResource($processo);
    }

    public function publicar(Processo $processo, PublicarProcessoAction $action)
    {
        // Verifica se pode publicar ESTE processo
        $this->authorize('publish', $processo);

        $action->execute($processo);

        return response()->json([
            'message' => 'Processo publicado com sucesso',
        ]);
    }

    public function homologar(Processo $processo, HomologarProcessoAction $action)
    {
        $this->authorize('approve', $processo);

        $action->execute($processo);

        return response()->json([
            'message' => 'Processo homologado',
        ]);
    }
}
```

O `$this->authorize()` lança `AuthorizationException` automaticamente se a Policy retornar `false`. O controller não precisa de `if/else` para tratar o caso negativo — o middleware de exceções converte em resposta 403.

### Gates para permissões avultas

Gates são úteis para permissões que não estão ligadas a um Model específico:

```php
// bootstrap/app.php ou AuthServiceProvider
Gate::define('access-admin-panel', function (User $user) {
    return $user->hasPermission('admin.usuarios.gerenciar')
        || $user->hasPermission('admin.logs.visualizar');
});

Gate::define('process-payroll', function (User $user) {
    return $user->hasPermission('rh.folha.processar');
});

// Uso
if (Gate::allows('access-admin-panel')) {
    // Mostrar link para painel admin
}

// No Blade/Inertia
@can('access-admin-panel')
    <a href="/admin">Painel Administrativo</a>
@endcan
```

### Middleware de permissão customizado

```php
// app/Http/Middleware/EnsureUserHasPermission.php
class EnsureUserHasPermission
{
    public function handle(Request $request, Closure $next, string ...$permissions): Response
    {
        $user = $request->user();

        if (! $user) {
            abort(401);
        }

        foreach ($permissions as $permission) {
            if (! $user->hasPermission($permission)) {
                abort(403, "Permissão necessária: {$permission}");
            }
        }

        return $next($request);
    }
}

// bootstrap/app.php
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'permission' => EnsureUserHasPermission::class,
    ]);
})

// Uso nas rotas
Route::middleware('permission:licitacoes.visualizar')->group(function () {
    Route::get('/licitacoes', [ProcessoController::class, 'index']);
});

Route::middleware('permission:rh.folha.processar')->group(function () {
    Route::post('/folha/processar', [FolhaController::class, 'processar']);
});
```

---

## Proteção contra IDOR

IDOR (Insecure Direct Object Reference) é a vulnerabilidade mais comum em APIs Laravel — e a mais perigosa em sistemas governamentais. Acontece quando o sistema permite que um usuário acesse recursos de outro apenas mudando o ID na URL.

### O problema

```php
// ❌ Vulnerável a IDOR — qualquer usuário autenticado vê qualquer requerimento
Route::get('/api/requerimentos/{id}', function ($id) {
    return Requerimento::findOrFail($id);
});

// Ataque: usuário logado como cidadão José troca o ID
// GET /api/requerimentos/1   → Seu requerimento ✅
// GET /api/requerimentos/2   → Requerimento de Maria ❌ IDOR!
// GET /api/requerimentos/3   → Requerimento de João ❌ IDOR!
```

### Solução 1: Policies (mais robusto)

```php
// Policy verifica propriedade
class RequerimentoPolicy
{
    public function view(User $user, Requerimento $requerimento): bool
    {
        // Cidadão só vê os próprios
        if ($user->role === UserRole::CIDADAO) {
            return $requerimento->cidadao_id === $user->id;
        }

        // Atendente vê os do seu setor
        if ($user->hasRole('atendente')) {
            return $requerimento->setor_id === $user->setor_id;
        }

        // Gestor vê todos
        return $user->hasPermission('requerimentos.visualizar');
    }
}

// Controller usa a Policy
public function show(Requerimento $requerimento)
{
    $this->authorize('view', $requerimento);  // 403 se não autorizado

    return new RequerimentoResource($requerimento);
}
```

### Solução 2: Scope de query (defense in depth)

Mesmo com Policy, é uma boa prática filtrar queries no nível do Repository ou Scope, garantindo que o usuário **nunca** tenha acesso a dados fora do seu escopo — nem por acidente:

```php
// app/Models/Requerimento.php
class Requerimento extends Model
{
    public function scopeVisibleTo(Builder $query, User $user): Builder
    {
        if ($user->isAdmin()) {
            return $query; // Admin vê tudo
        }

        if ($user->hasRole('atendente')) {
            return $query->where('setor_id', $user->setor_id);
        }

        // Cidadão: apenas os próprios
        return $query->where('cidadao_id', $user->id);
    }
}

// Controller
public function index(Request $request)
{
    $this->authorize('viewAny', Requerimento::class);

    return RequerimentoResource::collection(
        Requerimento::query()
            ->visibleTo($request->user())  // IDOR impossível na listagem
            ->latest()
            ->paginate()
    );
}
```

### Solução 3: UUIDs ao invés de IDs sequenciais

IDs auto-incrementais (`1, 2, 3...`) são triviais de enumerar. UUIDs são aleatórios e imprevisíveis — um atacante não consegue adivinhar o próximo ID:

```php
// Migration
Schema::create('requerimentos', function (Blueprint $table) {
    $table->uuid('id')->primary();
    // ...
});

// Model
class Requerimento extends Model
{
    use HasUuids;
}

// URL resultante: /api/requerimentos/9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d
// Impossível adivinhar outros IDs
```

UUIDs não substituem Policies — são uma camada adicional de defesa. A Policy garante que mesmo quem acertar o UUID não acessará se não tiver permissão.

### Solução 4: Forçar escopo via Route Model Binding

```php
// routes/api.php — Binding aninhado que força escopo
Route::middleware('auth:sanctum')->group(function () {
    // O requerimento DEVE pertencer ao cidadão autenticado
    Route::get('/meus-requerimentos/{requerimento}', function (Requerimento $requerimento) {
        return new RequerimentoResource($requerimento);
    })->scopeBindings()->can('view', 'requerimento');
});

// Ou com binding customizado no Model
class Requerimento extends Model
{
    public function resolveRouteBinding($value, $field = null)
    {
        $user = request()->user();

        // Se cidadão, só resolve se for seu
        if ($user->role === UserRole::CIDADAO) {
            return $this->where('id', $value)
                ->where('cidadao_id', $user->id)
                ->firstOrFail();
        }

        return parent::resolveRouteBinding($value, $field);
    }
}
```

### A abordagem de defesa em profundidade

Nenhuma camada sozinha é suficiente. A combinação é o que cria segurança real:

```
Camada 1: UUID (dificulta enumeração)
    ↓
Camada 2: Middleware de autenticação (garante identidade)
    ↓
Camada 3: Policy (verifica permissão específica)
    ↓
Camada 4: Scope de query (limita resultados ao escopo do usuário)
    ↓
Camada 5: Resource (filtra campos sensíveis na resposta)
```

Se qualquer camada falhar, as demais ainda protegem.

---

## Sanitização e Validação de Dados

### Validação no FormRequest

A primeira linha de defesa é nunca confiar em dados que vêm do frontend:

```php
// app/Http/Requests/StoreRequerimentoRequest.php
class StoreRequerimentoRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->hasPermission('requerimentos.criar');
    }

    public function rules(): array
    {
        return [
            'tipo_id' => ['required', 'exists:tipos_requerimento,id'],
            'descricao' => ['required', 'string', 'min:10', 'max:5000'],
            'cpf_interessado' => ['required', 'string', 'cpf'], // Regra customizada
            'documentos' => ['nullable', 'array', 'max:5'],
            'documentos.*' => [
                'file',
                'mimes:pdf,jpg,png',
                'max:10240', // 10MB
            ],
        ];
    }

    /**
     * Sanitizar dados ANTES da validação.
     */
    protected function prepareForValidation(): void
    {
        $this->merge([
            'cpf_interessado' => preg_replace('/\D/', '', $this->cpf_interessado ?? ''),
            'descricao' => strip_tags($this->descricao ?? ''),
        ]);
    }

    public function messages(): array
    {
        return [
            'descricao.min' => 'A descrição deve ter pelo menos 10 caracteres.',
            'documentos.*.mimes' => 'Apenas PDF, JPG e PNG são aceitos.',
            'documentos.*.max' => 'Cada documento deve ter no máximo 10MB.',
        ];
    }
}
```

### Regra de validação customizada para CPF

```php
// app/Rules/CpfRule.php
class CpfRule implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        $cpf = preg_replace('/\D/', '', $value);

        if (strlen($cpf) !== 11) {
            $fail('O :attribute deve ter 11 dígitos.');
            return;
        }

        if (preg_match('/^(\d)\1{10}$/', $cpf)) {
            $fail('O :attribute é inválido.');
            return;
        }

        // Validação dos dígitos verificadores
        for ($t = 9; $t < 11; $t++) {
            $sum = 0;
            for ($c = 0; $c < $t; $c++) {
                $sum += $cpf[$c] * (($t + 1) - $c);
            }
            $digit = ((10 * $sum) % 11) % 10;
            if ((int) $cpf[$t] !== $digit) {
                $fail('O :attribute é inválido.');
                return;
            }
        }
    }
}
```

### Protegendo contra Mass Assignment

O Mass Assignment é uma das vulnerabilidades mais clássicas do Laravel. Sem proteção, um atacante pode enviar campos extras na requisição para modificar dados que não deveria:

```php
// ❌ Vulnerável — aceita qualquer campo
public function update(Request $request, User $user)
{
    $user->update($request->all());
    // Atacante envia: {"name": "Maria", "role": "admin"} → escalação de privilégios!
}

// ✅ Seguro — usa dados validados com campos explícitos
public function update(UpdateUserRequest $request, User $user)
{
    $this->authorize('update', $user);

    $user->update($request->validated());
    // FormRequest define exatamente quais campos são aceitos
}

// ✅ Duplamente seguro — $fillable no Model restringe ainda mais
class User extends Authenticatable
{
    protected $fillable = [
        'name',
        'email',
        'phone',
    ];

    // 'role', 'password', 'email_verified_at' NÃO estão no fillable
    // Mesmo que passados em update(), são ignorados silenciosamente

    protected $hidden = [
        'password',
        'remember_token',
    ];
}
```

### Validação de parâmetros de busca

```php
// ❌ SQL Injection via campo de ordenação
public function index(Request $request)
{
    return Product::orderBy($request->input('sort', 'id'))->paginate();
    // Atacante: ?sort=id; DROP TABLE products; --
}

// ✅ Whitelist de campos permitidos
public function index(Request $request)
{
    $request->validate([
        'sort' => ['nullable', Rule::in(['name', 'price', 'created_at'])],
        'direction' => ['nullable', Rule::in(['asc', 'desc'])],
        'search' => ['nullable', 'string', 'max:100'],
    ]);

    return Product::query()
        ->when($request->search, fn ($q, $search) =>
            $q->where('name', 'ilike', '%' . $search . '%')
        )
        ->orderBy(
            $request->input('sort', 'created_at'),
            $request->input('direction', 'desc')
        )
        ->paginate();
}
```

---

## Proteção de Dados Sensíveis na API

### O problema: Model direto na resposta

```php
// ❌ NUNCA retorne Models diretamente
public function show(Funcionario $funcionario)
{
    return $funcionario;
    // Expõe: cpf, salario, conta_bancaria, dados_medicos, observacoes_internas
}
```

### API Resources com filtragem contextual

```php
// app/Http/Resources/FuncionarioResource.php
class FuncionarioResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        $user = $request->user();

        $data = [
            'id' => $this->id,
            'nome' => $this->nome,
            'matricula' => $this->matricula,
            'cargo' => $this->cargo,
            'setor' => $this->setor?->nome,
            'data_admissao' => $this->data_admissao->format('d/m/Y'),
        ];

        // CPF mascarado por padrão, completo apenas para RH
        $data['cpf'] = $user->hasPermission('rh.funcionarios.editar')
            ? $this->cpf
            : $this->cpfMascarado();

        // Dados financeiros apenas para quem tem permissão
        if ($user->hasPermission('rh.folha.visualizar')) {
            $data['salario'] = $this->salario;
            $data['conta_bancaria'] = [
                'banco' => $this->banco,
                'agencia' => $this->agencia,
                'conta' => $this->contaMascarada(),
            ];
        }

        // Dados administrativos internos — nunca para cidadãos
        if ($user->hasPermission('rh.funcionarios.editar')) {
            $data['observacoes_internas'] = $this->observacoes_internas;
            $data['avaliacao_desempenho'] = $this->avaliacao_desempenho;
        }

        return $data;
    }
}
```

```php
// Métodos auxiliares no Model
class Funcionario extends Model
{
    public function cpfMascarado(): string
    {
        // 123.456.789-09 → ***.456.789-**
        return '***.' . substr($this->cpf, 3, 7) . '-**';
    }

    public function contaMascarada(): string
    {
        // 12345-6 → ****5-6
        return str_repeat('*', strlen($this->conta) - 3)
            . substr($this->conta, -3);
    }
}
```

### Resource para cidadão (dados mínimos)

```php
// Quando o cidadão consulta seu próprio requerimento, vê dados limitados
class RequerimentoPublicoResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'protocolo' => $this->protocolo,
            'tipo' => $this->tipo->nome,
            'status' => $this->status->label(),
            'data_abertura' => $this->created_at->format('d/m/Y'),
            'ultima_movimentacao' => $this->updated_at->format('d/m/Y H:i'),

            // NÃO inclui: observações internas, CPF completo do cidadão,
            // dados do servidor que despachou, IP de origem
        ];
    }
}

// Resource interno — para atendentes e gestores
class RequerimentoInternoResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'protocolo' => $this->protocolo,
            'tipo' => $this->tipo->nome,
            'status' => $this->status->label(),
            'cidadao' => [
                'nome' => $this->cidadao->nome,
                'cpf' => $this->cidadao->cpf,
                'telefone' => $this->cidadao->telefone,
            ],
            'descricao' => $this->descricao,
            'observacoes_internas' => $this->observacoes_internas,
            'despachos' => DespachoResource::collection($this->despachos),
            'documentos' => DocumentoResource::collection($this->documentos),
            'historico' => $this->auditLogs->map(fn ($log) => [
                'acao' => $log->action,
                'usuario' => $log->user->nome,
                'data' => $log->created_at->format('d/m/Y H:i'),
            ]),
        ];
    }
}
```

```php
// Controller decide qual Resource usar
public function show(Requerimento $requerimento, Request $request)
{
    $this->authorize('view', $requerimento);

    if ($request->user()->role === UserRole::CIDADAO) {
        return new RequerimentoPublicoResource($requerimento);
    }

    return new RequerimentoInternoResource(
        $requerimento->load(['cidadao', 'despachos', 'documentos', 'auditLogs'])
    );
}
```

### Campos que NUNCA devem aparecer na API

```php
// app/Models/User.php
class User extends Authenticatable
{
    // $hidden remove campos da serialização JSON do Model
    protected $hidden = [
        'password',
        'remember_token',
        'two_factor_secret',
        'two_factor_recovery_codes',
    ];

    // Mas NÃO confie apenas no $hidden — use Resources
}
```

`$hidden` é uma rede de segurança, não a solução primária. A solução primária é **nunca retornar Models diretamente** — sempre use Resources que listam explicitamente quais campos são incluídos.

---

## CSRF, XSS e SQL Injection

### CSRF (Cross-Site Request Forgery)

O Laravel protege automaticamente contra CSRF em formulários web via middleware `VerifyCsrfToken`. Para APIs com Sanctum usando autenticação por sessão, o frontend Vue.js precisa obter o cookie CSRF antes de fazer requisições:

```javascript
// No frontend Vue.js — chamar antes do primeiro POST
await axios.get('/sanctum/csrf-cookie');

// Depois, axios envia o cookie automaticamente
await axios.post('/api/v1/orders', { /* ... */ });
```

Para APIs com autenticação por token, CSRF não se aplica (tokens são imunes por design).

### XSS (Cross-Site Scripting)

O Blade escapa HTML automaticamente com `{{ }}`. O Vue.js faz o mesmo com `{{ }}` em templates. O risco aparece em situações onde o HTML é renderizado diretamente:

```php
// ❌ Perigoso no Blade
{!! $requerimento->descricao !!}
// Se descricao contiver <script>alert('XSS')</script>, executa

// ✅ Seguro — escapa HTML
{{ $requerimento->descricao }}

// ❌ Perigoso no Vue
<div v-html="requerimento.descricao"></div>

// ✅ Seguro — texto puro
<div>{{ requerimento.descricao }}</div>
```

Para casos onde HTML é necessário (editor WYSIWYG), sanitize no backend:

```php
// No FormRequest
protected function prepareForValidation(): void
{
    if ($this->has('conteudo_html')) {
        $this->merge([
            'conteudo_html' => strip_tags(
                $this->conteudo_html,
                '<p><br><strong><em><ul><ol><li><h2><h3>' // Tags permitidas
            ),
        ]);
    }
}
```

### SQL Injection

O Eloquent e o Query Builder do Laravel usam prepared statements por padrão, prevenindo SQL Injection na maioria dos cenários. O risco aparece com raw queries:

```php
// ❌ Vulnerável — interpolação direta
DB::select("SELECT * FROM users WHERE name = '$name'");

// ❌ Vulnerável — orderBy com input do usuário
DB::table('products')->orderBy($request->input('sort'))->get();

// ✅ Seguro — bindings parametrizados
DB::select('SELECT * FROM users WHERE name = ?', [$name]);

// ✅ Seguro — whitelist para orderBy
$allowedSorts = ['name', 'price', 'created_at'];
$sort = in_array($request->input('sort'), $allowedSorts)
    ? $request->input('sort')
    : 'created_at';

DB::table('products')->orderBy($sort)->get();
```

---

## Rate Limiting e Proteção contra Abuso

### Rate Limiting por papel e contexto

```php
// bootstrap/app.php
->withMiddleware(function (Middleware $middleware) {
    // ...
})

// app/Providers/AppServiceProvider.php
public function boot(): void
{
    RateLimiter::for('api', function (Request $request) {
        $user = $request->user();

        if (! $user) {
            // Não autenticado: 30 requisições por minuto por IP
            return Limit::perMinute(30)->by($request->ip());
        }

        // Rate limit baseado no papel
        $limit = match ($user->role) {
            UserRole::ADMIN => 300,
            UserRole::SERVIDOR => 120,
            UserRole::CIDADAO => 60,
            default => 30,
        };

        return Limit::perMinute($limit)->by($user->id);
    });

    // Rate limit específico para login — proteção contra brute force
    RateLimiter::for('login', function (Request $request) {
        return [
            Limit::perMinute(5)->by($request->ip()),
            Limit::perMinute(10)->by($request->input('email', '')),
        ];
    });

    // Rate limit para operações sensíveis
    RateLimiter::for('sensitive', function (Request $request) {
        return Limit::perHour(10)->by($request->user()?->id ?: $request->ip());
    });
}
```

```php
// Uso nas rotas
Route::middleware('throttle:login')->group(function () {
    Route::post('/login', [AuthController::class, 'login']);
});

Route::middleware(['auth:sanctum', 'throttle:api'])->group(function () {
    Route::apiResource('orders', OrderController::class);
});

Route::middleware(['auth:sanctum', 'throttle:sensitive'])->group(function () {
    Route::post('/rh/folha/processar', [FolhaController::class, 'processar']);
    Route::delete('/admin/users/{user}', [UserController::class, 'destroy']);
});
```

---

## Auditoria e Logging de Segurança

Em sistemas governamentais, auditoria não é opcional — é requisito legal. Todo acesso a dados sensíveis e toda operação que muda estado deve ser rastreável.

### Trait de auditoria automática

```php
// app/Traits/Auditable.php
trait Auditable
{
    public static function bootAuditable(): void
    {
        static::created(function (Model $model) {
            self::logAudit($model, 'created');
        });

        static::updated(function (Model $model) {
            if ($model->isDirty()) {
                self::logAudit($model, 'updated', [
                    'changes' => $model->getChanges(),
                    'original' => array_intersect_key(
                        $model->getOriginal(),
                        $model->getChanges()
                    ),
                ]);
            }
        });

        static::deleted(function (Model $model) {
            self::logAudit($model, 'deleted');
        });
    }

    private static function logAudit(Model $model, string $action, array $extra = []): void
    {
        $user = auth()->user();

        DB::table('audit_logs')->insert([
            'auditable_type' => get_class($model),
            'auditable_id' => $model->getKey(),
            'action' => $action,
            'user_id' => $user?->id,
            'user_name' => $user?->nome ?? 'Sistema',
            'ip_address' => request()->ip(),
            'user_agent' => request()->userAgent(),
            'data' => json_encode($extra),
            'created_at' => now(),
        ]);
    }
}
```

```php
// Uso nos Models que precisam de auditoria
class Processo extends Model
{
    use Auditable;
    // Toda criação, atualização e deleção é registrada automaticamente
}

class Funcionario extends Model
{
    use Auditable;
}
```

### Logging de acessos a dados sensíveis

Além de auditar mudanças, registre **visualizações** de dados sensíveis:

```php
// app/Http/Middleware/LogSensitiveAccess.php
class LogSensitiveAccess
{
    private const SENSITIVE_PATTERNS = [
        'funcionarios/*' => 'Acesso a dados de funcionário',
        'folha/*' => 'Acesso a dados de folha de pagamento',
        'cidadaos/*/saude' => 'Acesso a dados de saúde',
    ];

    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request);

        if ($response->isSuccessful()) {
            $this->logIfSensitive($request);
        }

        return $response;
    }

    private function logIfSensitive(Request $request): void
    {
        $path = $request->path();

        foreach (self::SENSITIVE_PATTERNS as $pattern => $description) {
            if (Str::is("api/v1/{$pattern}", $path)) {
                DB::table('access_logs')->insert([
                    'user_id' => $request->user()?->id,
                    'path' => $path,
                    'method' => $request->method(),
                    'description' => $description,
                    'ip_address' => $request->ip(),
                    'created_at' => now(),
                ]);
                break;
            }
        }
    }
}
```

### Consulta de logs de auditoria

```php
// Controller para o painel de auditoria (apenas admins)
class AuditLogController extends Controller
{
    public function index(Request $request)
    {
        $this->authorize('viewAny', AuditLog::class);

        $request->validate([
            'user_id' => ['nullable', 'exists:users,id'],
            'action' => ['nullable', Rule::in(['created', 'updated', 'deleted'])],
            'from' => ['nullable', 'date'],
            'to' => ['nullable', 'date', 'after_or_equal:from'],
        ]);

        $logs = DB::table('audit_logs')
            ->when($request->user_id, fn ($q, $id) => $q->where('user_id', $id))
            ->when($request->action, fn ($q, $action) => $q->where('action', $action))
            ->when($request->from, fn ($q, $from) => $q->where('created_at', '>=', $from))
            ->when($request->to, fn ($q, $to) => $q->where('created_at', '<=', $to))
            ->orderByDesc('created_at')
            ->paginate(50);

        return response()->json($logs);
    }
}
```

---

## LGPD e Dados de Cidadãos

A LGPD (Lei Geral de Proteção de Dados) impõe obrigações específicas para o tratamento de dados pessoais por entes públicos. Em sistemas municipais, as principais preocupações são:

### Base legal para tratamento

Entes públicos não precisam de consentimento para tratar dados — a base legal é o cumprimento de obrigação legal ou execução de políticas públicas (Art. 7°, II e III da LGPD). Mas isso não significa que podem tratar dados sem restrição: o tratamento deve ser proporcional e limitado ao necessário.

```php
// O sistema só deve armazenar dados necessários para a finalidade

// ❌ Coleta excessiva
class StoreRequerimentoRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'cpf' => 'required',
            'nome' => 'required',
            'rg' => 'required',
            'estado_civil' => 'required',  // Necessário para requerimento?
            'religiao' => 'required',       // Necessário? Quase certamente não
            'etnia' => 'required',          // Dado sensível sem justificativa
            // ...
        ];
    }
}

// ✅ Coleta proporcional — apenas o necessário
class StoreRequerimentoRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'cpf' => ['required', new CpfRule()],
            'nome' => ['required', 'string', 'max:255'],
            'telefone' => ['required', 'string'],
            'tipo_id' => ['required', 'exists:tipos_requerimento,id'],
            'descricao' => ['required', 'string', 'min:10'],
        ];
    }
}
```

### Direito de acesso e retificação

O cidadão tem direito de acessar e corrigir seus dados. O sistema deve oferecer endpoints para isso:

```php
// Endpoints para o cidadão gerenciar seus dados
Route::middleware('auth:sanctum')->prefix('meus-dados')->group(function () {
    Route::get('/', [MeusDadosController::class, 'show']);
    Route::put('/', [MeusDadosController::class, 'update']);
    Route::get('/historico-acesso', [MeusDadosController::class, 'accessHistory']);
    Route::post('/solicitar-exclusao', [MeusDadosController::class, 'requestDeletion']);
});
```

```php
class MeusDadosController extends Controller
{
    public function show(Request $request): JsonResponse
    {
        $user = $request->user();

        return response()->json([
            'dados_pessoais' => [
                'nome' => $user->nome,
                'cpf' => $user->cpf,
                'email' => $user->email,
                'telefone' => $user->telefone,
            ],
            'dados_coletados' => [
                'requerimentos' => $user->requerimentos()->count(),
                'ultimo_acesso' => $user->last_login_at?->format('d/m/Y H:i'),
            ],
        ]);
    }

    public function update(UpdateDadosRequest $request): JsonResponse
    {
        $user = $request->user();

        $user->update($request->validated());

        // Registrar a alteração para auditoria
        DB::table('audit_logs')->insert([
            'auditable_type' => User::class,
            'auditable_id' => $user->id,
            'action' => 'self_update',
            'user_id' => $user->id,
            'data' => json_encode(['campos_alterados' => array_keys($request->validated())]),
            'ip_address' => $request->ip(),
            'created_at' => now(),
        ]);

        return response()->json(['message' => 'Dados atualizados']);
    }

    public function accessHistory(Request $request): JsonResponse
    {
        $logs = DB::table('access_logs')
            ->where('user_id', $request->user()->id)
            ->orderByDesc('created_at')
            ->limit(100)
            ->get(['path', 'method', 'created_at']);

        return response()->json(['acessos' => $logs]);
    }
}
```

### Anonimização e retenção

Dados pessoais não devem ser mantidos indefinidamente. Defina políticas de retenção e implemente anonimização:

```php
// app/Console/Commands/AnonymizeOldData.php
class AnonymizeOldData extends Command
{
    protected $signature = 'data:anonymize {--months=60}';
    protected $description = 'Anonimizar dados pessoais de requerimentos antigos';

    public function handle(): void
    {
        $cutoff = now()->subMonths((int) $this->option('months'));

        $count = Requerimento::query()
            ->where('status', 'arquivado')
            ->where('updated_at', '<', $cutoff)
            ->whereNotNull('cidadao_cpf') // Ainda não anonimizado
            ->update([
                'cidadao_cpf' => null,
                'cidadao_nome' => 'Cidadão anonimizado',
                'cidadao_telefone' => null,
                'cidadao_email' => null,
                'anonimizado_em' => now(),
            ]);

        $this->info("Anonimizados: {$count} requerimentos");
    }
}

// Agendar execução mensal
// app/Console/Kernel.php
$schedule->command('data:anonymize')->monthly();
```

---

## Anti-patterns de Segurança

### 1. Autorização apenas no frontend

```php
// ❌ O Vue.js esconde o botão, mas a API aceita a requisição
// Frontend Vue.js
<button v-if="user.role === 'admin'" @click="deleteUser">Excluir</button>

// Backend — sem verificação
public function destroy(User $user)
{
    $user->delete();  // Qualquer autenticado pode chamar diretamente via API!
}

// ✅ Autorização SEMPRE no backend, frontend é apenas UX
public function destroy(User $user)
{
    $this->authorize('delete', $user);  // Policy verifica no backend

    $user->delete();
}
```

Esconder botões no frontend é UX, não segurança. Todo controle de acesso real deve estar no backend.

### 2. Verificação de permissão por ID

```php
// ❌ Frágil — depende do ID numérico do papel
if ($user->role_id === 1) {
    // Admin
}

// ❌ Comparação de strings mágicas espalhadas pelo código
if ($user->role === 'admin') {
    // Admin
}

// ✅ Enum tipado + verificação semântica
if ($user->role === UserRole::ADMIN) {
    // Admin
}

// ✅ Melhor ainda — Policy encapsula a lógica
$this->authorize('approve', $processo);
// A Policy decide internamente quem pode aprovar
```

### 3. Informações de debug em produção

```php
// ❌ Respostas de erro que revelam informações internas
{
    "message": "SQLSTATE[42S02]: Base table or view not found: 
     1146 Table 'prefeitura_db.processos' doesn't exist",
    "file": "/var/www/app/Repositories/ProcessoRepository.php",
    "line": 47,
    "trace": [...]
}

// ✅ .env de produção
APP_DEBUG=false
APP_ENV=production

// Resposta segura em produção
{
    "message": "Erro interno do servidor"
}
```

Nunca exponha stack traces, nomes de tabelas, paths do servidor ou queries SQL em respostas de erro em produção.

### 4. Secrets no código-fonte

```php
// ❌ Credenciais hardcoded
class PaymentService
{
    private string $apiKey = 'sk_live_abc123def456';
}

// ❌ No .env commitado no Git
// .env (no repositório)
PAYMENT_API_KEY=sk_live_abc123def456

// ✅ .env local (não commitado) + .env.example sem valores
// .env.example (commitado)
PAYMENT_API_KEY=

// .env (ignorado pelo .gitignore)
PAYMENT_API_KEY=sk_live_abc123def456
```

---

## Checklist de Segurança

Use esta lista como verificação antes de cada deploy:

**Autenticação.**
Todas as rotas que exigem login estão protegidas com `auth:sanctum`?
O login tem rate limiting?
Sessões são regeneradas após login?
Tokens expiram?

**Autorização.**
Toda operação de escrita usa `$this->authorize()` ou Policy?
Endpoints de leitura filtram por escopo do usuário (`visibleTo`)?
Segregação de funções está implementada para operações críticas?

**IDOR.**
Endpoints que recebem IDs verificam propriedade/permissão?
Listagens usam scope de query para limitar resultados?
IDs sequenciais expostos foram substituídos por UUIDs onde possível?

**Dados sensíveis.**
Todas as respostas usam Resources (nunca Model direto)?
Campos sensíveis são mascarados ou omitidos por padrão?
`$hidden` está configurado em todos os Models?
`APP_DEBUG=false` em produção?

**Input.**
Todos os endpoints usam FormRequest com regras de validação?
Parâmetros de ordenação e filtro usam whitelist?
Uploads têm restrição de tipo e tamanho?

**Auditoria.**
Operações em dados sensíveis são logadas?
Logs incluem user_id, IP e timestamp?
Logs de auditoria não são apagáveis por usuários comuns?

---

## Exercícios Práticos

### Exercício 1: Implementar IDOR Protection

Pegue um endpoint existente do seu sistema que recebe um ID na URL e implemente proteção em múltiplas camadas: Policy que verifica propriedade, scope de query que filtra por permissão, e teste de feature que verifica que um usuário não pode acessar recurso de outro.

### Exercício 2: Sistema de Permissões

Implemente o sistema completo de papéis e permissões: migrations, Models (Role, Permission), Seeder com permissões iniciais para o seu domínio (Licitações, Atendimento, RH), e middleware de verificação de permissão. Escreva testes que verificam que um atendente não pode acessar dados de folha de pagamento.

### Exercício 3: Resource com filtragem contextual

Para o endpoint `GET /api/funcionarios/{id}`, crie um `FuncionarioResource` que retorna campos diferentes baseado no papel do usuário autenticado: cidadão vê apenas nome e cargo, servidor do RH vê dados pessoais e financeiros, e admin vê tudo incluindo observações internas. Escreva testes para cada nível de acesso.

### Exercício 4: Auditoria completa

Implemente o trait `Auditable` e aplique nos Models mais sensíveis do seu sistema. Configure logging de acesso para endpoints que retornam dados pessoais. Crie um endpoint de consulta de logs acessível apenas para administradores.

### Exercício 5: Compliance LGPD

Implemente os endpoints de "Meus Dados" que permitem ao cidadão: ver quais dados pessoais o sistema possui, corrigir dados incorretos, e ver o histórico de quem acessou seus dados.

---

## Recursos Adicionais

### Documentação

- [Laravel Authorization](https://laravel.com/docs/authorization) — Policies, Gates e middleware de autorização.
- [Laravel Sanctum](https://laravel.com/docs/sanctum) — Autenticação para SPAs e APIs.
- [OWASP Top 10](https://owasp.org/www-project-top-ten/) — As 10 vulnerabilidades web mais críticas, com mitigações.
- [OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/) — Vulnerabilidades específicas de APIs.

### Livros

- **Web Application Security** — Andrew Hoffman. Guia prático sobre segurança de aplicações web, cobrindo XSS, CSRF, injection e mais.
- **The Web Application Hacker's Handbook** — Dafydd Stuttard. Referência detalhada sobre como atacantes exploram vulnerabilidades — entender o ataque é o primeiro passo para a defesa.

### LGPD

- [Lei 13.709/2018](https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm) — Texto completo da LGPD.
- [Guia de Boas Práticas LGPD](https://www.gov.br/governodigital/pt-br/privacidade-e-seguranca/guias/guia_lgpd.pdf) — Guia do governo federal para adequação à LGPD em órgãos públicos.
- [ANPD](https://www.gov.br/anpd/) — Autoridade Nacional de Proteção de Dados, com orientações e resoluções.

---

## Conclusão

Segurança não é uma camada que se adiciona sobre um sistema pronto. É uma propriedade que emerge de decisões tomadas em cada linha de código: no FormRequest que valida e sanitiza input, na Policy que verifica permissões, no Resource que filtra campos sensíveis, no scope que delimita queries, no middleware que limita requisições, e no log que registra acessos.

Os princípios que guiam segurança em sistemas municipais são:

**Defesa em profundidade.** Nenhuma camada sozinha é suficiente. UUID + autenticação + Policy + scope + Resource — cada camada protege contra uma classe diferente de falha.

**Autorização no backend, sempre.** O frontend decide o que mostrar; o backend decide o que é permitido. `v-if` no Vue.js é UX. `$this->authorize()` no controller é segurança.

**Menor privilégio.** Cada usuário, cada token, cada role tem acesso apenas ao mínimo necessário para cumprir sua função. Um atendente não precisa ver folha de pagamento; um cidadão não precisa ver observações internas.

**Não confiar em input.** Todo dado que vem do frontend, da URL, de headers, de cookies — tudo é potencialmente malicioso. Valide, sanitize, e use whitelist ao invés de blacklist.

**Auditoria como padrão.** Em sistemas governamentais, toda operação em dados sensíveis deve ser rastreável: quem fez, quando, de onde, e o que mudou. Isso é requisito legal (LGPD) e boa prática de governança.

**LGPD como premissa.** Coletar apenas o necessário, reter pelo tempo mínimo, dar acesso ao titular, e proteger contra vazamento. A LGPD não é obstáculo — é o padrão mínimo de respeito aos dados dos cidadãos.

A conexão com os documentos anteriores é direta: a arquitetura do documento 4 cria as camadas onde as verificações de segurança vivem. A lógica de negócio do documento 3 encapsula regras como segregação de funções. Os testes do documento 6 verificam que proteções contra IDOR e escalação de privilégios funcionam corretamente. E as fronteiras de API do documento 2 são o ponto onde validação, rate limiting e filtragem de dados sensíveis acontecem.

Segurança não é um checklist que se completa uma vez — é uma prática contínua que evolui com o sistema.
