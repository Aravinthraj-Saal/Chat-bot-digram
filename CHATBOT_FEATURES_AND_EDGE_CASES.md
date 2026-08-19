# Chatbot Features & Edge Cases Explained

> For each difficulty/edge case, this file explains:
> 1. **What feature exists** in the legacy app
> 2. **Which file** implements it (click to view)
> 3. **Why** we need a specific approach in the web app
> 4. **What happens if we skip it**

---

## 1. Proactive Conversations Without Workers

### What Feature Is This?

The chatbot **actively pushes messages to the student** without them asking. Examples:
- "Hey Ahmad, you have a new assignment due tomorrow!"
- "Your practice exam is ready - want to start?"
- "Your instructor answered your question!"
- "You haven't completed your AL checkpoint this week."

This is essentially a **notification system embedded in the chat** — the bot talks first.

### Where It Lives in Legacy

| File | What It Does |
|---|---|
| `e2-student-portal/src/workers/chat.worker.js` (lines 22-228) | Runs a loop every 5s, reads QUEUED conversations from RxDB, creates messages |
| `e2-student-portal/src/conversation-config.js` | Defines 13 conversation types with timing and messages |
| `e2-student-portal/src/workers/sync.worker.js` (syncChats) | During download sync, discovers assignments/recommendations → adds to queue |

### How It Works in Legacy

```
sync.worker.js discovers: "Student has a new assignment"
  → Adds to ChatbotConversationQueue (RxDB) with status: QUEUED
    → chat.worker.js picks it up every 5s
      → Creates a bot message: "Hey, you have an assignment!"
      → Shows bubble on avatar: "New task for you!"
      → After 10s timeout: "Take your time, I'll be here."
```

### Why We Need Something in the Web App

Without this, the chatbot becomes **passive only** — it never speaks first. Students would miss:
- Assignment reminders
- Practice recommendations from adaptive learning
- Instructor answers to their questions
- At-risk intervention messages

### Do We Need Polling/Cache?

**YES, but only if you want the avatar bubble + proactive messages.**

- **If you want the badge to show "3 unread"** → You need to poll `GET /chat` periodically to check for new bot-initiated messages
- **If you want the bubble tooltip** ("Hey, you have homework!") → You need to poll or fetch on app load
- **If you skip this entirely** → Chat only responds when the student asks. No badge, no bubble, no proactive nudges.

### Recommendation

**Phase 1:** On chat open, fetch all messages including server-generated proactive ones. The backend (`e2-chatbot`) already stores these in MongoDB. No polling needed yet — just show them when user opens chat.

**Phase 2:** Poll unread count every 60s to show the badge number on the avatar. This is just:
```
GET /e2-sp-service/chat?userId=X&courseId=Y&readAt=null → count the results
```

---

## 2. Avatar Bubble Messages Without Workers

### What Feature Is This?

The floating chatbot avatar shows a **speech bubble tooltip** above it with personalized greetings:
- "Hi Ahmad! How can I help you today?"
- "You have a pending practice exam!"
- "Great job on your assignment!"

The bubble appears for 7 seconds then disappears.

### Where It Lives in Legacy

| File | What It Does |
|---|---|
| `e2-student-portal/src/pages/chat-bot/chat-bot-avatar/chat-bot-avatar.js` (lines 44-85) | Subscribes to `getLatestBotBubbleMsg$()`, shows bubble for 7s |
| `e2-student-portal/src/workers/chat.worker.js` (lines 38-58, 102-116) | `initializeChatGreetingQueue()` picks random greeting; `updateChatbubbleMsg()` sets bubbles for conversation events |
| `e2-student-portal/src/data/models/chat-bot-bubble-msg.js` | Model for bubble messages stored in RxDB |

### How It Works in Legacy

```
chat.worker.js starts → picks random greeting → saves to RxDB 'chatbotbubblemessage'
  → chat-bot-avatar.js subscribes to getLatestBotBubbleMsg$()
    → When new bubble message appears in RxDB:
      → Shows tooltip above avatar for 7 seconds
      → After 7s: hides it (setTimeout)
```

### Why We Need Something in the Web App

This is a **user engagement feature** — it draws attention to the chatbot. Without it, the avatar just sits there silently and students may forget it exists.

### Do We Need Polling for This?

