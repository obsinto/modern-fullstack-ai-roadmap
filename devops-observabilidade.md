# DevOps, Deployment e Observabilidade

## Introdução
Uma aplicação moderna não termina quando o código é "mergeado". O ciclo de vida continua na infraestrutura, garantindo que o sistema seja escalável, resiliente e monitorável. Este guia cobre as práticas essenciais de DevOps para o ecossistema Laravel.

---

## 1. Containerização com Docker

O Docker garante que o ambiente de desenvolvimento seja idêntico ao de produção, eliminando o clássico "na minha máquina funciona".

### 1.1 Laravel Sail (Desenvolvimento)

O Sail é uma interface CLI leve para interagir com o ambiente Docker padrão do Laravel.

```bash
# Instalar Sail em projeto existente
composer require laravel/sail --dev
php artisan sail:install

# Escolha os serviços: mysql, redis, meilisearch, etc.

# Iniciar ambiente
./vendor/bin/sail up -d

# Comandos via Sail
./vendor/bin/sail artisan migrate
./vendor/bin/sail composer require package/name
./vendor/bin/sail npm run dev
./vendor/bin/sail tinker

# Parar ambiente
./vendor/bin/sail down
```

**Alias recomendado:**
```bash
# ~/.bashrc ou ~/.zshrc
alias sail='./vendor/bin/sail'
```

### 1.2 Dockerfile de Produção (Multi-stage)

```dockerfile
# Estágio 1: Build de Assets
FROM node:20-alpine AS assets-builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY resources/ resources/
COPY vite.config.js tailwind.config.js postcss.config.js ./
RUN npm run build

# Estágio 2: Dependências PHP
FROM composer:2.7 AS composer-builder
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist

COPY . .
RUN composer dump-autoload --optimize

# Estágio 3: Imagem Final
FROM php:8.3-fpm-alpine

# Instalar extensões necessárias
RUN apk add --no-cache \
    nginx \
    supervisor \
    libpng-dev \
    libzip-dev \
    icu-dev \
    && docker-php-ext-install \
    pdo_mysql \
    gd \
    zip \
    bcmath \
    intl \
    opcache

# Configurar OPcache para produção
RUN echo "opcache.enable=1" >> /usr/local/etc/php/conf.d/opcache.ini \
    && echo "opcache.memory_consumption=256" >> /usr/local/etc/php/conf.d/opcache.ini \
    && echo "opcache.interned_strings_buffer=16" >> /usr/local/etc/php/conf.d/opcache.ini \
    && echo "opcache.max_accelerated_files=20000" >> /usr/local/etc/php/conf.d/opcache.ini \
    && echo "opcache.validate_timestamps=0" >> /usr/local/etc/php/conf.d/opcache.ini

WORKDIR /var/www

# Copiar aplicação
COPY --from=composer-builder /app/vendor vendor/
COPY --from=assets-builder /app/public/build public/build/
COPY . .

# Permissões
RUN chown -R www-data:www-data storage bootstrap/cache \
    && chmod -R 775 storage bootstrap/cache

# Configurações Nginx e Supervisor
COPY docker/nginx.conf /etc/nginx/nginx.conf
COPY docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Cache de configuração
RUN php artisan config:cache \
    && php artisan route:cache \
    && php artisan view:cache

EXPOSE 80

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

### 1.3 Docker Compose para Produção

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    environment:
      - APP_ENV=production
      - APP_DEBUG=false
    volumes:
      - storage:/var/www/storage
    depends_on:
      - mysql
      - redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/up"]
      interval: 30s
      timeout: 10s
      retries: 3

  mysql:
    image: mysql:8.0
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_DATABASE}
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx-proxy.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app

volumes:
  storage:
  mysql_data:
  redis_data:
```

---

## 2. CI/CD: Automação Total

Continuous Integration (CI) e Continuous Deployment (CD) são fundamentais para entregas rápidas e seguras.

### 2.1 GitHub Actions (Pipeline Completo)

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  PHP_VERSION: '8.3'
  NODE_VERSION: '20'

