# Comunicação em Tempo Real (WebSockets)

## Introdução
Aplicações modernas exigem interatividade instantânea. Em vez do cliente perguntar ao servidor se há novos dados (polling), o servidor "empurra" a informação para o cliente no momento em que ela acontece. No Laravel 11, isso é feito nativamente com o **Laravel Reverb**.

---

## 1. O Ecossistema Real-time

### 1.1 Laravel Reverb

Um servidor de WebSockets de alta performance, escrito em PHP puro e otimizado para o Laravel. Ele substitui soluções pagas como Pusher ou configurações complexas como o Soketi.

**Características:**
- Escrito em PHP (usa ReactPHP)
- Suporte a milhares de conexões simultâneas
- Integração nativa com Laravel Echo
- Horizontal scaling via Redis
- Zero dependências externas

### 1.2 Laravel Echo

Uma biblioteca JavaScript que facilita a subscrição em canais e a escuta de eventos transmitidos pelo Laravel.

```javascript
// Instalação
npm install laravel-echo pusher-js
```

### 1.3 Arquitetura do Sistema

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│    Browser      │────▶│  Laravel App    │────▶│     Queue       │
│  (Laravel Echo) │     │   (Backend)     │     │   (Redis)       │
└────────┬────────┘     └─────────────────┘     └────────┬────────┘
         │                                               │
         │ WebSocket                                     │ Job
         │                                               │
         ▼                                               ▼
┌─────────────────┐                             ┌─────────────────┐
│  Laravel Reverb │◀────────────────────────────│  Queue Worker   │
│   (WebSocket    │      Redis Pub/Sub          │                 │
│    Server)      │                             └─────────────────┘
└─────────────────┘
```

---

## 2. Instalação e Configuração

### 2.1 Instalação do Reverb

```bash
# Laravel 11+
php artisan install:broadcasting

# Ou manualmente
composer require laravel/reverb
php artisan reverb:install
```

### 2.2 Configuração do .env

```env
# Broadcasting
BROADCAST_CONNECTION=reverb

# Reverb Configuration
REVERB_APP_ID=my-app-id
REVERB_APP_KEY=my-app-key
REVERB_APP_SECRET=my-app-secret
REVERB_HOST="localhost"
REVERB_PORT=8080
REVERB_SCHEME=http

# Para o frontend (Vite)
VITE_REVERB_APP_KEY="${REVERB_APP_KEY}"
VITE_REVERB_HOST="${REVERB_HOST}"
VITE_REVERB_PORT="${REVERB_PORT}"
VITE_REVERB_SCHEME="${REVERB_SCHEME}"
```

### 2.3 Configuração do Echo (Frontend)

```javascript
// resources/js/bootstrap.js
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: import.meta.env.VITE_REVERB_PORT ?? 80,
    wssPort: import.meta.env.VITE_REVERB_PORT ?? 443,
    forceTLS: (import.meta.env.VITE_REVERB_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss'],
});
```

### 2.4 Iniciando o Servidor

```bash
# Desenvolvimento
php artisan reverb:start

# Com debug
php artisan reverb:start --debug

# Produção (via Supervisor)
php artisan reverb:start --host=0.0.0.0 --port=8080
```

---

## 3. Tipos de Canais

### 3.1 Public Channels

Qualquer um pode ouvir. Sem autenticação necessária.

```php
// Evento
class CurrencyRateUpdated implements ShouldBroadcast
{
    public function __construct(
        public string $currency,
        public float $rate
    ) {}

    public function broadcastOn(): array
    {
        return [
            new Channel('currency-rates'),
        ];
    }
}

// Disparar
event(new CurrencyRateUpdated('USD', 5.23));
```

```javascript
// Frontend
Echo.channel('currency-rates')
    .listen('CurrencyRateUpdated', (e) => {
        console.log(`${e.currency}: R$ ${e.rate}`);
    });
```

### 3.2 Private Channels

Exigem autenticação. O usuário deve ter permissão para ouvir.

```php
// Evento
class OrderStatusUpdated implements ShouldBroadcast
{
    public function __construct(public Order $order) {}

    public function broadcastOn(): array
    {
        return [
            new PrivateChannel("orders.{$this->order->user_id}"),
        ];
    }
}
```

```php
// routes/channels.php - Autorização
use App\Models\Order;

