# Chatbot Implementation Diagrams - e2-student-app-web

> Visual diagrams for implementing the chatbot in the new web application.
> Render these with any Mermaid-compatible viewer (GitHub, VS Code extension, Mermaid Live Editor).

---

## 1. High-Level Architecture (Online-Only)

```mermaid
graph TB
    subgraph Browser["🌐 Browser (e2-student-app-web)"]
        direction TB
        UI["React UI<br/>(Chat Components)"]
        Hooks["Custom Hooks<br/>(useChatMessages, useChatSend,<br/>useChatFeedback, useChatSession)"]
        Service["Chat Service Layer<br/>(chatbot.service.ts)"]
        Store["Chat Store<br/>(React Context / Zustand)<br/>UI state only"]
        Cache["In-Memory Cache<br/>(React Query or manual)"]

        UI --> Hooks
        Hooks --> Service
        Hooks --> Store
        Hooks --> Cache
        Service -->|"Axios"| Proxy
    end

    subgraph Proxy["setupProxy.ts"]
        P1["/e2-sp-service → localhost:5017<br/>or remote server"]
    end

    subgraph SPService["🔀 e2-sp-service (unchanged)"]
        GW["FeathersJS Gateway<br/>/chat, /ai-question-suggestion<br/>/qa-feedback, /alerts"]
    end

    subgraph Chatbot["🤖 e2-chatbot (unchanged)"]
        CS["Chat Service"]
        MONGO[(MongoDB)]
        CS --> MONGO
    end

    subgraph AI["🧠 smart-learning-companion-aai-service"]
        AAI["RAG/LLM Engine"]
    end

    Proxy --> SPService
    GW --> CS
    CS --> AAI
```

---

## 2. Comparison: Legacy vs New Architecture

```mermaid
graph LR
    subgraph Legacy["❌ Legacy (e2-student-portal)"]
        direction TB
        L1["React UI"]
        L2["DataStore (data-store.js)"]
        L3["RxDB (IndexedDB)"]
        L4["Chat Worker (5s loop)"]
        L5["Sync Worker (interval)"]
        L6["PendingUpload Queue"]
        L7["Comlink (worker bridge)"]

        L1 --> L2
        L2 --> L3
        L4 --> L3
        L5 --> L3
        L5 --> L6
    end

    subgraph New["✅ New (e2-student-app-web)"]
        direction TB
        N1["React UI (TypeScript)"]
        N2["Custom Hooks"]
        N3["Chat Service (Axios)"]
        N4["React Context (UI state)"]
        N5["Polling (setInterval)"]

        N1 --> N2
        N2 --> N3
        N2 --> N4
        N2 --> N5
    end

    Legacy -.->|"Migrates to"| New
```

---

## 3. Component Architecture

```mermaid
graph TD
    MainLayout["MainLayout.tsx<br/>(existing)"]
    MainLayout --> Provider["ChatbotProvider.tsx<br/>🔧 Context + Session"]

    Provider --> Avatar["ChatbotAvatar.tsx<br/>🟢 Floating Button + Badge"]
    Provider --> Container["ChatbotContainer.tsx<br/>📦 Ant Design Drawer/Modal"]

    Avatar --> AvatarBubble["ChatbotAvatarBubble.tsx<br/>💬 Tooltip Message"]

    Container --> Tabs["Ant Design Tabs"]
    Tabs --> ChatTab["Chat Tab"]
    Tabs --> NotifTab["Notifications Tab"]

    ChatTab --> Dialog["ChatbotDialog.tsx<br/>📝 Message List + Input"]
    Dialog --> MsgList["Message List<br/>(paginated, scroll-up to load more)"]
    Dialog --> Typing["ChatTypingIndicator.tsx<br/>⏳ Animated Dots"]
    Dialog --> Input["ChatInput.tsx<br/>⌨️ Textarea + Send"]

    MsgList --> Message["ChatMessage.tsx<br/>Individual Bubble"]
    Message --> Content["ChatMessageContent.tsx<br/>📄 Files, Images, Games"]
    Message --> Actions["ChatMessageActions.tsx<br/>👍 👎 Buttons"]
    Message --> Related["ChatMessageRelated.tsx<br/>❓ Question Chips"]

    Input --> Suggestions["ChatSuggestions.tsx<br/>🔍 Autocomplete Dropdown"]

    Actions --> InstrPicker["ChatInstructorPicker.tsx<br/>👨‍🏫 Modal"]

    NotifTab --> Notifications["ChatNotifications.tsx"]
    Notifications --> NotifCard["ChatNotificationCard.tsx<br/>📋 Alert Cards"]
```

