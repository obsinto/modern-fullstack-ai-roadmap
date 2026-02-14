# Fundamentos do JavaScript Moderno (ES6+)

## Introdução

Este guia cobre os recursos essenciais do JavaScript moderno que todo desenvolvedor full stack precisa dominar. O JavaScript evoluiu drasticamente desde o ES6 (2015), e conhecer esses fundamentos é crucial para trabalhar com Vue.js, Node.js e qualquer framework moderno.

---

## Let, Const e Escopo

### Var vs Let vs Const

```javascript
// var - escopo de função, hoisting problemático
function problematicVar() {
    console.log(x); // undefined (não erro!)
    var x = 5;

    if (true) {
        var x = 10; // Mesma variável!
    }
    console.log(x); // 10
}

// let - escopo de bloco, sem hoisting "silencioso"
function betterLet() {
    // console.log(y); // ReferenceError!
    let y = 5;

    if (true) {
        let y = 10; // Nova variável
        console.log(y); // 10
    }
    console.log(y); // 5
}

// const - constante de referência
const config = { apiUrl: 'http://api.example.com' };
// config = {}; // Error! Não pode reatribuir
config.apiUrl = 'http://new-api.example.com'; // OK! Pode modificar propriedades
```

### Boas Práticas

```javascript
// SEMPRE use const por padrão
const MAX_RETRIES = 3;
const user = { name: 'John' };
const items = [1, 2, 3];

// Use let APENAS quando precisar reatribuir
let count = 0;
count++;

let status = 'pending';
status = 'completed';

// NUNCA use var em código novo
```

### Temporal Dead Zone

```javascript
// TDZ - variável existe mas não pode ser acessada
function example() {
    // console.log(x); // ReferenceError (TDZ)

    const x = 10;
    console.log(x); // 10
}

// Útil para prevenir bugs de uso antes da declaração
```

---

## Arrow Functions

### Sintaxe

```javascript
// Function tradicional
function soma(a, b) {
    return a + b;
}

// Arrow function - forma completa
const soma = (a, b) => {
    return a + b;
};

// Arrow function - retorno implícito
const soma = (a, b) => a + b;

// Um parâmetro - parênteses opcionais
const dobro = n => n * 2;

// Sem parâmetros
const getTimestamp = () => Date.now();

// Retornando objeto (precisa parênteses)
const createUser = (name, email) => ({
    name,
    email,
    createdAt: new Date()
});
```

### This Léxico

```javascript
// Function tradicional - this dinâmico
const obj = {
    name: 'Object',
    traditional: function() {
        setTimeout(function() {
            console.log(this.name); // undefined! this é window/global
        }, 100);
    },
    arrow: function() {
        setTimeout(() => {
            console.log(this.name); // "Object" - this é capturado
        }, 100);
    }
};

// Classes - arrow functions preservam this
class Counter {
    count = 0;

    // Método tradicional - this pode se perder
    increment() {
        this.count++;
    }

    // Arrow function como propriedade - this sempre correto
    decrement = () => {
        this.count--;
    }
}

const counter = new Counter();
const inc = counter.increment;
const dec = counter.decrement;

// inc(); // Error! this é undefined
dec(); // Funciona! this é counter
```

### Quando Usar Cada Um

```javascript
// USE arrow functions para:
// - Callbacks
array.map(item => item.name);
array.filter(item => item.active);
setTimeout(() => this.update(), 1000);

// - Funções curtas
const isEven = n => n % 2 === 0;

// USE function tradicional para:
// - Métodos de objeto que precisam de this dinâmico
const api = {
    baseUrl: 'http://api.example.com',
    fetch(endpoint) {
        return fetch(this.baseUrl + endpoint);
    }
};

// - Quando precisa de arguments ou new
function variadic() {
    console.log(arguments); // Arrow não tem arguments
}
```

---

## Template Literals

### Interpolação

```javascript
const name = 'João';
const age = 25;

// Concatenação antiga
const message = 'Olá, ' + name + '! Você tem ' + age + ' anos.';

// Template literal
const message = `Olá, ${name}! Você tem ${age} anos.`;

// Expressões
const price = 99.99;
const quantity = 3;
const total = `Total: R$ ${(price * quantity).toFixed(2)}`;

// Chamadas de função
const upper = `Nome: ${name.toUpperCase()}`;
```

