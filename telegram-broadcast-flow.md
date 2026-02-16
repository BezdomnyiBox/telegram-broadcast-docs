# Telegram Broadcast Flow Logic

## Component Architecture
```mermaid
graph TD;
    A[User] -->|Send Message| B(Controller);
    B --> C[Service];
    C --> D[Database];
    D --> E[APIs];
```

## Broadcast Flow
```mermaid
sequenceDiagram;
    User->>Controller: Initiate Broadcast;
    Controller->>Service: Process Request;
    Service->>Database: Retrieve Eligibility;
    Database-->>Service: Return Response;
    Service->>Controller: Send Approval;
    Controller->>User: Notify Status;
```

## Customer Eligibility
```mermaid
flowchart TD;
    A[Start] --> B{Eligibility Check};
    B -->|Eligible| C[Proceed to Broadcast];
    B -->|Not Eligible| D[End];
```

## Bonus Reminders
```mermaid
timeline;
    2026-02-16 : User signed up;
    2026-02-30 : First Reminder;
    2026-03-01 : Second Reminder;
```

## Cron Execution
```mermaid
cronGraph;
    A[Daily Job] --> B{Check Time};
    B -->|Execute| C[Run Broadcast Flow];
```

## API Endpoints
```mermaid
classDiagram;
    class API {
        +POST /broadcast
        +GET /status
        +POST /reminders
    }
```

## Template Variables
```mermaid
mindmap;
    root((Template Variables))
        child((User Info))
        child((Broadcast Content))
        child((Metadata))
```

## Data Schema
```mermaid
classDiagram;
    class Broadcast {
        +id: String
        +content: String
    }
    class User {
        +id: String
        +name: String
        +eligibility: Boolean
    }
    Broadcast <-- User;
```