jobs:
  # Job 1: Testes e Análise Estática
  test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3

      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd="redis-cli ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
          extensions: mbstring, pdo, pdo_mysql, redis, gd, zip
          coverage: xdebug

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Cache Composer
        uses: actions/cache@v4
        with:
          path: vendor
          key: composer-${{ hashFiles('**/composer.lock') }}
          restore-keys: composer-

      - name: Install PHP Dependencies
        run: composer install --no-progress --prefer-dist

      - name: Install Node Dependencies
        run: npm ci

      - name: Build Assets
        run: npm run build

      - name: Prepare Environment
        run: |
          cp .env.example .env
          php artisan key:generate

      - name: Run PHPStan
        run: ./vendor/bin/phpstan analyse --memory-limit=2G

      - name: Run Pint (Code Style)
        run: ./vendor/bin/pint --test

      - name: Run Tests
        run: php artisan test --parallel --coverage-clover coverage.xml
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: root
          REDIS_HOST: 127.0.0.1

      - name: Upload Coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: coverage.xml
          token: ${{ secrets.CODECOV_TOKEN }}

  # Job 2: Build e Push da Imagem Docker
  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # Job 3: Deploy para Produção
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - name: Deploy to Server
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /var/www/app
            docker compose pull
            docker compose up -d --remove-orphans
            docker compose exec -T app php artisan migrate --force
            docker compose exec -T app php artisan config:cache
            docker compose exec -T app php artisan route:cache
            docker compose exec -T app php artisan view:cache
            docker compose exec -T app php artisan queue:restart
```

### 2.2 Estratégias de Deploy

#### Blue-Green Deployment

Dois ambientes idênticos. Deploy no "green" enquanto "blue" serve tráfego, depois troca.

```bash
# Estrutura de diretórios
/var/www/
├── blue/     # Ambiente atual
├── green/    # Ambiente novo
└── current -> blue  # Symlink

# Script de deploy
#!/bin/bash
NEW_ENV=$([ "$(readlink current)" = "blue" ] && echo "green" || echo "blue")

# Deploy no ambiente inativo
cd /var/www/$NEW_ENV
git pull
composer install --no-dev
php artisan migrate --force

# Trocar symlink (atômico)
ln -sfn /var/www/$NEW_ENV /var/www/current

# Recarregar
sudo systemctl reload php-fpm
sudo systemctl reload nginx
```

#### Rolling Deployment (Kubernetes)

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
        - name: app
          image: ghcr.io/user/app:latest
          readinessProbe:
            httpGet:
              path: /up
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /up
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 20
```

---

## 3. Estratégias de Deployment

### 3.1 Laravel Forge (PaaS)

A forma mais comum de deploy para servidores VPS. O Forge gerencia Nginx, SSL, Queues e Cronjobs automaticamente.