### Multiline Strings

```javascript
// Antes - escape de newlines
const html = '<div>\n' +
    '  <h1>Title</h1>\n' +
    '  <p>Content</p>\n' +
    '</div>';

// Template literal - preserva formatação
const html = `
<div>
  <h1>Title</h1>
  <p>Content</p>
</div>
`;

// SQL queries
const query = `
    SELECT u.id, u.name, u.email
    FROM users u
    WHERE u.status = 'active'
    ORDER BY u.created_at DESC
    LIMIT 10
`;
```

### Tagged Templates

```javascript
// Para processamento customizado
function sql(strings, ...values) {
    // strings = ['SELECT * FROM users WHERE id = ', ' AND status = ', '']
    // values = [userId, status]

    // Escape valores para prevenir SQL injection
    const escaped = values.map(v => escape(v));

    return strings.reduce((result, str, i) => {
        return result + str + (escaped[i] || '');
    }, '');
}

const userId = 1;
const status = 'active';
const query = sql`SELECT * FROM users WHERE id = ${userId} AND status = ${status}`;

// Uso real: styled-components, GraphQL queries
const Button = styled.button`
    background: ${props => props.primary ? 'blue' : 'white'};
    color: ${props => props.primary ? 'white' : 'blue'};
`;
```

---

## Destructuring

### Arrays

```javascript
const numbers = [1, 2, 3, 4, 5];

// Básico
const [first, second] = numbers;
console.log(first, second); // 1, 2

// Pular elementos
const [, , third] = numbers;
console.log(third); // 3

// Rest
const [head, ...tail] = numbers;
console.log(head); // 1
console.log(tail); // [2, 3, 4, 5]

// Default values
const [a, b, c = 10] = [1, 2];
console.log(c); // 10

// Swap
let x = 1, y = 2;
[x, y] = [y, x];
console.log(x, y); // 2, 1
```

### Objects

```javascript
const user = {
    name: 'John',
    email: 'john@example.com',
    age: 30,
    address: {
        city: 'São Paulo',
        country: 'Brasil'
    }
};

// Básico
const { name, email } = user;

// Renomear
const { name: userName, email: userEmail } = user;

// Default values
const { phone = 'N/A' } = user;

// Nested
const { address: { city } } = user;

// Rest
const { name: n, ...rest } = user;
console.log(rest); // { email, age, address }

// Em parâmetros de função
function createUser({ name, email, role = 'user' }) {
    return { name, email, role, createdAt: new Date() };
}

createUser({ name: 'Jane', email: 'jane@example.com' });
```

### Uso Prático

```javascript
// Importar específicos
import { ref, computed, watch } from 'vue';

// Parâmetros de função
function processOrder({ id, items, customer: { email } }) {
    console.log(`Processing order ${id} for ${email}`);
    items.forEach(item => console.log(item.name));
}

// Retorno de múltiplos valores
function getStats(data) {
    return {
        min: Math.min(...data),
        max: Math.max(...data),
        avg: data.reduce((a, b) => a + b, 0) / data.length
    };
}

const { min, max, avg } = getStats([1, 2, 3, 4, 5]);

// Composables Vue
function useCounter(initial = 0) {
    const count = ref(initial);
    const increment = () => count.value++;
    const decrement = () => count.value--;

    return { count, increment, decrement };
}

const { count, increment } = useCounter();
```

---

## Spread Operator

### Arrays

```javascript
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];

// Concatenar
const combined = [...arr1, ...arr2];
// [1, 2, 3, 4, 5, 6]

// Copiar (shallow)
const copy = [...arr1];

// Inserir no meio
const withMiddle = [...arr1.slice(0, 1), 'new', ...arr1.slice(1)];
// [1, 'new', 2, 3]

// Converter iterables
const chars = [...'hello'];
// ['h', 'e', 'l', 'l', 'o']

// Como argumentos
const numbers = [1, 2, 3];
Math.max(...numbers); // 3
```

### Objects