---

## 4. Data Flow: Send Message (Online-Only)

```mermaid
sequenceDiagram
    participant User
    participant Input as ChatInput
    participant Hook as useChatSend
    participant Store as ChatStore
    participant Service as chatbot.service.ts
    participant API as e2-sp-service
    participant CB as e2-chatbot
    participant AI as AI Service

    User->>Input: Types message, clicks Send
    Input->>Hook: sendMessage(query)
    Hook->>Store: setTyping(true)
    Hook->>Store: addOptimisticMessage({type:'User', text: query})
    Note over Store: UI shows user message immediately

    Hook->>Service: chatbotService.sendMessage({query, feedbackState, ...})
    Service->>API: POST /e2-sp-service/chat
    API->>CB: POST /chat (proxied)
    CB->>AI: POST /chat (RAG/LLM query)
    AI-->>CB: {answer, relatedQuestions, content}
    CB-->>API: {data, response}
    API-->>Service: Response

    Service-->>Hook: Parsed response
    Hook->>Store: setTyping(false)
    Hook->>Store: addMessage({type:'Bot', ...response})
    Hook->>Store: updateFeedbackState(response)
    Note over Store: UI shows bot response
```

---

## 5. Data Flow: Like / Dislike with Escalation

```mermaid
sequenceDiagram
    participant User
    participant Actions as ChatMessageActions
    participant Hook as useChatFeedback
    participant Store as ChatStore
    participant Service as chatbot.service.ts
    participant API as e2-sp-service

    User->>Actions: Clicks 👎 Dislike
    Actions->>Hook: submitFeedback('dislike', messageId)

    Hook->>Service: chatbotService.updateFeedback(id, {feedbackGiven:'dislike'})
    Service->>API: PATCH /e2-sp-service/chat/:id
    Note over API: Feedback saved in MongoDB

    Hook->>Store: Update feedbackState.feedbacks[]
    Hook->>Hook: sendMessage with updated feedbackState
    Hook->>Service: chatbotService.sendMessage({feedbackState, query})
    Service->>API: POST /e2-sp-service/chat

    alt 1st dislike
        API-->>Hook: {actionType: NO_ACTION, relatedQuestions: [...]}
        Hook->>Store: Show new answer + related question chips
    else 2nd dislike
        API-->>Hook: {actionType: ASK_EXTERNAL_RESOURCE}
        Hook->>Store: Show "Instructor" + "Smart Library" buttons
    else 3rd dislike
        API-->>Hook: {actionType: ASK_INSTRUCTOR}
        Hook->>Store: Show "Ask Instructor" button
    end
```

---

## 6. State Management Design

```mermaid
graph TB
    subgraph ServerState["📡 Server State (fetched from API)"]
        direction TB
        S1["Chat Messages<br/>(paginated history)"]
        S2["Unread Count"]
        S3["Suggestions<br/>(autocomplete)"]
        S4["Notifications<br/>(alerts)"]
        S5["Healthcheck<br/>(chatbot enabled?)"]
    end

    subgraph ClientState["💾 Client UI State (React Context)"]
        direction TB
        C1["isOpen: boolean"]
        C2["activeTab: 'chat' | 'notifications'"]
        C3["isTyping: boolean"]
        C4["sessionId: string"]
        C5["feedbackState: FeedbackState"]
        C6["error: string | null"]
    end

    subgraph Hooks["🪝 Custom Hooks (bridge)"]
        direction TB
        H1["useChatMessages()"]
        H2["useChatSend()"]
        H3["useChatFeedback()"]
        H4["useChatSuggestions()"]
        H5["useChatSession()"]
        H6["useChatHealthcheck()"]
        H7["useChatUnreadCount()"]
        H8["useChatNotifications()"]
    end

    H1 --> S1
    H2 --> S1
    H2 --> C3
    H2 --> C5
    H3 --> S1
    H3 --> C5
    H4 --> S3
    H5 --> C4
    H6 --> S5
    H7 --> S2
    H8 --> S4
```