**Recursos:**
- Provisionamento automático de servidores
- Deploy via Git (push to deploy)
- SSL automático (Let's Encrypt)
- Gerenciamento de queues e scheduler
- Monitoring básico

### 3.2 Laravel Vapor (Serverless)

Deploy para AWS Lambda. Escalabilidade infinita sem gerenciar servidores.

```yaml
# vapor.yml
id: 12345
name: my-app
environments:
  production:
    memory: 1024
    cli-memory: 512
    runtime: php-8.3:al2
    build:
      - 'composer install --no-dev'
      - 'npm ci && npm run build'
    deploy:
      - 'php artisan migrate --force'
      - 'php artisan config:cache'
    database: my-database
    cache: my-cache
    queues:
      - default
      - emails
```

**Considerações:**
- Limite de 15 min por execução Lambda
- Cold starts (mitigado com provisioned concurrency)
- Custo variável baseado em uso
- Excelente para APIs e workloads variáveis

### 3.3 Kubernetes Básico

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: laravel-app

---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: laravel-app
data:
  APP_ENV: "production"
  APP_DEBUG: "false"
  LOG_CHANNEL: "stderr"

---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: laravel-app
type: Opaque
stringData:
  APP_KEY: "base64:your-key-here"
  DB_PASSWORD: "your-db-password"

---
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-app
  namespace: laravel-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: laravel
  template:
    metadata:
      labels:
        app: laravel
    spec:
      containers:
        - name: app
          image: ghcr.io/user/laravel-app:latest
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /up
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /up
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 20

---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: laravel-service
  namespace: laravel-app
spec:
  selector:
    app: laravel
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: laravel-ingress
  namespace: laravel-app
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: laravel-service
                port:
                  number: 80
```

### 3.4 Horizontal Pod Autoscaler

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: laravel-hpa
  namespace: laravel-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: laravel-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## 4. Observabilidade e Monitoramento

Você não pode consertar o que não consegue ver.

### 4.1 Os Três Pilares

```
┌─────────────────────────────────────────────────────────┐
│                   OBSERVABILIDADE                        │
├───────────────────┬─────────────────┬───────────────────┤
│       LOGS        │     MÉTRICAS    │      TRACES       │
│                   │                 │                   │
│ O que aconteceu?  │ Como está?      │ Por que demorou?  │
│                   │                 │                   │
│ Texto detalhado   │ Números ao      │ Caminho da        │
│ de eventos        │ longo do tempo  │ requisição        │
│                   │                 │                   │
│ Sentry, Loki      │ Prometheus,     │ Jaeger,           │
│ ELK Stack         │ Datadog         │ OpenTelemetry     │
└───────────────────┴─────────────────┴───────────────────┘
```

### 4.2 Logs Estruturados

Evite logs de texto puro. Use JSON para facilitar busca e análise.

```php
// config/logging.php
'channels' => [
    'production' => [
        'driver' => 'stack',
        'channels' => ['daily', 'stderr'],
    ],

    'stderr' => [
        'driver' => 'monolog',
        'handler' => StreamHandler::class,
        'formatter' => JsonFormatter::class,
        'with' => [
            'stream' => 'php://stderr',
        ],
    ],
],

// Uso no código
Log::info('Order created', [
    'order_id' => $order->id,
    'user_id' => $order->user_id,
    'total' => $order->total,
    'items_count' => $order->items->count(),
]);

// Output JSON
{
    "message": "Order created",
    "context": {
        "order_id": 123,
        "user_id": 456,
        "total": 99.99,
        "items_count": 3
    },
    "level": 200,
    "level_name": "INFO",
    "datetime": "2024-01-15T10:30:00+00:00"
}
```

### 4.3 Níveis de Log Apropriados

| Nível | Quando Usar |
|-------|-------------|
| `emergency` | Sistema inutilizável |
| `alert` | Ação imediata necessária |
| `critical` | Condições críticas |
| `error` | Erros que não quebram o sistema |
| `warning` | Situações anormais mas não erros |
| `notice` | Eventos normais mas significativos |
| `info` | Eventos informativos |
| `debug` | Informações detalhadas para debug |

```php
// Exemplos práticos
Log::emergency('Database server is down');
Log::alert('Payment gateway not responding after 5 retries');
Log::critical('Memory usage at 95%');
Log::error('Failed to send email', ['exception' => $e->getMessage()]);
Log::warning('API rate limit at 80%');
Log::notice('New user registered', ['user_id' => $user->id]);
Log::info('Order shipped', ['order_id' => $order->id]);
Log::debug('Cache hit for user preferences', ['user_id' => $userId]);
```

### 4.4 Sentry (Monitoramento de Erros)

```bash
composer require sentry/sentry-laravel
php artisan sentry:publish --dsn=YOUR_DSN
```

```php
// config/sentry.php
return [
    'dsn' => env('SENTRY_LARAVEL_DSN'),
    'release' => env('SENTRY_RELEASE', trim(exec('git rev-parse --short HEAD'))),
    'environment' => env('APP_ENV', 'production'),
    'traces_sample_rate' => 0.2, // 20% das requisições
    'profiles_sample_rate' => 0.1, // 10% para profiling
];

// Adicionar contexto extra
Sentry\configureScope(function (Scope $scope) use ($user) {
    $scope->setUser([
        'id' => $user->id,
        'email' => $user->email,
        'username' => $user->name,
    ]);
});

// Capturar exceção manualmente
try {
    riskyOperation();
} catch (Exception $e) {
    Sentry\captureException($e);
    // Continue execution
}
```

### 4.5 Laravel Pulse (Monitoramento Nativo)

Dashboard nativo do Laravel para monitorar performance em tempo real.

```bash
composer require laravel/pulse
php artisan vendor:publish --provider="Laravel\Pulse\PulseServiceProvider"
php artisan migrate
```

```php
// config/pulse.php
return [
    'ingest' => [
        'trim_lottery' => [1, 1000], // Limpa dados antigos
    ],

    'recorders' => [
        // Requisições lentas
        Recorders\SlowRequests::class => [
            'enabled' => true,
            'threshold' => 1000, // ms
        ],

        // Jobs lentos
        Recorders\SlowJobs::class => [
            'enabled' => true,
            'threshold' => 5000, // ms
        ],

        // Queries lentas
        Recorders\SlowQueries::class => [
            'enabled' => true,
            'threshold' => 100, // ms
        ],

        // Exceções
        Recorders\Exceptions::class => [
            'enabled' => true,
        ],

        // Cache
        Recorders\CacheInteractions::class => [
            'enabled' => true,
        ],
    ],
];
```

Acesse em `/pulse` (proteja com middleware de autenticação).

### 4.6 Métricas com Prometheus

```php
// app/Providers/PrometheusServiceProvider.php
use Prometheus\CollectorRegistry;
use Prometheus\Storage\Redis;

class PrometheusServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->singleton(CollectorRegistry::class, function () {
            return new CollectorRegistry(new Redis([
                'host' => config('database.redis.default.host'),
                'port' => config('database.redis.default.port'),
            ]));
        });
    }
}