```javascript
const defaults = {
    theme: 'light',
    language: 'pt-BR',
    notifications: true
};

const userPrefs = {
    theme: 'dark'
};

// Merge (último vence)
const settings = { ...defaults, ...userPrefs };
// { theme: 'dark', language: 'pt-BR', notifications: true }

// Adicionar/sobrescrever propriedades
const user = { name: 'John', age: 30 };
const updatedUser = { ...user, age: 31, email: 'john@example.com' };

// Copiar (shallow!)
const copy = { ...user };

// Cuidado com nested objects
const original = { a: 1, b: { c: 2 } };
const shallow = { ...original };
shallow.b.c = 3;
console.log(original.b.c); // 3! Referência compartilhada

// Deep copy
const deep = JSON.parse(JSON.stringify(original));
// Ou
const deep = structuredClone(original); // Moderno
```

---

## Rest Parameters

### Funções

```javascript
// Coletar argumentos extras
function log(level, ...messages) {
    console.log(`[${level}]`, ...messages);
}

log('INFO', 'Starting', 'application', 'v2.0');
// [INFO] Starting application v2.0

// Substituindo arguments
function sum(...numbers) {
    return numbers.reduce((total, n) => total + n, 0);
}

sum(1, 2, 3, 4); // 10

// Combinando com parâmetros normais
function createRequest(method, url, ...options) {
    return { method, url, options };
}
```

### Destructuring

```javascript
// Arrays
const [first, ...rest] = [1, 2, 3, 4, 5];
// first = 1, rest = [2, 3, 4, 5]

// Objects
const { id, ...data } = { id: 1, name: 'John', age: 30 };
// id = 1, data = { name: 'John', age: 30 }

// Útil para "omitir" propriedades
function omit(obj, ...keys) {
    const result = { ...obj };
    keys.forEach(key => delete result[key]);
    return result;
}

const user = { id: 1, name: 'John', password: 'secret' };
const safe = omit(user, 'password');
// { id: 1, name: 'John' }
```

---

## Optional Chaining

### Acesso Seguro a Propriedades

```javascript
const user = {
    name: 'John',
    address: {
        city: 'São Paulo'
    }
};

// Antes - verificações manuais
const city = user && user.address && user.address.city;

// Agora - optional chaining
const city = user?.address?.city;

// Com arrays
const users = [{ name: 'John' }];
const firstName = users?.[0]?.name;

// Com métodos
const result = obj?.method?.();

// Com delete
delete user?.address?.temp;
```

### Casos de Uso

```javascript
// API responses
async function getUser(id) {
    const response = await fetch(`/api/users/${id}`);
    const data = await response.json();

    return {
        name: data?.user?.profile?.name ?? 'Unknown',
        avatar: data?.user?.profile?.avatar?.url ?? '/default.png'
    };
}

// Event handlers
function handleClick(event) {
    const value = event?.target?.value;
    const dataset = event?.target?.dataset?.id;
}

// Config objects
const config = loadConfig();
const timeout = config?.api?.timeout ?? 5000;
const retries = config?.api?.retries ?? 3;
```

---

## Nullish Coalescing

### ?? vs ||

```javascript
// || retorna o primeiro "truthy" value
// Problema: 0, '', false são considerados "falsy"

const count = 0;
const result = count || 10;
// result = 10 (indesejado se 0 é válido!)

// ?? retorna o primeiro valor que NÃO é null/undefined
const result = count ?? 10;
// result = 0 (correto!)

// Exemplos
const a = null ?? 'default';    // 'default'
const b = undefined ?? 'default'; // 'default'
const c = 0 ?? 'default';       // 0
const d = '' ?? 'default';      // ''
const e = false ?? 'default';   // false

// Use || para strings onde vazio deve usar default
const name = inputName || 'Anonymous';

// Use ?? para valores onde 0/false/'' são válidos
const count = inputCount ?? 0;
const enabled = inputEnabled ?? true;
```

### Combinando com Optional Chaining

```javascript
const user = {
    settings: {
        notifications: false,
        volume: 0
    }
};

// ?? preserva false e 0
const notifications = user?.settings?.notifications ?? true;
// false (valor do user)

const volume = user?.settings?.volume ?? 100;
// 0 (valor do user)

const theme = user?.settings?.theme ?? 'light';
// 'light' (default, pois theme é undefined)
```

---

