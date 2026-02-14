# Git e GitHub Profissional: O Fluxo de Trabalho de Elite

## Introdução
Git não é apenas uma ferramenta de backup de código; é uma ferramenta de colaboração e uma "máquina do tempo". Para um desenvolvedor sênior, a forma como você organiza seus commits e branches é tão importante quanto a qualidade do seu código.

---

## 1. Filosofia do Commit Atômico

Um commit deve fazer **apenas uma coisa**. Se você está corrigindo um bug e percebe um erro de digitação em outro arquivo, resista à tentação de incluir tudo no mesmo commit.

**Vantagens:**
- Facilita o `git revert` se algo der errado
- Simplifica o code review (commits menores são mais fáceis de revisar)
- Torna o histórico legível e navegável
- Permite `git cherry-pick` preciso

### 1.1 Conventional Commits

Siga um padrão semântico para suas mensagens. Isso permite automação (changelogs, versionamento semântico):

```
<tipo>(<escopo opcional>): <descrição>

<corpo opcional>

<rodapé opcional>
```

**Tipos principais:**

| Tipo | Descrição | Exemplo |
|------|-----------|---------|
| `feat` | Nova funcionalidade | `feat(auth): add password reset flow` |
| `fix` | Correção de bug | `fix(cart): resolve race condition on checkout` |
| `docs` | Documentação | `docs: update API endpoint examples` |
| `style` | Formatação (sem mudança de lógica) | `style: fix indentation in UserController` |
| `refactor` | Refatoração (sem mudança de comportamento) | `refactor(orders): extract validation logic` |
| `test` | Adição/correção de testes | `test: add unit tests for OrderService` |
| `chore` | Tarefas de build, CI, deps | `chore: upgrade Laravel to 11.5` |
| `perf` | Melhorias de performance | `perf(queries): add index to orders.user_id` |

### 1.2 Anatomia de uma Boa Mensagem

```bash
# Ruim
git commit -m "fix bug"
git commit -m "wip"
git commit -m "changes"

# Bom
git commit -m "fix(checkout): prevent duplicate order submission

The submit button was not being disabled after the first click,
allowing users to accidentally submit multiple orders.

Closes #142"
```

**Regras:**
1. Linha de assunto: máximo 50 caracteres, imperativo ("add" não "added")
2. Linha em branco após o assunto
3. Corpo: explique o **porquê**, não o **o quê** (o diff mostra o quê)
4. Rodapé: referências a issues, breaking changes

---

## 2. Comandos Essenciais (Revisão)

### 2.1 Fluxo Básico

```bash
# Clonar repositório
git clone git@github.com:user/repo.git
cd repo

# Verificar status
git status

# Adicionar arquivos ao staging
git add arquivo.php           # Arquivo específico
git add .                     # Todos os arquivos
git add -p                    # Interativo (por hunks)

# Commit
git commit -m "feat: add new feature"

# Push
git push origin main

# Pull (fetch + merge)
git pull origin main

# Fetch (apenas baixa, não faz merge)
git fetch origin
```

### 2.2 Branches

```bash
# Listar branches
git branch                    # Locais
git branch -r                 # Remotas
git branch -a                 # Todas

# Criar e trocar para branch
git checkout -b feature/login # Antigo
git switch -c feature/login   # Moderno (Git 2.23+)

# Trocar de branch
git checkout main             # Antigo
git switch main               # Moderno

# Deletar branch
git branch -d feature/login   # Seguro (só se merged)
git branch -D feature/login   # Forçado

# Deletar branch remota
git push origin --delete feature/login
```

---

## 3. Comandos Avançados (A "Caixa de Ferramentas" Sênior)

### 3.1 Git Rebase vs. Merge

```
# Merge: Cria um commit de merge
main:    A---B---C
              \   \
feature:       D---E---M (merge commit)

# Rebase: Reaplica commits sobre a base atualizada
main:    A---B---C
                  \
feature:           D'---E' (commits reaplicados)
```

**Quando usar cada um:**

| Situação | Recomendação |
|----------|--------------|
| Atualizar feature branch com main | `git rebase main` |
| Finalizar feature para main | `git merge feature` (ou PR) |
| Branch compartilhada com outros | `git merge` (nunca rebase) |
| Histórico linear limpo | `git rebase` |