// Middleware para métricas de requisição
class PrometheusMiddleware
{
    public function handle($request, Closure $next)
    {
        $start = microtime(true);
        $response = $next($request);
        $duration = microtime(true) - $start;

        $histogram = app(CollectorRegistry::class)
            ->getOrRegisterHistogram(
                'app',
                'http_request_duration_seconds',
                'HTTP request duration',
                ['method', 'endpoint', 'status']
            );

        $histogram->observe(
            $duration,
            [$request->method(), $request->path(), $response->status()]
        );

        return $response;
    }
}

// Endpoint /metrics
Route::get('/metrics', function () {
    $renderer = new RenderTextFormat();
    $registry = app(CollectorRegistry::class);

    return response($renderer->render($registry->getMetricFamilySamples()))
        ->header('Content-Type', RenderTextFormat::MIME_TYPE);
});
```

### 4.7 Health Checks

```php
// Laravel 11+ (bootstrap/app.php)
->withRouting(
    health: '/up',
)

// Health check customizado
// routes/web.php
Route::get('/health', function () {
    $checks = [
        'database' => fn() => DB::connection()->getPdo(),
        'redis' => fn() => Redis::ping(),
        'storage' => fn() => Storage::exists('.gitignore'),
        'queue' => fn() => Queue::size('default') < 1000,
    ];

    $results = [];
    $healthy = true;

    foreach ($checks as $name => $check) {
        try {
            $check();
            $results[$name] = 'ok';
        } catch (Exception $e) {
            $results[$name] = 'failed: ' . $e->getMessage();
            $healthy = false;
        }
    }

    return response()->json([
        'status' => $healthy ? 'healthy' : 'unhealthy',
        'checks' => $results,
        'timestamp' => now()->toISOString(),
    ], $healthy ? 200 : 503);
});
```

---

## 5. Backup e Disaster Recovery

### 5.1 Backup Automatizado

```php
// app/Console/Commands/BackupDatabase.php
class BackupDatabase extends Command
{
    protected $signature = 'backup:database';

    public function handle()
    {
        $filename = 'backup-' . now()->format('Y-m-d-H-i-s') . '.sql.gz';

        $command = sprintf(
            'mysqldump -h %s -u %s -p%s %s | gzip > %s',
            config('database.connections.mysql.host'),
            config('database.connections.mysql.username'),
            config('database.connections.mysql.password'),
            config('database.connections.mysql.database'),
            storage_path('backups/' . $filename)
        );

        exec($command);

        // Upload para S3
        Storage::disk('s3')->put(
            'backups/' . $filename,
            file_get_contents(storage_path('backups/' . $filename))
        );

        // Limpar backups locais antigos
        $this->cleanOldBackups();

        $this->info("Backup created: {$filename}");
    }
}

// Schedule
// app/Console/Kernel.php
$schedule->command('backup:database')->dailyAt('03:00');
```

### 5.2 Usando Laravel Backup (Spatie)

```bash
composer require spatie/laravel-backup
php artisan vendor:publish --provider="Spatie\Backup\BackupServiceProvider"
```

```php
// config/backup.php
return [
    'backup' => [
        'name' => env('APP_NAME', 'laravel-backup'),
        'source' => [
            'files' => [
                'include' => [base_path()],
                'exclude' => [
                    base_path('vendor'),
                    base_path('node_modules'),
                    storage_path('logs'),
                ],
            ],
            'databases' => ['mysql'],
        ],
        'destination' => [
            'disks' => ['s3'],
        ],
    ],
    'cleanup' => [
        'strategy' => DefaultStrategy::class,
        'default_strategy' => [
            'keep_all_backups_for_days' => 7,
            'keep_daily_backups_for_days' => 30,
            'keep_weekly_backups_for_weeks' => 8,
            'keep_monthly_backups_for_months' => 4,
        ],
    ],
];
```

---

## 6. Segurança em Produção

### 6.1 Hardening do Servidor

```bash
# Desabilitar funções perigosas no php.ini
disable_functions = exec,passthru,shell_exec,system,proc_open,popen

