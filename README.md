# Chat-bot-digram
# Chatbot Architecture Diagrams - e2-student-portal

> Visual diagrams explaining how the chatbot system works end-to-end.
> Render these with any Mermaid-compatible viewer (GitHub, VS Code extension, Mermaid Live Editor).

---

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph Electron["🖥️ Electron App (e2-student-portal)"]
        direction TB
        UI["React UI<br/>(Chat Components)"]
        DS["DataStore<br/>(data-store.js)"]
        DB[(RxDB<br/>IndexedDB)]
        CW["Chat Worker<br/>(chat.worker.js)<br/>⏱️ 5s loop"]
        SW["Sync Worker<br/>(sync.worker.js)<br/>⏱️ interval"]
        PU[(PendingUpload<br/>Queue)]

        UI -->|"read/write"| DS
        DS -->|"CRUD"| DB
        DS -->|"queue offline actions"| PU
        CW -->|"read queue, write messages"| DB
        SW -->|"read pending"| PU
        SW -->|"bulk upsert"| DB
    end

    subgraph Gateway["🔀 e2-sp-service (port 5017/8000)"]
        GW["FeathersJS Gateway<br/>/chat, /ai-question-suggestion<br/>/qa-feedback, /alerts"]
    end

    subgraph Chatbot["🤖 e2-chatbot (port 5020)"]
        CS["Chat Service<br/>(chat.service.js)"]
        MONGO[(MongoDB<br/>Chat History)]
        LOG["Chat Logs"]
        CS --> MONGO
        CS --> LOG
    end

    subgraph AI["🧠 AI Service"]
        AAI["smart-learning-companion<br/>-aai-service<br/>(RAG/LLM Engine)"]
    end

    DS -->|"POST /e2-chatbot/chat<br/>(real-time query)"| GW
    SW -->|"GET/POST/PATCH<br/>(sync uploads/downloads)"| GW
    GW -->|"proxy"| CS
    CS -->|"POST /chat<br/>POST /library/search<br/>GET /chat/complete"| AAI
```

---

## 2. Component Hierarchy

```mermaid
graph TD
    Layout["Layout.js<br/>(Global Wrapper)"]
    Layout --> Avatar["ChatBotAvatar<br/>🟢 Floating Button"]
    Layout --> Container["ChatBotContainer<br/>📦 Popup Modal"]

    Avatar --> Badge["Unread Badge<br/>(useObservable)"]
    Avatar --> Bubble["AvatarGreetings<br/>💬 Bubble Tooltip"]
    Avatar --> AlertBubble["AvatarAlerts<br/>⚠️ Alert Bubble"]

    Container --> Tabs["Tab Navigation"]
    Tabs --> ChatTab["Chat Tab"]
    Tabs --> NotifTab["Notifications Tab"]

    ChatTab --> Dialog["ChatDialogContainer<br/>📝 Main Logic"]
    Dialog --> MsgList["Message List<br/>(infinite scroll)"]
    Dialog --> Input["MessageInput<br/>⌨️ + Suggestions"]

    MsgList --> Msg1["ChatMessage<br/>(Bot - text)"]
    MsgList --> Msg2["ChatMessage<br/>(Bot - file card)"]
    MsgList --> Msg3["ChatMessage<br/>(User - query)"]
    MsgList --> Msg4["ChatMessage<br/>(Bot - escalation)"]

    Msg1 --> Actions["👍 👎 Buttons"]
    Msg4 --> InstrPicker["InstructorPopup<br/>👨‍🏫 Picker"]
    Msg4 --> SmartLib["Smart Library<br/>📚 Button"]

    Input --> Suggestions["Question Suggestions<br/>Dropdown"]

    NotifTab --> Alerts["ChatBotAlerts"]
    Alerts --> AssignCard["Assignment Card"]
    Alerts --> RecoCard["Recommendation Card"]
    Alerts --> QACard["QA Feedback Card"]
    Alerts --> GameCard["Game Alert Card"]
    Alerts --> AICard["AI Intervention Card"]