## Promises e Async/Await

### Promises

```javascript
// Criando uma Promise
const fetchUser = (id) => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            if (id > 0) {
                resolve({ id, name: 'John' });
            } else {
                reject(new Error('Invalid ID'));
            }
        }, 1000);
    });
};

// Usando com .then/.catch
fetchUser(1)
    .then(user => console.log(user))
    .catch(error => console.error(error))
    .finally(() => console.log('Done'));

// Encadeamento
fetchUser(1)
    .then(user => fetchOrders(user.id))
    .then(orders => processOrders(orders))
    .catch(error => console.error(error));
```

### Async/Await

```javascript
// Forma mais legível de trabalhar com Promises
async function loadUserData(userId) {
    try {
        const user = await fetchUser(userId);
        const orders = await fetchOrders(user.id);
        const processed = await processOrders(orders);

        return { user, orders, processed };
    } catch (error) {
        console.error('Error:', error);
        throw error;
    }
}

// Arrow function async
const loadData = async (id) => {
    const data = await fetch(`/api/data/${id}`);
    return data.json();
};

// IIFE async
(async () => {
    const result = await loadData(1);
    console.log(result);
})();
```

### Execução Paralela

```javascript
// Sequencial (lento)
async function loadSequential() {
    const users = await fetchUsers();     // 1s
    const products = await fetchProducts(); // 1s
    const orders = await fetchOrders();     // 1s
    // Total: 3s
}

// Paralelo (rápido)
async function loadParallel() {
    const [users, products, orders] = await Promise.all([
        fetchUsers(),
        fetchProducts(),
        fetchOrders()
    ]);
    // Total: ~1s (o mais lento)
}

// Promise.allSettled - não falha se uma rejeitar
async function loadAllSettled() {
    const results = await Promise.allSettled([
        fetchUsers(),
        fetchProducts(),
        mightFail()
    ]);

    results.forEach(result => {
        if (result.status === 'fulfilled') {
            console.log('Success:', result.value);
        } else {
            console.log('Failed:', result.reason);
        }
    });
}

// Promise.race - primeiro a resolver/rejeitar
async function withTimeout(promise, ms) {
    const timeout = new Promise((_, reject) => {
        setTimeout(() => reject(new Error('Timeout')), ms);
    });

    return Promise.race([promise, timeout]);
}

// Promise.any - primeiro a resolver (ignora rejeições)
async function fetchFromMirrors(urls) {
    return Promise.any(urls.map(url => fetch(url)));
}
```

### Error Handling

```javascript
// Try/catch com async/await
async function safeLoad() {
    try {
        const data = await riskyOperation();
        return { success: true, data };
    } catch (error) {
        console.error('Failed:', error);
        return { success: false, error: error.message };
    }
}

// Wrapper para evitar try/catch repetitivo
async function to(promise) {
    try {
        const data = await promise;
        return [null, data];
    } catch (error) {
        return [error, null];
    }
}

// Uso
const [error, user] = await to(fetchUser(1));
if (error) {
    console.error(error);
    return;
}
console.log(user);
```

---

## Modules (ESM)

### Export

```javascript
// Named exports
export const API_URL = 'http://api.example.com';

export function formatDate(date) {
    return date.toISOString();
}

export class UserService {
    // ...
}

// Export no final (equivalente)
const API_URL = 'http://api.example.com';
function formatDate(date) { /* ... */ }
class UserService { /* ... */ }

export { API_URL, formatDate, UserService };

// Renomear no export
export { formatDate as formatDateISO };

// Default export (um por arquivo)
export default class ApiClient {
    // ...
}

// Ou
const client = new ApiClient();
export default client;
```

### Import

```javascript
// Named imports
import { API_URL, formatDate } from './utils.js';

// Renomear
import { formatDate as format } from './utils.js';

// Default import
import ApiClient from './api-client.js';

// Ambos
import ApiClient, { API_URL, formatDate } from './api.js';

// Importar tudo como namespace
import * as Utils from './utils.js';
Utils.formatDate(new Date());

// Side-effect import (apenas executa)
import './polyfills.js';
import './styles.css';

// Dynamic import
const module = await import('./heavy-module.js');
module.doSomething();

// Conditional import
if (condition) {
    const { feature } = await import('./feature.js');
    feature.init();
}
```