```bash
# Rebase interativo da sua branch em cima de main
git fetch origin
git rebase origin/main

# Se houver conflitos
git status                    # Ver arquivos com conflito
# Resolver conflitos manualmente
git add arquivo-resolvido.php
git rebase --continue

# Abortar rebase se necessário
git rebase --abort
```

### 3.2 Interactive Rebase (`git rebase -i`)

A ferramenta mais poderosa para limpar seu histórico **antes** de um Pull Request.

```bash
# Rebase dos últimos 5 commits
git rebase -i HEAD~5

# Ou desde onde sua branch divergiu de main
git rebase -i main
```

**Comandos disponíveis:**

```
pick   = usar commit como está
reword = usar commit, mas editar mensagem
edit   = usar commit, mas parar para amendments
squash = fundir com commit anterior (mantém mensagem)
fixup  = fundir com commit anterior (descarta mensagem)
drop   = remover commit
```

**Exemplo: Limpar commits antes de PR:**

```bash
# Editor abre com:
pick a1b2c3d feat: add user model
pick e4f5g6h fix typo
pick h7i8j9k add user controller
pick l0m1n2o fix another typo
pick p3q4r5s add user routes

# Você edita para:
pick a1b2c3d feat: add user model
fixup e4f5g6h fix typo
pick h7i8j9k add user controller
fixup l0m1n2o fix another typo
squash p3q4r5s add user routes

# Resultado: 2 commits limpos em vez de 5
```

### 3.3 Git Stash

Salva alterações pendentes em uma "pilha" temporária.

```bash
# Guardar alterações (tracked files)
git stash

# Guardar com mensagem
git stash push -m "WIP: login feature"

# Guardar incluindo arquivos não rastreados
git stash -u

# Listar stashes
git stash list
# stash@{0}: WIP: login feature
# stash@{1}: On main: debugging

# Aplicar último stash (mantém na pilha)
git stash apply

# Aplicar e remover da pilha
git stash pop

# Aplicar stash específico
git stash apply stash@{1}

# Ver conteúdo de um stash
git stash show -p stash@{0}

# Deletar stash
git stash drop stash@{0}

# Limpar todos os stashes
git stash clear
```

### 3.4 Git Cherry-pick

Pega um commit específico de um branch e aplica em outro.

```bash
# Aplicar commit específico
git cherry-pick abc123

# Aplicar múltiplos commits
git cherry-pick abc123 def456

# Aplicar range de commits
git cherry-pick abc123^..def456

# Cherry-pick sem commit automático
git cherry-pick -n abc123

# Se houver conflito
git cherry-pick --continue
git cherry-pick --abort
```

**Caso de uso comum: Hotfix**

```bash
# Você está em feature/big-feature mas precisa de um hotfix em produção
git checkout main
git checkout -b hotfix/critical-bug
# Faz o fix, commit
git commit -m "fix: critical security issue"
# Merge em main
git checkout main
git merge hotfix/critical-bug
git push origin main

# Agora traz o fix para sua feature branch também
git checkout feature/big-feature
git cherry-pick <commit-hash-do-fix>
```

### 3.5 Git Reflog

A rede de segurança. O `reflog` registra todos os movimentos da sua HEAD.

```bash
# Ver reflog
git reflog

# Exemplo de output:
# abc123 HEAD@{0}: commit: feat: add new feature
# def456 HEAD@{1}: checkout: moving from main to feature
# ghi789 HEAD@{2}: commit: fix: bug fix
# jkl012 HEAD@{3}: rebase -i (finish): returning to refs/heads/feature
# mno345 HEAD@{4}: rebase -i (start): checkout main

# Recuperar commit "perdido" após rebase mal feito
git checkout HEAD@{4}
# Ou criar branch a partir dele
git branch recovery HEAD@{4}
```

### 3.6 Git Bisect

Encontra o commit que introduziu um bug usando busca binária.

```bash
# Iniciar bisect
git bisect start

# Marcar commit atual como ruim
git bisect bad

# Marcar commit antigo conhecido como bom
git bisect good v1.0.0

# Git faz checkout de um commit no meio
# Você testa e marca:
git bisect good  # ou
git bisect bad

# Repita até encontrar o commit culpado
# Git mostrará: "abc123 is the first bad commit"

# Finalizar
git bisect reset

# Automatizado com script de teste
git bisect start HEAD v1.0.0
git bisect run php artisan test --filter=SpecificTest
```

