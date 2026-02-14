# Fundamentos de HTML, CSS e Tailwind CSS

## Introdução
No desenvolvimento moderno, o frontend não é apenas sobre "fazer bonito", mas sobre **usabilidade, acessibilidade e performance**. O Tailwind CSS revolucionou a forma como estilizamos aplicações, movendo-nos do "CSS baseado em componentes" para o "CSS utilitário".

---

## 1. HTML5 Semântico: A Base

HTML semântico não é apenas para SEO; é para acessibilidade (leitores de tela) e para que o navegador entenda a estrutura do seu conteúdo.

### 1.1 Elementos Estruturais

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Aplicação Moderna</title>
</head>
<body>
    <header>
        <nav aria-label="Navegação principal">
            <ul>
                <li><a href="/">Home</a></li>
                <li><a href="/sobre">Sobre</a></li>
            </ul>
        </nav>
    </header>

    <main>
        <article>
            <h1>Título Principal</h1>
            <section>
                <h2>Seção do Artigo</h2>
                <p>Conteúdo...</p>
            </section>
        </article>

        <aside>
            <h2>Conteúdo Relacionado</h2>
        </aside>
    </main>

    <footer>
        <p>&copy; 2024 Minha Empresa</p>
    </footer>
</body>
</html>
```

### 1.2 Quando Usar Cada Elemento

| Elemento | Uso Correto |
|----------|-------------|
| `<header>` | Cabeçalho da página ou de uma seção |
| `<nav>` | Navegação principal ou secundária |
| `<main>` | Conteúdo principal (único por página) |
| `<article>` | Conteúdo independente e auto-contido |
| `<section>` | Agrupamento temático de conteúdo |
| `<aside>` | Conteúdo tangencialmente relacionado |
| `<footer>` | Rodapé da página ou de uma seção |
| `<figure>` | Imagens, diagramas, código com legenda |

### 1.3 Acessibilidade (ARIA)

```html
<!-- Botão com estado -->
<button
    aria-pressed="false"
    aria-label="Ativar modo escuro"
    onclick="toggleDarkMode()"
>
    <svg aria-hidden="true">...</svg>
</button>

<!-- Região dinâmica (live region) -->
<div aria-live="polite" aria-atomic="true" id="notifications">
    <!-- Notificações aparecem aqui e são anunciadas -->
</div>

<!-- Skip link para acessibilidade -->
<a href="#main-content" class="sr-only focus:not-sr-only">
    Pular para o conteúdo principal
</a>

<!-- Formulário acessível -->
<form>
    <div>
        <label for="email">E-mail</label>
        <input
            type="email"
            id="email"
            name="email"
            aria-describedby="email-hint"
            required
        >
        <small id="email-hint">Nunca compartilharemos seu e-mail.</small>
    </div>
</form>
```

---

## 2. CSS Moderno: Flexbox e Grid

Antes de usar Tailwind, você deve entender os dois pilares do layout:

### 2.1 Flexbox (Unidimensional)

Ideal para alinhar itens em uma linha ou coluna (menus, barras laterais, cards em linha).

```css
/* Container flex */
.container {
    display: flex;
    flex-direction: row;        /* row | column | row-reverse | column-reverse */
    justify-content: center;    /* flex-start | flex-end | center | space-between | space-around | space-evenly */
    align-items: center;        /* flex-start | flex-end | center | stretch | baseline */
    gap: 1rem;                  /* Espaço entre itens */
    flex-wrap: wrap;            /* nowrap | wrap | wrap-reverse */
}

/* Itens flex */
.item {
    flex: 1;                    /* grow shrink basis */
    flex-grow: 1;               /* Quanto pode crescer */
    flex-shrink: 0;             /* Quanto pode encolher */
    flex-basis: 200px;          /* Tamanho base */
    align-self: flex-start;     /* Override do align-items para este item */
    order: 2;                   /* Ordem de exibição */
}
```

#### Exemplo Prático: Navbar Responsiva

```html
<nav class="navbar">
    <div class="logo">Logo</div>
    <ul class="nav-links">
        <li><a href="#">Home</a></li>
        <li><a href="#">Sobre</a></li>
        <li><a href="#">Contato</a></li>
    </ul>
    <button class="cta">Entrar</button>