```

---

## 3. Message Send Flow (Online)

```mermaid
sequenceDiagram
    participant User
    participant UI as ChatDialogContainer
    participant DS as DataStore
    participant RxDB as RxDB (Local)
    participant SP as e2-sp-service
    participant CB as e2-chatbot
    participant AI as AI Service (RAG/LLM)

    User->>UI: Types message & clicks Send
    UI->>UI: closeRunningConversationFlow()
    UI->>DS: handleSaveQnaRequest(query)
    DS->>RxDB: createQuestionAndAnswer({type:'User'})
    RxDB-->>UI: Reactive update → shows user message

    UI->>UI: setLoading(true) → typing indicator
    UI->>DS: sendChatMessage({query, feedbackState, typeOfChat:'chat'})
    DS->>SP: POST /e2-chatbot/chat
    SP->>CB: POST /chat (proxied)
    CB->>AI: POST /chat (with query, userId, courseId)
    AI-->>CB: {answer, relatedQuestions, content}
    CB->>CB: Save to MongoDB + Log
    CB-->>SP: {data, response}
    SP-->>DS: {data, response}

    DS-->>UI: Response returned
    UI->>UI: setLoading(false)
    UI->>DS: handleSaveQnaResponse(response)
    DS->>RxDB: createQuestionAndAnswer({type:'Bot', message, relatedQuestions})
    RxDB-->>UI: Reactive update → shows bot message

    UI->>DS: sendChatMessage({data}) [notify backend of save]
    DS->>SP: POST /e2-chatbot/chat (no typeOfChat)
```

---

## 4. Like / Dislike Escalation Flow

```mermaid
sequenceDiagram
    participant User
    participant UI as ChatMessage
    participant DS as DataStore
    participant RxDB as RxDB
    participant SP as e2-sp-service
    participant CB as e2-chatbot
    participant AI as AI Service

    User->>UI: Clicks 👎 (Dislike)
    UI->>DS: updateQnaFeedback({feedbackGiven:'dislike'})
    DS->>RxDB: Patch message locally
    DS->>RxDB: createPendingUpload('update-qna-feedback')
    Note over RxDB: Queued for sync

    UI->>DS: sendChatMessage({feedbackState:{feedbacks:[{value:'dislike'}]}})
    DS->>SP: POST /e2-chatbot/chat
    SP->>CB: POST /chat

    alt 1st dislike
        CB->>AI: POST /chat (re-query with feedback)
        AI-->>CB: New answer + relatedQuestions
        CB-->>SP: {actionType: NO_ACTION, relatedQuestions: [...]}
        SP-->>UI: Shows new answer + related question chips
    else 2nd dislike
        CB-->>SP: {actionType: ASK_EXTERNAL_RESOURCE}
        SP-->>UI: Shows "Ask Instructor" + "Ask Smart Library" buttons
    else 3rd dislike
        CB-->>SP: {actionType: ASK_INSTRUCTOR}
        SP-->>UI: Shows "Ask Your Instructor" button only
    end
```

---

## 5. Offline Sync Architecture

```mermaid
graph TB
    subgraph Online["🟢 Online Mode"]
        direction LR
        A1["User sends message"] --> A2["Direct API call"]
        A2 --> A3["Response saved to RxDB"]
    end

    subgraph Offline["🔴 Offline Mode"]
        direction LR
        B1["User likes message"] --> B2["Save to RxDB locally"]
        B2 --> B3["Create PendingUpload"]
    end

    subgraph SyncProcess["🔄 Sync Worker (when back online)"]
        direction TB
        C1["Check connectivity<br/>GET /e2-irp/healthcheck"]
        C2{"Online?"}
        C3["syncPendingUploads()"]
        C4["syncDownloads()"]

        C1 --> C2
        C2 -->|Yes| C3
        C3 --> C4
        C2 -->|No| C5["Retry later"]
    end

    subgraph Upload["📤 Upload Pending"]
        direction TB
        D1["Read PendingUpload queue"]
        D2["Group by entity type"]
        D3["chatbot-chat → POST /chat?bulk=true"]
        D4["update-qna-feedback → PATCH /chat/:id"]
        D5["bulk-patch-chat → POST /chat-bulk-patch"]
        D6["Delete from queue on success"]

        D1 --> D2
        D2 --> D3
        D2 --> D4
        D2 --> D5
        D3 --> D6
        D4 --> D6
        D5 --> D6
    end

    subgraph Download["📥 Download Data"]
        direction TB
        E1["GET /chat?userId&courseId<br/>(paginated, up to 500)"]
        E2["Generate local messages from:<br/>• Assignments<br/>• Recommendations<br/>• QA Feedbacks<br/>• Games"]
        E3["bulkUpsertQna() → RxDB"]

        E1 --> E2
        E2 --> E3
    end

    SyncProcess --> Upload
    SyncProcess --> Download