**NO, not necessarily.** The simplest approach:

- On app load: pick a random greeting and show it once above the avatar (client-side only, no API needed)
- For conversation-triggered bubbles: only show these if you implement proactive conversations (Phase 2+)

### What Happens If We Skip It?

The avatar shows a badge number (unread count) but no speech bubble. Users still know there are unread messages. This is **nice-to-have**, not critical.

---

## 3. Chat History Pagination

### What Feature Is This?

When the user opens the chat, they see **their conversation history** — old messages they sent, bot replies, etc. The user can **scroll up** to see older messages.

### Where It Lives in Legacy

| File | What It Does |
|---|---|
| `e2-student-portal/src/workers/sync.worker.js` (syncChats) | Downloads ALL messages (up to 500) from server into local RxDB |
| `e2-student-portal/src/pages/chat-bot/chat-dialog-container/chat-dialog-container.js` (handleScroll) | Shows last 10 messages, loads 10 more on scroll-up from LOCAL RxDB |

### How It Works in Legacy

```
sync.worker.js runs periodically:
  → GET /e2-chatbot/chat?userId=X&courseId=Y (fetches ALL history)
  → Stores everything in local RxDB
  
User opens chat:
  → Reads last 10 messages from LOCAL RxDB (instant, no network)
  → User scrolls up → loads next 10 from LOCAL RxDB (still instant)
```

### Why the Web App Is Different

In the web app there is **no local RxDB**. Every read must come from the server:
```
User opens chat:
  → GET /e2-sp-service/chat?userId=X&courseId=Y&skip=0&limit=20 (network call!)
  → User scrolls up → GET ...&skip=20&limit=20 (another network call!)
```

### Do We Need Caching Here?

**YES** — but simple caching, not a full library:
- Once you fetch page 1 (latest 20 messages), keep it in component state
- When user scrolls up, fetch page 2 and **prepend** to the existing array
- Don't re-fetch page 1 every time (that's the "cache")

This is just `useState` with an array that grows. React Query's `useInfiniteQuery` automates this pattern, but you can do it manually too.

### What Happens If We Skip Pagination?

If you fetch ALL messages at once (like legacy sync does), it works but:
- First load is slow (potentially 500 messages to download)
- Wastes bandwidth for users who only read the last few messages

---

## 4. Optimistic Updates for Feedback

### What Feature Is This?

When the student clicks **👍 Like** or **👎 Dislike**, the button should **highlight immediately** without waiting for the server response.

### Where It Lives in Legacy

| File | What It Does |
|---|---|
| `e2-student-portal/src/pages/chat-bot/chat-dialog-container/chat-dialog-container.js` (handleAcceptFeedback, line 277) | Updates RxDB instantly (`updateQnaFeedback`), THEN queues server update |
| `e2-student-portal/src/data/data-store.js` (updateQnaFeedback) | Patches RxDB locally + creates PendingUpload for server sync |

### How It Works in Legacy

```
User clicks 👍:
  1. Instantly update RxDB (feedbackGiven = 'like') → UI refreshes immediately
  2. Create PendingUpload → sync worker will PATCH server later
  
User sees: button highlighted instantly (no loading spinner)
```

### Why the Web App Is Different

No local DB means:
```
User clicks 👍:
  1. Send PATCH /chat/:id to server → WAIT for response (200-500ms)
  2. THEN update UI
  
User sees: button does nothing for half a second → feels laggy
```

### Do We Need React Query / Special Library for This?

**NO.** You can do optimistic updates with plain `useState`:

```typescript
// Simple approach:
const [feedbackGiven, setFeedbackGiven] = useState(message.feedbackGiven);

const handleLike = async () => {
  setFeedbackGiven('like'); // Instant UI update
  try {
    await chatbotService.updateFeedback(id, { feedbackGiven: 'like' });
  } catch {
    setFeedbackGiven(null); // Rollback on error
  }
};
```

React Query's `useMutation` with `onMutate` does the same thing but with more structure. Either works.

### What Happens If We Skip Optimistic Updates?

The Like/Dislike buttons feel slow (300-500ms delay). Not terrible, but noticeable. The button could also show a tiny spinner during the API call.

---

## 5. Session Management

### What Feature Is This?