### 3.7 Git Worktrees

Trabalhe em múltiplas branches simultaneamente sem stash.

```bash
# Criar worktree
git worktree add ../project-hotfix hotfix/critical

# Listar worktrees
git worktree list
# /home/user/project        abc1234 [main]
# /home/user/project-hotfix def5678 [hotfix/critical]

# Trabalhar no hotfix
cd ../project-hotfix
# Fazer alterações, commit, push

# Remover worktree
git worktree remove ../project-hotfix
```

---

## 4. Configuração Profissional

### 4.1 Aliases Úteis

```bash
# ~/.gitconfig ou via comando
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.visual '!gitk'

# Aliases avançados
git config --global alias.lg "log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
git config --global alias.undo 'reset --soft HEAD~1'
git config --global alias.amend 'commit --amend --no-edit'
git config --global alias.wip 'commit -am "WIP"'
git config --global alias.branches 'branch -a --sort=-committerdate'
```

### 4.2 Arquivo .gitconfig Completo

```ini
[user]
    name = Seu Nome
    email = seu@email.com
    signingkey = YOUR_GPG_KEY_ID

[core]
    editor = code --wait
    autocrlf = input
    excludesfile = ~/.gitignore_global
    pager = delta  # ou less

[init]
    defaultBranch = main

[pull]
    rebase = true

[push]
    autoSetupRemote = true
    default = current

[fetch]
    prune = true

[merge]
    conflictstyle = diff3

[diff]
    colorMoved = default

[commit]
    gpgsign = true

[rerere]
    enabled = true

[alias]
    st = status -sb
    co = checkout
    br = branch
    ci = commit
    lg = log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
    undo = reset --soft HEAD~1
    amend = commit --amend --no-edit
```

### 4.3 .gitignore Global

```bash
# Criar arquivo
touch ~/.gitignore_global
git config --global core.excludesfile ~/.gitignore_global
```

```gitignore
# ~/.gitignore_global

# OS
.DS_Store
Thumbs.db

# IDEs
.idea/
.vscode/
*.sublime-project
*.sublime-workspace

# Temporários
*.swp
*.swo
*~
```

### 4.4 .gitignore do Projeto (Laravel)

```gitignore
# Laravel
/vendor/
/node_modules/
/public/build/
/public/hot
/public/storage
/storage/*.key
.env
.env.backup
.env.production
.phpunit.result.cache
Homestead.json
Homestead.yaml
npm-debug.log
yarn-error.log

# IDE
/.idea
/.vscode
*.sublime-project
*.sublime-workspace

# Testing
.phpunit.result.cache
/coverage/

# Build
/public/build/

# OS
.DS_Store
Thumbs.db
```

### 4.5 .gitattributes

```gitattributes
# Normalização de line endings
* text=auto

# Arquivos que devem ser sempre LF
*.php text eol=lf
*.js text eol=lf
*.css text eol=lf
*.html text eol=lf
*.json text eol=lf
*.yml text eol=lf
*.yaml text eol=lf
*.md text eol=lf

# Binários
*.png binary
*.jpg binary
*.gif binary
*.ico binary
*.pdf binary
*.zip binary

# Arquivos que não devem ser diffed
*.lock binary
package-lock.json binary

# Exportar (git archive)
.gitattributes export-ignore
.gitignore export-ignore
.github/ export-ignore
tests/ export-ignore
```

---

## 5. Estratégias de Branching (Workflows)

### 5.1 GitHub Flow (Recomendado para maioria)

Simples e eficiente. Uma branch `main` sempre estável e feature branches curtas.

```
main ─────●─────●─────●─────●─────●─────
           \         /       \     /
feature/A   ●───●───●         ●───●
```

**Regras:**
1. `main` está sempre deployável
2. Crie branches a partir de `main` para cada feature
3. Abra PR quando pronto para review
4. Merge via PR após aprovação
5. Delete branch após merge

### 5.2 Git Flow (Para releases programados)

Mais complexo. Usa múltiplas branches de longa duração.

```
main     ─────●─────────────────●─────────
              │                 │
              │    ┌────────────┘
              │    │
release  ─────┼────●────●
              │         │
develop  ●────●────●────●────●────●────●──
          \       /          \       /
feature    ●─────●            ●─────●
```

