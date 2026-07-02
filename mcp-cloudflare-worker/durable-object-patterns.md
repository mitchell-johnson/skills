# Durable Object Patterns for McpAgent

Patterns that use the Durable Object storage and alarm APIs available on `this.ctx` inside an `McpAgent` subclass.

## Caching with Daily Expiry

Cache expensive upstream calls in Durable Object storage, invalidated by date stamp:

```typescript
export class MyMCP extends McpAgent<Env, Record<string, never>, UserProps> {
  /** Today's date stamp for cache invalidation */
  private todayStamp(): string {
    return new Date().toISOString().slice(0, 10);
  }

  /** Get from cache if same day */
  private async cacheGet<T>(key: string): Promise<T | undefined> {
    const entry = await this.ctx.storage.get<{ date: string; data: T }>(`cache:${key}`);
    if (entry && entry.date === this.todayStamp()) {
      return entry.data;
    }
    return undefined;
  }

  /** Set cache with today's date */
  private async cacheSet<T>(key: string, data: T): Promise<void> {
    await this.ctx.storage.put(`cache:${key}`, {
      date: this.todayStamp(),
      data,
    });
  }

  /** Get cached or fetch fresh */
  private async cached<T>(key: string, fetcher: () => Promise<T>): Promise<T> {
    const hit = await this.cacheGet<T>(key);
    if (hit !== undefined) return hit;
    const fresh = await fetcher();
    await this.cacheSet(key, fresh);
    return fresh;
  }

  async init() {
    this.server.tool(
      "get_data",
      "Get data (cached daily)",
      {},
      async () => {
        const data = await this.cached("all_data", () => this.fetchAllData());
        return { content: [{ type: "text", text: JSON.stringify(data) }] };
      }
    );
  }
}
```

## WebSocket Keepalive with Alarms

Keep a long-lived upstream connection (e.g. a WebSocket to an external service) alive with periodic pings, and disconnect after idle so the Durable Object can hibernate:

```typescript
const KEEPALIVE_INTERVAL_MS = 30_000;
const IDLE_TIMEOUT_MS = 10 * 60_000;

export class MyMCP extends McpAgent<Env, Record<string, never>, UserProps> {
  private client: ExternalClient | null = null;
  private lastActivity = Date.now();

  private scheduleNextAlarm(): void {
    try {
      this.ctx.storage.setAlarm(Date.now() + KEEPALIVE_INTERVAL_MS);
    } catch {
      // Alarm API may not be available in tests
    }
  }

  override readonly alarm = async (): Promise<void> => {
    if (!this.client) return;

    const idleTime = Date.now() - this.lastActivity;
    if (idleTime > IDLE_TIMEOUT_MS) {
      // Idle too long - disconnect to allow hibernation
      this.client.disconnect();
      this.client = null;
      return;
    }

    // Send keepalive ping
    this.client.ping();
    this.scheduleNextAlarm();
  };

  private async ensureClient(): Promise<ExternalClient> {
    this.lastActivity = Date.now();
    
    if (this.client) return this.client;

    this.client = new ExternalClient();
    await this.client.connect();
    this.scheduleNextAlarm();

    return this.client;
  }
}
```