A **chat session** starts when the user opens the chat and ends when they close it. The backend uses sessions to:
- Group messages into conversation threads
- Know when a user is "done" (closure message)
- Log engagement duration

### Where It Lives in Legacy

| File | What It Does |
|---|---|
| `e2-student-portal/src/pages/chat-bot/chat-dialog-container/chat-dialog-container.js` (lines 121-148) | `closeChatAndSession()` → marks messages as read, sends closure to backend |
| `e2-student-portal/src/data/data-store.js` (sendChatMessage with typeOfChat:'closure') | Sends POST /chat with typeOfChat:'closure' |

### How It Works in Legacy

```
User opens chat:
  → Generate sessionId (shortid)
  → All messages get this sessionId
  
User closes chat (clicks X button):
  → closeChatAndSession()
    → markQnaAsRead() (update all unread)
    → closeSession() → POST /chat { typeOfChat: 'closure' }
    → Backend responds with "Let me know if you need anything!"
```

### Why the Web App Has Edge Cases

In Electron, the app controls the window lifecycle completely. In a browser:

| Scenario | Electron (Legacy) | Web (New) |
|---|---|---|
| User clicks X button | ✅ `closeChatAndSession()` runs | ✅ Same — runs cleanup |
| User closes browser tab | ✅ Electron's `before-quit` event | ⚠️ `beforeunload` is unreliable |
| User's laptop dies | ❌ No cleanup | ❌ No cleanup |
| Multiple tabs open | ❌ Not possible (Electron is single instance) | ⚠️ Multiple sessions? |
| Token expires | ❌ Token doesn't expire in offline app | ⚠️ API call fails silently |

### What We Need

1. **`beforeunload` event** — try to send closure (not guaranteed to fire)
2. **Server-side timeout** — if no messages for 10 minutes, server auto-closes session (this is a backend enhancement)
3. **Single session enforcement** — generate sessionId per tab, backend handles multiple

### What Happens If We Skip It?

Sessions never properly close. The backend has stale "open" sessions. Impact is minor — it only affects analytics/logging, not functionality.

---

## 6. Real-time Message Updates

### What Feature Is This?

After you send a message, the **bot reply appears instantly** in the chat UI without refreshing.

### Where It Lives in Legacy

| File | What It Does |
|---|---|
| `e2-student-portal/src/pages/chat-bot/chat-dialog-container/chat-dialog-container.js` (handleSaveQnaResponse) | Saves bot message to RxDB → `useObservable` subscription triggers re-render |
| `e2-student-portal/src/hooks/use-observable.js` | Subscribes to RxDB reactive queries → component re-renders on data change |

### How It Works in Legacy

```
RxDB is reactive (like a live database):
  → Component subscribes: "show me all messages for this course"
  → When handleSaveQnaResponse() inserts a new message into RxDB
    → RxDB automatically notifies all subscribers
    → Component re-renders with new message visible
    
No manual state management needed — the database IS the state.
```

### Why the Web App Is Different

No RxDB means you must manually manage the message list state:
```typescript
const [messages, setMessages] = useState<ChatMessage[]>([]);

// After sending and receiving response:
setMessages(prev => [...prev, userMessage, botMessage]);
```

### Do We Need a Special Library?

**NO.** This is just standard React state management. When `sendMessage()` returns:
1. Add user message to state (immediately)
2. Add bot message to state (after API response)

The UI re-renders automatically because state changed. This is how React already works.

### What Happens If We Skip It?

You can't skip this — it's the core chat functionality. But you don't need any special library. Plain `useState` or Context works.

---

## 7. File Viewer Integration

### What Feature Is This?

Bot messages sometimes include **files** (PDFs, documents, images) from the course content. Users can click to view them inline.

### Where It Lives in Legacy

| File | What It Does |
|---|---|
| `e2-student-portal/src/pages/chat-bot/chat-dialog-container/chat-message.js` (lines 300+) | Renders file cards with title, extension icon; opens `FileViewer` component |
| `e2-student-portal/src/utils.js` (openFileExternally) | Opens file with system default application (Electron API) |

### How It Works in Legacy

```
Bot message contains: message.content.file = { path, title, extension, id }
  → ChatMessage renders a file card with icon + title
  → User clicks → opens Fullscreen FileViewer component (PDF.js based)
  → OR clicks "external" button → Electron opens system viewer (e.g., Adobe Acrobat)
```