---

## 7. API Routes Map

```mermaid
graph LR
    subgraph Frontend["Frontend Calls"]
        F1["sendMessage()"]
        F2["getMessages()"]
        F3["updateFeedback()"]
        F4["getSuggestions()"]
        F5["healthcheck()"]
        F6["createQaFeedback()"]
        F7["markAsRead()"]
        F8["getNotifications()"]
    end

    subgraph Routes["e2-sp-service Routes"]
        R1["POST /chat"]
        R2["GET /chat"]
        R3["PATCH /chat/:id"]
        R4["GET /ai-question-suggestion"]
        R5["GET /healthcheck"]
        R6["POST /qa-feedback"]
        R7["PATCH /chat/:id (readAt)"]
        R8["GET /alerts"]
    end

    subgraph Backend["e2-chatbot Handles"]
        B1["create() → AI query"]
        B2["find() → MongoDB"]
        B3["patch() → MongoDB"]
        B4["AI suggestion → GET /chat/complete"]
        B5["Feature toggle"]
        B6["Course service"]
    end

    F1 --> R1 --> B1
    F2 --> R2 --> B2
    F3 --> R3 --> B3
    F4 --> R4 --> B4
    F5 --> R5 --> B5
    F6 --> R6 --> B6
    F7 --> R7 --> B3
    F8 --> R8
```

---

## 8. File Structure Implementation Map

```mermaid
graph TD
    subgraph Services["src/services/chatbot/"]
        SVC1["chatbot.service.ts<br/>All API calls"]
        SVC2["chatbot.types.ts<br/>TypeScript interfaces"]
        SVC3["chatbot.constants.ts<br/>ActionTypes, Sources"]
        SVC4["chatbot.utils.ts<br/>Locale, RTL, dates"]
    end

    subgraph HooksDir["src/hooks/chatbot/"]
        HK1["useChatMessages.ts"]
        HK2["useChatSend.ts"]
        HK3["useChatFeedback.ts"]
        HK4["useChatSuggestions.ts"]
        HK5["useChatSession.ts"]
        HK6["useChatHealthcheck.ts"]
        HK7["useChatUnreadCount.ts"]
        HK8["useChatNotifications.ts"]
    end

    subgraph Components["src/components/organisms/chatbot/"]
        CP1["ChatbotProvider.tsx"]
        CP2["ChatbotAvatar.tsx"]
        CP3["ChatbotAvatarBubble.tsx"]
        CP4["ChatbotContainer.tsx"]
        CP5["ChatbotDialog.tsx"]
        CP6["ChatMessage.tsx"]
        CP7["ChatMessageContent.tsx"]
        CP8["ChatMessageActions.tsx"]
        CP9["ChatMessageRelated.tsx"]
        CP10["ChatInput.tsx"]
        CP11["ChatSuggestions.tsx"]
        CP12["ChatTypingIndicator.tsx"]
        CP13["ChatInstructorPicker.tsx"]
        CP14["ChatFileViewer.tsx"]
        CP15["ChatNotifications.tsx"]
        CP16["ChatNotificationCard.tsx"]
        CP17["index.ts (barrel)"]
    end

    HooksDir -->|"uses"| Services
    Components -->|"uses"| HooksDir
    CP1 -->|"wraps"| CP2
    CP1 -->|"wraps"| CP4
    CP4 -->|"contains"| CP5
    CP5 -->|"contains"| CP6
    CP5 -->|"contains"| CP10
```

---

## 9. Implementation Phases (Timeline)