</nav>

<style>
.navbar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1rem 2rem;
}

.nav-links {
    display: flex;
    gap: 2rem;
    list-style: none;
}

/* Mobile */
@media (max-width: 768px) {
    .navbar {
        flex-direction: column;
        gap: 1rem;
    }
}
</style>
```

### 2.2 CSS Grid (Bidimensional)

Ideal para layouts de página completos, galerias e estruturas complexas.

```css
/* Container grid */
.grid-container {
    display: grid;
    grid-template-columns: repeat(3, 1fr);           /* 3 colunas iguais */
    grid-template-columns: 200px 1fr 200px;          /* Sidebar fixa, conteúdo flexível */
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); /* Responsivo automático */
    grid-template-rows: auto 1fr auto;               /* Header, main, footer */
    gap: 1rem;                                       /* row-gap e column-gap */
    grid-template-areas:
        "header header header"
        "sidebar main aside"
        "footer footer footer";
}

/* Itens grid */
.header { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main { grid-area: main; }
.aside { grid-area: aside; }
.footer { grid-area: footer; }

/* Span de colunas/linhas */
.featured {
    grid-column: span 2;        /* Ocupa 2 colunas */
    grid-row: 1 / 3;            /* Da linha 1 até a 3 */
}
```

#### Exemplo Prático: Layout de Dashboard

```html
<div class="dashboard">
    <header class="header">Header</header>
    <nav class="sidebar">Sidebar</nav>
    <main class="content">
        <div class="card">Card 1</div>
        <div class="card">Card 2</div>
        <div class="card featured">Card Destaque</div>
        <div class="card">Card 3</div>
    </main>
</div>

<style>
.dashboard {
    display: grid;
    grid-template-columns: 250px 1fr;
    grid-template-rows: 60px 1fr;
    grid-template-areas:
        "sidebar header"
        "sidebar content";
    min-height: 100vh;
}

.header { grid-area: header; }
.sidebar { grid-area: sidebar; }
.content {
    grid-area: content;
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
    gap: 1rem;
    padding: 1rem;
}

.featured {
    grid-column: span 2;
}

@media (max-width: 768px) {
    .dashboard {
        grid-template-columns: 1fr;
        grid-template-areas:
            "header"
            "content";
    }
    .sidebar { display: none; }
    .featured { grid-column: span 1; }
}
</style>
```

---

## 3. Tailwind CSS: O Padrão Utilitário

O Tailwind permite que você escreva estilos diretamente nas suas classes HTML, evitando que você "fuja" para um arquivo CSS separado e perca o contexto.

### 3.1 Por que Tailwind?

- **Velocidade:** Não precisa inventar nomes de classes (`.user-card-inner-wrapper`).
- **Consistência:** Você usa uma escala pré-definida de cores, espaçamentos e fontes.
- **Tamanho do Bundle:** O Tailwind remove todo o CSS não utilizado em produção (JIT/Purge).
- **Manutenção:** O estilo está junto do HTML, facilitando refatorações.

### 3.2 Escala de Espaçamento

O Tailwind usa uma escala consistente baseada em `rem`:

| Classe | Valor | Pixels (base 16px) |
|--------|-------|-------------------|
| `p-0` | 0 | 0px |
| `p-1` | 0.25rem | 4px |
| `p-2` | 0.5rem | 8px |
| `p-4` | 1rem | 16px |
| `p-6` | 1.5rem | 24px |
| `p-8` | 2rem | 32px |
| `p-12` | 3rem | 48px |
| `p-16` | 4rem | 64px |

### 3.3 Conceitos Chave

```html
<!-- Espaçamento -->
<div class="p-4 m-2 space-x-4 space-y-2">
    <!-- p = padding, m = margin -->
    <!-- px, py = horizontal/vertical -->
    <!-- space-x/y = gap entre children -->
</div>

<!-- Tipografia -->
<p class="text-xl font-bold text-slate-900 leading-relaxed tracking-wide">
    <!-- text-{size}: xs, sm, base, lg, xl, 2xl... -->
    <!-- font-{weight}: thin, light, normal, medium, semibold, bold, black -->
    <!-- leading-{spacing}: tight, snug, normal, relaxed, loose -->
</p>

<!-- Cores -->
<div class="bg-blue-500 text-white border-2 border-blue-700">
    <!-- Escala: 50, 100, 200, 300, 400, 500, 600, 700, 800, 900, 950 -->
    <!-- Paletas: slate, gray, zinc, red, orange, amber, yellow, lime, green, emerald, teal, cyan, sky, blue, indigo, violet, purple, fuchsia, pink, rose -->
</div>

<!-- Layout -->
<div class="flex items-center justify-between gap-4">
    <div class="grid grid-cols-3 gap-4">
        <div class="col-span-2">Ocupa 2 colunas</div>
    </div>
</div>

<!-- Dimensões -->
<div class="w-full h-screen max-w-md min-h-[500px]">
    <!-- w/h: full, screen, auto, 1/2, 1/3, 1/4, números (w-64 = 16rem) -->
    <!-- Valores arbitrários: [500px], [50vh], [calc(100%-2rem)] -->
</div>

<!-- Bordas e Sombras -->
<div class="rounded-lg border border-gray-200 shadow-md">
    <!-- rounded: none, sm, md, lg, xl, 2xl, full -->
    <!-- shadow: sm, md, lg, xl, 2xl -->
</div>
```

### 3.4 Responsividade (Mobile First)

O Tailwind é **mobile first**. Classes sem prefixo aplicam a mobile, prefixos aplicam a partir de breakpoints:

| Prefixo | Breakpoint | Largura Mínima |
|---------|------------|----------------|
| (nenhum) | Mobile | 0px |
| `sm:` | Small | 640px |
| `md:` | Medium | 768px |
| `lg:` | Large | 1024px |
| `xl:` | Extra Large | 1280px |
| `2xl:` | 2X Large | 1536px |

```html
<!-- Mobile: 1 coluna, Tablet: 2 colunas, Desktop: 4 colunas -->
<div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
    <div>Item</div>
</div>

<!-- Mobile: stack vertical, Desktop: horizontal -->
<div class="flex flex-col md:flex-row gap-4">
    <div class="w-full md:w-1/3">Sidebar</div>
    <div class="w-full md:w-2/3">Conteúdo</div>
</div>

<!-- Esconder/mostrar -->
<div class="hidden md:block">Só aparece em desktop</div>
<div class="md:hidden">Só aparece em mobile</div>
```

### 3.5 Estados e Interatividade

```html
<!-- Hover, Focus, Active -->
<button class="
    bg-blue-600
    hover:bg-blue-700
    focus:ring-2
    focus:ring-blue-500
    focus:ring-offset-2
    active:bg-blue-800
    disabled:opacity-50
    disabled:cursor-not-allowed
    transition-colors
    duration-200
">
    Botão
</button>

<!-- Focus Visible (apenas para teclado) -->
<a href="#" class="focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-500">
    Link acessível
</a>

<!-- Group: Estilizar children baseado no pai -->
<div class="group cursor-pointer p-4 hover:bg-gray-100">
    <h3 class="text-gray-900 group-hover:text-blue-600">Título</h3>
    <p class="text-gray-500 group-hover:text-gray-700">Descrição</p>
</div>

<!-- Peer: Estilizar siblings -->
<input type="checkbox" id="toggle" class="peer sr-only">
<label for="toggle" class="cursor-pointer">Toggle</label>
<div class="hidden peer-checked:block">
    Conteúdo visível quando checkbox está marcado
</div>
```

---

## 4. Formulários com Tailwind

### 4.1 Inputs Estilizados

```html
<div class="space-y-4">
    <!-- Input básico -->
    <div>
        <label for="name" class="block text-sm font-medium text-gray-700 mb-1">
            Nome
        </label>
        <input
            type="text"
            id="name"
            class="
                w-full px-3 py-2
                border border-gray-300 rounded-lg
                focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent
                placeholder-gray-400
            "
            placeholder="Seu nome"
        >
    </div>

    <!-- Input com erro -->
    <div>
        <label for="email" class="block text-sm font-medium text-gray-700 mb-1">
            E-mail
        </label>
        <input
            type="email"
            id="email"
            class="
                w-full px-3 py-2
                border border-red-500 rounded-lg
                focus:outline-none focus:ring-2 focus:ring-red-500
                bg-red-50
            "
            aria-invalid="true"
            aria-describedby="email-error"
        >
        <p id="email-error" class="mt-1 text-sm text-red-600">
            Por favor, insira um e-mail válido.
        </p>
    </div>

    <!-- Select -->
    <div>
        <label for="country" class="block text-sm font-medium text-gray-700 mb-1">
            País
        </label>
        <select
            id="country"
            class="
                w-full px-3 py-2
                border border-gray-300 rounded-lg
                focus:outline-none focus:ring-2 focus:ring-blue-500
                bg-white
            "
        >
            <option value="">Selecione...</option>
            <option value="br">Brasil</option>
            <option value="us">Estados Unidos</option>
        </select>
    </div>

    <!-- Checkbox -->
    <div class="flex items-center gap-2">
        <input
            type="checkbox"
            id="terms"
            class="
                w-4 h-4
                text-blue-600
                border-gray-300 rounded
                focus:ring-blue-500
            "
        >
        <label for="terms" class="text-sm text-gray-700">
            Aceito os termos de uso
        </label>
    </div>
</div>
```

### 4.2 Validação Visual com Peer

```html
<div>
    <label for="password" class="block text-sm font-medium text-gray-700 mb-1">
        Senha
    </label>
    <input
        type="password"
        id="password"
        required
        minlength="8"
        class="
            peer
            w-full px-3 py-2
            border border-gray-300 rounded-lg
            focus:outline-none focus:ring-2 focus:ring-blue-500
            invalid:border-red-500 invalid:focus:ring-red-500
        "
    >
    <p class="mt-1 text-sm text-gray-500 peer-invalid:text-red-600">
        Mínimo de 8 caracteres
    </p>
</div>
```

---

## 5. Animações e Transições

### 5.1 Transições

```html
<!-- Transição suave de cores -->
<button class="
    bg-blue-600 text-white px-4 py-2 rounded
    transition-colors duration-200 ease-in-out
    hover:bg-blue-700
">
    Hover me
</button>

<!-- Transição de transform -->
<div class="
    transition-transform duration-300 ease-out
    hover:scale-105 hover:-translate-y-1
">
    Card com lift effect
</div>

<!-- Múltiplas propriedades -->
<div class="
    transition-all duration-300
    opacity-0 translate-y-4
    group-hover:opacity-100 group-hover:translate-y-0
">
    Fade in + slide up
</div>
```

### 5.2 Animações Built-in

```html
<!-- Spin (loading) -->
<svg class="animate-spin h-5 w-5 text-blue-600">...</svg>

<!-- Ping (notification) -->
<span class="relative flex h-3 w-3">
    <span class="animate-ping absolute inline-flex h-full w-full rounded-full bg-red-400 opacity-75"></span>
    <span class="relative inline-flex rounded-full h-3 w-3 bg-red-500"></span>
</span>

<!-- Pulse (skeleton loading) -->
<div class="animate-pulse bg-gray-200 h-4 w-3/4 rounded"></div>

<!-- Bounce -->
<div class="animate-bounce">
    <svg>Arrow down icon</svg>
</div>
```

### 5.3 Animações Customizadas

```javascript
// tailwind.config.js
module.exports = {
    theme: {
        extend: {
            animation: {
                'fade-in': 'fadeIn 0.5s ease-out',
                'slide-up': 'slideUp 0.3s ease-out',
                'shake': 'shake 0.5s ease-in-out',
            },
            keyframes: {
                fadeIn: {
                    '0%': { opacity: '0' },
                    '100%': { opacity: '1' },
                },
                slideUp: {
                    '0%': { transform: 'translateY(20px)', opacity: '0' },
                    '100%': { transform: 'translateY(0)', opacity: '1' },
                },
                shake: {
                    '0%, 100%': { transform: 'translateX(0)' },
                    '25%': { transform: 'translateX(-5px)' },
                    '75%': { transform: 'translateX(5px)' },
                },
            },
        },
    },
}
```

```html
<div class="animate-fade-in">Conteúdo com fade in</div>
<div class="animate-shake">Erro! Campo inválido</div>
```

---

## 6. Dark Mode

### 6.1 Configuração

```javascript
// tailwind.config.js
module.exports = {
    darkMode: 'class', // ou 'media' para seguir preferência do sistema
}
```

### 6.2 Uso

```html
<!-- Adicione 'dark' na tag html para ativar -->
<html class="dark">

<!-- Estilos adaptáveis -->
<div class="bg-white dark:bg-slate-900 text-slate-900 dark:text-white">
    <h1 class="text-gray-900 dark:text-gray-100">Título</h1>
    <p class="text-gray-600 dark:text-gray-400">Parágrafo</p>
    <button class="bg-blue-600 dark:bg-blue-500 hover:bg-blue-700 dark:hover:bg-blue-400">
        Botão
    </button>
</div>

<!-- Toggle com JavaScript -->
<button onclick="document.documentElement.classList.toggle('dark')">
    Toggle Dark Mode
</button>
```

### 6.3 Persistência

```javascript
// Verificar preferência salva ou do sistema
if (localStorage.theme === 'dark' ||
    (!('theme' in localStorage) && window.matchMedia('(prefers-color-scheme: dark)').matches)) {
    document.documentElement.classList.add('dark')
} else {
    document.documentElement.classList.remove('dark')
}

// Salvar preferência
function setTheme(theme) {
    localStorage.theme = theme
    document.documentElement.classList.toggle('dark', theme === 'dark')
}
```

---

## 7. Design Systems e Reutilização

### 7.1 Componentização no Vue

No Vue 3, não criamos classes CSS para reutilizar estilos. **Criamos componentes**.

```vue
<!-- components/ui/AppButton.vue -->
<script setup>
defineProps({
    variant: {
        type: String,
        default: 'primary',
        validator: (v) => ['primary', 'secondary', 'danger'].includes(v)
    },
    size: {
        type: String,
        default: 'md',
        validator: (v) => ['sm', 'md', 'lg'].includes(v)
    },
    loading: Boolean,
    disabled: Boolean,
})

const variants = {
    primary: 'bg-blue-600 hover:bg-blue-700 text-white focus:ring-blue-500',
    secondary: 'bg-gray-200 hover:bg-gray-300 text-gray-900 focus:ring-gray-500',
    danger: 'bg-red-600 hover:bg-red-700 text-white focus:ring-red-500',
}

const sizes = {
    sm: 'px-3 py-1.5 text-sm',
    md: 'px-4 py-2 text-base',
    lg: 'px-6 py-3 text-lg',
}
</script>

<template>
    <button
        :class="[
            'inline-flex items-center justify-center gap-2',
            'font-medium rounded-lg',
            'focus:outline-none focus:ring-2 focus:ring-offset-2',
            'transition-colors duration-200',
            'disabled:opacity-50 disabled:cursor-not-allowed',
            variants[variant],
            sizes[size],
        ]"
        :disabled="disabled || loading"
    >
        <svg v-if="loading" class="animate-spin h-4 w-4" viewBox="0 0 24 24">
            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" fill="none"/>
            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"/>
        </svg>
        <slot />
    </button>
