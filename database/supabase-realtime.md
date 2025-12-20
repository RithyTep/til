# Supabase Realtime - Live Data Subscriptions

<div align="center">

![Supabase](https://img.shields.io/badge/Supabase-3FCF8E?style=for-the-badge&logo=supabase&logoColor=white)
![Realtime](https://img.shields.io/badge/Realtime-WebSocket-blue?style=for-the-badge)
![Postgres](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)

*Subscribe to database changes in real-time with PostgreSQL and WebSockets.*

![Supabase](https://supabase.com/_next/image?url=%2Fimages%2Fproduct%2Frealtime%2Fheader--dark.png&w=1920&q=75)

</div>

## Realtime Features

```
Feature                        Use Case
──────────────────────────     ──────────────────────────
Postgres Changes               Live data updates
Broadcast                      User presence, cursors
Presence                       Who's online
```

## Installation

```bash
npm install @supabase/supabase-js
```

## Setup

```typescript
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)
```

## Subscribe to Database Changes

```typescript
// Subscribe to all changes on a table
const channel = supabase
  .channel('db-changes')
  .on(
    'postgres_changes',
    {
      event: '*',  // INSERT, UPDATE, DELETE, or *
      schema: 'public',
      table: 'messages',
    },
    (payload) => {
      console.log('Change received:', payload)
      // payload.eventType: 'INSERT' | 'UPDATE' | 'DELETE'
      // payload.new: new row data
      // payload.old: old row data (for UPDATE/DELETE)
    }
  )
  .subscribe()

// Cleanup
channel.unsubscribe()
```

## Filter Changes

```typescript
// Only listen to specific rows
const channel = supabase
  .channel('room-messages')
  .on(
    'postgres_changes',
    {
      event: 'INSERT',
      schema: 'public',
      table: 'messages',
      filter: 'room_id=eq.123',  // Only room 123
    },
    (payload) => {
      addMessage(payload.new)
    }
  )
  .subscribe()

// Multiple filters
const channel2 = supabase
  .channel('user-updates')
  .on(
    'postgres_changes',
    {
      event: 'UPDATE',
      schema: 'public',
      table: 'users',
      filter: 'status=eq.online',
    },
    handleOnlineUsers
  )
  .subscribe()
```

## React Hook Pattern

```typescript
import { useEffect, useState } from 'react'
import { supabase } from '@/lib/supabase'

interface Message {
  id: number
  content: string
  user_id: string
  created_at: string
}

export function useMessages(roomId: string) {
  const [messages, setMessages] = useState<Message[]>([])

  useEffect(() => {
    // Initial fetch
    const fetchMessages = async () => {
      const { data } = await supabase
        .from('messages')
        .select('*')
        .eq('room_id', roomId)
        .order('created_at', { ascending: true })

      setMessages(data || [])
    }

    fetchMessages()

    // Subscribe to changes
    const channel = supabase
      .channel(`room-${roomId}`)
      .on(
        'postgres_changes',
        {
          event: 'INSERT',
          schema: 'public',
          table: 'messages',
          filter: `room_id=eq.${roomId}`,
        },
        (payload) => {
          setMessages((prev) => [...prev, payload.new as Message])
        }
      )
      .on(
        'postgres_changes',
        {
          event: 'DELETE',
          schema: 'public',
          table: 'messages',
          filter: `room_id=eq.${roomId}`,
        },
        (payload) => {
          setMessages((prev) =>
            prev.filter((m) => m.id !== payload.old.id)
          )
        }
      )
      .subscribe()

    return () => {
      supabase.removeChannel(channel)
    }
  }, [roomId])

  return messages
}
```

## Broadcast (Custom Events)

```typescript
// Send events to other clients (no database)
const channel = supabase.channel('room-1')

// Subscribe
channel
  .on('broadcast', { event: 'cursor' }, ({ payload }) => {
    updateCursor(payload.userId, payload.x, payload.y)
  })
  .subscribe()

// Send
channel.send({
  type: 'broadcast',
  event: 'cursor',
  payload: { userId: 'user-123', x: 100, y: 200 },
})
```

## Presence (Who's Online)

```typescript
const channel = supabase.channel('online-users')

// Track presence
channel
  .on('presence', { event: 'sync' }, () => {
    const state = channel.presenceState()
    console.log('Online users:', state)
    // { 'user-1': [{ online_at: '...', user_id: '...' }], ... }
  })
  .on('presence', { event: 'join' }, ({ key, newPresences }) => {
    console.log('User joined:', key, newPresences)
  })
  .on('presence', { event: 'leave' }, ({ key, leftPresences }) => {
    console.log('User left:', key, leftPresences)
  })
  .subscribe(async (status) => {
    if (status === 'SUBSCRIBED') {
      // Track this user
      await channel.track({
        user_id: currentUser.id,
        online_at: new Date().toISOString(),
      })
    }
  })

// Untrack when leaving
await channel.untrack()
```

## Chat Room Example

```typescript
// components/ChatRoom.tsx
'use client'
import { useEffect, useState, useRef } from 'react'
import { supabase } from '@/lib/supabase'

interface Message {
  id: string
  content: string
  user_id: string
  user_name: string
  created_at: string
}

export function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<Message[]>([])
  const [onlineUsers, setOnlineUsers] = useState<string[]>([])
  const [input, setInput] = useState('')
  const bottomRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    // Fetch existing messages
    supabase
      .from('messages')
      .select('*')
      .eq('room_id', roomId)
      .order('created_at')
      .then(({ data }) => setMessages(data || []))

    // Subscribe to new messages
    const channel = supabase
      .channel(`room-${roomId}`)
      .on(
        'postgres_changes',
        {
          event: 'INSERT',
          schema: 'public',
          table: 'messages',
          filter: `room_id=eq.${roomId}`,
        },
        (payload) => {
          setMessages((m) => [...m, payload.new as Message])
        }
      )
      .on('presence', { event: 'sync' }, () => {
        const state = channel.presenceState()
        setOnlineUsers(Object.keys(state))
      })
      .subscribe(async (status) => {
        if (status === 'SUBSCRIBED') {
          await channel.track({ user_id: 'current-user' })
        }
      })

    return () => {
      supabase.removeChannel(channel)
    }
  }, [roomId])

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' })
  }, [messages])

  const sendMessage = async () => {
    if (!input.trim()) return

    await supabase.from('messages').insert({
      room_id: roomId,
      content: input,
      user_id: 'current-user',
      user_name: 'Me',
    })

    setInput('')
  }

  return (
    <div className="flex flex-col h-screen">
      <div className="p-2 border-b">
        Online: {onlineUsers.length} users
      </div>

      <div className="flex-1 overflow-y-auto p-4 space-y-2">
        {messages.map((msg) => (
          <div key={msg.id} className="p-2 bg-gray-100 rounded">
            <strong>{msg.user_name}:</strong> {msg.content}
          </div>
        ))}
        <div ref={bottomRef} />
      </div>

      <div className="p-4 border-t flex gap-2">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === 'Enter' && sendMessage()}
          className="flex-1 border rounded px-3 py-2"
          placeholder="Type a message..."
        />
        <button
          onClick={sendMessage}
          className="px-4 py-2 bg-blue-500 text-white rounded"
        >
          Send
        </button>
      </div>
    </div>
  )
}
```

## Enable Realtime on Tables

```sql
-- In Supabase SQL Editor
ALTER PUBLICATION supabase_realtime ADD TABLE messages;

-- Or for specific operations
ALTER PUBLICATION supabase_realtime ADD TABLE messages
  WITH (publish = 'insert, update, delete');
```

## Row Level Security

```sql
-- Ensure RLS works with realtime
CREATE POLICY "Users can view messages in their rooms"
  ON messages FOR SELECT
  USING (
    room_id IN (
      SELECT room_id FROM room_members WHERE user_id = auth.uid()
    )
  );
```

---

*Learned: December 20, 2025*
*Tags: Supabase, Realtime, WebSocket, PostgreSQL, Live Data*