```mermaid
gantt
    title Chatbot Implementation Phases
    dateFormat  YYYY-MM-DD
    axisFormat  %b %d

    section Phase 1 - Core MVP
    Types & Constants           :p1a, 2026-08-25, 2d
    Service Layer (API calls)   :p1b, after p1a, 2d
    Chat Store (Context)        :p1c, after p1a, 1d
    ChatbotProvider             :p1d, after p1c, 1d
    ChatbotAvatar (basic)       :p1e, after p1d, 2d
    ChatbotContainer            :p1f, after p1d, 2d
    ChatbotDialog               :p1g, after p1f, 3d
    ChatMessage (text only)     :p1h, after p1g, 2d
    ChatInput (no suggestions)  :p1i, after p1g, 2d
    ChatTypingIndicator         :p1j, after p1i, 1d
    useChatSend hook            :p1k, after p1b, 2d
    useChatMessages hook        :p1l, after p1b, 2d
    useChatSession hook         :p1m, after p1k, 1d
    Mount in MainLayout         :p1n, after p1j, 1d

    section Phase 2 - Feedback
    ChatMessageActions          :p2a, after p1n, 2d
    useChatFeedback hook        :p2b, after p1n, 2d
    FeedbackState tracking      :p2c, after p2b, 1d
    ChatMessageRelated          :p2d, after p2a, 2d
    ChatInstructorPicker        :p2e, after p2d, 2d
    Smart Library redirect      :p2f, after p2d, 1d

    section Phase 3 - Rich Content
    ChatMessageContent          :p3a, after p2e, 3d
    ChatFileViewer              :p3b, after p3a, 2d
    Infinite scroll (history)   :p3c, after p3a, 2d
    Unread badge                :p3d, after p3c, 1d

    section Phase 4 - Polish
    ChatSuggestions             :p4a, after p3d, 2d
    ChatNotifications           :p4b, after p3d, 3d
    Error boundaries            :p4c, after p4a, 1d
    RTL / Bilingual             :p4d, after p4c, 2d
    Testing                     :p4e, after p4d, 3d
```

---

## 10. Session Lifecycle (Web vs Legacy)

```mermaid
stateDiagram-v2
    [*] --> Idle: App loads

    state Idle {
        [*] --> CheckHealth: Poll every 60s
        CheckHealth --> Enabled: enableChatbot: true
        CheckHealth --> Disabled: enableChatbot: false
    }

    Enabled --> AvatarVisible: Show floating button
    Disabled --> Hidden: Hide chatbot entirely

    AvatarVisible --> ChatOpen: User clicks avatar
    
    state ChatOpen {
        [*] --> StartSession: Generate sessionId
        StartSession --> Active: Mark messages as read
        Active --> Typing: User sends message
        Typing --> Active: Bot responds
        Active --> FeedbackFlow: User likes/dislikes
        FeedbackFlow --> Active: Escalation resolved
    }

    ChatOpen --> SessionEnd: User closes chat
    
    state SessionEnd {
        [*] --> SendClosure: POST /chat {typeOfChat:'closure'}
        SendClosure --> Cleanup: Clear sessionId
    }

    SessionEnd --> AvatarVisible: Back to idle

    note right of ChatOpen
        Web-specific additions:
        - beforeunload sends closure
        - visibilitychange pauses polling
        - Token expiry → re-auth prompt
    end note
```

---

## 11. Error Handling Strategy

```mermaid
flowchart TD
    Start["API Call Made"] --> Response{"Response?"}

    Response -->|"200 OK"| Success["✅ Process response<br/>Update UI"]
    Response -->|"Network Error"| NetErr["Show inline error<br/>+ Retry button"]
    Response -->|"401 Unauthorized"| Auth["Token expired<br/>→ Refresh token<br/>→ Retry once"]
    Response -->|"500 Server Error"| ServerErr["Show toast notification<br/>'Service temporarily unavailable'"]
    Response -->|"Timeout (5min)"| Timeout["Show 'Still thinking...'<br/>+ Cancel button"]

    Auth -->|"Refresh failed"| Logout["Redirect to login"]
    Auth -->|"Refresh success"| Retry["Retry original request"]

    NetErr -->|"User clicks Retry"| Start
    Timeout -->|"User cancels"| Cancel["Remove typing indicator<br/>Show 'Request cancelled'"]

    subgraph HealthcheckFailure["Healthcheck Returns false"]
        H1["Hide chatbot avatar entirely"]
        H2["If chat was open: show banner<br/>'Chatbot is currently unavailable'"]
    end
```