</template>
```

```vue
<!-- Uso -->
<AppButton variant="primary" size="lg" :loading="isSubmitting">
    Salvar
</AppButton>

<AppButton variant="danger" size="sm">
    Excluir
</AppButton>
```

### 7.2 Quando Usar @apply

O `@apply` deve ser usado com moderação. Prefira componentes Vue. Use apenas para:

```css
/* styles.css - Estilos base que afetam TODOS os elementos */
@layer base {
    body {
        @apply bg-white dark:bg-slate-900 text-slate-900 dark:text-white;
    }

    h1 {
        @apply text-3xl font-bold;
    }
}

/* Para bibliotecas de terceiros que você não controla */
@layer components {
    .prose {
        @apply max-w-none;
    }

    .prose a {
        @apply text-blue-600 hover:text-blue-800;
    }
}
```

---

## 8. Plugins Essenciais

### 8.1 @tailwindcss/forms

Reseta e estiliza elementos de formulário nativos:

```bash
npm install @tailwindcss/forms
```

```javascript
// tailwind.config.js
module.exports = {
    plugins: [
        require('@tailwindcss/forms'),
    ],
}
```

### 8.2 @tailwindcss/typography

Estilização automática para conteúdo rich-text (Markdown, WYSIWYG):

```bash
npm install @tailwindcss/typography
```

```html
<article class="prose lg:prose-xl dark:prose-invert">
    <h1>Título do Artigo</h1>
    <p>Parágrafo com <a href="#">link</a> e <strong>negrito</strong>.</p>
    <ul>
        <li>Item 1</li>
        <li>Item 2</li>
    </ul>
    <pre><code>console.log('Hello')</code></pre>