Broadcast::channel('orders.{userId}', function ($user, $userId) {
    return (int) $user->id === (int) $userId;
});
```

```javascript
// Frontend
Echo.private(`orders.${userId}`)
    .listen('OrderStatusUpdated', (e) => {
        console.log('Pedido atualizado:', e.order.status);
    });
```

### 3.3 Presence Channels

Como canais privados, mas informam quem está online no canal.

```php
// Evento
class UserJoinedDocument implements ShouldBroadcast
{
    public function __construct(public Document $document) {}

    public function broadcastOn(): array
    {
        return [
            new PresenceChannel("document.{$this->document->id}"),
        ];
    }
}
```

```php
// routes/channels.php
Broadcast::channel('document.{documentId}', function ($user, $documentId) {
    $document = Document::find($documentId);

    if ($user->canView($document)) {
        return [
            'id' => $user->id,
            'name' => $user->name,
            'avatar' => $user->avatar_url,
        ];
    }

    return false;
});
```

```javascript
// Frontend
Echo.join(`document.${documentId}`)
    .here((users) => {
        // Lista inicial de usuários online
        console.log('Usuários online:', users);
    })
    .joining((user) => {
        // Novo usuário entrou
        console.log(`${user.name} entrou`);
    })
    .leaving((user) => {
        // Usuário saiu
        console.log(`${user.name} saiu`);
    })
    .listen('DocumentUpdated', (e) => {
        // Evento customizado
    });
```

---

## 4. Eventos Broadcasting

### 4.1 Estrutura Completa de um Evento

```php
namespace App\Events;

use App\Models\Message;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class MessageSent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public Message $message
    ) {}

    /**
     * Canais onde o evento será transmitido.
     */
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel("chat.{$this->message->chat_id}"),
        ];
    }

    /**
     * Nome do evento no frontend (opcional).
     * Padrão: nome da classe (MessageSent)
     */
    public function broadcastAs(): string
    {
        return 'message.sent';
    }

    /**
     * Dados enviados ao frontend.
     * Por padrão, todas as propriedades públicas são enviadas.
     */
    public function broadcastWith(): array
    {
        return [
            'id' => $this->message->id,
            'content' => $this->message->content,
            'user' => [
                'id' => $this->message->user->id,
                'name' => $this->message->user->name,
            ],
            'sent_at' => $this->message->created_at->toISOString(),
        ];
    }

    /**
     * Condição para transmitir (opcional).
     */
    public function broadcastWhen(): bool
    {
        return $this->message->is_visible;
    }
}
```

### 4.2 ShouldBroadcast vs ShouldBroadcastNow

```php
// ShouldBroadcast - Vai para a fila (recomendado)
class OrderCreated implements ShouldBroadcast {}

// ShouldBroadcastNow - Síncrono, imediato
class UrgentAlert implements ShouldBroadcastNow {}
```

### 4.3 Disparando Eventos

```php
// Via event()
event(new MessageSent($message));

// Via broadcast() com opções
broadcast(new MessageSent($message))->toOthers();

// Excluir o usuário atual (evitar eco)
broadcast(new MessageSent($message))->toOthers();
```

---

## 5. Client Events (Whisper)

Eventos enviados diretamente do cliente para outros clientes, sem passar pelo servidor Laravel.

```javascript
// Enviar evento client-side
Echo.private(`chat.${chatId}`)
    .whisper('typing', {
        user: currentUser.name
    });

// Ouvir eventos client-side
Echo.private(`chat.${chatId}`)
    .listenForWhisper('typing', (e) => {
        console.log(`${e.user} está digitando...`);
    });
```

**Casos de uso:**
- Indicador "está digitando..."
- Posição do cursor em editores colaborativos
- Movimento do mouse em whiteboards

---

## 6. Notificações em Tempo Real

### 6.1 Criando uma Notificação Broadcast

```php
namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Notifications\Notification;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Notifications\Messages\BroadcastMessage;

class InvoicePaid extends Notification implements ShouldBroadcast
{
    use Queueable;

    public function __construct(
        public Invoice $invoice
    ) {}

    public function via($notifiable): array
    {
        return ['database', 'broadcast'];
    }