### Padrões de Organização

```javascript
// index.js - Re-exports (barrel file)
export { UserService } from './user-service.js';
export { OrderService } from './order-service.js';
export { ProductService } from './product-service.js';

// Uso
import { UserService, OrderService } from './services';

// Estrutura típica
// services/
//   index.js
//   user-service.js
//   order-service.js
// composables/
//   index.js
//   useAuth.js
//   useCart.js
```

---

## Classes

### Sintaxe Básica

```javascript
class User {
    // Propriedades públicas
    name;
    email;

    // Propriedade privada (# prefix)
    #password;

    // Propriedade estática
    static DEFAULT_ROLE = 'user';

    constructor(name, email, password) {
        this.name = name;
        this.email = email;
        this.#password = password;
    }

    // Getter
    get displayName() {
        return `${this.name} <${this.email}>`;
    }

    // Setter
    set password(value) {
        if (value.length < 8) {
            throw new Error('Password too short');
        }
        this.#password = value;
    }

    // Método de instância
    greet() {
        return `Hello, ${this.name}!`;
    }

    // Método privado
    #hashPassword(password) {
        // ...
    }

    // Método estático
    static create(data) {
        return new User(data.name, data.email, data.password);
    }
}

const user = new User('John', 'john@example.com', 'secret123');
console.log(user.displayName);
console.log(User.DEFAULT_ROLE);
```

### Herança

```javascript
class Animal {
    constructor(name) {
        this.name = name;
    }

    speak() {
        console.log(`${this.name} makes a sound.`);
    }
}

class Dog extends Animal {
    constructor(name, breed) {
        super(name); // Chama construtor pai
        this.breed = breed;
    }

    speak() {
        console.log(`${this.name} barks.`);
    }

    fetch() {
        console.log(`${this.name} fetches the ball.`);
    }
}

const dog = new Dog('Rex', 'German Shepherd');
dog.speak(); // Rex barks.
```

### Class Fields

```javascript
class Counter {
    // Public field com valor inicial
    count = 0;

    // Private field
    #step = 1;

    // Arrow function como propriedade (this sempre correto)
    increment = () => {
        this.count += this.#step;
    };

    decrement = () => {
        this.count -= this.#step;
    };

    setStep(step) {
        this.#step = step;
    }
}

// Útil para event handlers
const counter = new Counter();
button.addEventListener('click', counter.increment);
```

---

## Array Methods

### Transformação

```javascript
const users = [
    { id: 1, name: 'John', age: 30, active: true },
    { id: 2, name: 'Jane', age: 25, active: false },
    { id: 3, name: 'Bob', age: 35, active: true }
];

// map - transformar cada item
const names = users.map(user => user.name);
// ['John', 'Jane', 'Bob']

// filter - filtrar items
const activeUsers = users.filter(user => user.active);
// [{ id: 1, ... }, { id: 3, ... }]

// find - primeiro que atende condição
const john = users.find(user => user.name === 'John');
// { id: 1, name: 'John', ... }

// findIndex - índice do primeiro que atende
const johnIndex = users.findIndex(user => user.name === 'John');
// 0

// some - algum atende?
const hasInactive = users.some(user => !user.active);
// true

// every - todos atendem?
const allActive = users.every(user => user.active);
// false

// includes - contém valor?
const numbers = [1, 2, 3];
numbers.includes(2); // true
```

### Agregação

```javascript
const numbers = [1, 2, 3, 4, 5];

// reduce - agregar em um valor
const sum = numbers.reduce((total, n) => total + n, 0);
// 15

// Com objeto inicial
const users = [
    { id: 1, name: 'John' },
    { id: 2, name: 'Jane' }
];

const byId = users.reduce((acc, user) => {
    acc[user.id] = user;
    return acc;
}, {});
// { 1: { id: 1, name: 'John' }, 2: { id: 2, name: 'Jane' } }

// flat - achatar arrays aninhados
const nested = [[1, 2], [3, 4], [5]];
const flat = nested.flat();
// [1, 2, 3, 4, 5]

// flatMap - map + flat
const sentences = ['Hello World', 'Goodbye World'];
const words = sentences.flatMap(s => s.split(' '));
// ['Hello', 'World', 'Goodbye', 'World']
```