### Why the Web App Is Different

- No Electron shell → can't call `openFileExternally()`
- Files are served from `/e2-content` service → accessible via URL
- The web app **already has a FileViewer component** at `src/components/molecules/file-viewer/FileViewer.tsx`

### What We Need

Reuse the existing `FileViewer` component. For external viewing, open in a new browser tab:
```typescript
window.open(`${contentServiceUrl}/${file.path}`, '_blank');
```

### What Happens If We Skip It?

Bot answers reference course content but users can't view it. This significantly reduces the chatbot's value since RAG answers often link to source materials.

---

## 8. Network Error Handling

### What Feature Is This?

Gracefully handling network failures, server errors, and AI service timeouts.

### Where It Lives in Legacy

| File | What It Does |
|---|---|
| `e2-student-portal/src/data/data-store.js` (sendChatMessage) | Catches errors, creates PendingUpload if offline |
| `e2-student-portal/src/pages/chat-bot/chat-dialog-container/message-input/message-input.js` (line 30) | Uses `useSync()` → disables input when offline |
| `e2-student-portal/src/workers/sync.worker.js` (ensureConnection) | Pings `/e2-irp/healthcheck` to check connectivity |

### How It Works in Legacy

```
User sends message while offline:
  → sendChatMessage catches error
  → Creates PendingUpload (queues for later)
  → Shows error message in input area
  → Input is disabled (useSync.isOnline = false)
  
When back online:
  → Sync worker detects connectivity
  → Processes PendingUpload queue
  → Messages get sent to server
```

### Why the Web App Is Different

- No offline queue → failed messages are just **lost** unless handled
- AI service timeout is **5 MINUTES** (set in `register-services.js` line 14)
- No `useSync` hook → must detect errors from API responses

### What We Need

```typescript
const handleSend = async (query: string) => {
  try {
    setTyping(true);
    const response = await chatbotService.sendMessage({...});
    addBotMessage(response);
  } catch (error) {
    if (error.code === 'ECONNABORTED') {
      // Timeout after 5 min
      showError('The AI is taking too long. Please try again.');
    } else if (error.response?.status === 503) {
      // Chatbot disabled
      showError('Chatbot is temporarily unavailable.');
    } else {
      // Network error
      showError('Could not send message. Check your connection.');
      // Keep user's message in input so they can retry
    }
  } finally {
    setTyping(false);
  }
};
```

### What Happens If We Skip It?

Unhandled promise rejection → user sees nothing, message disappears, no feedback. Very bad UX. **This is critical to implement.**

---

## 9. Concurrent Like/Dislike Race Conditions

### What Feature Is This?

Preventing bugs when a user **rapidly clicks Like/Dislike** on the same or different messages.

### Where It Lives in Legacy

| File | What It Does |
|---|---|
| `e2-student-portal/src/data/data-store.js` (updateQnaFeedback) | Writes to RxDB (atomic local write — no race condition possible) |

### How It Works in Legacy

```
User clicks 👎 on message A:
  → Instantly writes to RxDB (atomic, synchronous-like)
  → No race condition because it's a local database write
  → PendingUpload queues it for server (async, but already committed locally)
```

### Why the Web App Has This Problem

```
User clicks 👎 on message A → API call starts (200ms pending...)
User clicks 👍 on message A → Another API call starts (200ms pending...)
  → Both PATCH requests hit the server → last one wins
  → UI shows inconsistent state during the overlap
```

### What We Need

Simple: **disable the buttons while a feedback API call is in progress** for that specific message.

```typescript
const [feedbackLoading, setFeedbackLoading] = useState(false);

<Button 
  disabled={feedbackLoading}
  onClick={() => handleFeedback('like')}
/>
```

### What Happens If We Skip It?

Edge case only — most users won't click fast enough. But if they do, the feedback value could be wrong. Low priority fix but easy to implement.

---

## 10. Large Message History Performance

### What Feature Is This?

Handling users with **hundreds of messages** without the browser becoming slow.

### Where It Lives in Legacy