    public function toBroadcast($notifiable): BroadcastMessage
    {
        return new BroadcastMessage([
            'invoice_id' => $this->invoice->id,
            'amount' => $this->invoice->amount,
            'message' => "Fatura #{$this->invoice->id} foi paga!",
        ]);
    }

    public function toArray($notifiable): array
    {
        return [
            'invoice_id' => $this->invoice->id,
            'amount' => $this->invoice->amount,
        ];
    }
}
```

### 6.2 Escutando Notificações

```javascript
// Canal de notificações do usuário (automático)
Echo.private(`App.Models.User.${userId}`)
    .notification((notification) => {
        console.log(notification.type); // App\Notifications\InvoicePaid
        console.log(notification.message);
    });
```

---

## 7. Integração com Vue 3 / Inertia

### 7.1 Composable Reutilizável

```javascript
// composables/useChannel.js
import { onMounted, onUnmounted, ref } from 'vue';

export function useChannel(channelName, events = {}) {
    const channel = ref(null);

    onMounted(() => {
        channel.value = Echo.channel(channelName);

        Object.entries(events).forEach(([event, callback]) => {
            channel.value.listen(event, callback);
        });
    });

    onUnmounted(() => {
        Echo.leaveChannel(channelName);
    });

    return channel;
}

export function usePrivateChannel(channelName, events = {}) {
    const channel = ref(null);

    onMounted(() => {
        channel.value = Echo.private(channelName);

        Object.entries(events).forEach(([event, callback]) => {
            channel.value.listen(event, callback);
        });
    });

    onUnmounted(() => {
        Echo.leaveChannel(`private-${channelName}`);
    });

    return channel;
}

export function usePresenceChannel(channelName, handlers = {}) {
    const channel = ref(null);
    const users = ref([]);

    onMounted(() => {
        channel.value = Echo.join(channelName)
            .here((initialUsers) => {
                users.value = initialUsers;
                handlers.here?.(initialUsers);
            })
            .joining((user) => {
                users.value.push(user);
                handlers.joining?.(user);
            })
            .leaving((user) => {
                users.value = users.value.filter(u => u.id !== user.id);
                handlers.leaving?.(user);
            });

        if (handlers.events) {
            Object.entries(handlers.events).forEach(([event, callback]) => {
                channel.value.listen(event, callback);
            });
        }
    });

    onUnmounted(() => {
        Echo.leaveChannel(`presence-${channelName}`);
    });

    return { channel, users };
}
```

### 7.2 Uso em Componentes

```vue
<script setup>
import { ref } from 'vue';
import { usePresenceChannel } from '@/composables/useChannel';

const props = defineProps({
    chatId: Number,
    auth: Object,
});

const messages = ref([]);
const typingUsers = ref([]);

const { users: onlineUsers } = usePresenceChannel(`chat.${props.chatId}`, {
    here: (users) => console.log('Usuários online:', users),
    joining: (user) => console.log(`${user.name} entrou`),
    leaving: (user) => console.log(`${user.name} saiu`),
    events: {
        'MessageSent': (e) => {
            messages.value.push(e.message);
        },
    },
});

// Whisper para "está digitando"
let typingTimeout;
function handleTyping() {
    Echo.private(`chat.${props.chatId}`).whisper('typing', {
        user: props.auth.user,
    });
}

// Ouvir whisper
Echo.private(`chat.${props.chatId}`)
    .listenForWhisper('typing', (e) => {
        if (!typingUsers.value.find(u => u.id === e.user.id)) {
            typingUsers.value.push(e.user);
        }

        clearTimeout(typingTimeout);
        typingTimeout = setTimeout(() => {
            typingUsers.value = typingUsers.value.filter(u => u.id !== e.user.id);
        }, 2000);
    });
</script>

<template>
    <div class="chat">
        <!-- Usuários online -->
        <div class="online-users">
            <span v-for="user in onlineUsers" :key="user.id" class="avatar">
                {{ user.name }}
            </span>
        </div>

        <!-- Mensagens -->
        <div class="messages">
            <div v-for="msg in messages" :key="msg.id">
                <strong>{{ msg.user.name }}:</strong> {{ msg.content }}
            </div>
        </div>

        <!-- Indicador de digitação -->
        <div v-if="typingUsers.length" class="typing">
            {{ typingUsers.map(u => u.name).join(', ') }}
            {{ typingUsers.length === 1 ? 'está' : 'estão' }} digitando...
        </div>

        <!-- Input -->
        <input @input="handleTyping" @keyup.enter="sendMessage" />
    </div>
