# 03 — Transport abstraction

## Why this exists

Socket.IO and Phoenix Channels have different room models, ack callbacks, and reconnect payloads. UI must not care. Tests must run without a broker.

## Interface (TypeScript sketch)

```ts
export type ConnectionState =
  | "offline"
  | "connecting"
  | "connected"
  | "reconnecting"
  | "dead"; // auth failed / protocol mismatch

export interface SendMessageInput {
  chatId: string;
  type: "text";
  body: string;
  idempotencyKey: string;
  localId: string;
  clientTs: number; // unix ms
  replyToMessageId?: string;
}

export interface SendAck {
  idempotencyKey: string;
  localId: string;
  messageId: string;
  seq: number;
  serverTs: number;
  duplicate: boolean; // true if server already had this key
}

export interface TransportEvents {
  "connection.state": ConnectionState;
  "message.created": ServerMessage;
  "message.delivered": ReceiptEvent;
  "message.read": ReceiptEvent;
  "typing": TypingEvent;
  "presence": PresenceEvent;
  "chat.seq.gap": { chatId: string; expected: number; got: number };
}

export interface ChatTransport {
  connect(opts: { accessToken: string; deviceId: string }): Promise<void>;
  disconnect(): Promise<void>;
  sendMessage(input: SendMessageInput): Promise<SendAck>;
  sendReceipt(input: ReceiptInput): Promise<void>;
  sendTyping(input: { chatId: string; isTyping: boolean }): Promise<void>;
  joinChat(chatId: string): Promise<void>;
  leaveChat(chatId: string): Promise<void>;
  on<K extends keyof TransportEvents>(
    event: K,
    handler: (payload: TransportEvents[K]) => void
  ): () => void;
}
```

`ChatClient` owns outbox + this interface. `SocketIOTransport` and `PhoenixTransport` implement `ChatTransport`.

## Ack model

Phase 1 Socket.IO: `socket.emit("message.send", payload, ackCallback)` with a **client timeout** (e.g. 10s). Timeout ≠ failure of persist; it means unknown. Outbox stays `in_flight` until:

- ack arrives (success or duplicate), or
- server later sends `message.created` echoing the same `idempotencyKey` to the sender, or
- a sync pull returns that key.

Never generate a new `idempotencyKey` on retry.

Phoenix: `push("message:send", payload)` + reply. Same `SendAck` shape.

## Capability flags

```ts
interface TransportCaps {
  binary: boolean;          // protobuf later
  serverAckTimeoutMs: number;
  needsStickyLb: boolean;   // Socket.IO long-polling fallback: true
  roomsAreServerSide: boolean;
}
```

Use flags for behavior, not `if (transportName === "socketio")` in domain code.

## Connection bootstrap

On `connect`:

1. Transport authenticates (`auth` in Socket.IO handshake / Phoenix `connect` params) with the **access token from credential login** (not a WhatsApp primary-device link).
2. Server returns `{ userId, deviceId, protocolVersion }`. Multiple devices of the same `userId` may be connected at once.
3. Client does **HTTP sync** (chat list + cursors), not a giant WS dump.
4. Client `joinChat` for currently open chat + optionally all unmuted chats (Phase 1: join all memberships, cap later).

## Error taxonomy (transport → ChatClient)

| Code | Meaning | Outbox action |
| --- | --- | --- |
| `UNAUTHENTICATED` | Token dead | Pause worker, refresh/login |
| `FORBIDDEN` | Not a member | Mark local message `failed_terminal` |
| `RATE_LIMITED` | 429 | Backoff, keep same key |
| `VALIDATION` | Bad body | Terminal fail |
| `CONFLICT` | Protocol | Dev error |
| `UNAVAILABLE` | 5xx / disconnect | Retry |
| `TIMEOUT` | No ack | Retry / wait for echo |

## Testing

`LoopbackTransport`: in-memory bus. UI tests and outbox tests never start Redis.

Contract tests (same suite, two adapters):

- send → ack with `messageId`
- send twice same key → `duplicate: true`, same `messageId`
- disconnect before ack → retry → still one server row