</article>
```

### 8.3 @tailwindcss/aspect-ratio

Mantém proporção de elementos (útil para vídeos e imagens):

```html
<div class="aspect-w-16 aspect-h-9">
    <iframe src="https://youtube.com/..." class="w-full h-full"></iframe>
</div>

<!-- Ou com classes nativas do Tailwind 3.x -->
<div class="aspect-video">
    <img src="thumbnail.jpg" class="object-cover w-full h-full">
</div>
```

---

## 9. Performance e Otimização

### 9.1 Purge CSS (JIT)

O Tailwind automaticamente remove classes não utilizadas em produção:

```javascript
// tailwind.config.js
module.exports = {
    content: [
        './resources/**/*.blade.php',
        './resources/**/*.js',
        './resources/**/*.vue',
    ],
}
```

### 9.2 Evite Classes Dinâmicas Quebradas

```javascript
// ERRADO - O Tailwind não consegue detectar
const color = 'red'
<div class={`bg-${color}-500`}>...</div>

// CERTO - Use classes completas
const bgColors = {
    red: 'bg-red-500',
    blue: 'bg-blue-500',
    green: 'bg-green-500',
}
<div class={bgColors[color]}>...</div>
```

### 9.3 Safelist (Quando Necessário)

```javascript
// tailwind.config.js
module.exports = {
    safelist: [
        'bg-red-500',
        'bg-blue-500',
        { pattern: /^bg-(red|blue|green)-/ },
    ],
}
```

---

## 10. Boas Práticas

1. **Mobile First:** Sempre estilize primeiro para telas pequenas e use prefixos (`md:`, `lg:`) para telas maiores.

2. **Evite Classes Longas:** Se uma linha tem mais de 10 classes, considere criar um componente.

3. **Consistência de Espaçamento:** Use a escala do Tailwind (`p-4`, `gap-6`) em vez de valores arbitrários.

4. **Acessibilidade:** Sempre use `focus-visible:` em vez de apenas `focus:` para indicadores de foco.

5. **Dark Mode:** Planeje o dark mode desde o início, não como uma feature adicional.

6. **Semantic First:** Use elementos HTML semânticos antes de adicionar ARIA.

---

## 11. Exercícios Práticos

### Exercício 1: Card de Produto
Crie um card de produto responsivo com:
- Imagem com aspect ratio 4:3
- Badge de desconto posicionado
- Título, preço e botão de compra
- Hover effect com elevação

### Exercício 2: Formulário de Login
Crie um formulário de login com:
- Validação visual (estados de erro)
- Dark mode completo
- Loading state no botão
- Link "Esqueci minha senha"

### Exercício 3: Dashboard Layout
Crie um layout de dashboard com:
- Sidebar fixa (colapsável em mobile)
- Header com busca e avatar
- Grid de cards de métricas
- Responsivo para mobile

### Exercício 4: Componente de Notificação
Crie um componente de toast/notificação com:
- Variantes (success, error, warning, info)
- Animação de entrada e saída
- Auto-dismiss com progress bar
- Botão de fechar

---

## Conclusão

HTML e CSS são as ferramentas que o seu usuário realmente vê. Dominar o Tailwind CSS permite que você transforme ideias em interfaces profissionais em minutos, mantendo um código limpo e escalável dentro do ecossistema Laravel/Vue. A chave é entender os fundamentos (Flexbox, Grid, acessibilidade) antes de depender das abstrações do Tailwind.