</template>
```

---

## 8. Tratamento de Erros e Reconexão

### 8.1 Estado da Conexão

```javascript
// Verificar estado da conexão
Echo.connector.pusher.connection.bind('state_change', (states) => {
    console.log('Estado anterior:', states.previous);
    console.log('Estado atual:', states.current);
});

// Estados possíveis:
// - initialized
// - connecting
// - connected
// - unavailable
// - failed
// - disconnected
```

### 8.2 Reconexão Automática

```javascript
// O Echo/Pusher já faz reconexão automática, mas você pode customizar
Echo.connector.pusher.connection.bind('connected', () => {
    console.log('Conectado ao WebSocket');
    // Re-subscrever em canais se necessário
});

Echo.connector.pusher.connection.bind('disconnected', () => {
    console.log('Desconectado do WebSocket');
    // Mostrar indicador de offline
});

Echo.connector.pusher.connection.bind('error', (error) => {
    console.error('Erro no WebSocket:', error);
});
```

### 8.3 Composable com Estado de Conexão

```javascript
// composables/useWebSocketStatus.js
import { ref, onMounted, onUnmounted } from 'vue';

export function useWebSocketStatus() {
    const isConnected = ref(false);
    const connectionState = ref('initialized');

    const handleStateChange = (states) => {
        connectionState.value = states.current;
        isConnected.value = states.current === 'connected';
    };

    onMounted(() => {
        Echo.connector.pusher.connection.bind('state_change', handleStateChange);
        isConnected.value = Echo.connector.pusher.connection.state === 'connected';
    });

    onUnmounted(() => {
        Echo.connector.pusher.connection.unbind('state_change', handleStateChange);
    });

    return {
        isConnected,
        connectionState,
    };
}
```

```vue
<script setup>
import { useWebSocketStatus } from '@/composables/useWebSocketStatus';

const { isConnected, connectionState } = useWebSocketStatus();
</script>

<template>
    <div class="connection-indicator">
        <span :class="isConnected ? 'bg-green-500' : 'bg-red-500'" class="w-2 h-2 rounded-full" />
        <span>{{ isConnected ? 'Online' : 'Offline' }}</span>
    </div>
</template>
```

---

## 9. Configuração de Produção

### 9.1 Supervisor

```ini
; /etc/supervisor/conf.d/reverb.conf
[program:reverb]
process_name=%(program_name)s
command=php /var/www/app/artisan reverb:start --host=0.0.0.0 --port=8080
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/app/storage/logs/reverb.log
stopwaitsecs=3600
```

### 9.2 Nginx (Proxy para WebSocket)

```nginx
# /etc/nginx/sites-available/app.conf
server {
    listen 443 ssl http2;
    server_name app.com;

    # SSL config...

    # WebSocket proxy
    location /app {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }

    # Laravel app
    location / {
        # ...
    }
}
```

### 9.3 Configuração de Produção no .env

```env
REVERB_HOST="app.com"
REVERB_PORT=443
REVERB_SCHEME=https

VITE_REVERB_HOST="app.com"
VITE_REVERB_PORT=443
VITE_REVERB_SCHEME=https
```

### 9.4 Scaling Horizontal com Redis

```env
# .env
REVERB_SCALING_ENABLED=true
REVERB_SCALING_CHANNEL=reverb

REDIS_HOST=redis-cluster.internal
REDIS_PASSWORD=secret
REDIS_PORT=6379
```

```php
// config/reverb.php
'scaling' => [
    'enabled' => env('REVERB_SCALING_ENABLED', false),
    'channel' => env('REVERB_SCALING_CHANNEL', 'reverb'),
],
```

---

## 10. Broadcasting vs. Polling

| Aspecto | Polling | Broadcasting (Reverb) |
|---------|---------|----------------------|
| **Latência** | 1-30 segundos | Instantâneo |
| **Carga no servidor** | Alta (requisições constantes) | Baixa (conexão persistente) |
| **Carga no banco** | Alta (queries repetidas) | Nenhuma (push-based) |
| **Complexidade** | Baixa | Média |
| **Escalabilidade** | Limitada | Alta (com Redis) |
| **Custo** | Alto em escala | Baixo em escala |

**Quando usar Polling:**
- Dados que mudam raramente
- Aplicações simples sem infraestrutura WebSocket
- Fallback quando WebSocket não está disponível

**Quando usar Broadcasting:**
- Notificações em tempo real
- Chat e mensagens
- Dashboards ao vivo
- Colaboração multi-usuário
- Jogos e aplicações interativas

---

## 11. Casos de Uso Completos

### 11.1 Sistema de Notificações

```php
// app/Events/NotificationCreated.php
class NotificationCreated implements ShouldBroadcast
{
    public function __construct(
        public Notification $notification,
        public User $user
    ) {}