### Encadeamento

```javascript
const orders = [
    { id: 1, customer: 'John', total: 100, status: 'completed' },
    { id: 2, customer: 'Jane', total: 200, status: 'pending' },
    { id: 3, customer: 'John', total: 150, status: 'completed' },
    { id: 4, customer: 'Bob', total: 300, status: 'completed' }
];

// Pipeline de transformações
const johnCompletedTotal = orders
    .filter(order => order.customer === 'John')
    .filter(order => order.status === 'completed')
    .map(order => order.total)
    .reduce((sum, total) => sum + total, 0);
// 250

// Ordenação + transformação
const topCustomers = orders
    .filter(order => order.status === 'completed')
    .reduce((acc, order) => {
        acc[order.customer] = (acc[order.customer] || 0) + order.total;
        return acc;
    }, {});

const sorted = Object.entries(topCustomers)
    .sort(([, a], [, b]) => b - a)
    .map(([customer, total]) => ({ customer, total }));
// [{ customer: 'Bob', total: 300 }, { customer: 'John', total: 250 }]
```

### Métodos Mutáveis vs Imutáveis

```javascript
// MUTÁVEIS (modificam o array original)
const arr = [3, 1, 2];
arr.sort();        // arr é agora [1, 2, 3]
arr.reverse();     // arr é agora [3, 2, 1]
arr.push(4);       // arr é agora [3, 2, 1, 4]
arr.pop();         // arr é agora [3, 2, 1]
arr.splice(1, 1);  // arr é agora [3, 1]

// IMUTÁVEIS (retornam novo array)
const arr = [1, 2, 3];
const mapped = arr.map(n => n * 2);    // arr inalterado
const filtered = arr.filter(n => n > 1); // arr inalterado
const sliced = arr.slice(0, 2);        // arr inalterado
const concatenated = arr.concat([4, 5]); // arr inalterado

// Novos métodos imutáveis (ES2023)
const sorted = arr.toSorted();         // arr inalterado
const reversed = arr.toReversed();     // arr inalterado
const spliced = arr.toSpliced(1, 1);   // arr inalterado
```

---

## Object Methods

### Criação e Manipulação

```javascript
// Object.assign - merge/clone
const defaults = { a: 1, b: 2 };
const options = { b: 3, c: 4 };
const merged = Object.assign({}, defaults, options);
// { a: 1, b: 3, c: 4 }

// Object.keys/values/entries
const user = { name: 'John', age: 30 };

Object.keys(user);    // ['name', 'age']
Object.values(user);  // ['John', 30]
Object.entries(user); // [['name', 'John'], ['age', 30]]

// Object.fromEntries - inverso de entries
const entries = [['name', 'John'], ['age', 30]];
const obj = Object.fromEntries(entries);
// { name: 'John', age: 30 }

// Útil para transformar objetos
const prices = { apple: 1, banana: 2, orange: 3 };
const doubled = Object.fromEntries(
    Object.entries(prices).map(([fruit, price]) => [fruit, price * 2])
);
// { apple: 2, banana: 4, orange: 6 }
```

### Property Shorthand

```javascript
const name = 'John';
const age = 30;

// Antes
const user = {
    name: name,
    age: age
};

// Property shorthand
const user = { name, age };

// Com métodos
const calculator = {
    value: 0,

    // Method shorthand
    add(n) {
        this.value += n;
        return this;
    },

    subtract(n) {
        this.value -= n;
        return this;
    }
};
```

### Computed Property Names

```javascript
const key = 'dynamicKey';

const obj = {
    [key]: 'value',
    [`computed_${key}`]: 'another value',
    ['method' + 'Name']() {
        return 'Hello';
    }
};

// obj.dynamicKey === 'value'
// obj.computed_dynamicKey === 'another value'
// obj.methodName() === 'Hello'

// Útil para reducers
const field = 'email';
const value = 'john@example.com';

const update = { [field]: value };
// { email: 'john@example.com' }
```

---

## Map e Set

### Map