**Branches:**
- `main`: Produção, tags de versão
- `develop`: Integração contínua
- `feature/*`: Novas funcionalidades
- `release/*`: Preparação de release
- `hotfix/*`: Correções urgentes em produção

### 5.3 Trunk-Based Development

Para equipes maduras com CI/CD robusto. Branches curtíssimas (< 1 dia).

```
main ─●─●─●─●─●─●─●─●─●─●─●─●─●─●─●─
       \ /   \ /   \ /   \ /
        ●     ●     ●     ●
```

**Requisitos:**
- Feature flags para código incompleto
- Testes automatizados extensivos
- CI/CD rápido (< 10 min)
- Code review assíncrono ou pair programming

---

## 6. GitHub: Além do Código

### 6.1 Pull Requests de Alta Qualidade

**Template de PR (.github/pull_request_template.md):**

```markdown
## Descrição
<!-- Descreva as mudanças e o motivo -->

## Tipo de Mudança
- [ ] Bug fix
- [ ] Nova feature
- [ ] Breaking change
- [ ] Documentação

## Como Testar
<!-- Passos para testar as mudanças -->

## Checklist
- [ ] Código segue os padrões do projeto
- [ ] Self-review realizado
- [ ] Testes adicionados/atualizados
- [ ] Documentação atualizada (se aplicável)

## Screenshots (se aplicável)
<!-- Adicione screenshots de mudanças visuais -->

## Issues Relacionadas
Closes #123
```

### 6.2 Code Review Etiquette

**Para o Autor:**
- PRs pequenos (< 400 linhas é ideal)
- Descrição clara do contexto
- Self-review antes de pedir review
- Responda comentários prontamente

**Para o Revisor:**
- Seja construtivo, não crítico
- Use perguntas em vez de ordens
- Elogie boas soluções
- Aprove quando estiver satisfeito (não espere perfeição)

```markdown
# Bom
"O que você acha de extrair essa lógica para um método separado?
Facilitaria o teste unitário."

# Ruim
"Extraia isso para um método."
```

### 6.3 GitHub Actions (CI Básico)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  tests:
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

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: mbstring, pdo, pdo_mysql
          coverage: xdebug

      - name: Cache Composer
        uses: actions/cache@v4
        with:
          path: vendor
          key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
          restore-keys: ${{ runner.os }}-composer-

      - name: Install Dependencies
        run: composer install --no-progress --prefer-dist

      - name: Copy .env
        run: cp .env.example .env

      - name: Generate Key
        run: php artisan key:generate

      - name: Run Tests
        run: php artisan test --parallel
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: root

      - name: Run Static Analysis
        run: ./vendor/bin/phpstan analyse
```

---

## 7. Git Hooks

### 7.1 Hooks Locais

```bash
# Hooks ficam em .git/hooks/
# Renomeie .sample para ativar

# .git/hooks/pre-commit
#!/bin/sh

# Rodar linter antes de commit
./vendor/bin/pint --test

if [ $? -ne 0 ]; then
    echo "Code style issues found. Run './vendor/bin/pint' to fix."
    exit 1
fi

# .git/hooks/commit-msg
#!/bin/sh

# Validar formato de commit message (Conventional Commits)
commit_regex='^(feat|fix|docs|style|refactor|test|chore|perf)(\(.+\))?: .{1,50}'

if ! grep -qE "$commit_regex" "$1"; then
    echo "Invalid commit message format."
    echo "Expected: type(scope): description"
    echo "Example: feat(auth): add password reset"
    exit 1
fi
```

### 7.2 Husky + lint-staged (Recomendado)

```bash
npm install -D husky lint-staged
npx husky init
```

```json
// package.json
{
    "lint-staged": {
        "*.php": [
            "./vendor/bin/pint"
        ],
        "*.{js,vue}": [
            "eslint --fix",
            "prettier --write"
        ]
    }
}
```

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npx lint-staged
```

---

## 8. Segurança no Git

### 8.1 Signed Commits (GPG)

```bash
# Gerar chave GPG
gpg --full-generate-key

# Listar chaves
gpg --list-secret-keys --keyid-format=long

# Configurar Git
git config --global user.signingkey YOUR_KEY_ID
git config --global commit.gpgsign true

# Exportar chave pública (para GitHub)
gpg --armor --export YOUR_KEY_ID
```

