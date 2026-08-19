# Chatbot Implementation Plan - New Web Application (e2-student-app-web)

## Table of Contents

1. [Migration Overview](#migration-overview)
2. [Key Differences: Legacy vs New](#key-differences-legacy-vs-new)
3. [What to Keep, Replace, and Remove](#what-to-keep-replace-and-remove)
4. [Difficulties & Edge Cases](#difficulties--edge-cases)
5. [Architecture Design (Online-Only)](#architecture-design-online-only)
6. [Component Structure](#component-structure)
7. [State Management Strategy](#state-management-strategy)
8. [API Integration Layer](#api-integration-layer)
9. [New Backend Endpoints Needed](#new-backend-endpoints-needed)
10. [Implementation Phases](#implementation-phases)
11. [File-by-File Implementation Guide](#file-by-file-implementation-guide)
12. [Feature Parity Checklist](#feature-parity-checklist)
13. [Things to Consider](#things-to-consider)

---

## Migration Overview

### From
- **e2-student-portal** (Electron, offline-first, RxDB, Web Workers, JavaScript)

### To
- **e2-student-app-web** (Browser SPA, online-only, React Query/Zustand, TypeScript)

### Core Philosophy Change

| Aspect | Legacy (Electron) | New (Web) |
|---|---|---|
| Connectivity | Offline-first, eventual sync | Online-only, real-time |
| Data Storage | RxDB (IndexedDB) local-first | Server-authoritative, client cache |
| Workers | chat.worker.js + sync.worker.js | None needed |
| Language | JavaScript | TypeScript (strict) |
| State | RxDB observables + React state | React Query + lightweight store |
| UI Library | Custom SCSS + reactjs-popup | Existing design system in app |
| Routing | React Router v5 | React Router v6 (already in app) |
| Backend Gateway | e2-sp-service (bundled locally) | e2-sp-service (remote, via proxy) |

---

## Key Differences: Legacy vs New

### 1. No Offline Support Needed
The entire `PendingUpload` queue, sync workers, and local-first pattern can be **eliminated**. All operations are direct API calls with immediate responses.

### 2. No Web Workers
- No `chat.worker.js` → Proactive conversations handled differently (push notifications, server-sent events, or polling)
- No `sync.worker.js` → No sync needed; data is always live from server

### 3. No RxDB
- No local database
- No reactive observables (`useObservable` replaced by React Query subscriptions)
- No schema definitions for local models

### 4. TypeScript Instead of JavaScript
- All components, hooks, services typed
- API response types defined
- Props interfaces for all components

### 5. Proxy Already Configured
`e2-student-app-web/src/setupProxy.ts` already has:
```typescript
'/e2-sp-service': 'http://localhost:5017'
```
All chatbot calls will go through this existing proxy → `e2-sp-service` → `e2-chatbot`.

### 6. No Bundled Backend
Unlike the Electron app (which runs `e2-sp-service` locally at port 8000), the web app calls a remote `e2-sp-service`. This means:
- No cold-start issues
- No local MongoDB dependency
- Server handles all persistence

---

## What to Keep, Replace, and Remove

### ✅ Keep (Port to TypeScript)
| Feature | Reason |
|---|---|
| Chat UI components | Same UX (avatar, container, dialog, messages) |
| Like/Dislike flow | Same backend logic in e2-chatbot |
| Instructor escalation | Same backend flow |
| Smart Library integration | Same external app |
| Question suggestions (autocomplete) | Same API endpoint |
| Feedback state tracking | Same escalation logic |
| File viewer in messages | Same content display |
| Infinite scroll / pagination | Same UX pattern |
| Bilingual support (EN/AR + RTL) | Same requirement |
| Related questions chips | Same backend feature |
| Message timestamp formatting | Same UX |

### 🔄 Replace (Different Implementation)
| Feature | Legacy | New |
|---|---|---|
| Local data storage | RxDB | React Query cache |
| Message observables | `useObservable` → RxDB | `useQuery` → API |
| Background workers | chat.worker.js | Polling or WebSocket |
| Offline queue | PendingUpload | Direct API calls (error handling) |
| API layer | `data-store.js` | Service modules with Axios/fetch |
| Proactive conversations | Worker + local queue | Server push / polling |
| Unread badge count | Local RxDB count | API call or WebSocket |
| Guided tour | react-shepherd | App's existing tour system or new |
| Chat history fetch | Sync download + merge | Direct paginated API call |
| Session management | Client-generated sessionId | Can stay same or server-managed |
| Theme/locale | Custom hooks (useTheme, useLocale) | App's existing context providers |

### ❌ Remove (Not Needed)
| Feature | Reason |
|---|---|
| RxDB schemas/models | No local DB |
| sync.worker.js | No offline sync |
| chat.worker.js | No background worker |
| PendingUpload queue | No offline queue |
| worker-handler.js | No workers |
| Comlink integration | No inter-thread communication |
| DataStore class (as-is) | Replaced by service modules |
| `ensureConnection()` | Always online (use error boundaries) |
| `handleOffline` param | Not relevant |
| Local message generation from alerts | Server provides all messages |
| `addConversationsToQueue()` | Proactive convos managed server-side |
| `bulkUpsertQna()` | No local bulk operations |
| conversation-config.js (partially) | Simplify for UI display only |

---

## Difficulties & Edge Cases

### 1. Proactive Conversations Without Workers
**Problem:** Legacy uses `chat.worker.js` running every 5s to pop conversation messages from a local queue. Web app has no workers.

**Solutions (pick one):**
- **Option A - Server Push (WebSocket/SSE):** Backend sends proactive messages via WebSocket. Best UX but most complex.
- **Option B - Polling:** Poll an endpoint every 30s for pending proactive messages. Simpler, slightly delayed.
- **Option C - On-Open Fetch:** Fetch pending proactive messages when chat opens. Simplest, but conversations only show when user opens chat.

**Recommendation:** Start with **Option C** (fetch on open), upgrade to **Option B** (polling) for avatar bubble, then eventually **Option A** (WebSocket).

### 2. Avatar Bubble Messages Without Workers
**Problem:** Worker updates `chatbotbubblemessage` collection for floating avatar bubbles. No local DB in web app.

**Solution:** Poll a lightweight endpoint (e.g., `GET /e2-chatbot/pending-bubble`) every 30-60s, or combine with unread count polling.

### 3. Chat History Pagination
**Problem:** Legacy loads ALL history into RxDB during sync, then paginates locally. Web app must paginate from API.

**Solution:** Use cursor-based or offset-based pagination. `GET /e2-chatbot/chat?userId=X&courseId=Y&skip=0&limit=20&sortByDate=desc` for newest-first, then load older on scroll-up.

**Edge case:** New messages arriving while user is scrolled up reviewing history. Use React Query's `refetchInterval` or WebSocket for new message notifications.

### 4. Optimistic Updates for Feedback
**Problem:** Legacy saves feedback to RxDB instantly (optimistic), then queues server update. Web app must wait for server.

**Solution:** Use React Query's `useMutation` with `onMutate` for optimistic updates. Rollback on error.

```typescript
const feedbackMutation = useMutation({
  mutationFn: (data) => chatService.updateFeedback(id, data),
  onMutate: async (newData) => {
    // Optimistic update in cache
    queryClient.setQueryData(['chat-message', id], old => ({...old, feedbackGiven: newData.feedback}));
  },
  onError: (err, data, context) => {
    // Rollback
    queryClient.setQueryData(['chat-message', id], context.previousData);
  }
});
```

### 5. Session Management
**Problem:** Legacy creates a `sessionId` on chat open, sends closure on chat close. Works because Electron app controls the lifecycle.

**Edge cases in web:**
- User closes browser tab without closing chat → session never closed
- Multiple browser tabs open with chat
- Token expiry mid-conversation

**Solution:**
- Use `beforeunload` event to send closure (unreliable but helps)
- Server-side session timeout (expire sessions after 10min inactivity)
- `visibilitychange` API to detect tab switches
- Session heartbeat every 60s

### 6. Real-time Message Updates
**Problem:** Legacy uses RxDB observables for real-time UI updates. When a message is created locally, UI updates instantly.

**Solution:** After sending a message and receiving response:
1. Add user message to React Query cache immediately (optimistic)
2. Show typing indicator
3. On API response, add bot message to cache
4. React Query's cache invalidation handles stale data

### 7. File Viewer Integration
**Problem:** Legacy opens files in an embedded viewer or Electron's system viewer. Web must use browser-based viewer.

**Solution:** Use the app's existing file viewer component or embed a PDF.js viewer. External files open in new tab.

### 8. Network Error Handling
**Problem:** Legacy queues failed requests for later. Web app must handle errors gracefully in real-time.

**Edge cases:**
- AI service timeout (currently 5 minutes!) - show "still thinking" state
- Network loss mid-message - retry with exponential backoff
- 503 chatbot disabled - hide chatbot entirely (already has healthcheck)

**Solution:** Error boundaries per feature area, toast notifications for transient errors, graceful degradation for chatbot unavailability.

### 9. Concurrent Like/Dislike Race Conditions
**Problem:** User rapidly clicks like/dislike on multiple messages. Legacy handles via local RxDB (atomic). Web must handle concurrent API calls.

**Solution:** Disable feedback buttons during mutation, use `mutationKey` per message ID to prevent duplicate calls.

### 10. Large Message History Performance
**Problem:** Legacy loads 500 messages into RxDB (indexed). Web must handle large history without memory issues.

**Solution:** Virtualized list (e.g., `react-window` or `@tanstack/virtual`) for rendering only visible messages. Keep paginated fetching.

### 11. Bilingual Content Rendering
**Problem:** Messages have `locales: [{text, locale}]` array. Must display correct language and detect RTL.

**Solution:** Utility function that picks text based on current locale:
```typescript
function getLocalizedText(message: ChatMessage, locale: string): string {
  const localized = message.message.locales?.find(l => l.locale === locale);
  return localized?.text || message.message.text;
}
```

### 12. Smart Library Deep Link
**Problem:** Legacy opens Smart Library app in Electron webview or external browser with complex URL params.

**Solution:** Open in new browser tab with same URL params. Ensure CORS and auth token passing are handled.

---

## Architecture Design (Online-Only)

### System Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ Browser (e2-student-app-web)                                      │
│                                                                    │
│  ┌────────────────┐   ┌───────────────┐   ┌──────────────────┐  │
│  │ React UI       │──▶│ Chat Service  │──▶│ React Query      │  │
│  │ (Components)   │   │ (API layer)   │   │ Cache + Mutations │  │
│  └────────────────┘   └───────┬───────┘   └──────────────────┘  │
│                                │                                   │
│  ┌────────────────┐            │                                   │
│  │ Chat Store     │◀───────────┘                                   │
│  │ (Zustand/      │  State: session, feedbackState,                │
│  │  Context)      │  isTyping, activeConversation                  │
│  └────────────────┘                                                │
└────────────────────────────────┬──────────────────────────────────┘
                                 │ HTTP via proxy
                                 ▼
┌────────────────────────────────────────────────────────────────────┐
│ e2-sp-service (remote)                                              │
│ Proxy: /e2-sp-service → http://sp-service-host                     │
│ Routes: /chat, /ai-question-suggestion, /healthcheck               │
└────────────────────────────────┬──────────────────────────────────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────┐
│ e2-chatbot (unchanged)                                              │
│ - Same escalation logic                                             │
│ - Same AI service integration                                       │
│ - Same MongoDB persistence                                          │
└────────────────────────────────────────────────────────────────────┘
```

### Data Flow for Sending a Message

```
User types → MessageInput → onSend()
  │
  ├── 1. chatStore.addMessage({type: 'User', text: query})  // Optimistic UI
  ├── 2. chatStore.setTyping(true)                           // Show indicator
  ├── 3. chatService.sendMessage({query, feedbackState, ...})
  │       └── POST /e2-sp-service/chat
  │           └── (proxied to) e2-chatbot → AI Service
  │               └── Response returned
  ├── 4. chatStore.addMessage({type: 'Bot', ...response})    // Add bot reply
  ├── 5. chatStore.setTyping(false)                          // Hide indicator
  └── 6. chatStore.updateFeedbackState(response)             // Track for escalation
```

---

## Component Structure

### Proposed File Tree

```
src/components/
├── organisms/
│   └── chatbot/
│       ├── ChatbotProvider.tsx           # Context provider (session, state)
│       ├── ChatbotAvatar.tsx             # Floating button + badge
│       ├── ChatbotAvatarBubble.tsx       # Tooltip bubble above avatar
│       ├── ChatbotContainer.tsx          # Main popup with tabs
│       ├── ChatbotDialog.tsx             # Chat conversation panel
│       ├── ChatMessage.tsx               # Individual message bubble
│       ├── ChatMessageContent.tsx        # Content renderer (files, images, games)
│       ├── ChatMessageActions.tsx        # Like/Dislike/Escalation buttons
│       ├── ChatMessageRelated.tsx        # Related question chips
│       ├── ChatInput.tsx                 # Text input with suggestions
│       ├── ChatSuggestions.tsx           # Autocomplete dropdown
│       ├── ChatTypingIndicator.tsx       # Animated typing dots
│       ├── ChatInstructorPicker.tsx      # Instructor selection modal
│       ├── ChatFileViewer.tsx            # Embedded file/PDF viewer
│       ├── ChatNotifications.tsx         # Notifications/Alerts tab
│       ├── ChatNotificationCard.tsx      # Individual alert card
│       └── index.ts                      # Barrel exports
│
├── molecules/
│   └── (shared small components like Badge, Tooltip, etc.)
│
└── atoms/
    └── (shared atomic elements)

src/services/
├── chatbot/
│   ├── chatbot.service.ts               # API calls to e2-chatbot
│   ├── chatbot.types.ts                 # TypeScript interfaces
│   ├── chatbot.constants.ts             # Action types, sources, etc.
│   └── chatbot.utils.ts                 # Text direction, date formatting

src/hooks/
├── chatbot/
│   ├── useChatMessages.ts              # React Query hook for messages
│   ├── useChatSend.ts                  # Mutation hook for sending
│   ├── useChatFeedback.ts             # Mutation hook for like/dislike
│   ├── useChatSuggestions.ts          # Query hook for autocomplete
│   ├── useChatSession.ts             # Session lifecycle management
│   ├── useChatHealthcheck.ts          # Chatbot enabled/disabled
│   ├── useChatNotifications.ts        # Alerts/notifications query
│   └── useChatUnreadCount.ts          # Unread badge count

src/store/
├── chatbot.store.ts                    # Zustand store for UI state
```

### Component Hierarchy

```
<MainLayout>
  <ChatbotProvider>                    # Wraps entire app
    <ChatbotAvatar />                  # Always visible (when enabled)
      ├── <Badge count={unread} />
      └── <ChatbotAvatarBubble />      # Proactive message tooltip
    <ChatbotContainer open={isOpen}>   # Popup/Drawer
      ├── <Tabs>
      │   ├── Tab: Chat
      │   │   └── <ChatbotDialog>
      │   │       ├── <ChatMessage /> × N    (virtualized list)
      │   │       │   ├── <ChatMessageContent />
      │   │       │   ├── <ChatMessageActions />
      │   │       │   └── <ChatMessageRelated />
      │   │       ├── <ChatTypingIndicator />
      │   │       └── <ChatInput>
      │   │           └── <ChatSuggestions />
      │   └── Tab: Notifications
      │       └── <ChatNotifications>
      │           └── <ChatNotificationCard /> × N
      └── <ChatInstructorPicker />      # Modal overlay
  </ChatbotProvider>
</MainLayout>
```

---

## State Management Strategy

### React Query (Server State)

```typescript
// Messages (paginated, newest first)
useInfiniteQuery(['chat-messages', courseId], fetchMessages, {
  getNextPageParam: (lastPage) => lastPage.nextCursor,
  refetchInterval: 30000, // Poll for new messages every 30s
});

// Unread count
useQuery(['chat-unread', courseId], fetchUnreadCount, {
  refetchInterval: 60000,
});

// Suggestions (debounced)
useQuery(['chat-suggestions', debouncedQuery], fetchSuggestions, {
  enabled: debouncedQuery.length >= 3,
});

// Healthcheck
useQuery(['chat-enabled'], fetchHealthcheck, {
  refetchInterval: 60000,
  staleTime: 30000,
});

// Notifications
useInfiniteQuery(['chat-notifications', courseId], fetchNotifications);
```

### Zustand/Context Store (Client UI State)

```typescript
interface ChatbotStore {
  // UI state
  isOpen: boolean;
  activeTab: 'chat' | 'notifications';
  isTyping: boolean;
  sessionId: string | null;

  // Feedback tracking (not persisted to server separately)
  feedbackState: FeedbackState | null;

  // Actions
  openChat: () => void;
  closeChat: () => void;
  setActiveTab: (tab: 'chat' | 'notifications') => void;
  startSession: () => void;
  endSession: () => void;
  setTyping: (typing: boolean) => void;
  updateFeedbackState: (feedback: FeedbackState) => void;
  resetFeedbackState: () => void;
}
```

---

## API Integration Layer

### Service Module (`src/services/chatbot/chatbot.service.ts`)

```typescript
import { apiClient } from '@/lib/api'; // App's existing Axios/fetch instance

const BASE = '/e2-sp-service/e2-chatbot';

export const chatbotService = {
  // Send message and get AI response
  sendMessage: (data: SendMessageRequest): Promise<SendMessageResponse> =>
    apiClient.post(`${BASE}/chat`, data),

  // Get chat history (paginated)
  getMessages: (params: GetMessagesParams): Promise<PaginatedMessages> =>
    apiClient.get(`${BASE}/chat`, { params }),

  // Update message feedback
  updateFeedback: (id: string, data: UpdateFeedbackRequest): Promise<ChatMessage> =>
    apiClient.patch(`${BASE}/chat/${id}`, data),

  // Get autocomplete suggestions
  getSuggestions: (query: string): Promise<string[]> =>
    apiClient.get(`${BASE}/ai-question-suggestion`, { params: { query } }),

  // Check if chatbot is enabled
  healthcheck: (): Promise<{ enableChatbot: boolean }> =>
    apiClient.get(`${BASE}/healthcheck`),

  // Create QA feedback (instructor escalation)
  createQaFeedback: (data: QaFeedbackRequest): Promise<void> =>
    apiClient.post('/e2-sp-service/e2-course/qa-feedback/', data),

  // Mark messages as read
  markAsRead: (ids: string[]): Promise<void> =>
    apiClient.post(`${BASE}/chat-bulk-patch`, { ids, readAt: new Date().toISOString() }),

  // Get notifications/alerts
  getNotifications: (params: NotificationParams): Promise<PaginatedNotifications> =>
    apiClient.get('/e2-sp-service/course-alerts', { params }),
};
```

### Type Definitions (`src/services/chatbot/chatbot.types.ts`)

```typescript
export interface ChatMessage {
  id: string;
  sessionId: string;
  questionId?: string;
  answerId?: string;
  courseId: string;
  userId: string;
  date: string;
  type: 'Bot' | 'User';
  source: MessageSource;
  isGreeting: boolean;
  readAt: string | null;
  hideFromChat: boolean;
  feedbackGiven: 'like' | 'dislike' | 'neutral' | null;
  feedbackState: FeedbackState | null;
  actionType: ActionType;
  query: string;
  relatedQuestions: string[];
  message: MessageContent;
}

export type ActionType = 'NO_ACTION' | 'ASK_INSTRUCTOR' | 'SHOW_INSTRUCTORS' | 'ASK_EXTERNAL_RESOURCE';

export type MessageSource =
  | 'I-STUDY' | 'SMART_LIBRARY' | 'small_talk' | 'thank_you_message'
  | 'ALERT_ASSIGNMENT' | 'ALERT_RECOMMENDATION' | 'ALERT_QA_FEEDBACK'
  | 'ALERT_STUDENT_AI_INTERVENTION' | 'ALERT_GAMES_AVAILABLE'
  | string; // conversation types

export interface MessageContent {
  text: string;
  locales?: { text: string; locale: string }[];
  question?: string;
  answer?: string;
  content?: {
    file?: FileContent;
    displayImage?: string;
    path?: string;
    pageNumber?: number;
    id?: string;
  };
  data?: {
    type: 'CONTENTS' | 'PRACTICE' | 'REMINDER';
    contents?: ContentItem[];
    programItemMetas?: ProgramItemMeta[];
  };
}

export interface FeedbackState {
  question: string;
  questionId: string;
  answerId?: string;
  feedbacks: {
    answerId: string;
    answer: string;
    value: 'like' | 'dislike';
  }[];
}

export interface SendMessageRequest {
  courseId: string;
  userId: string;
  type: 'User';
  typeOfChat: 'chat' | 'ask-smart-library' | 'qa-feedback' | 'closure';
  query: string;
  sessionId: string;
  feedbackState?: FeedbackState;
  locale?: string;
  askToSource?: string;
}

export interface SendMessageResponse {
  data: ChatMessage;
  response: {
    answer: string;
    relatedQuestions: string[];
    actionType: ActionType;
    content?: MessageContent['content'];
  };
}
```

---

## New Backend Endpoints Needed

The existing `e2-chatbot` and `e2-sp-service` backends mostly support the web app already. However, some new endpoints or modifications may be needed:

### 1. Chat History Pagination Enhancement
**Current:** `GET /chat?userId=X&courseId=Y&sortByDate=true` returns up to 500 records.
**Needed:** `GET /chat?userId=X&courseId=Y&skip=0&limit=20&sort=-date` with proper cursor/offset pagination.

**Impact:** Modify `e2-chatbot/src/services/chat/chat.service.js` → `find()` method.

### 2. Unread Count Endpoint
**Current:** No dedicated endpoint; legacy counts locally from RxDB.
**Needed:** `GET /chat/unread-count?userId=X&courseId=Y` returns `{ count: number }`.

**Impact:** New method in `e2-chatbot` chat service.

### 3. Proactive Messages Endpoint (Optional)
**Current:** Chat worker generates messages from local conversation queue.
**Needed:** `GET /chat/pending-proactive?userId=X&courseId=Y` returns messages that should be shown proactively.

**Impact:** New service in `e2-chatbot` or `e2-sp-service` that checks for pending alerts/recommendations.

### 4. Bulk Mark-as-Read (Already Exists)
**Current:** `POST /chat-bulk-patch` exists.
**Needed:** Ensure it's accessible via `e2-sp-service` proxy.

### 5. Session Management Enhancement (Optional)
**Current:** Client-managed sessions.
**Needed:** Server-side session timeout. Add TTL-based cleanup for sessions inactive >10min.

---

## Implementation Phases

### Phase 1: Core Chat (MVP) — 2-3 weeks
**Goal:** Basic send/receive with the AI chatbot.

| Task | File(s) | Priority |
|---|---|---|
| Types + constants | `chatbot.types.ts`, `chatbot.constants.ts` | P0 |
| API service layer | `chatbot.service.ts` | P0 |
| Chat store (Zustand) | `chatbot.store.ts` | P0 |
| Healthcheck hook | `useChatHealthcheck.ts` | P0 |
| ChatbotProvider | `ChatbotProvider.tsx` | P0 |
| ChatbotAvatar (basic) | `ChatbotAvatar.tsx` | P0 |
| ChatbotContainer (shell) | `ChatbotContainer.tsx` | P0 |
| ChatbotDialog | `ChatbotDialog.tsx` | P0 |
| ChatMessage (text only) | `ChatMessage.tsx` | P0 |
| ChatInput (no suggestions) | `ChatInput.tsx` | P0 |
| ChatTypingIndicator | `ChatTypingIndicator.tsx` | P0 |
| Send message hook | `useChatSend.ts` | P0 |
| Messages query hook | `useChatMessages.ts` | P0 |
| Session hook | `useChatSession.ts` | P0 |
| Mount in MainLayout | `MainLayout.tsx` modification | P0 |

**Deliverable:** User can open chat, send messages, see AI responses.

### Phase 2: Feedback & Escalation — 1-2 weeks
**Goal:** Like/dislike with escalation flow.

| Task | File(s) | Priority |
|---|---|---|
| ChatMessageActions | `ChatMessageActions.tsx` | P1 |
| Feedback mutation hook | `useChatFeedback.ts` | P1 |
| FeedbackState tracking in store | `chatbot.store.ts` update | P1 |
| Related questions | `ChatMessageRelated.tsx` | P1 |
| Instructor picker | `ChatInstructorPicker.tsx` | P1 |
| Smart Library redirect | In `ChatMessageActions.tsx` | P1 |
| QA Feedback API call | In `chatbot.service.ts` | P1 |

**Deliverable:** Full like/dislike → escalation → instructor/smart library flow.

### Phase 3: Rich Content & History — 1-2 weeks
**Goal:** File viewing, images, pagination, and history.

| Task | File(s) | Priority |
|---|---|---|
| ChatMessageContent | `ChatMessageContent.tsx` | P2 |
| ChatFileViewer | `ChatFileViewer.tsx` | P2 |
| Infinite scroll (history) | Update `ChatbotDialog.tsx` | P2 |
| Paginated messages hook | Update `useChatMessages.ts` | P2 |
| Mark as read | In message hooks | P2 |
| Unread badge | Update `ChatbotAvatar.tsx` | P2 |
| Unread count hook | `useChatUnreadCount.ts` | P2 |

**Deliverable:** View old messages, see files/images, unread indicators.

### Phase 4: Suggestions & Polish — 1 week
**Goal:** Autocomplete, notifications, final polish.

| Task | File(s) | Priority |
|---|---|---|
| ChatSuggestions | `ChatSuggestions.tsx` | P3 |
| Suggestions hook | `useChatSuggestions.ts` | P3 |
| Debounced input | Update `ChatInput.tsx` | P3 |
| ChatNotifications tab | `ChatNotifications.tsx` | P3 |
| ChatNotificationCard | `ChatNotificationCard.tsx` | P3 |
| Notifications hook | `useChatNotifications.ts` | P3 |
| Bilingual text utility | `chatbot.utils.ts` | P3 |
| RTL detection | `chatbot.utils.ts` | P3 |
| Error boundaries | Wrap components | P3 |
| Loading/empty states | All components | P3 |

**Deliverable:** Feature-complete chatbot matching legacy functionality.

### Phase 5: Proactive Conversations (Optional) — 1-2 weeks
**Goal:** Avatar bubbles and proactive messages without workers.

| Task | File(s) | Priority |
|---|---|---|
| Avatar bubble component | `ChatbotAvatarBubble.tsx` | P4 |
| Proactive message polling | New hook | P4 |
| Backend endpoint | `e2-chatbot` modification | P4 |
| Conversation greeting | On first open logic | P4 |

**Deliverable:** Avatar shows contextual bubbles for pending items.

---

## File-by-File Implementation Guide

### `src/services/chatbot/chatbot.types.ts`
**What:** All TypeScript interfaces and types.
**Port from:** `e2-student-portal/src/data/models/question-and-answer.js` (schema) + `e2-chatbot/src/constants.js`
**Key decision:** Define strict union types for `actionType`, `source`, `type`.

### `src/services/chatbot/chatbot.constants.ts`
**What:** Enum-like constants for action types, message sources, alert types.
**Port from:** `e2-student-portal/src/constants.js` + `e2-chatbot/src/constants.js`

### `src/services/chatbot/chatbot.service.ts`
**What:** All API calls to the chatbot backend.
**Port from:** `e2-student-portal/src/data/data-store.js` (extract API calls only)
**Key difference:** No offline handling, no PendingUpload queue. Direct axios calls.

### `src/services/chatbot/chatbot.utils.ts`
**What:** Utility functions for text direction, date formatting, locale selection.
**Port from:** `e2-student-portal/src/utils.js` (subset of chatbot-related functions)
**Functions needed:**
- `getLocalizedText(message, locale)` - pick correct language text
- `getTextDirection(text, defaultDir)` - detect RTL/LTR
- `formatMessageDate(date, locale)` - relative time display
- `formatMessageTime(date, locale)` - time only
- `isSmallTalk(source)` - check if message is casual (hide feedback)
- `shouldShowFeedback(message)` - determine if like/dislike shown

### `src/store/chatbot.store.ts`
**What:** Zustand store for client-side UI state only.
**Port from:** Multiple sources:
- `chat-bot-container.js` → tab state
- `chat-dialog-container.js` → session state, feedback state, typing
- `layout.js` → bot open/close state

### `src/hooks/chatbot/useChatMessages.ts`
**What:** React Query infinite query for chat messages.
**Port from:** `data-store.js` → `findQna$()` reactive query + `sync.worker.js` → `syncChats()`
**Key difference:** Paginated API fetch instead of local RxDB subscription.

### `src/hooks/chatbot/useChatSend.ts`
**What:** React Query mutation for sending messages.
**Port from:** `chat-dialog-container.js` → `handleSendMessage()` flow
**Contains:** Optimistic update logic, error handling, feedback state management.

### `src/hooks/chatbot/useChatFeedback.ts`
**What:** Mutation for like/dislike + re-query.
**Port from:** `chat-dialog-container.js` → `handleAcceptFeedback()` / `handleRejectFeedback()`
**Key difference:** No PendingUpload queue. Direct PATCH + new POST.

### `src/hooks/chatbot/useChatSuggestions.ts`
**What:** Query for autocomplete suggestions.
**Port from:** `data-store.js` → `getQuestionSuggestions()`
**Debounce:** 300ms, min 3 chars.

### `src/hooks/chatbot/useChatSession.ts`
**What:** Session lifecycle (start, end, heartbeat).
**Port from:** `chat-dialog-container.js` → `startSession()`, `closeChatAndSession()`, `closeSession()`
**Enhancement:** Add `beforeunload` listener and visibility change detection.

### `src/hooks/chatbot/useChatHealthcheck.ts`
**What:** Poll chatbot availability.
**Port from:** `utils.js` → `checkChatBotEnabled()`
**Behavior:** Every 60s, determines if chatbot components render.

### `src/hooks/chatbot/useChatUnreadCount.ts`
**What:** Query for unread message count (badge).
**Port from:** `chat-bot-avatar.js` → `findQnaCount$` with unread filter
**New requirement:** Backend endpoint for count, OR fetch recent messages and count client-side.

### `src/components/organisms/chatbot/ChatbotProvider.tsx`
**What:** Context provider wrapping the app. Manages session, exposes store.
**Port from:** Combination of `layout.js` logic + worker initialization
**Key difference:** No worker init. Just session management + healthcheck.

### `src/components/organisms/chatbot/ChatbotAvatar.tsx`
**What:** Floating circular button with unread badge.
**Port from:** `chat-bot-avatar.js`
**Key difference:** Uses React Query for unread count instead of RxDB observable.

### `src/components/organisms/chatbot/ChatbotContainer.tsx`
**What:** Popup/drawer with Chat and Notifications tabs.
**Port from:** `chat-bot-container.js`
**Key difference:** Use app's existing modal/drawer component instead of `reactjs-popup`.

### `src/components/organisms/chatbot/ChatbotDialog.tsx`
**What:** Chat conversation panel with messages list + input.
**Port from:** `chat-dialog-container.js` (THE MOST COMPLEX MIGRATION)
**Key differences:**
- No RxDB subscription - uses `useChatMessages()` hook
- Infinite scroll via React Query's `fetchNextPage()`
- No greeting messages from local worker (handle via first-message logic)
- TypeScript throughout

### `src/components/organisms/chatbot/ChatMessage.tsx`
**What:** Individual message bubble.
**Port from:** `chat-message.js`
**Key difference:** Decomposed into sub-components (`ChatMessageContent`, `ChatMessageActions`, `ChatMessageRelated`) for maintainability.

### `src/components/organisms/chatbot/ChatInput.tsx`
**What:** Text input with send button and suggestion integration.
**Port from:** `message-input.js`
**Key difference:** No `useSync()` → always enabled (or disabled only on healthcheck fail).

---

## Feature Parity Checklist

### Must Have (Parity with Legacy)
- [ ] Send message and receive AI response
- [ ] View chat history with pagination (scroll up for older)
- [ ] Like/Dislike feedback buttons on bot messages
- [ ] Dislike escalation: 1→new answer, 2→external resource, 3→instructor
- [ ] Ask Instructor flow (instructor picker + QA feedback creation)
- [ ] Ask Smart Library flow (redirects to smart library app)
- [ ] Related questions chips (clickable to send as new query)
- [ ] Question autocomplete suggestions (debounced, 3+ chars)
- [ ] Typing indicator while waiting for AI
- [ ] Unread message badge on avatar
- [ ] Message timestamps (relative)
- [ ] Bilingual support (EN/AR) with RTL detection per message
- [ ] File/content cards in messages
- [ ] Healthcheck-based enable/disable
- [ ] Session management (open/close)
- [ ] Notifications/Alerts tab
- [ ] Dark/Light theme support

### Nice to Have (Enhanced for Web)
- [ ] Real-time new message notifications (WebSocket/SSE)
- [ ] Proactive avatar bubbles
- [ ] Keyboard shortcuts (Escape to close, Enter to send)
- [ ] Message search within history
- [ ] Copy message text button
- [ ] Responsive design (mobile-friendly)
- [ ] Accessibility (ARIA labels, screen reader support)
- [ ] Toast notifications for errors
- [ ] Retry mechanism for failed sends

### Not Needed (Legacy-Only Features)
- [ ] ~~Offline message queue~~
- [ ] ~~Background sync worker~~
- [ ] ~~Proactive conversation queue (local)~~
- [ ] ~~RxDB local storage~~
- [ ] ~~Learning recommendation reminders (worker-generated)~~
- [ ] ~~Thank you message generation (worker-generated)~~
- [ ] ~~Guided tour (react-shepherd)~~ (use app's own system if exists)

---

## Things to Consider

### 1. Performance
- **Virtualized message list:** For users with 100+ messages, render only visible items
- **Image lazy loading:** Don't load content images until scrolled into view
- **Debounced autocomplete:** Prevent API spam with 300ms debounce
- **Memoization:** `React.memo` on `ChatMessage` to prevent re-renders when other messages update

### 2. Security
- **No API keys in frontend:** All AI service calls go through backend
- **XSS prevention:** Bot messages contain HTML (`dangerouslySetInnerHTML` in legacy) — sanitize with DOMPurify
- **CSRF:** Ensure token-based auth on all mutation endpoints
- **Rate limiting:** Prevent rapid-fire message sends (disable button during pending)

### 3. Error Handling
- **AI timeout:** The AI service has a 5-minute timeout. Show "still thinking" with cancel option after 30s.
- **Network errors:** Show inline error with retry button, not just a toast
- **Session expiry:** If auth token expires mid-chat, prompt re-login without losing drafted message
- **Service unavailable:** If healthcheck fails, hide chatbot gracefully (don't crash app)

### 4. Testing Strategy
- **Unit tests:** Service functions, utility functions, store actions
- **Component tests:** Each component with React Testing Library
- **Integration tests:** Full send/receive flow with MSW (Mock Service Worker)
- **E2E tests:** Playwright/Cypress for critical paths (send message, feedback, escalation)

### 5. Accessibility
- **Focus management:** When chat opens, focus moves to input. On close, returns to avatar.
- **ARIA live regions:** New messages announced to screen readers
- **Keyboard navigation:** Tab through messages, Enter to send, Escape to close
- **Color contrast:** Ensure message bubbles meet WCAG AA
- **Reduced motion:** Respect `prefers-reduced-motion` for typing indicator

### 6. Internationalization
- **Existing i18n setup:** Use app's existing i18n system (likely react-i18next or similar)
- **Message-level locale:** Each message has its own text direction (Arabic bot reply to English question)
- **UI chrome locale:** Chat container UI follows app locale
- **Date formatting:** Use Intl.DateTimeFormat or app's date library with locale

### 7. Monitoring & Analytics
- **Error tracking:** Integrate with app's existing error service (Sentry, etc.)
- **Usage analytics:** Track chat opens, messages sent, escalations, feedback ratios
- **Performance metrics:** Time-to-first-response, message render time

### 8. Bundle Size
- **Lazy load:** Chat components should be code-split (loaded only when avatar clicked)
- **Tree shaking:** Import only needed utilities
- **Avoid large dependencies:** No need for RxDB, Comlink, or heavy libraries

```typescript
// Lazy load example
const ChatbotContainer = React.lazy(() => import('./ChatbotContainer'));

// In ChatbotAvatar click handler:
if (!isOpen) {
  setIsOpen(true); // This triggers React.lazy to load the chat bundle
}
```

### 9. API Compatibility
- **Backend unchanged:** `e2-chatbot` service stays the same. Only the frontend interaction pattern changes.
- **Request format:** Keep same request body shape as legacy to avoid backend changes
- **Response parsing:** Handle both old and new response formats defensively

### 10. Migration Strategy
- **Feature flag:** Ship behind a feature flag to allow A/B testing against Electron chatbot
- **Gradual rollout:** Enable for test users first, then broader audience
- **Fallback:** If critical issues, disable via healthcheck without deployment
