# pulseloop-chat-system
Chat Architecture — PulseLoop

Transport Layer

Socket.io over WebSocket (fallback to HTTP long-polling)
One socket connection per authenticated user (agent or customer), authenticated via JWT on handshake
Server maintains userId → socketId map for direct addressing

Conversation Layer

Each conversation = a Socket.io room (room:conversation_id)
On join_conversation, server verifies participant, adds socket to room, loads recent message history from MongoDB
Conversation state (participants, status: active/closed) persisted in a Conversation collection
Messages persisted in a Message collection with conversation_id, sender_id, sequence_number, client_message_id (for de-dup)

Presence Layer

Separate from conversation rooms — tracked per-user, not per-conversation
In-memory store (Redis in prod, Map in dev) of userId → {status, lastSeen}
Status changes broadcast only to users who share an active conversation with that user, not globally

Event Flow

connect → authenticate → join_conversation → send_message → server persists + assigns sequence_number
  → broadcast to room → recipient emits delivered_ack → sender sees "delivered"
  → recipient views message → emits read_ack → sender sees "read"

Reliability rules

Every outbound message carries a client_message_id; server dedupes on this key
On reconnect, client requests events since last_known_sequence_number per conversation (replay, not full resync)
Delivery status: pending → sent → delivered → read, with failed on timeout + retry

Architecture diagram :
<img width="2720" height="2240" alt="pulseloop_chat_architecture" src="https://github.com/user-attachments/assets/5f72e72c-dc2d-4823-a364-88580191db6e" />