| File | What It Does |
|---|---|
| `e2-student-portal/src/pages/chat-bot/chat-dialog-container/chat-dialog-container.js` (MESSAGES_PER_PAGE=10) | Only renders 10 messages at a time, loads more on scroll |
| RxDB (IndexedDB) | Stores 500 messages locally but only queries what's needed |

### How It Works in Legacy

```
RxDB stores 500 messages (indexed by date)
  → UI queries: "give me the latest 10" → renders only 10 DOM nodes
  → Scroll up → query: "give me the next 10" → now 20 DOM nodes
  → Maximum in DOM at any time: however many the user scrolled through
```

### Why the Web App Has This Problem

If you fetch 20 messages per page and the user scrolls through 500 messages, you eventually have 500 message components in the DOM. Each message can contain images, files, buttons — heavy DOM.

### Do We Need a Virtualized List Library?

**PROBABLY NOT for MVP.** Here's why:

- Most users have 50-100 messages total
- Legacy shows 10 per page — the UI never shows all 500 at once
- If you paginate (20 per page) and the user scrolls through 5 pages = 100 DOM nodes. This is fine.
- React can handle 200-300 simple divs without issues

**Only consider `react-virtuoso`** if you discover real performance issues (>300ms render time).

### What Happens If We Skip It?

Nothing bad for 95% of users. The 5% with 500+ messages might see slight jank on scroll. Acceptable for MVP.

---

## 11. Bilingual Content Rendering (EN/AR + RTL)

### What Feature Is This?

Every bot message has text in **both English and Arabic**. The UI must:
- Show the correct language based on user's locale setting
- Detect **text direction** per message (Arabic = RTL, English = LTR)
- Handle mixed content (Arabic message with English course name inside)

### Where It Lives in Legacy

| File | What It Does |
|---|---|
| `e2-student-portal/src/utils.js` (isArabic, getChatMessageTextDirection) | Detects if text contains Arabic characters → sets `dir="rtl"` |
| `e2-student-portal/src/pages/chat-bot/chat-dialog-container/chat-message.js` | Uses `getChatMessageTextDirection()` per message |
| `e2-student-portal/src/hooks/use-locale.js` | Provides `i(key, data, locale)` translation function |
| `e2-student-portal/src/locales/en.json` + `ar.json` | Message templates in both languages |

### How It Works in Legacy

```
Bot message from server:
  message: {
    text: "Your assignment is due tomorrow",           // English (default)
    locales: [{ text: "موعد تسليم واجبك غدًا", locale: "ar" }]  // Arabic
  }

UI logic:
  1. Get current locale from LocaleContext
  2. If locale === 'ar': use locales[0].text (Arabic)
  3. Else: use message.text (English)
  4. Detect direction: isArabic(displayedText) ? 'rtl' : 'ltr'
  5. Set <div dir="rtl"> on the message bubble
```

### Why We Need This in the Web App

The web app **already has locale support** (`src/contexts/locale-context/`). This is not optional — SAAL's platform serves Arabic-speaking students.

### What We Need

One utility function + CSS:
```typescript
function getLocalizedText(message: MessageContent, locale: string): string {
  if (locale === 'ar') {
    const arabic = message.locales?.find(l => l.locale === 'ar');
    return arabic?.text || message.text;
  }
  return message.text;
}

function getTextDirection(text: string): 'rtl' | 'ltr' {
  const arabicRegex = /[\u0600-\u06FF]/;
  return arabicRegex.test(text) ? 'rtl' : 'ltr';
}
```

### What Happens If We Skip It?

Arabic-speaking users see English messages (or broken Arabic without RTL). **This is a must-have — SAAL is UAE-based, Arabic is primary.**

---

## 12. Smart Library Deep Link

### What Feature Is This?

When the bot says "I found relevant content in the Smart Library", the user can click a link that opens the **Smart Library viewer app** with the specific document.

### Where It Lives in Legacy

| File | What It Does |
|---|---|
| `e2-student-portal/src/pages/chat-bot/chat-dialog-container/chat-message.js` (SMART_LIBRARY source) | Renders a "View in Smart Library" link |
| `e2-student-portal/src/utils.js` (constructUrlWithQueryParams, openUrlExternally) | Builds URL with params and opens in Electron's external browser |
| `e2-chatbot/src/services/chat/chat.service.js` (askFromSmartLibrary, line 391) | Calls `POST /library/search` on AI service |