    public function broadcastOn(): array
    {
        return [new PrivateChannel("notifications.{$this->user->id}")];
    }

    public function broadcastWith(): array
    {
        return [
            'id' => $this->notification->id,
            'type' => $this->notification->type,
            'title' => $this->notification->title,
            'message' => $this->notification->message,
            'read' => false,
            'created_at' => $this->notification->created_at->diffForHumans(),
        ];
    }
}
```

### 11.2 Dashboard em Tempo Real

```php
// app/Events/MetricsUpdated.php
class MetricsUpdated implements ShouldBroadcast
{
    public function __construct(
        public array $metrics
    ) {}

    public function broadcastOn(): array
    {
        return [new Channel('dashboard.metrics')];
    }
}

// Disparo periódico via Schedule
// app/Console/Kernel.php
$schedule->call(function () {
    $metrics = [
        'sales_today' => Order::whereDate('created_at', today())->sum('total'),
        'active_users' => User::where('last_active_at', '>', now()->subMinutes(5))->count(),
        'pending_orders' => Order::where('status', 'pending')->count(),
    ];

    broadcast(new MetricsUpdated($metrics));
})->everyMinute();
```

### 11.3 Edição Colaborativa

```php
// app/Events/DocumentCursorMoved.php
class DocumentCursorMoved implements ShouldBroadcastNow
{
    public function __construct(
        public int $documentId,
        public int $userId,
        public array $position
    ) {}

    public function broadcastOn(): array
    {
        return [new PresenceChannel("document.{$this->documentId}")];
    }
}
```

---

## 12. Troubleshooting

### Problemas Comuns

**1. Eventos não chegam no frontend**
- Verifique se o Reverb está rodando (`php artisan reverb:start --debug`)
- Verifique o canal correto no `channels.php`
- Confirme que o evento implementa `ShouldBroadcast`

**2. Erro de autenticação em canais privados**
- Verifique o `channels.php` retorna `true` ou um array
- Confirme que o usuário está autenticado
- Verifique o CSRF token no Echo

**3. Conexão fecha após 60 segundos**
- Configure `proxy_read_timeout` no Nginx
- Verifique firewall/load balancer timeouts

**4. "Client not found"**
- Verifique `REVERB_APP_KEY` e `VITE_REVERB_APP_KEY`
- Confirme que o Reverb está no host/porta corretos

---

## 13. Exercícios Práticos

### Exercício 1: Chat em Tempo Real
Implemente um chat com:
- Canais privados por conversa
- Indicador "está digitando"
- Lista de usuários online (presence)
- Histórico persistido no banco

### Exercício 2: Notificações Toast
Crie um sistema de notificações com:
- Notificações push via broadcast
- Toast UI com animações
- Marcar como lida
- Contador de não lidas

### Exercício 3: Dashboard Ao Vivo
Construa um dashboard com:
- Métricas atualizadas a cada 30s
- Gráficos que se atualizam
- Alerta quando um limite é atingido

### Exercício 4: Editor Colaborativo
Implemente um editor onde:
- Múltiplos usuários veem o mesmo documento
- Cursores de outros usuários são visíveis
- Mudanças são sincronizadas em tempo real

---

## Conclusão

Dominar o tempo real é o que diferencia aplicações "estáticas" de experiências modernas de alta interatividade. Com o Reverb, o ecossistema Laravel fechou a última lacuna que o separava de frameworks focados em Node.js para este fim. A combinação de Reverb + Echo + Vue 3 permite criar experiências colaborativas e responsivas com a mesma produtividade que você já tem no restante da stack Laravel.