### 8.2 SSH Keys

```bash
# Gerar chave SSH (Ed25519, mais seguro)
ssh-keygen -t ed25519 -C "seu@email.com"

# Adicionar ao ssh-agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Copiar chave pública
cat ~/.ssh/id_ed25519.pub
# Cole no GitHub: Settings > SSH Keys
```

### 8.3 Secrets Management

**NUNCA commite:**
- Arquivos `.env`
- Chaves de API
- Senhas
- Certificados privados

```bash
# Se você commitou um secret por acidente:

# 1. Revogue o secret imediatamente (no provedor)

# 2. Remova do histórico (BFG é mais rápido que filter-branch)
brew install bfg  # ou baixe o JAR

# Remover arquivo
bfg --delete-files .env

# Remover string específica
echo "API_KEY=xyz123" > secrets.txt
bfg --replace-text secrets.txt

# Limpar e forçar push
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force
```

### 8.4 Ferramentas de Detecção

- **GitGuardian**: Monitora commits públicos
- **git-secrets**: Pre-commit hook para AWS keys
- **truffleHog**: Busca secrets no histórico
- **GitHub Secret Scanning**: Nativo do GitHub

---

## 9. IA no Fluxo Git

### 9.1 Commit Messages com IA

```bash
# Usando Claude/GPT para gerar mensagem
git diff --staged | llm "Generate a conventional commit message for these changes"

# Ou com ferramenta específica
# https://github.com/Nutlope/aicommits
npx aicommits
```

### 9.2 PR Summaries

GitHub Copilot e outras ferramentas podem:
- Resumir mudanças automaticamente
- Sugerir reviewers baseado no código alterado
- Identificar potenciais problemas

### 9.3 Code Review Assistido

```bash
# Analisar PR com IA
gh pr diff 123 | llm "Review this code for potential issues, security concerns, and best practices"
```

---

## 10. Troubleshooting

### 10.1 Problemas Comuns

**"Detached HEAD"**
```bash
# Você está em um commit, não em uma branch
git checkout main  # ou
git switch main
```

**Desfazer último commit (mantendo alterações)**
```bash
git reset --soft HEAD~1
```

**Desfazer último commit (descartando alterações)**
```bash
git reset --hard HEAD~1
```

**Arquivo adicionado por engano ao staging**
```bash
git restore --staged arquivo.php
```

**Descartar alterações em arquivo**
```bash
git restore arquivo.php
```

**Conflito de merge: aceitar "ours" ou "theirs"**
```bash
git checkout --ours arquivo.php   # Sua versão
git checkout --theirs arquivo.php # Versão deles
git add arquivo.php
```

**Push rejeitado (non-fast-forward)**
```bash
git pull --rebase origin main
git push origin main
```

---

## 11. Exercícios Práticos

### Exercício 1: Limpeza de Histórico
1. Crie 5 commits em uma feature branch (inclua "WIP" e "fix typo")
2. Use `git rebase -i` para consolidar em 2 commits limpos
3. Force push para sua branch

### Exercício 2: Resolução de Conflitos
1. Crie uma branch A e modifique um arquivo
2. Crie uma branch B e modifique o mesmo arquivo de forma diferente
3. Merge A em main
4. Tente fazer merge de B em main
5. Resolva o conflito manualmente

### Exercício 3: Recuperação
1. Crie commits importantes
2. Faça um `git reset --hard HEAD~3` (simule perda)
3. Use `git reflog` para encontrar e recuperar os commits

### Exercício 4: Git Bisect
1. Clone um projeto com bugs conhecidos
2. Use `git bisect` para encontrar o commit que introduziu um bug
3. Automatize com `git bisect run`

### Exercício 5: Workflow Completo
1. Configure hooks (pre-commit, commit-msg)
2. Crie uma feature branch
3. Faça commits seguindo Conventional Commits
4. Abra um PR com descrição completa
5. Faça merge via PR

---

## Conclusão

Dominar o Git e o GitHub é sobre ter controle total sobre o ciclo de vida do software. Um histórico de commits bem estruturado é um sinal de maturidade técnica e respeito pelos seus colegas de equipe. As ferramentas avançadas como rebase interativo, bisect e reflog são o que separam desenvolvedores juniores de seniores no dia a dia.
