# Chatbot Architecture - Legacy Student Portal (e2-student-portal)

## Table of Contents

1. [System Overview](#system-overview)
2. [Component File Map](#component-file-map)
3. [Data Flow & Workflow](#data-flow--workflow)
4. [File-by-File Breakdown - Frontend](#file-by-file-breakdown---frontend)
5. [File-by-File Breakdown - Backend (e2-chatbot)](#file-by-file-breakdown---backend-e2-chatbot)
6. [File-by-File Breakdown - Backend (e2-sp-service)](#file-by-file-breakdown---backend-e2-sp-service)
7. [API Endpoints](#api-endpoints)
8. [Offline Architecture](#offline-architecture)
9. [Conversation Flow System](#conversation-flow-system)
10. [Feedback & Escalation Logic](#feedback--escalation-logic)
11. [Reusable Functions & Hooks](#reusable-functions--hooks)
12. [Data Models](#data-models)
13. [Constants & Configuration](#constants--configuration)

---

## System Overview

The chatbot ("iStudy" / "Khaled") is an AI-powered conversational assistant built into an **offline-first Electron application**. It consists of:

- **Frontend UI** (React, SCSS, RxDB for local storage)
- **Background Worker** (`chat.worker.js` - manages proactive conversations)
- **Sync Worker** (`sync.worker.js` - syncs data between local DB and server)
- **Gateway Service** (`e2-sp-service` at port 5017/8000 - proxies to backend)
- **Chatbot Service** (`e2-chatbot` at port 5020 - the AI orchestrator)
- **AI Service** (`smart-learning-companion-aai-service` - actual LLM/RAG engine)

### Architecture Diagram

```
┌────────────────────────────────────────────────────────────────────────┐
│ Electron App (e2-student-portal)                                        │
│                                                                          │
│  ┌────────────────┐   ┌──────────────┐   ┌─────────────────────────┐  │
│  │ React UI       │──▶│ DataStore    │──▶│ RxDB (IndexedDB)        │  │
│  │ (Components)   │   │ (data-store) │   │ - questionandanswer     │  │
│  └────────────────┘   └──────┬───────┘   │ - chatbotbubblemessage  │  │
│                               │           │ - chatbotConversationQ  │  │
│         ┌─────────────────────┤           │ - pendingupload        │  │
│         │                     │           └─────────────────────────┘  │
│         ▼                     ▼                                         │
│  ┌──────────────┐   ┌──────────────────┐                               │
│  │ Chat Worker  │   │ Sync Worker      │                               │
│  │ (5s loop)    │   │ (interval sync)  │                               │
│  │ - queue mgmt │   │ - upload pending │                               │
│  │ - greetings  │   │ - download data  │                               │
│  │ - reminders  │   │ - push feedback  │                               │
│  └──────────────┘   └────────┬─────────┘                               │
└───────────────────────────────┼─────────────────────────────────────────┘
                                │ HTTP (REACT_APP_API_URL = localhost:8000)
                                ▼
┌───────────────────────────────────────────────────────────────────────┐
│ e2-sp-service (port 5017, exposed at port 8000 in Electron)            │
│ - Proxies /chat to CHATBOT_SERVICE                                     │
│ - Proxies /ai-question-suggestion to CHATBOT_SERVICE                   │
│ - Handles other service routing (courses, exams, etc.)                 │
└───────────────────────────────┬───────────────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────────────┐
│ e2-chatbot (port 5020)                                                 │
│ - MongoDB for chat persistence                                         │
│ - Like/dislike escalation logic                                        │
│ - Routes to AI service                                                 │
│ - Chat logging                                                         │
└───────────────────────────────┬───────────────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────────────┐
│ smart-learning-companion-aai-service (SAAL's RAG/LLM)                  │
│ - POST /chat (or /{workspace}/chat for DATA_PRISM provider)            │
│ - GET /chat/complete (question autocomplete)                           │
│ - POST /library/search (smart library)                                 │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Component File Map

```
src/
├── pages/chat-bot/
│   ├── chat-bot-container/
│   │   ├── chat-bot-container.js          # Main popup - tabs (Chat + Notifications)
│   │   └── chat-bot-container.scss
│   ├── chat-bot-avatar/
│   │   ├── chat-bot-avatar.js             # Floating avatar button with badge
│   │   ├── chat-bot-avatar.scss
│   │   ├── avatar-greetings.js            # Bubble message above avatar
│   │   └── avatar-alerts.js               # Alert bubbles above avatar
│   ├── chat-dialog-container/
│   │   ├── chat-dialog-container.js       # Chat conversation UI (MAIN LOGIC)
│   │   ├── chat-dialog-container.scss
│   │   ├── chat-message.js                # Individual message bubble
│   │   ├── message-input/
│   │   │   ├── message-input.js           # Text input with suggestions
│   │   │   ├── message-input.scss
│   │   │   └── instructor-popup.js        # Instructor picker modal
│   │   └── message-timestamp/
│   │       └── message-timestamp.js       # Relative time display
│   ├── chat-bot-alerts/
│   │   ├── chat-bot-alerts.js             # Notifications tab container
│   │   ├── chat-bot-alerts.scss
│   │   ├── assignment-submission.js       # Assignment alert card
│   │   ├── recommendation.js             # Recommendation alert card
│   │   ├── instructor-answer.js          # QA feedback alert card
│   │   ├── game-alert.js                 # Game recommendation card
│   │   ├── ai-intervention.js            # AI intervention alert card
│   │   └── alert-date.js                 # Date grouping header
│   └── qa-feedback-card/
│       ├── qa-feedback-card.js            # QA feedback display card
│       └── qa-feedback-card.scss
├── components/layout/
│   └── layout.js                          # Mounts ChatBotAvatar + ChatBotContainer
├── data/
│   ├── data-store.js                      # All DB operations + API calls
│   ├── api.js                             # Axios instance (baseURL: localhost:8000)
│   └── models/
│       ├── question-and-answer.js         # Chat message model + RxDB schema
│       ├── chat-bot-bubble-msg.js         # Avatar bubble message model
│       └── chatbot-conversation-queue.js  # Conversation queue model
├── workers/
│   ├── chat.worker.js                     # Background conversation service
│   └── sync.worker.js                     # Data sync (upload + download)
├── worker-handler.js                      # Initializes all workers
├── conversation-config.js                 # Conversation type definitions + timing
├── constants.js                           # ActionTypes, QuestionAndAnswerType, etc.
├── utils.js                               # checkChatBotEnabled, date utils, etc.
├── hooks/
│   ├── use-store.js                       # Access DataStore methods
│   ├── use-observable.js                  # RxDB reactive queries
│   ├── use-locale.js                      # i18n translation
│   ├── use-theme.js                       # Dark/light mode
│   ├── use-sync.js                        # Online/offline status
│   ├── use-promise.js                     # Async data fetching
│   ├── use-promise-interval.js            # Polling data fetcher
│   └── use-debounce.js                    # Input debouncing
├── locales/
│   ├── en.json                            # English translations
│   └── ar.json                            # Arabic translations
└── setupProxy.js                          # Dev proxy → localhost:8000
```

---

## Data Flow & Workflow

### Flow 1: User Sends a Message (Online)

```
1. User types in MessageInput
2. MessageInput.onClickSendButton()
   └── closeRunningConversationFlow()  // stop any active proactive conversation
   └── ChatDialogContainer.handleSendMessage(query)
       ├── handleSaveQnaRequest(query)
       │   └── createQuestionAndAnswer({type: 'User', query, ...})
       │       └── RxDB insert → UI updates reactively
       ├── sendChatMessage({data, typeOfChat: 'chat', query, feedbackState})
       │   └── POST /e2-chatbot/chat → AI response returned
       ├── handleSaveQnaResponse(response)
       │   └── createQuestionAndAnswer({type: 'Bot', message, ...})
       │       └── RxDB insert → UI updates reactively
       └── sendChatMessage({data}) // notify backend of saved response (no typeOfChat)
```

### Flow 2: Like/Dislike Feedback

```
1. User clicks 👍 on bot message
2. ChatMessage → handleAcceptFeedback(questionAndAnswer)
   ├── updateQnaFeedback({feedbackGiven: 'like'})
   │   ├── RxDB patch (local update)
   │   └── createPendingUpload({entity: 'update-qna-feedback'})
   │       └── Sync worker later: PATCH /e2-chatbot/chat/{id}
   ├── handleSaveQnaRequest("I liked the answer")  // local save
   ├── sendChatMessage({feedbackState: {feedbacks: [{value:'like', ...}]}})
   │   └── POST /e2-chatbot/chat → acknowledgment response
   └── handleSaveQnaResponse(response) // save bot reply

3. User clicks 👎 on bot message
   ├── Same as above but feedbackGiven: 'dislike'
   └── Bot returns alternative answer OR escalation action:
       ├── 1 dislike: new answer + relatedQuestions from AI
       ├── 2 dislikes: ActionTypes.ASK_EXTERNAL_RESOURCE
       └── 3 dislikes: ActionTypes.ASK_INSTRUCTOR
```

### Flow 3: Ask Instructor Escalation

```
1. Bot message has actionType === 'SHOW_INSTRUCTORS'
2. ChatMessage renders instructor list
3. User clicks instructor → handleSendToInstructor(instructor, feedbackState)
   ├── handleSaveQnaRequest(instructor.name)
   ├── sendChatMessage({typeOfChat: 'chat', query, feedbackState})
   ├── createQaFeedback({userId, instructorId, question, botAnswers})
   │   └── POST /e2-course/qa-feedback/
   ├── handleSaveQnaResponse("Query sent to instructor")
   └── handleSaveQnaResponse("Is there anything else?")
```

### Flow 4: Ask Smart Library

```
1. Bot message has actionType === 'ASK_EXTERNAL_RESOURCE'
2. User clicks "Ask Smart Library"
3. handleAskFromSmartLibrary(questionAndAnswer)
   ├── handleSaveQnaRequest('ask-smart-library', hideFromChat=true)
   ├── sendChatMessage({typeOfChat: 'ask-smart-library', query})
   │   └── e2-chatbot calls POST /library/search on AI service
   └── handleSaveQnaResponse(response) // shows smart library content link
```

### Flow 5: Proactive Conversation (Background Worker)

```
1. chat.worker.js runs every 5 seconds
2. ChatService.run()
   ├── getConversationQueue() from RxDB
   ├── Find QUEUED conversations, group by first type
   ├── Mark group as ACTIVE + assign groupId
   ├── updateChatbubbleMsg() → shows bubble on avatar
   ├── addConversationToChatBot() → saves message to RxDB chat
   │   └── User sees message when they open chat
   └── addClosureMsgAfterTimeout({closureDelay: 10000})
       └── After 10s: mark as DONE, add closure message
```

### Flow 6: Sync Download (Chat History)

```
1. SyncService.run() → syncDownloads() → syncChats()
2. GET /e2-chatbot/chat?userId=X&courseId=Y&sortByDate=true (paginated)
   └── Fetches up to 500 messages from server
3. Merges with local RxDB data
4. Generates additional local-only messages from:
   ├── Active assignments → assignment chat messages
   ├── Lesson recommendations → recommendation messages
   ├── AI recommendations → AI content messages
   └── QA feedbacks → feedback notification messages
5. bulkUpsertQna() → stores all in RxDB
```

### Flow 7: Sync Upload (Pending Changes)

```
1. SyncService.syncPendingUploads()
2. Reads all PendingUpload records from RxDB
3. Groups by entity type:
   ├── 'chatbot-chat' / 'create-qna-entry' → POST /e2-chatbot/chat?bulk=true
   ├── 'update-qna-feedback' → PATCH /e2-chatbot/chat/{id}
   └── 'bulk-patch-chat' (readAt updates) → POST /e2-chatbot/chat-bulk-patch
4. Deletes PendingUpload records on success
```

---

## File-by-File Breakdown - Frontend

### `src/components/layout/layout.js`
**Purpose:** Global layout wrapper that mounts the chatbot globally.

**Key behaviors:**
- Polls `checkChatBotEnabled()` every 60s to check if chatbot is toggled on/off server-side
- Manages `botState = {isBotOpen, isChatTabOpen, data}` via `useState`
- Hides avatar during exams (`THEORETICAL_EXAMS`, `PRACTICAL_EXAMS`, `PRACTICAL_LESSONS`)
- Disables chatbot when `viewMode !== StudentMode`
- Renders `<ChatBotAvatar>` (always when enabled) and `<ChatBotContainer>` (when open)

---

### `src/pages/chat-bot/chat-bot-avatar/chat-bot-avatar.js`
**Purpose:** Floating circular avatar button (entry point to chatbot).

**Key behaviors:**
- Shows unread message count badge via `useObservable` → `findQnaCount$` (filters: type=Bot, no readAt, not greeting, not closure)
- Subscribes to `getLatestBotBubbleMsg$()` for bubble greeting messages
- Displays bubble message for 7 seconds then auto-hides
- Marks all unread messages as read when clicked via `markQnaAsRead()`
- Uses `React.memo` for performance

**Dependencies:** `useObservable`, `useStore`, `useLocale`, `useTheme`

---

### `src/pages/chat-bot/chat-bot-avatar/avatar-greetings.js`
**Purpose:** Speech bubble tooltip that appears above the avatar.

- Pure presentational component
- Renders a `div.chat-bot-avatar__bubble` with the message text
- Supports dark/light theme styling

---

### `src/pages/chat-bot/chat-bot-avatar/avatar-alerts.js`
**Purpose:** Alert notification bubbles above the avatar.

- Subscribes to `findAlerts$({})` for unread alerts
- Filters for assignment-type alerts only
- Shows random alert text ("Writer's block?" or "Due soon!")
- Clicking opens chatbot to notifications tab with the alert data
- CSS bubble animation (4s cubic-bezier)

---

### `src/pages/chat-bot/chat-bot-container/chat-bot-container.js`
**Purpose:** Main chatbot popup dialog with tab navigation.

**Key behaviors:**
- Renders as a `<Popup>` modal (reactjs-popup library)
- Two tabs: Chat (`ChatDialogContainer`) and Notifications (`ChatBotAlerts`)
- Tab state via `useState({chat: true/false, notifications: true/false})`
- Shows unread alerts count on Notifications tab badge
- Integrates with `react-shepherd` guided tour system
- Saves tour completion via `createUserData` → `SP_TOUR_DATA`
- RTL support via `textDirection` based on locale
- Uses `useIntervalPromise` to poll `alertCount()` for badge

---

### `src/pages/chat-bot/chat-dialog-container/chat-dialog-container.js`
**Purpose:** The core chat conversation logic and UI. **MOST COMPLEX FILE (828 lines).**

**Key state:**
- `loading` - typing indicator timestamp (false when idle, Date when loading)
- `isSessionOpen` - whether a conversation session is active
- `feedbackState` - tracks current question's feedback history for escalation
- `displayedMessageCount` - for pagination/infinite scroll
- `shouldMaintainScrollPosition` - prevents scroll jump on loading older messages
- `scrollBehavior` - smooth scroll enabled after 3s delay

**Key functions:**

| Function | Lines | Purpose |
|---|---|---|
| `handleSendMessage(query)` | 234-274 | Main message send flow |
| `handleSaveQnaRequest(query, hideFromChat)` | 94-119 | Save user message to RxDB |
| `handleSaveQnaResponse(responseData)` | 166-226 | Parse + save bot response to RxDB |
| `handleAcceptFeedback(qna)` | 277-307 | Like → re-query bot with positive feedback |
| `handleRejectFeedback(qna)` | 310-344 | Dislike → re-query bot with negative feedback |
| `handleSendToInstructor(instructor, feedbackState)` | 407-434 | Instructor escalation + create QA feedback |
| `handleAskYourInstructor(feedbackState)` | 436-447 | Request "ask instructor" flow from bot |
| `handleAskFromSmartLibrary(qna)` | 449-484 | Query smart library as alternative source |
| `handleOnLoadBook(qna)` / `handleOnUnloadBook(qna)` | 350-405 | Track learning records for file viewing |
| `handleConversationActionClosureMsg()` | 486-518 | Close conversation with message |
| `closeRunningConversationFlow()` | 520-524 | Mark active conversations as DONE |
| `closeChatAndSession()` | 121-136 | Session cleanup + mark read + goodbye |
| `closeSession()` | 138-148 | Send closure message to backend |
| `sendGreetingMsgs()` | 729-744 | Show welcome messages on first open |
| `handleScroll(e)` | 527-575 | Infinite scroll - load older messages |
| `scrollToBottom()` | 578-585 | Scroll chat to newest message |
| `scrollToFirstUnread()` | 588-601 | Scroll to first unread message |

**Lifecycle:**
1. On mount: `startSession()`, `sendGreetingMsgs()`, mark all unread as read
2. On new messages: auto-scroll to bottom (unless scrolling history)
3. On unmount: `closeChatAndSession()` → sends closure to backend

**Pagination:** Shows last 10 messages (`MESSAGES_PER_PAGE`), loads 10 more on scroll-up.

---

### `src/pages/chat-bot/chat-dialog-container/chat-message.js`
**Purpose:** Individual message bubble renderer. **SECOND MOST COMPLEX FILE (975 lines).**

**Renders differently based on:**
- `type` (Bot vs User) → alignment, avatar placement, styling
- `source` (ALERT_ASSIGNMENT, ALERT_RECOMMENDATION, SMART_LIBRARY, conversation types, etc.)
- `actionType` (NO_ACTION, ASK_INSTRUCTOR, SHOW_INSTRUCTORS, ASK_EXTERNAL_RESOURCE)
- `feedbackGiven` (like/dislike/neutral) → button highlight state
- `relatedQuestions` (shown after exactly 1 dislike)

**Conditional renders (in order):**
1. **Message text** - with HTML rendering (`dangerouslySetInnerHTML`)
2. **QA Feedback question/answer** - for QA_FEEDBACK_NOTIFICATIONS
3. **Smart Library link** - opens external viewer with URL params
4. **Display image** - clickable content preview from RAG response
5. **Game cards** - game recommendation with controller icon + title
6. **AL Engine content cards** - file cards with recommended content
7. **Recommendation content cards** - from recommendation alerts
8. **Standard file card** - PDF/DOCX/PPTX with file viewer
9. **Ask External Resource buttons** - Instructor + Smart Library side by side
10. **Like/Dislike buttons** - thumbs up/down (conditional display)
11. **Related questions** - clickable question chips after 1 dislike
12. **Instructor picker** - avatar + name list for selection
13. **Ask Instructor button** - standalone after 3 dislikes
14. **Action button ("Take Action")** - redirect to relevant page
15. **AL Weekly Reminder links** - module checkpoint list with "View More"

**File Viewer integration:**
- `Fullscreen` component wraps `FileViewer`
- Tracks learning records (`LEARN_STARTED` / `LEARN_ENDED`) on open/close
- External file button for non-native formats

---

### `src/pages/chat-bot/chat-dialog-container/message-input/message-input.js`
**Purpose:** Chat text input area with autocomplete suggestions.

**Key behaviors:**
- Textarea with send button (PaperPlaneRight icon)
- Debounces input (3 chars minimum) → calls `getQuestionSuggestions(query)`
- Shows suggestion dropdown; clicking suggestion fills input
- Disables when offline (`!isOnline`) or server error
- Uses `useDebounce` with 600ms for session closure detection
- Shows error messages for network/server issues
- Closes running conversation flows when user sends new message
- Send disabled when no text entered

---

### `src/pages/chat-bot/chat-dialog-container/message-input/instructor-popup.js`
**Purpose:** Modal popup for selecting an instructor to forward question to.

- Fetches course instructors via `findCourseUsers$` (filtered by INSTRUCTOR role)
- Radio button selection with avatars and names
- Auto-selects first instructor
- "Send Question" button → calls `onSend(selectedInstructor)`
- Close button (CrossButton)

---

### `src/pages/chat-bot/chat-bot-alerts/chat-bot-alerts.js`
**Purpose:** Notifications/Alerts tab in chatbot container.

**Key behaviors:**
- Polls alerts via `useIntervalPromise` → `findCourseAlerts()`
- Groups alerts by day index (today=0, yesterday=1, older=2)
- Infinite scroll pagination (10 alerts per page)
- Scroll maintains position when loading older alerts
- Renders alert type-specific card components:
  - `AlertTypes.QA_FEEDBACK` → `InstructorAnswer`
  - `AlertTypes.ASSIGNMENT` → `AssignmentSubmission`
  - `AlertTypes.LESSON_RECOMMENDATION` / `RECOMMENDATION` / `AL_ENGINE` → `Recommendation`
  - `AlertTypes.STUDENT_AI_INTERVENTION` → `AIIntervention`
  - `AlertTypes.GAMES_AVAILABLE` → `GameAlert`
- Only shows `UN_READ` status alerts

---

### `src/workers/chat.worker.js`
**Purpose:** Background service managing proactive chatbot conversations.

**Class: `ChatService`**

**Initialization (`init()`):**
- Creates own `DataStore` instance
- Subscribes to `chatbotbubblemessage` RxDB collection changes → triggers UI updates
- Calls `initializeChatGreetingQueue()` → picks random greeting, creates bubble message
- Starts `run()` loop

**`run()` method (every 5 seconds via `setTimeout`):**
1. Get conversation queue from RxDB
2. Check if any conversation is ACTIVE
3. If no ACTIVE and QUEUED items exist:
   - Group QUEUED by first item's `type`
   - Assign shared `groupId`
   - Mark group as ACTIVE
   - Set `addClosureMsgAfterTimeout()` timer
   - Show bubble message (`updateChatbubbleMsg()`)
   - Add chat messages per item (`addConversationToChatBot()`)
4. Run periodic checks:
   - `sendLearningRecommendationReminders()` - AL practice reminders
   - `sendStudentAIInterventionMessage()` - AI intervention alerts
   - `sendThankYouMessage()` - course completion gratitude

**`addClosureMsgAfterTimeout({configData, source, messages, isFinalStage})` logic:**
- Waits `configData.closureDelay` (typically 10s)
- Checks if conversations still ACTIVE (user might have dismissed them)
- If `hasTwoStages` and not final:
  - Shows intermediate closure message
  - Recursively calls with `isFinalStage: true`
- Else (final stage):
  - Marks all group conversations as DONE
  - Shows bubble closure message
  - Shows chat closure message

**`addConversationToChatBot({id, conversationId, source, msgKey, enMsgData, arMsgData, path, ...})` - Core reusable:**
- Creates bilingual `QuestionAndAnswer` using localized message keys
- Saves to RxDB via `saveToChat()`
- `saveToChat()` → `createQuestionAndAnswer()` + `sendChatMessage({}, true)` (offline-queued)

---

### `src/workers/sync.worker.js` (Chat-related sections)
**Purpose:** Syncs local data with server when online.

**`syncPendingUploads()` (line 894):**
1. Reads all `PendingUpload` records from RxDB
2. Separates by entity type:
   - `chatbotEntities`: `['create-qna-entry', 'al-engine-chat-message', 'chatbot-chat']`
   - `feedbackEntities`: `['update-qna-feedback']`
   - `readAtEntities`: `['bulk-patch-chat']`
3. Bulk-creates chatbot messages: `POST /e2-chatbot/chat?bulk=true`
4. Bulk-patches feedback: concatenated `PATCH /e2-chatbot/chat/{id}` calls
5. Bulk-patches readAt: `POST /e2-chatbot/chat-bulk-patch`
6. Deletes processed PendingUpload records

**`syncChats()` (line 3364):**
1. Fetches all messages: `GET /e2-chatbot/chat?userId=X&courseId=Y&sortByDate=true`
2. Maps to `QuestionAndAnswer` models
3. Generates additional local messages from:
   - Active assignment exam papers with active sessions → `ALERT_ASSIGNMENT`
   - Lesson recommendations → `ALERT_LESSON_RECOMMENDATION`
   - AI recommendations → `ALERT_RECOMMENDATION`
   - QA feedbacks from instructors → `ALERT_QA_FEEDBACK`
   - Games available → `ALERT_GAMES_AVAILABLE`
4. Merges server + generated messages
5. `bulkUpsertQna()` → stores all in RxDB

---

### `src/worker-handler.js`
**Purpose:** Initializes all background services on app load.

**Chat service initialization (line 78-80):**
```javascript
const chatService = await new ChatService();
await chatService.init(loggedInUser, Comlink.proxy(dataStore.db.triggerChange), Comlink.proxy(console));
window.chatService = chatService;
```

Uses **Comlink** library for structured cloning proxy between main thread and workers.

---

### `src/conversation-config.js`
**Purpose:** Defines all proactive conversation types and their timing behavior.

**Statuses:** `QUEUED` → `ACTIVE` → `DONE`

**13 Conversation Types:**
| Type | Bubble Key | Closure Delay | Has Two Stages |
|---|---|---|---|
| `CHAT_PP_ASSIGNMENT_NOTIFICATIONS` | pp-assignment-notification | 10s | No |
| `CHAT_PP_ASSIGNMENT_REMINDERS` | pp-assignment-reminder | 10s | No |
| `CHAT_TA_PRACTICE_NOTIFICATIONS` | ta-practice-notification | 10s | No |
| `CHAT_TA_PRACTICE_REMINDERS` | ta-practice-reminder | 10s | No |
| `CHAT_PENDING_AL_NOTIFICATIONS` | pending-al-notification | 10s | Yes |
| `CHAT_QA_FEEDBACK_NOTIFICATIONS` | qa-feedback-notification | 10s | No |
| `CHAT_UNDERACHIEVER_NOTIFICATIONS` | (none) | 10s | No |
| `CHAT_DISCIPLINE_NOTIFICATIONS` | (none) | 10s | No |
| `CHAT_ATTENDANCE_NOTIFICATIONS` | (none) | 10s | No |
| `CHAT_PUNCTUALITY_NOTIFICATIONS` | (none) | 10s | No |
| `CHAT_PENDING_AL_WEEKLY_REMINDERS` | pending-al-checkpoint | 10s | Yes |
| `CHAT_FIRST_SYSTEM_PRACTICE` | first-system-practice | 10s | Yes |
| `CHAT_SECOND_SYSTEM_PRACTICE` | second-system-practice | 10s | Yes |

---

## File-by-File Breakdown - Backend (e2-chatbot)

### `src/app.js`
**Purpose:** Application entry using `@saal/feathers-starter` framework.

Creates Feathers app with: services, MongoDB, Kafka events, health checks, migrations.

### `src/register-services.js`
**Purpose:** Registers external HTTP service connections as Axios instances.

- `CHAT_SERVICE` → the AI engine (configurable URL)
- `COURSE_SERVICE` → course data service

### `src/constants.js`
**Purpose:** Shared constants for the chatbot service.

```
ActionTypes: NO_ACTION, ASK_INSTRUCTOR, SHOW_INSTRUCTORS, ASK_EXTERNAL_RESOURCE
QuestionAndAnswerType: Bot, User
ChatLogTypes: INFO, ERROR
ChatMessageDataTypes: CONTENTS, PRACTICE, REMINDER
```

### `src/services/chat/chat.service.js`
**Purpose:** Core chatbot orchestration logic. **THE BRAIN OF THE SYSTEM.**

**`create(data, params)` - Main Decision Tree:**
```
If message type is User OR typeOfChat is 'closure':
  │
  ├─ If 2+ dislikes AND typeOfChat is 'qa-feedback':
  │   └─ Return: { actionType: SHOW_INSTRUCTORS, answer: "pick your instructor" }
  │
  ├─ If typeOfChat is 'ask-smart-library':
  │   └─ Call askFromSmartLibrary(query, userId) → POST /library/search
  │
  ├─ If 3 dislikes (hasThreeDislikes):
  │   └─ Return: { actionType: ASK_INSTRUCTOR, answer: "ask instructor message" }
  │
  ├─ If 2 dislikes (hasTwoDisLikes):
  │   └─ Return: { actionType: ASK_EXTERNAL_RESOURCE, answer: "ask external source" }
  │
  ├─ If askToSource is set:
  │   └─ Same escalation logic as above
  │
  └─ Else (normal query - call AI service):
      ├─ If CHAT_PROVIDER === 'DATA_PRISM':
      │   └─ Get course name → build workspace slug
      │   └─ POST /{workspaceName}/chat with Bearer API key
      └─ Else (AI_SERVICE):
          └─ POST /chat with formatted request
```

**After response:**
1. Save to MongoDB (`collection.findOneAndUpdate` with upsert)
2. Log to `chat-logs` service (request + response)
3. Publish event via `app-events` (Kafka)
4. Return `{data: savedMessage, response: formattedResponse}`

**`doesDisLikeLastN(feedbacks, lastN)` - Escalation checker:**
Checks if the last N items in feedbacks array all have `value !== 'like'`.

**`getDataRequiredForResponse(data, usingNewAiService)` - Request formatter:**
- For DATA_PRISM: `{message, mode: 'query', previousFeedback}`
- For AI_SERVICE: `{type, query, courseId, userId, previousFeedback, language}`

**`getFormattedResponse({response, useNewAiService, query})` - Response normalizer:**
- For DATA_PRISM: maps `textResponse`, `sources[0]`, `chatId` → standard format
- For AI_SERVICE: passes through unchanged

### `src/services/chat/chat.schema.js`
**Purpose:** JSON Schema validation for chat CRUD.

**Create:** requires `courseId`, `userId`. If type is User, also requires `query` + `typeOfChat`.
**Find:** requires at least one of `userId`, `courseId`, or `ids`.
**Patch:** requires `id`, cannot update `version` field.

### `src/services/ai-questions-suggestion/ai-questions-suggestion.service.js`
**Purpose:** Autocomplete suggestions from AI service.

Calls `GET CHAT_SERVICE/chat/complete?query=...` → returns `candidates` array.

### `src/services/chat-logs/chat-logs.service.js`
**Purpose:** Audit logging for all chatbot interactions (MongoDB collection: `chatLogs`).

### `.env`
```
PORT=5020
MONGODB_URL=mongodb://127.0.0.1:27017/e2-chatbot
CHAT_PROVIDER=AI_SERVICE                    # or DATA_PRISM
CHAT_SERVICE=https://dev3-e2.saal.ai/smart-learning-companion-aai-service
CHAT_SERVICE_API_KEY=N0DG8PW-DY54MP2-JAMKTVX-T63MT6F
COURSE_SERVICE=https://qa3-e2.saal.ai/e2-course
```

---

## File-by-File Breakdown - Backend (e2-sp-service)

### `src/data-source/api/chatbot.api.js`
**Purpose:** Thin proxy between the student app and `e2-chatbot` service.

| Method | Route Called | Purpose |
|---|---|---|
| `findChats(params)` | `GET CHATBOT_SERVICE/chat` | Fetch chat history |
| `createChats(data, params)` | `POST CHATBOT_SERVICE/chat` | Send message + get response |
| `updateChats(id, data, params)` | `PATCH CHATBOT_SERVICE/chat/{id}` | Update feedback/readAt |
| `findAiQuestionSuggestion(params)` | `GET CHATBOT_SERVICE/ai-question-suggestion` | Autocomplete |

### `src/services/ai-question-suggestion/ai-question-suggestion.service.js`
**Purpose:** Service wrapper that delegates to data source chatbot API.

### `src/data-source/api/index.js`
**Purpose:** Aggregated data source interface. Chat exports:
- `findChats`, `createChats`, `updateChats`, `findAiQuestionSuggestion`

### `src/register-services.js`
**Purpose:** Creates Axios instances for all downstream services.
Registers `CHATBOT_SERVICE` → `http://localhost:5020` from env.

### `src/data-source/db/user-db.service.js` (offline mode)
**Purpose:** In offline/online mode, stores chat messages locally in SQLite.
Posts to `CHATBOT_SERVICE` AND saves to local MongoDB collection.

---

## API Endpoints

### Frontend → e2-sp-service (via proxy)

| Endpoint | Method | Function | Purpose |
|---|---|---|---|
| `/e2-chatbot/chat` | POST | `sendChatMessage()` | Send query, get AI response |
| `/e2-chatbot/chat` | GET | `syncChats()` | Download chat history |
| `/e2-chatbot/chat/{id}` | PATCH | sync worker | Update feedback/readAt |
| `/e2-chatbot/chat?bulk=true` | POST | sync worker | Batch create messages |
| `/e2-chatbot/chat-bulk-patch` | POST | sync worker | Batch update messages |
| `/e2-chatbot/ai-question-suggestion` | GET | `getQuestionSuggestions()` | Autocomplete |
| `/e2-chatbot/healthcheck` | GET | `checkChatBotEnabled()` | Feature toggle check |
| `/e2-course/qa-feedback/` | POST | `createQaFeedback()` | Instructor escalation record |

### e2-chatbot → AI Service

| Endpoint | Method | Purpose |
|---|---|---|
| `POST /chat` | Standard query (AI_SERVICE provider) |
| `POST /{workspace}/chat` | Workspace-scoped query (DATA_PRISM) |
| `GET /chat/complete` | Question autocomplete candidates |
| `POST /library/search` | Smart library content search |

---

## Offline Architecture

### Local-First Pattern

```
┌─────────────────────────────────────────────────────┐
│  USER INTERACTION                                     │
│  (Always reads from RxDB, never directly from API)   │
└──────────────────────────┬──────────────────────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
     ┌─────────────┐ ┌──────────┐ ┌──────────────┐
     │ Read data   │ │ Write    │ │ Real-time AI │
     │ from RxDB   │ │ to RxDB  │ │ query (POST) │
     │ (always     │ │ + queue  │ │ (online only)│
     │  works)     │ │ upload   │ │              │
     └─────────────┘ └────┬─────┘ └──────────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │ PendingUpload   │
                  │ Queue (RxDB)    │
                  └────────┬────────┘
                           │ (when online)
                           ▼
                  ┌─────────────────┐
                  │ Sync Worker     │
                  │ pushes to server│
                  └─────────────────┘
```

### What Works Offline

| Feature | Works Offline? | Reason |
|---|---|---|
| View chat history | ✅ Yes | Stored in local RxDB |
| See unread badge count | ✅ Yes | Computed from local RxDB query |
| View notifications/alerts | ✅ Yes | Alerts stored locally during sync |
| Proactive conversations | ✅ Yes | Generated by chat.worker.js locally |
| Open file attachments | ✅ Yes | Files downloaded during sync to local disk |
| Send new message | ❌ No | Requires real-time AI response |
| Question suggestions | ❌ No | Requires API call to AI service |
| Like/Dislike feedback | ⚠️ Partial | Saves locally, queues server update |
| Session closure message | ⚠️ Partial | Queued if offline |

### PendingUpload Queue Mechanism

```javascript
// Any action that needs to reach the server:
await this.createPendingUpload(new PendingUpload({
  entity: 'update-qna-feedback',  // identifies the handler in sync worker
  data: { id, feedbackGiven: 'like', ... },
  id: shortid(),
}));

// Sync worker (when online) routes by entity:
// 'chatbot-chat'         → POST /e2-chatbot/chat
// 'update-qna-feedback'  → PATCH /e2-chatbot/chat/{id}
// 'bulk-create-chat'     → POST /e2-chatbot/chat?bulk=true
// 'bulk-patch-chat'      → POST /e2-chatbot/chat-bulk-patch
```

### Online Detection

```javascript
// sync.worker.js
async ensureConnection() {
  return axios.get(`${REACT_APP_API_URL}/e2-irp/healthcheck`, { timeout: 5000 });
}
// Sets this.isConnected = true/false

// UI checks via hook:
const { isOnline } = useSync();
// MessageInput disables textarea when !isOnline
```

---

## Conversation Flow System

### Queue Lifecycle Diagram

```
External trigger (sync discovers assignment, recommendation, etc.)
  └── addConversationsToQueue([{type, content, courseId}])
      └── Status: QUEUED in RxDB
          │
          ▼ (chat.worker.js picks it up every 5s)
      Status: ACTIVE + groupId assigned
          │
          ├── Shows bubble message on floating avatar
          ├── Adds chat message(s) to conversation in RxDB
          │
          ├── Path A: User opens chat and clicks action button
          │   └── handleConversationClick(redirectPath)
          │       ├── After actionClosureDelay (2s):
          │       │   ├── Shows actionClosureMessage in chat
          │       │   └── closeRunningConversationFlow() → Status: DONE
          │       └── Navigates to redirectPath (lesson page, etc.)
          │
          └── Path B: User ignores (doesn't interact)
              └── After closureDelay (10s):
                  ├── If hasTwoStages:
                  │   ├── Shows intermediate closureMessage
                  │   └── After another closureDelay:
                  │       ├── Shows finalClosureMessage + bubble
                  │       └── Status: DONE
                  └── Else:
                      ├── Shows closureMessage + bubble closure
                      └── Status: DONE
```

### What Triggers Conversations Being Queued

Conversations are added by the **alerts worker** and **sync worker** based on system events:
- Assignment exam papers becoming active
- Practice due dates approaching
- AL engine checkpoint detection
- Instructor answering a QA feedback
- Discipline/attendance/punctuality violations detected
- AI intervention risk thresholds exceeded
- System practice milestones

---

## Feedback & Escalation Logic

### Backend Decision (e2-chatbot chat.service.js)

```javascript
const hasThreeDislikes = this.doesDisLikeLastN(previousFeedback, 3);
const hasTwoDisLikes = this.doesDisLikeLastN(previousFeedback, 2);

// Escalation ladder:
// 0 dislikes → normal AI answer + relatedQuestions[]
// 1 dislike  → new AI answer + relatedQuestions[] (shown as chips)
// 2 dislikes → ASK_EXTERNAL_RESOURCE (Smart Library OR Instructor buttons)
// 3 dislikes → ASK_INSTRUCTOR (only "Ask Instructor" button)
// qa-feedback type + 2+ → SHOW_INSTRUCTORS (instructor picker list)
```

### `feedbackState` Object (Client → Server)

```javascript
{
  question: "What is quantum physics?",      // Original question text
  questionId: "abc123",                       // Original question ID
  answerId: "def456",                         // Current answer being evaluated
  feedbacks: [
    { answerId: "ans1", answer: "First answer", value: "dislike" },
    { answerId: "ans2", answer: "Second answer", value: "dislike" },
    { answerId: "ans3", answer: "Third answer" },  // no value = pending
  ]
}
```

This entire object is passed with every subsequent `sendChatMessage()` call so the backend has full context of the feedback history for the current conversation thread.

---

## Reusable Functions & Hooks

### Custom Hooks Used by Chatbot

| Hook | File | Purpose |
|---|---|---|
| `useStore()` | `hooks/use-store.js` | Access all DataStore methods as functions |
| `useObservable(selector)` | `hooks/use-observable.js` | Subscribe to RxDB reactive queries |
| `useLocale()` | `hooks/use-locale.js` | Get `i(key, data, locale)` translator + current locale |
| `useTheme()` | `hooks/use-theme.js` | Get current theme (DARK/LIGHT) |
| `useAuth()` | `hooks/use-auth.js` | Get current logged-in user |
| `useSync()` | `hooks/use-sync.js` | Get `isOnline` / `isSyncing` status |
| `usePromise(fn, opts)` | `hooks/use-promise.js` | Async data fetching with dependencies |
| `useIntervalPromise(fn, opts)` | `hooks/use-promise-interval.js` | Polling data fetcher |
| `useDebounce(deps, fn, delay)` | `hooks/use-debounce.js` | Debounced side-effect callback |

### Key DataStore Functions (src/data/data-store.js)

| Function | Purpose |
|---|---|
| `sendChatMessage(data, handleOffline)` | POST /e2-chatbot/chat or queue for later |
| `createQuestionAndAnswer(data)` | Save message to local RxDB |
| `updateQnaFeedback(data)` | Patch feedback in RxDB + queue upload |
| `markQnaAsRead(selector, readAt)` | Batch mark messages as read |
| `createQaFeedback(data)` | POST /e2-course/qa-feedback/ (instructor escalation) |
| `getQuestionSuggestions(query)` | GET /e2-chatbot/ai-question-suggestion |
| `initialWelcomeMessage(messages)` | First-time welcome messages |
| `addChatGreeting(messages)` | Add greeting if last message isn't recent |
| `addMessageToChatBotBubble(data)` | Set floating avatar bubble message |
| `getLatestBotBubbleMsg$()` | Observable for current bubble message |
| `getConversationQueue(query)` | Read conversation queue from RxDB |
| `addConversationsToQueue(data)` | Insert items into conversation queue |
| `updateConversations(query, data)` | Update conversation status |
| `bulkUpsertGreetings(data)` | Replace all bubble messages |
| `findQna(query)` | Query messages (non-reactive, returns array) |
| `findQna$(query)` | Query messages (reactive RxDB observable) |
| `findQnaCount$(query)` | Count messages (reactive observable) |
| `createPendingUpload(data)` | Queue an action for sync upload |

### Utility Functions (src/utils.js)

| Function | Purpose |
|---|---|
| `checkChatBotEnabled()` | GET /e2-chatbot/healthcheck → returns `{enableChatbot}` |
| `secondsDifference(date)` | Seconds elapsed since given date |
| `getChatbotDate(date, format, locale)` | Format date for chatbot display |
| `daysDifference(date)` | Days between now and given date |
| `isArabic(text)` | Check if text contains Arabic characters |
| `getChatMessageTextDirection(text, defaultDir)` | Auto-detect RTL/LTR per message |
| `getUserName(user)` | Format user display name from model |
| `getContentUrl(path)` | Build full content service URL |
| `openFileExternally(file)` | Open file in system default viewer |
| `openUrlExternally(url)` | Open URL in external browser |
| `constructUrlWithQueryParams(url, params)` | Build URL with encoded query params |
| `getTranslation(key, data, locale)` | Used in workers (no React context) |

---

## Data Models

### QuestionAndAnswer (Chat Message)

```javascript
{
  id: "string",                   // Unique message ID (shortid)
  sessionId: "string",            // Conversation session ID
  questionId: "string",           // Links question to its answer chain
  answerId: "string",             // Specific answer identifier for feedback
  courseId: "string",             // Course context
  userId: "string",              // Sender user ID
  date: "ISO date string",       // Message timestamp
  type: "Bot" | "User",          // Who sent the message
  source: "string",              // Origin identifier (see Sources below)
  isGreeting: boolean,            // Is it a greeting/welcome message
  readAt: "ISO string" | null,   // When message was read (null = unread)
  hideFromChat: boolean,          // Hidden from chat UI (used for internal messages)
  feedbackGiven: "like" | "dislike" | "neutral" | null,
  feedbackState: {
    question: "string",           // Original question text
    questionId: "string",
    feedbacks: [{
      answerId: "string",
      answer: "string",           // Answer text
      value: "like" | "dislike"   // User's feedback
    }]
  },
  actionType: "NO_ACTION" | "ASK_INSTRUCTOR" | "SHOW_INSTRUCTORS" | "ASK_EXTERNAL_RESOURCE",
  query: "string",               // Original user question text
  relatedQuestions: ["string"],   // AI-suggested follow-up questions
  sender: { User object },        // For recommendation alerts - who sent it
  message: {
    text: "string",               // Display text (English default)
    locales: [{ text: "string", locale: "ar" }],  // Arabic translation
    question: "string",           // For QA feedback display
    answer: "string",             // For QA feedback display
    content: {
      file: { path, id, title, extension, fileName },
      displayImage: "string",     // Image URL from RAG
      path: "string",             // Redirect path for action buttons
      pageNumber: number,         // Book page reference
      id: "string",              // Content ID
      teachingPointIds: ["string"],
      moduleId: "string",
      lessonId: "string"
    },
    data: {
      type: "CONTENTS" | "PRACTICE" | "REMINDER",
      lessonId: "string",
      contents: [{ file, path, text, id, itemType, ... }],
      programItemMetas: [{ id, name, code, type, endDate, locales }]
    },
    gamecontent: { title, gameConfigurationId, locales }
  }
}
```

**RxDB Schema version:** 4 | **Primary key:** `clientId` | **Indexes:** `['id']`

### ChatbotBubbleMsg

```javascript
{
  id: "string",
  message: "string",              // Display text (English)
  type: "closure" | "greeting" | "conversation",
  locales: [{ locale: "ar", message: "Arabic text" }]
}
```

**RxDB Schema version:** 0 | **Primary key:** `clientId`

### ChatbotConversationQueue

```javascript
{
  id: "string",
  type: "CHAT_PP_ASSIGNMENT_NOTIFICATIONS" | ... (13 types),
  status: "QUEUED" | "ACTIVE" | "DONE",
  courseId: "string",
  groupId: "string",              // Links related conversations in same batch
  content: {                       // Context payload (varies by type)
    examPaper, exam, lesson, module, course,
    feedback, redirectPath, data, ...
  }
}
```

**RxDB Schema version:** 0 | **Primary key:** `id`

---

## Constants & Configuration

### ActionTypes (determines UI behavior of bot messages)
| Constant | UI Effect |
|---|---|
| `NO_ACTION` | Normal message with like/dislike buttons |
| `ASK_INSTRUCTOR` | Shows "Ask Your Instructor" button |
| `SHOW_INSTRUCTORS` | Shows instructor picker list |
| `ASK_EXTERNAL_RESOURCE` | Shows both "Instructor" + "Smart Library" buttons |

### QuestionAndAnswerType
| Value | Meaning |
|---|---|
| `Bot` | Message from the chatbot/system |
| `User` | Message from the student |

### Message Sources (determines rendering + button visibility)
| Source | Origin |
|---|---|
| `'I-STUDY'` | User query via chat interface |
| `'SMART_LIBRARY'` | Answer from smart library |
| `'small_talk'` / `'smalltalk'` | Casual conversation (no feedback buttons) |
| `'thank_you_message'` | Course completion gratitude |
| `'ALERT_ASSIGNMENT'` | Assignment notification |
| `'ALERT_LESSON_RECOMMENDATION'` | Instructor-sent recommendation |
| `'ALERT_RECOMMENDATION'` | AI-generated recommendation |
| `'ALERT_AL_ENGINE'` | Adaptive learning engine message |
| `'ALERT_QA_FEEDBACK'` | Instructor answered question |
| `'ALERT_STUDENT_AI_INTERVENTION'` | AI risk intervention |
| `'ALERT_GAMES_AVAILABLE'` | Game recommendation |
| `'CHAT_PP_ASSIGNMENT_*'` | Conversation flow messages |
| `'CHAT_TA_PRACTICE_*'` | Practice conversation flows |
| `'CHAT_PENDING_AL_*'` | AL checkpoint flows |

### Environment Configuration
```
REACT_APP_API_URL=http://localhost:8000     # Gateway (e2-sp-service)
REACT_APP_CONTENT_URL=http://localhost:8000/e2-content
REACT_APP_DEMO_LOGO=false                   # Demo mode flag
```

---

## Key Integration Points

1. **Tour System:** `react-shepherd` in `ChatBotContainer` - first-time user guided onboarding
2. **Learning Records:** Tracked via `createLearningRecord()` when files opened/closed from chat
3. **Matomo Analytics:** Page views tracked in Layout
4. **Keycloak Auth:** JWT token passed with all API calls via Axios interceptor
5. **Content Service:** Files referenced in messages served from `/e2-content`
6. **Kafka Events:** `e2-chatbot` publishes chat events for other microservices
7. **Smart Library:** External app opened with URL params (userId, courseId, locale, contentId)
8. **Symbology App:** External practice app accessible from layout
