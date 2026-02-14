# Comunicação em Tempo Real (WebSockets)

## Introdução
Aplicações modernas exigem interatividade instantânea. Em vez do cliente perguntar ao servidor se há novos dados (polling), o servidor "empurra" a informação para o cliente no momento em que ela acontece. No Laravel 11, isso é feito nativamente com o **Laravel Reverb**.

---

## 1. O Ecossistema Real-time

### 1.1 Laravel Reverb
Um servidor de WebSockets de alta performance, escrito em PHP e otimizado para o Laravel. Ele substitui soluções pagas como Pusher ou configurações complexas como o Soketi.

### 1.2 Laravel Echo
Uma biblioteca JavaScript que facilita a subscrição em canais e a escuta de eventos transmitidos pelo Laravel.

---

## 2. Tipos de Canais

1.  **Public Channels:** Qualquer um pode ouvir (ex: cotação de moedas, notícias).
2.  **Private Channels:** Exigem autenticação via Laravel Sanctum/Fortify (ex: notificações do usuário, mensagens privadas).
3.  **Presence Channels:** Como os canais privados, mas informam quem mais está online no canal (ex: "quem está visualizando este documento").

---

## 3. Implementação Prática

### 3.1 Criando um Evento Transmissível (Broadcast)
```php
namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderStatusUpdated implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(public $order) {}

    public function broadcastOn(): array
    {
        return [
            new PrivateChannel("orders.{$this->order->user_id}"),
        ];
    }
}
```

### 3.2 Escutando no Vue 3 (Inertia)
```javascript
import { onMounted } from 'vue';

onMounted(() => {
    Echo.private(`orders.${props.auth.user.id}`)
        .listen('OrderStatusUpdated', (e) => {
            console.log('Status do pedido mudou:', e.order.status);
            // Atualizar estado local ou mostrar notificação
        });
});
```

---

## 4. Broadcasting vs. Polling
- **Polling:** Cliente pergunta a cada 5 segundos. Gera carga desnecessária no banco e atraso na informação.
- **Broadcasting (Reverb):** Conexão persistente. Latência zero e menor consumo de recursos para atualizações frequentes.

---

## 5. Casos de Uso Comuns
- **Notificações Push:** Avisar o usuário sem que ele precise dar F5.
- **Dashboards:** Gráficos que se movem sozinhos conforme vendas ocorrem.
- **Colaboração Multi-usuário:** Ver quem está digitando ou editando um recurso.

---

## 6. Configuração de Produção
O Reverb exige que o servidor mantenha milhares de conexões abertas.
- **Nginx:** Deve ser configurado para permitir `Upgrade` de conexões HTTP para WebSocket.
- **Supervisor:** Necessário para manter o processo do Reverb (`php artisan reverb:start`) rodando continuamente.

---

## Conclusão
Dominar o tempo real é o que diferencia aplicações "estáticas" de experiências modernas de alta interatividade. Com o Reverb, o ecossistema Laravel fechou a última lacuna que o separava de frameworks focados em Node.js para este fim.