```javascript
// Map - chaves podem ser qualquer tipo
const map = new Map();

// Adicionar
map.set('string', 'value1');
map.set(123, 'value2');
map.set({ id: 1 }, 'value3');

// Acessar
map.get('string'); // 'value1'
map.has('string'); // true
map.size;          // 3

// Deletar
map.delete('string');
map.clear();

// Iteração
for (const [key, value] of map) {
    console.log(key, value);
}

map.forEach((value, key) => {
    console.log(key, value);
});

// Converter de/para array
const entries = [['a', 1], ['b', 2]];
const map = new Map(entries);
const arr = [...map]; // ou Array.from(map)
```

### Set

```javascript
// Set - valores únicos
const set = new Set([1, 2, 3, 3, 3]);
// Set { 1, 2, 3 }

set.add(4);
set.has(2);     // true
set.delete(2);
set.size;       // 3

// Remover duplicatas de array
const numbers = [1, 2, 2, 3, 3, 3];
const unique = [...new Set(numbers)];
// [1, 2, 3]

// Operações de conjunto
const a = new Set([1, 2, 3]);
const b = new Set([2, 3, 4]);

// União
const union = new Set([...a, ...b]);
// Set { 1, 2, 3, 4 }

// Interseção
const intersection = new Set([...a].filter(x => b.has(x)));
// Set { 2, 3 }

// Diferença
const difference = new Set([...a].filter(x => !b.has(x)));
// Set { 1 }
```

### WeakMap e WeakSet

```javascript
// WeakMap - chaves são fracamente referenciadas
// Útil para cache que não impede garbage collection
const cache = new WeakMap();

function process(obj) {
    if (cache.has(obj)) {
        return cache.get(obj);
    }

    const result = expensiveOperation(obj);
    cache.set(obj, result);
    return result;
}

// Quando obj não tem mais referências,
// a entrada no WeakMap é automaticamente removida

// WeakSet - similar, para conjuntos
const processed = new WeakSet();

function processOnce(obj) {
    if (processed.has(obj)) {
        return;
    }

    processed.add(obj);
    doProcess(obj);
}
```

---

## Iterators e Generators

### Iterators

```javascript
// Qualquer objeto com [Symbol.iterator] é iterável
const range = {
    from: 1,
    to: 5,

    [Symbol.iterator]() {
        let current = this.from;
        const to = this.to;

        return {
            next() {
                if (current <= to) {
                    return { value: current++, done: false };
                }
                return { done: true };
            }
        };
    }
};

for (const n of range) {
    console.log(n); // 1, 2, 3, 4, 5
}

// Spread funciona com iteráveis
const arr = [...range]; // [1, 2, 3, 4, 5]
```

### Generators

```javascript
// Generator - função que pode pausar e continuar
function* numberGenerator() {
    yield 1;
    yield 2;
    yield 3;
}

const gen = numberGenerator();
gen.next(); // { value: 1, done: false }
gen.next(); // { value: 2, done: false }
gen.next(); // { value: 3, done: false }
gen.next(); // { value: undefined, done: true }

// Com loop
function* range(from, to) {
    for (let i = from; i <= to; i++) {
        yield i;
    }
}

for (const n of range(1, 5)) {
    console.log(n); // 1, 2, 3, 4, 5
}

// Generators infinitos
function* infiniteSequence() {
    let i = 0;
    while (true) {
        yield i++;
    }
}

const gen = infiniteSequence();
gen.next().value; // 0
gen.next().value; // 1
gen.next().value; // 2
// ...

// Async generators
async function* fetchPages(urls) {
    for (const url of urls) {
        const response = await fetch(url);
        yield await response.json();
    }
}

for await (const page of fetchPages(urls)) {
    console.log(page);
}
```

---

## Proxy e Reflect

### Proxy

```javascript
// Proxy - interceptar operações em objetos
const user = { name: 'John', age: 30 };

const proxy = new Proxy(user, {
    get(target, property) {
        console.log(`Getting ${property}`);
        return target[property];
    },

    set(target, property, value) {
        console.log(`Setting ${property} to ${value}`);
        target[property] = value;
        return true;
    }
});

proxy.name;      // Log: "Getting name", retorna "John"
proxy.age = 31;  // Log: "Setting age to 31"

// Validação automática
const validator = {
    set(target, property, value) {
        if (property === 'age' && typeof value !== 'number') {
            throw new TypeError('Age must be a number');
        }
        target[property] = value;
        return true;
    }
};

const validated = new Proxy({}, validator);
validated.age = 30;      // OK
// validated.age = 'old'; // TypeError!
```