### How It Works in Legacy

```
User clicks "Ask Smart Library" after 2 dislikes:
  → Frontend sends: POST /chat { typeOfChat: 'ask-smart-library', query: "..." }
  → e2-chatbot calls: POST /library/search on AI service
  → Response contains: { contentId, title, pageNumber }
  → UI renders a link card
  → Click → opens: https://smart-library.saal.ai?userId=X&courseId=Y&contentId=Z&locale=en
    → Electron: openUrlExternally(url)
```

### Why the Web App Is Different

- No `openUrlExternally()` Electron API
- Solution: `window.open(url, '_blank')` — opens in new tab
- May need auth token in URL params or cookie for the Smart Library app to authenticate

### What We Need

```typescript
const openSmartLibrary = (contentId: string) => {
  const params = new URLSearchParams({
    userId: user.id,
    courseId: course.id,
    contentId,
    locale: currentLocale,
  });
  window.open(`${SMART_LIBRARY_URL}?${params.toString()}`, '_blank');
};
```

### What Happens If We Skip It?

The "Ask Smart Library" button doesn't work. Users lose access to one of the three escalation options (Smart Library, Instructor, or new answer). **Implement it — it's trivial in a browser.**

---

## Summary: Priority Matrix

| # | Edge Case | Priority | Effort | Skip Impact |
|---|---|---|---|---|
| 1 | Proactive Conversations | P2 (Phase 2) | Medium | No push notifications in chat |
| 2 | Avatar Bubble | P3 (Phase 4) | Low | Avatar is silent (badge still works) |
| 3 | Chat History Pagination | **P0 (Phase 1)** | Low | Must have — can't show history otherwise |
| 4 | Optimistic Feedback | P1 (Phase 2) | Low | Like/dislike feels slightly slow |
| 5 | Session Management | P1 (Phase 1) | Low | Stale sessions in backend (minor) |
| 6 | Real-time Updates | **P0 (Phase 1)** | None | Must have — it's just React state |
| 7 | File Viewer | P2 (Phase 3) | Low | Bot can't show course content |
| 8 | Network Error Handling | **P0 (Phase 1)** | Low | Broken UX on any failure |
| 9 | Concurrent Feedback | P3 (Phase 4) | Trivial | Rare edge case |
| 10 | Large History Perf | P4 (optional) | Medium | Only affects power users |
| 11 | Bilingual RTL | **P0 (Phase 1)** | Low | Arabic users get broken text |
| 12 | Smart Library Link | P2 (Phase 2) | Trivial | One escalation path broken |

---

## Do You Need React Query / Zustand / WebSocket?

| Library | Verdict | Reason |
|---|---|---|
| **React Query** | ❌ Not needed for MVP | Your `StoreContext` + `useState` covers chat pagination, send, feedback. React Query helps if you later add complex caching or polling across many features. |
| **Zustand** | ❌ Not needed | React Context (`ChatbotContext`) handles: isOpen, activeTab, isTyping, sessionId, feedbackState. These don't change fast enough to cause re-render issues. |
| **WebSocket** | ❌ Not needed | The chatbot is student↔bot, not real-time multi-user. Polling every 30-60s for unread count is enough. Bot only responds after student sends a message. |
| **Virtualized List** | ❌ Not needed for MVP | Pagination (20 per page) keeps DOM small. Only add if performance issues appear with power users. |
| **Polling (setInterval)** | ✅ Needed (small) | Only for: unread badge count (60s) + healthcheck (60s). Two simple intervals. |

### The Only Thing You Actually Need to Add

**Nothing new in `package.json`.** Everything can be built with:
- `axios` (already installed) — for API calls
- `React Context` (built-in) — for chatbot UI state
- `useState` + `useEffect` (built-in) — for message list, pagination, polling
- `Ant Design` (already installed) — for Drawer, Tabs, Badge, Button UI
- `@phosphor-icons/react` (already installed) — for Send, ThumbsUp, ThumbsDown icons
- `date-fns` or `dayjs` (already installed) — for message timestamps
- Existing `FileViewer` component — for embedded file viewing
- Existing `LocaleContext` — for bilingual support
- Existing `ThemeProvider` — for dark/light mode
- Existing `api-helper.ts` — for authenticated Axios with token refresh
