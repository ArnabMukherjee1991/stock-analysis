# Day 5: WebSocket + Kafka Real-Time Fan-Out

## Goal (2 hours)
Add WebSocket (STOMP) to the consumer app. Bridge Kafka → WebSocket so real-time stock updates broadcast to all connected browser clients. Build on the `Sinks.Many` hot Flux from Day 4.

---

## Part 1 — WebSocket + STOMP Config in Consumer (30 min)

The dependency was already added in Day 4. Now wire the configuration.

### Create `WebSocketConfig.java` in consumer
**File:** `exchange-data-consumer/src/main/java/com/myexchange/consumer/config/WebSocketConfig.java`

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // In-memory broker for topics prefixed with /topic
        registry.enableSimpleBroker("/topic");
        // Messages from clients to server go to /app prefix
        registry.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // WebSocket handshake endpoint — connect here
        registry.addEndpoint("/ws")
            .setAllowedOriginPatterns("*") // dev only — restrict in prod
            .withSockJS();
    }
}
```

---

## Part 2 — Kafka → WebSocket Bridge Service (40 min)

### Create `KafkaWebSocketBridgeService.java` in consumer
**File:** `exchange-data-consumer/src/main/java/com/myexchange/consumer/service/KafkaWebSocketBridgeService.java`

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class KafkaWebSocketBridgeService {

    private final ReactiveStockConsumerService reactiveStockConsumerService;
    private final SimpMessagingTemplate messagingTemplate;

    @PostConstruct
    public void bridgeKafkaToWebSocket() {
        reactiveStockConsumerService.streamStockUpdates()
            .doOnNext(dto -> {
                // Broadcast to all subscribers on /topic/stocks
                messagingTemplate.convertAndSend("/topic/stocks", dto);
                log.debug("Broadcasted stock update: {}", dto.getStock());
            })
            .doOnNext(dto -> {
                // User-specific message: each stock symbol gets its own channel
                // Client subscribes to /topic/stock/AAPL for AAPL-only updates
                messagingTemplate.convertAndSend(
                    "/topic/stock/" + dto.getStock(), dto);
            })
            .subscribe(
                dto -> {},
                error -> log.error("WebSocket bridge error", error)
            );
    }
}
```

> **Architecture note:** `SimpMessagingTemplate` is thread-safe and works with both imperative and reactive code — it internally dispatches on the message broker's thread pool.

---

## Part 3 — WebSocket REST Endpoint for Connection Info (20 min)

### Add WebSocket info endpoint to `ListenerController.java`:
```java
// Add field:
private final KafkaWebSocketBridgeService bridgeService;

@GetMapping("/ws-info")
public Map<String, String> wsConnectionInfo() {
    return Map.of(
        "websocket_endpoint", "ws://localhost:8001/ws",
        "all_stocks_topic", "/topic/stocks",
        "per_stock_topic", "/topic/stock/{SYMBOL}",
        "stomp_client", "Use SockJS + STOMP.js in browser"
    );
}
```

---

## Part 4 — Test with Simple HTML WebSocket Client (20 min)

### Create test HTML file
**File:** `exchange-data-consumer/src/main/resources/static/ws-test.html`

```html
<!DOCTYPE html>
<html>
<head><title>Stock WebSocket Test</title></head>
<body>
<h2>Live Stock Feed</h2>
<div id="output" style="font-family: monospace; border: 1px solid #ccc; padding: 10px; height: 300px; overflow-y: scroll;"></div>
<input id="symbol" placeholder="Symbol (e.g., AAPL)" />
<button onclick="subscribeSymbol()">Subscribe to Symbol</button>

<script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
<script>
const socket = new SockJS('http://localhost:8001/ws');
const stompClient = Stomp.over(socket);

stompClient.connect({}, frame => {
    console.log('Connected:', frame);

    // Subscribe to ALL stock updates
    stompClient.subscribe('/topic/stocks', msg => {
        const data = JSON.parse(msg.body);
        const div = document.getElementById('output');
        div.innerHTML += `<p>${new Date().toISOString()} | ${data.stock}: O=${data.open} C=${data.close}</p>`;
        div.scrollTop = div.scrollHeight;
    });
});

function subscribeSymbol() {
    const sym = document.getElementById('symbol').value.toUpperCase();
    stompClient.subscribe(`/topic/stock/${sym}`, msg => {
        const data = JSON.parse(msg.body);
        console.log(`${sym} update:`, data);
    });
}
</script>
</body>
</html>
```

Open at: `http://localhost:8001/ws-test.html`, then trigger stock data from producer.

---

## Part 5 — Scaling Scenario Discussion (10 min)

Think through these scenarios — will your current setup handle them?

### Scenario 1: Multiple Consumer Instances
If you run 2 instances of `exchange-data-consumer`:
- Each Kafka partition goes to exactly ONE consumer instance
- STOMP messages go only to WebSocket clients connected to THAT instance
- **Problem:** If AAPL partition goes to Instance A, clients on Instance B won't see AAPL updates

**Solution (for later/interview):** Use a Redis pub/sub broker (`registry.enableStompBrokerRelay()` with Redis or RabbitMQ) instead of in-memory broker. Each consumer instance relays to shared broker → all WebSocket servers broadcast.

### Scenario 2: 100K WebSocket Clients
With in-memory STOMP broker, `convertAndSend("/topic/stocks", dto)` loops through all subscriptions synchronously.

**Solution:** Use `@SendTo` or async dispatch + topic-based segmentation (per-symbol topics limit fan-out scope).

---

## Theory Checkpoint (20 min)

1. What is STOMP? How does it differ from raw WebSocket?
2. If Kafka consumer crashes mid-broadcast to 100K clients — what's the recovery strategy?
3. Why is the in-memory broker NOT suitable for horizontally scaled deployments?
4. What is the "latency budget" from Kafka produce → WebSocket delivery? What are the bottlenecks?

---

## Day 5 Deliverables
- [ ] `WebSocketConfig.java` configured with STOMP and SockJS
- [ ] `KafkaWebSocketBridgeService.java` bridges Kafka Flux to STOMP topics
- [ ] Both `/topic/stocks` (all) and `/topic/stock/{SYMBOL}` (per-symbol) working
- [ ] `ws-test.html` opens, connects, and shows live stock updates when producer sends data
- [ ] Understood multi-instance WebSocket scaling limitation with in-memory broker