### Casos de Uso

```javascript
// Reactive system (como Vue 3)
function reactive(obj) {
    return new Proxy(obj, {
        get(target, key) {
            track(target, key); // Registrar dependência
            return target[key];
        },
        set(target, key, value) {
            target[key] = value;
            trigger(target, key); // Notificar mudança
            return true;
        }
    });
}

// Logging automático
function withLogging(obj) {
    return new Proxy(obj, {
        get(target, key) {
            const value = target[key];
            if (typeof value === 'function') {
                return function(...args) {
                    console.log(`Called ${key} with`, args);
                    return value.apply(target, args);
                };
            }
            return value;
        }
    });
}
```

---

## Boas Práticas

### Checklist de Código Moderno

```
JavaScript Moderno:
□ Use const por padrão, let quando necessário
□ NUNCA use var
□ Use arrow functions para callbacks
□ Use template literals para strings
□ Use destructuring para extrair valores
□ Use spread/rest para arrays e objetos
□ Use async/await ao invés de .then chains
□ Use optional chaining (?.) para acesso seguro
□ Use nullish coalescing (??) para defaults
□ Use métodos de array (map, filter, reduce)
□ Use ESM (import/export)
```

### Padrões Comuns

```javascript
// Default parameters
function createUser(name, role = 'user', active = true) {
    return { name, role, active };
}

// Object defaults com spread
function merge(defaults, options) {
    return { ...defaults, ...options };
}

// Pipeline de transformação
const result = data
    .filter(Boolean)
    .map(transform)
    .reduce(aggregate, initial);

// Guard clauses
function processUser(user) {
    if (!user) return null;
    if (!user.active) return null;

    // Lógica principal...
}

// Early return
async function loadData(id) {
    const cached = cache.get(id);
    if (cached) return cached;

    const data = await fetch(`/api/${id}`);
    cache.set(id, data);
    return data;
}
```

---

## Exercícios Práticos

### Exercício 1: Refatorar para ES6+

```javascript
// Código ES5
var users = [
    { name: 'John', age: 30 },
    { name: 'Jane', age: 25 },
    { name: 'Bob', age: 35 }
];

var names = [];
for (var i = 0; i < users.length; i++) {
    names.push(users[i].name);
}

var adults = [];
for (var j = 0; j < users.length; j++) {
    if (users[j].age >= 30) {
        adults.push(users[j]);
    }
}

function createPerson(name, age) {
    return {
        name: name,
        age: age,
        greet: function() {
            var self = this;
            setTimeout(function() {
                console.log('Hello, ' + self.name);
            }, 1000);
        }
    };
}

// Refatore usando:
// - const/let
// - Arrow functions
// - Template literals
// - Array methods
// - Property shorthand
// - This léxico
```

### Exercício 2: Implementar Utility Functions

```javascript
// Implemente usando ES6+:

// 1. debounce(fn, delay) - atrasa execução
// 2. throttle(fn, limit) - limita frequência
// 3. memoize(fn) - cache de resultados
// 4. pipe(...fns) - composição de funções
// 5. deepClone(obj) - clone profundo
```

### Exercício 3: Async Data Loading

```javascript
// Implemente uma função que:
// - Carrega dados de múltiplas APIs em paralelo
// - Tem timeout de 5 segundos por request
// - Retorna partial results se alguma falhar
// - Implementa retry (3 tentativas)

async function loadAllData(endpoints) {
    // Sua implementação
}
```

---

## Conclusão

O JavaScript moderno oferece recursos poderosos para escrever código mais limpo e expressivo. Dominar esses fundamentos é essencial antes de avançar para frameworks como Vue.js.

### Próximos Passos

1. **Pratique** os exercícios acima
2. **Refatore** código legado usando ES6+
3. **Configure** ESLint para validar padrões modernos
4. **Avance** para Vue 3 e Composition API

---

*O JavaScript evoluiu significativamente. Código moderno JavaScript é legível, conciso e poderoso.*
