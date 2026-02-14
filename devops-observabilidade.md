# DevOps, Deployment e Observabilidade

## Introdução
Uma aplicação moderna não termina quando o código é "mergeado". O ciclo de vida continua na infraestrutura, garantindo que o sistema seja escalável, resiliente e monitorável. Este guia cobre as práticas essenciais de DevOps para o ecossistema Laravel.

---

## 1. Containerização com Docker

O Docker garante que o ambiente de desenvolvimento seja idêntico ao de produção, eliminando o clássico "na minha máquina funciona".

### 1.1 Laravel Sail (Desenvolvimento)
O Sail é uma interface CLI leve para interagir com o ambiente Docker padrão do Laravel.
- **Vantagem**: Configuração instantânea de PHP, MySQL, Redis, Meilisearch e Selenium.
- **Comando**: `./vendor/bin/sail up`

### 1.2 Dockerfile de Produção (Otimizado)
Para produção, usamos imagens multi-stage para reduzir o tamanho final e aumentar a segurança.

```dockerfile
# Estágio 1: Build de Assets
FROM node:18 AS assets-builder
WORKDIR /app
COPY . .
RUN npm install && npm run build

# Estágio 2: Aplicação PHP
FROM php:8.3-fpm-alpine
WORKDIR /var/www

# Instalar extensões e dependências de sistema
RUN apk add --no-cache nginx supervisor mysql-client libpng-dev libzip-dev
RUN docker-php-ext-install pdo_mysql gd zip bcmath

# Copiar código e assets buildados
COPY --from=assets-builder /app /var/www
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Permissões
RUN chown -R www-data:www-data /var/www/storage /var/www/cache
```

---

## 2. CI/CD: Automação Total

Continuous Integration (CI) e Continuous Deployment (CD) são fundamentais para entregas rápidas e seguras.

### 2.1 GitHub Actions (Exemplo de Pipeline)
Localizado em `.github/workflows/main.yml`:

```yaml
name: CI-CD
on: [push]
jobs:
  tests:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: testing
        ports: ['3306:3306']
    steps:
      - uses: actions/checkout@v3
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
      - run: composer install
      - run: php artisan test --parallel
```

---

## 3. Estratégias de Deployment

### 3.1 Laravel Forge (PaaS)
A forma mais comum de deploy para servidores VPS (DigitalOcean, AWS, Hetzner). O Forge gerencia Nginx, Certificados SSL, Queues e Cronjobs automaticamente.

### 3.2 Laravel Vapor (Serverless)
Deploy para AWS Lambda. Escalabilidade infinita sem gerenciar servidores, mas exige uma arquitetura que respeite o limite de execução do Lambda.

### 3.3 Blue-Green Deployment
Técnica para evitar downtime. Você sobe a versão nova em um ambiente paralelo e, após os testes passarem, troca o tráfego do roteador para o novo ambiente.

---

## 4. Observabilidade e Monitoramento

Você não pode consertar o que não consegue ver.

### 4.1 Logs Estruturados
Evite logs de texto puro. Use o driver `monolog` para gerar logs em JSON, facilitando a busca em ferramentas como **ELK Stack** ou **Loki**.

### 4.2 Sentry (Tratamento de Erros)
O Sentry captura exceções em tempo real, informando o stack trace, o usuário afetado e as variáveis de ambiente no momento do erro.

### 4.3 Laravel Pulse (Saúde do Servidor)
Uma dashboard nativa do Laravel para monitorar:
- Requisições lentas.
- Jobs falhando.
- Uso de CPU e Memória.
- Queries pesadas no banco de dados.

### 4.4 Health Checks
Implemente endpoints de `/health` para que o balanceador de carga saiba se a aplicação está pronta para receber tráfego.

```php
// No Laravel 11 (bootstrap/app.php)
->withRouting(
    health: '/up',
)
```

---

## 5. Segurança em Produção

- **Segredos**: Nunca commite o `.env`. Use gerenciadores de segredos (AWS Secrets Manager, HashiCorp Vault).
- **Hardening**: Desabilite funções PHP perigosas (`exec`, `passthru`) no `php.ini`.
- **WAF**: Use Cloudflare ou AWS WAF para proteção contra ataques DDoS e SQL Injection na borda.

---

## Conclusão
O DevOps no Laravel evoluiu para ser acessível mas extremamente robusto. Seguir estas práticas garante que sua aplicação suporte o crescimento do tráfego mantendo a estabilidade e a facilidade de depuração.