```

---

## 6. Proactive Conversation System (Chat Worker)

```mermaid
stateDiagram-v2
    [*] --> QUEUED: External trigger<br/>(sync discovers assignment,<br/>recommendation, etc.)

    QUEUED --> ACTIVE: chat.worker.js picks up<br/>(every 5s loop)
    
    state ACTIVE {
        [*] --> ShowBubble: updateChatbubbleMsg()
        ShowBubble --> AddMessages: addConversationToChatBot()
        AddMessages --> WaitForUser: Start closureDelay timer (10s)
        
        WaitForUser --> UserActed: User clicks action
        WaitForUser --> TimedOut: 10s elapsed, no interaction
        
        state hasTwoStages <<choice>>
        TimedOut --> hasTwoStages
        hasTwoStages --> IntermediateClose: hasTwoStages = true
        hasTwoStages --> FinalClose: hasTwoStages = false
        
        IntermediateClose --> WaitAgain: Show closureMessage
        WaitAgain --> FinalClose: Another 10s elapsed
    }

    ACTIVE --> DONE: User acted → actionClosureMessage (2s)
    FinalClose --> DONE: finalClosureMessage shown
    
    DONE --> [*]
```

---

## 7. Data Model Relationships

```mermaid
erDiagram
    QuestionAndAnswer {
        string id PK
        string clientId UK
        string sessionId
        string questionId
        string answerId
        string courseId FK
        string userId FK
        string date
        enum type "Bot | User"
        string source
        boolean isGreeting
        string readAt
        boolean hideFromChat
        enum feedbackGiven "like | dislike | neutral"
        object feedbackState
        enum actionType "NO_ACTION | ASK_INSTRUCTOR | SHOW_INSTRUCTORS | ASK_EXTERNAL_RESOURCE"
        string query
        array relatedQuestions
        object message
    }

    ChatbotBubbleMsg {
        string id PK
        string clientId UK
        string message
        enum type "closure | greeting | conversation"
        array locales
    }

    ChatbotConversationQueue {
        string id PK
        string type
        enum status "QUEUED | ACTIVE | DONE"
        string courseId FK
        string groupId
        object content
    }

    PendingUpload {
        string id PK
        string entity
        object data
    }

    QuestionAndAnswer ||--o{ PendingUpload : "queues feedback updates"
    ChatbotConversationQueue ||--o{ QuestionAndAnswer : "generates messages"
    ChatbotConversationQueue ||--o{ ChatbotBubbleMsg : "triggers bubbles"
```

---

## 8. Service Communication Map

```mermaid
graph LR
    subgraph Frontend["Frontend (Electron)"]
        APP["e2-student-portal<br/>React App"]
    end

    subgraph BFF["Backend-for-Frontend"]
        SP["e2-sp-service<br/>:5017/:8000"]
    end

    subgraph Microservices["Backend Microservices"]
        CB["e2-chatbot<br/>:5020"]
        COURSE["e2-course"]
        IRP["e2-irp"]
        CONTENT["e2-content"]
    end

    subgraph External["External / AI"]
        AI["smart-learning-companion<br/>-aai-service"]
        KAFKA["Kafka<br/>(Events)"]
    end

    APP -->|"/e2-chatbot/chat<br/>/e2-chatbot/ai-question-suggestion<br/>/e2-chatbot/healthcheck"| SP
    APP -->|"/e2-course/qa-feedback"| SP
    APP -->|"/e2-irp/healthcheck"| SP
    APP -->|"/e2-content/..."| SP

    SP -->|"CHATBOT_SERVICE"| CB
    SP -->|"COURSE_SERVICE"| COURSE
    SP -->|"IRP_SERVICE"| IRP
    SP -->|"CONTENT_SERVICE"| CONTENT

    CB -->|"POST /chat"| AI
    CB -->|"GET /chat/complete"| AI
    CB -->|"POST /library/search"| AI
    CB -->|"Publish chatEvents"| KAFKA
```

---

## 9. User Journey: Complete Chat Session

```mermaid
journey
    title Student Chat Session Journey
    section Open Chat
      Click avatar: 5: Student
      See unread badge clear: 3: System
      Greeting messages appear: 4: System
    section Ask Question
      Type question: 5: Student
      See autocomplete suggestions: 4: System
      Click send: 5: Student
      See typing indicator: 3: System
      Receive AI answer: 5: System
    section Dislike Answer
      Click thumbs down: 5: Student
      See new answer + related questions: 4: System
      Click related question: 5: Student
      Receive better answer: 5: System
    section Escalate
      Dislike again (2nd time): 5: Student
      See "Ask Instructor" + "Smart Library": 4: System
      Click "Ask Instructor": 5: Student
      Select instructor from list: 5: Student
      Confirmation message: 4: System
    section Close
      Close chat window: 5: Student
      Session closure sent to backend: 3: System
```

---

## 10. Feature Toggle & Visibility Logic

```mermaid
flowchart TD
    Start["App Loads (Layout.js)"] --> Check{"Poll every 60s:<br/>GET /e2-chatbot/healthcheck"}
    
    Check -->|"enableChatbot: true"| ViewCheck{"viewMode === StudentMode?"}
    Check -->|"enableChatbot: false<br/>or network error"| Hide["Hide Chatbot Entirely"]
    
    ViewCheck -->|Yes| PathCheck{"Current path is<br/>exam or practical?"}
    ViewCheck -->|No| Hide
    
    PathCheck -->|No| ShowAvatar["✅ Show ChatBotAvatar"]
    PathCheck -->|Yes| Hide
    
    ShowAvatar --> UserClick{"User clicks avatar?"}
    UserClick -->|Yes| ShowContainer["Show ChatBotContainer<br/>(popup modal)"]
    UserClick -->|No| ShowAvatar

    ShowContainer --> TabSelect{"Which tab?"}
    TabSelect -->|Chat| ShowDialog["ChatDialogContainer"]
    TabSelect -->|Notifications| ShowAlerts["ChatBotAlerts"]
```

---

## 11. API Request Flow (with Proxy)

```mermaid
flowchart LR
    subgraph Browser["Electron Renderer"]
        FE["React Component"]
    end

    subgraph Proxy["setupProxy.js<br/>(dev only)"]
        P1["/e2-chatbot → localhost:8000"]
    end

    subgraph SPService["e2-sp-service (:8000)"]
        R1["/chat → ChatService"]
        R2["/ai-question-suggestion"]
        R3["/qa-feedback"]
        R4["/alerts"]
    end

    subgraph ChatbotSvc["e2-chatbot (:5020)"]
        C1["chat.service.js"]
        C2["ai-questions-suggestion.service.js"]
    end

    subgraph AISvc["AI Service"]
        AI1["POST /chat"]
        AI2["GET /chat/complete"]
        AI3["POST /library/search"]
    end

    FE -->|"axios baseURL=localhost:8000"| SPService
    FE -.->|"dev proxy"| Proxy
    Proxy -.-> SPService

    R1 -->|"CHATBOT_SERVICE.post('chat')"| C1
    R2 -->|"CHATBOT_SERVICE.get('ai-question-suggestion')"| C2

    C1 -->|"Normal query"| AI1
    C2 -->|"Suggestions"| AI2
    C1 -->|"Smart Library"| AI3
```

---

## Quick Reference: Port Map

| Service | Port | Role |
|---|---|---|
| e2-student-portal (React) | 3000 | Frontend UI |
| e2-sp-service | 5017 (internal), 8000 (exposed) | BFF Gateway |
| e2-chatbot | 5020 | Chatbot orchestrator |
| smart-learning-companion-aai-service | remote | AI/RAG engine |
| MongoDB | 27017 | Chat persistence |
| e2-course | varies | Course data |
| e2-irp | 5000 | Health/connectivity check |

---

## How to Read These Diagrams

- **Diagram 1** — Shows the overall system architecture and data flow between all layers
- **Diagram 2** — Shows how React components are nested and composed
- **Diagram 3** — Step-by-step message sending sequence
- **Diagram 4** — How the dislike escalation works over time
- **Diagram 5** — How offline sync uploads/downloads work
- **Diagram 6** — State machine for proactive conversation lifecycle
- **Diagram 7** — Database entity relationships
- **Diagram 8** — Service-to-service communication topology
- **Diagram 9** — User experience journey through a chat session
- **Diagram 10** — Decision tree for showing/hiding the chatbot
- **Diagram 11** — How HTTP requests flow through proxies to backend services
