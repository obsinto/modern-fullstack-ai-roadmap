# Vue 3, Inertia.js e Gerenciamento de Estado Moderno

## Introdução
Este guia explora o "The Modern Monolith", uma arquitetura que utiliza o Laravel como backend e o Vue.js como frontend, unidos pelo **Inertia.js**. Esta abordagem permite criar Single Page Applications (SPAs) sem a complexidade de gerenciar rotas no cliente, autenticação JWT ou estados de API complexos.

---

## 1. Vue 3: O Poder da Composition API

A Composition API não é apenas uma nova forma de escrever componentes; é uma forma de **organizar lógica por funcionalidade** e não por tipo de opção (data, methods, computed).

### 1.1 O Padrão `<script setup>`
O `<script setup>` é o açúcar sintático recomendado para usar a Composition API dentro de Single File Components (SFCs).

```vue
<script setup>
import { ref, reactive, computed, onMounted, watch } from 'vue'

// Estado Primitivo (Reatividade via .value no JS)
const count = ref(0)

// Estado de Objeto (Reatividade profunda)
const form = reactive({
    name: '',
    email: ''
})

// Propriedade Computada (Cacheada e Reativa)
const doubleCount = computed(() => count.value * 2)

// Watcher (Reagir a mudanças)
watch(count, (newValue, oldValue) => {
    console.log(`Mudou de ${oldValue} para ${newValue}`)
})

// Ciclo de Vida
onMounted(() => {
    console.log('Componente montado!')
})

function increment() {
    count.value++
}
</script>

<template>
    <div>
        <p>Contagem: {{ count }}</p>
        <p>Dobro: {{ doubleCount }}</p>
        <button @click="increment">Aumentar</button>
        <input v-model="form.name" placeholder="Nome">
    </div>
</template>
```

---

## 2. Inertia.js: A Ponte

O Inertia funciona como um adaptador. No backend, você retorna uma resposta do Inertia; no frontend, o Inertia troca o componente de página sem recarregar o navegador.

### 2.1 Recebendo Dados (Props)
O Laravel envia dados para o frontend automaticamente através do método `Inertia::render()`. No Vue, você os define com `defineProps`.

```javascript
// No Vue
const props = defineProps({
    users: Array,
    filters: Object
})
```

### 2.2 Links e Navegação
Nunca use `<a>` para links internos. Use o componente `<Link>` do Inertia para manter o estado da SPA.

```vue
<script setup>
import { Link } from '@inertiajs/vue3'
</script>

<template>
    <Link href="/users" method="get" :data="{ search: 'Deyvid' }" class="btn">
        Ver Usuários
    </Link>
</template>
```

### 2.3 O Form Helper (Essencial)
O Inertia fornece um helper de formulário que gerencia estados de envio, erros de validação e progresso de upload.

```javascript
import { useForm } from '@inertiajs/vue3'

const form = useForm({
    name: '',
    email: '',
    avatar: null
})

const submit = () => {
    form.post('/profile', {
        preserveScroll: true,
        onSuccess: () => form.reset('password'),
    })
}
```

---

## 3. Gerenciamento de Estado com Pinia

O Pinia é a store oficial do Vue. Use-o para estados que precisam persistir entre diferentes páginas (Inertia mantém o estado das stores ao navegar).

### 3.1 Definição da Store (Padrão Setup)
```javascript
// stores/useNotificationStore.js
import { defineStore } from 'pinia'
import { ref } from 'vue'

export const useNotificationStore = defineStore('notifications', () => {
    const messages = ref([])

    function add(message) {
        messages.value.push({ id: Date.now(), text: message })
        setTimeout(() => remove(messages.value[0].id), 3000)
    }

    function remove(id) {
        messages.value = messages.value.filter(m => m.id !== id)
    }

    return { messages, add, remove }
})
```

---

## 4. Arquitetura de Componentes

Para manter o código limpo, divida seus componentes em categorias:

1.  **Pages**: Componentes em `resources/js/Pages`. Eles recebem props do Laravel.
2.  **Shared/UI**: Componentes genéricos (Botões, Inputs, Modais) que não conhecem a regra de negócio.
3.  **Modules/Features**: Componentes específicos de uma funcionalidade (ex: `OrderCard.vue`) que podem ser usados em várias páginas.
4.  **Layouts**: Estruturas de página (Sidebar, Navbar) que envolvem o conteúdo.

---

## 5. TypeScript no Frontend
O uso de TypeScript com Vue 3 é altamente recomendado para evitar erros de "undefined" em props e estados.

```typescript
<script setup lang="ts">
interface User {
    id: number
    name: string
}

const props = defineProps<{
    user: User
    active?: boolean
}>()
</script>
```

---

## 6. Boas Práticas e Performance

### 6.1 Partial Reloads
Se você precisa atualizar apenas uma parte dos dados da página (ex: paginação ou filtros), o Inertia permite buscar apenas props específicas:
```javascript
router.reload({ only: ['users'] })
```

### 6.2 Shared Data
Use o `HandleInertiaRequests` middleware do Laravel para injetar dados globais (como usuário autenticado e mensagens flash) que devem estar disponíveis em todos os componentes Vue.

### 6.3 Persistent Layouts
Defina o layout no nível do componente de página para garantir que ele não seja remontado ao trocar de página:
```javascript
// Pages/Dashboard.vue
import AppLayout from '@/Layouts/AppLayout.vue'
defineOptions({ layout: AppLayout })
```

---

## Conclusão
Dominar o Vue 3 com Inertia significa entender que o frontend é uma extensão visual das suas rotas de backend. O estado da aplicação deve viver no Laravel (banco de dados) sempre que possível, usando o Pinia apenas para interações puras de UI.