# Expor menos informações
expose_php = Off
display_errors = Off
log_errors = On

# Limites
memory_limit = 256M
max_execution_time = 30
upload_max_filesize = 10M
post_max_size = 10M
```

### 6.2 Headers de Segurança (Nginx)

```nginx
# /etc/nginx/conf.d/security-headers.conf
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

### 6.3 Rate Limiting

```php
// app/Providers/RouteServiceProvider.php
RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});

RateLimiter::for('login', function (Request $request) {
    return [
        Limit::perMinute(5)->by($request->ip()),
        Limit::perMinute(10)->by($request->input('email')),
    ];
});

// routes/api.php
Route::middleware(['throttle:api'])->group(function () {
    // ...
});
```

### 6.4 Gerenciamento de Secrets

```bash
# AWS Secrets Manager
aws secretsmanager get-secret-value --secret-id prod/app/secrets

# HashiCorp Vault
vault kv get secret/app/production

# Laravel com AWS
composer require aws/aws-sdk-php

// Carregar secrets no boot
$client = new SecretsManagerClient([...]);
$result = $client->getSecretValue(['SecretId' => 'prod/app']);
$secrets = json_decode($result['SecretString'], true);

foreach ($secrets as $key => $value) {
    putenv("{$key}={$value}");
}
```

---

## 7. Supervisor e Processos

### 7.1 Configuração do Supervisor

```ini
; /etc/supervisor/conf.d/laravel-worker.conf
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/app/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=8
redirect_stderr=true
stdout_logfile=/var/www/app/storage/logs/worker.log
stopwaitsecs=3600

[program:laravel-scheduler]
process_name=%(program_name)s
command=/bin/bash -c "while [ true ]; do (php /var/www/app/artisan schedule:run --verbose --no-interaction &); sleep 60; done"
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/var/www/app/storage/logs/scheduler.log

[program:reverb]
process_name=%(program_name)s
command=php /var/www/app/artisan reverb:start --host=0.0.0.0 --port=8080
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/var/www/app/storage/logs/reverb.log
```

```bash
# Comandos do Supervisor
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start laravel-worker:*
sudo supervisorctl stop laravel-worker:*
sudo supervisorctl restart laravel-worker:*
sudo supervisorctl status
```

---

## 8. Exercícios Práticos

### Exercício 1: Docker Multi-stage
1. Crie um Dockerfile multi-stage para seu projeto Laravel
2. Otimize para tamanho mínimo de imagem
3. Configure OPcache para produção
4. Rode localmente e verifique performance

### Exercício 2: CI/CD Pipeline
1. Configure GitHub Actions para seu projeto
2. Inclua: testes, análise estática, build de imagem
3. Configure deploy automático para staging
4. Adicione aprovação manual para produção

### Exercício 3: Observabilidade
1. Integre Sentry para monitoramento de erros
2. Configure Laravel Pulse
3. Adicione métricas customizadas
4. Crie um endpoint de health check completo

### Exercício 4: Backup e Recovery
1. Configure backup automático com Spatie Backup
2. Faça upload para S3 ou outro storage remoto
3. Teste o processo de restauração
4. Documente o processo de disaster recovery

### Exercício 5: Kubernetes
1. Crie manifestos K8s para sua aplicação
2. Configure Ingress com TLS
3. Implemente HPA (Horizontal Pod Autoscaler)
4. Faça um rolling deployment

---

## Conclusão

O DevOps no Laravel evoluiu para ser acessível mas extremamente robusto. Docker garante consistência entre ambientes, CI/CD automatiza entregas, e ferramentas de observabilidade como Pulse e Sentry permitem que você identifique e resolva problemas rapidamente. Seguir estas práticas garante que sua aplicação suporte o crescimento do tráfego mantendo a estabilidade e a facilidade de depuração.