---

## 12. Polling Strategy (No WebSocket)

```mermaid
graph TB
    subgraph WhenChatClosed["When Chat is CLOSED"]
        A1["Poll unread count<br/>GET /chat?userId&courseId&readAt=null<br/>Every 60s"]
        A2["Poll healthcheck<br/>GET /healthcheck<br/>Every 60s"]
        A1 --> Badge["Update badge number"]
        A2 --> Toggle["Show/hide avatar"]
    end

    subgraph WhenChatOpen["When Chat is OPEN"]
        B1["Poll new messages<br/>GET /chat?since=lastMessageDate<br/>Every 30s"]
        B2["Stop unread polling<br/>(already viewing chat)"]
        B1 --> Merge["Merge new messages into list"]
    end

    subgraph WhenTyping["When User is Typing"]
        C1["Stop all polling<br/>(waiting for response)"]
        C2["Resume after response received"]
    end

    subgraph Optimization["⚡ Optimizations"]
        D1["Use visibilitychange API<br/>Pause polling when tab hidden"]
        D2["Exponential backoff on errors"]
        D3["Cancel pending requests on unmount"]
    end
```

---

## 13. Migration Path: What Maps Where

```mermaid
graph LR
    subgraph OldFiles["Legacy Files (e2-student-portal)"]
        O1["data-store.js<br/>(6437 lines!)"]
        O2["chat-dialog-container.js<br/>(828 lines)"]
        O3["chat-message.js<br/>(975 lines)"]
        O4["message-input.js"]
        O5["chat-bot-avatar.js"]
        O6["chat-bot-container.js"]
        O7["chat.worker.js"]
        O8["sync.worker.js"]
        O9["conversation-config.js"]
        O10["constants.js"]
    end

    subgraph NewFiles["New Files (e2-student-app-web)"]
        N1["chatbot.service.ts<br/>(~150 lines)"]
        N2["ChatbotDialog.tsx<br/>(~300 lines)"]
        N3["ChatMessage.tsx<br/>(~100 lines)<br/>+ ChatMessageContent.tsx<br/>+ ChatMessageActions.tsx<br/>+ ChatMessageRelated.tsx"]
        N4["ChatInput.tsx<br/>+ ChatSuggestions.tsx"]
        N5["ChatbotAvatar.tsx"]
        N6["ChatbotContainer.tsx"]
        N7["Polling in hooks<br/>(~50 lines)"]
        N8["❌ REMOVED"]
        N9["chatbot.constants.ts<br/>(simplified)"]
        N10["chatbot.constants.ts"]
    end

    O1 -->|"Extract API calls only"| N1
    O2 -->|"Simplify (no RxDB, no workers)"| N2
    O3 -->|"Split into 4 focused components"| N3
    O4 -->|"Port + add TypeScript"| N4
    O5 -->|"Port + use React Query/polling"| N5
    O6 -->|"Port + use Ant Design Drawer"| N6
    O7 -->|"Replace with polling hooks"| N7
    O8 -->|"Not needed (online-only)"| N8
    O9 -->|"Keep display config only"| N9
    O10 -->|"Port as TypeScript enums"| N10
```

---

## 14. Suggested Ant Design Components Usage

