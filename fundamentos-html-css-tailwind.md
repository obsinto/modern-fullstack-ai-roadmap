# Fundamentos de HTML, CSS e Tailwind CSS

## Introdução
No desenvolvimento moderno, o frontend não é apenas sobre "fazer bonito", mas sobre **usabilidade, acessibilidade e performance**. O Tailwind CSS revolucionou a forma como estilizamos aplicações, movendo-nos do "CSS baseado em componentes" para o "CSS utilitário".

---

## 1. HTML5 Semântico: A Base
HTML semântico não é apenas para SEO; é para acessibilidade (leitores de tela) e para que o navegador entenda a estrutura do seu conteúdo.

- **Use `<main>`, `<nav>`, `<header>`, `<footer>`, `<section>`, `<article>`** em vez de apenas `<div>`.
- **Acessibilidade (ARIA):** Use atributos como `aria-label` e `role` quando a semântica nativa não for suficiente.

---

## 2. CSS Moderno: Flexbox e Grid

Antes de usar Tailwind, você deve entender os dois pilares do layout:

### 2.1 Flexbox (Unidimensional)
Ideal para alinhar itens em uma linha ou coluna (menus, barras laterais).
- `display: flex;`
- `justify-content: center | space-between;`
- `align-items: center;`

### 2.2 CSS Grid (Bidimensional)
Ideal para layouts de página completos e galerias complexas.
- `display: grid;`
- `grid-template-columns: repeat(3, 1fr);`

---

## 3. Tailwind CSS: O Padrão Utilitário

O Tailwind permite que você escreva estilos diretamente nas suas classes HTML, evitando que você "fuja" para um arquivo CSS separado e perca o contexto.

### 3.1 Por que Tailwind?
- **Velocidade:** Não precisa inventar nomes de classes (`.user-card-inner-wrapper`).
- **Consistência:** Você usa uma escala pré-definida de cores, espaçamentos e fontes.
- **Tamanho do Bundle:** O Tailwind remove todo o CSS não utilizado em produção (Purge/JIT).

### 3.2 Conceitos Chave
- **Espaçamento:** `p-4` (padding), `m-2` (margin), `space-x-4` (espaço entre elementos).
- **Tipografia:** `text-xl` (tamanho), `font-bold` (peso), `text-slate-900` (cor).
- **Responsividade:** `block md:flex` (é block por padrão, vira flex em telas médias).
- **Estados:** `hover:bg-blue-700`, `focus:ring-2`.

---

## 4. Design Systems e Reutilização

No Vue 3, não criamos classes CSS para reutilizar estilos. **Criamos componentes**.

```vue
<!-- Em vez de .btn-primary no CSS, criamos o componente AppButton.vue -->
<template>
  <button class="bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-lg transition">
    <slot />
  </button>
</template>
```

---

## 5. Dark Mode e Customização
O Tailwind facilita o suporte a temas escuros usando a variante `dark:`.

```html
<div class="bg-white dark:bg-slate-900 text-slate-900 dark:text-white">
  Conteúdo adaptável.
</div>
```

---

## 6. Boas Práticas
1.  **Mobile First:** Sempre estilize primeiro para telas pequenas e use prefixos (`md:`, `lg:`) para telas maiores.
2.  **Evite `@apply`:** Use as classes utilitárias diretamente. O `@apply` quebra a vantagem de não precisar manter arquivos CSS gigantes.
3.  **Group e Peer:** Use `group` e `peer` para estilizar elementos baseados no estado de seus pais ou irmãos.

---

## Conclusão
HTML e CSS são as ferramentas que o seu usuário realmente vê. Dominar o Tailwind CSS permite que você transforme ideias em interfaces profissionais em minutos, mantendo um código limpo e escalável dentro do ecossistema Laravel/Vue.