```mermaid
graph TD
    subgraph AntDesign["🐜 Ant Design Components to Use"]
        AD1["Drawer<br/>→ ChatbotContainer<br/>(slide-in panel)"]
        AD2["Tabs<br/>→ Chat / Notifications toggle"]
        AD3["Badge<br/>→ Unread count on avatar"]
        AD4["Input.TextArea<br/>→ Message input"]
        AD5["Button<br/>→ Send, Like, Dislike, Action"]
        AD6["Tooltip<br/>→ Avatar bubble message"]
        AD7["Modal<br/>→ Instructor picker"]
        AD8["List + List.Item<br/>→ Message list"]
        AD9["Skeleton<br/>→ Loading states"]
        AD10["Empty<br/>→ No messages state"]
        AD11["Alert<br/>→ Error messages"]
        AD12["Tag<br/>→ Related question chips"]
        AD13["Avatar<br/>→ Bot/User avatars"]
        AD14["FloatButton<br/>→ Chat avatar button"]
        AD15["Spin<br/>→ Typing indicator"]
        AD16["Card<br/>→ Notification cards"]
        AD17["Popover<br/>→ Suggestions dropdown"]
    end
```

---

## 15. Feature Flag & Graceful Degradation

```mermaid
flowchart TD
    AppLoad["App Loads"] --> InitProvider["ChatbotProvider mounts"]
    InitProvider --> HealthPoll["Poll: GET /e2-sp-service/healthcheck<br/>or chatbot-specific endpoint"]

    HealthPoll -->|"Success: enabled=true"| Mount["Mount ChatbotAvatar"]
    HealthPoll -->|"Success: enabled=false"| Skip["Don't render chatbot"]
    HealthPoll -->|"Network error"| Retry["Retry in 60s<br/>(assume disabled)"]

    Mount --> UserOpens["User opens chat"]
    UserOpens --> CheckAgain{"Re-verify on open"}

    CheckAgain -->|"Still enabled"| ShowChat["Show ChatbotDialog"]
    CheckAgain -->|"Now disabled"| ShowBanner["Show 'Chatbot unavailable'<br/>banner in container"]

    ShowChat --> SendMsg["User sends message"]
    SendMsg -->|"AI service timeout"| ShowRetry["Show retry button<br/>+ 'Still processing...'"]
    SendMsg -->|"500 error"| ShowError["Show error inline<br/>'Could not get response'"]
    SendMsg -->|"Success"| ShowResponse["Show bot response"]

    ShowRetry -->|"After 3 retries"| Degrade["Disable input<br/>Show 'Service temporarily down'"]
```

---

## Quick Reference: Technology Decisions

| Decision | Recommendation | Rationale |
|---|---|---|
| State management | React Context (existing pattern) | Consistent with app, chatbot state is simple |
| Data fetching | StoreContext Axios + custom hooks | No new deps, team knows it |
| Real-time updates | `setInterval` polling (30-60s) | Simple, no backend changes needed |
| UI library | Ant Design (already installed) | Drawer, Tabs, Badge, Modal all ready |
| List rendering | Simple pagination (20 per page) | Adequate for chat, no virtualization needed |
| File viewing | Existing `FileViewer` component | Already built at `src/components/molecules/file-viewer/` |
| Typing | TypeScript strict | Matches app standard |
| Testing | Vitest + React Testing Library | Already configured |
| Styling | SCSS modules (existing pattern) | Consistent with app |
| Icons | `@phosphor-icons/react` (installed) | Send, ThumbsUp, ThumbsDown, etc. |

---

## How to Read These Diagrams

| # | Diagram | What It Explains |
|---|---|---|
| 1 | System Architecture | Overall service topology for the web app |
| 2 | Legacy vs New | Side-by-side comparison of what changes |
| 3 | Component Tree | React component hierarchy to build |
| 4 | Send Message Flow | Step-by-step sequence for sending a message |
| 5 | Feedback Flow | Like/dislike escalation sequence |
| 6 | State Design | What lives where (server vs client) |
| 7 | API Routes | Complete endpoint mapping |
| 8 | File Structure | What files to create and their relationships |
| 9 | Timeline | Gantt chart of implementation phases |
| 10 | Session Lifecycle | State machine for chat open/close/error |
| 11 | Error Handling | Decision tree for all error scenarios |
| 12 | Polling Strategy | When and how to poll for updates |
| 13 | Migration Map | Which legacy file becomes which new file |
| 14 | Ant Design Usage | Which existing UI components to leverage |
| 15 | Feature Flag | Graceful degradation flowchart |
