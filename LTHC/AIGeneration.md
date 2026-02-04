# Clinical Note AI Generation - Data Flow

## High-Level Overview

```mermaid
flowchart TB
    subgraph API["API Layer"]
        A[POST /clinical-notes/generate-with-ai]
    end

    subgraph Orchestration["ClinicalNoteAiService.generateNoteStream()"]
        B[Validate Thread & Get Patient]
        C[Gather Conversation History]
        D[Get Auto-Included Note IDs]
        E[Fetch Clinical Notes]
        F[Fetch Attachments from S3]
        G[Build Prompt Variables]
    end

    subgraph AIAssistant["AiAssistantService"]
        H[Prepare Bedrock Request]
        I[Invoke Stream with Retry]
    end

    subgraph Bedrock["BedrockService"]
        J[Build Content Blocks]
        K[Process Attachments]
        L[Send to AWS Bedrock]
    end

    subgraph Response["Response Processing"]
        M[Stream Chunks]
        N[Parse XML Tags]
        O[Match ICD/CPT Codes]
        P[Return Complete Result]
    end

    A --> B --> C --> D --> E --> F --> G
    G --> H --> I --> J --> K --> L
    L --> M --> N --> O --> P
```

## Data Sources Gathered

```mermaid
flowchart LR
    subgraph Sources["Data Sources"]
        MSG[Messages]
        ATT[Attachment Summaries]
        CALL[Call Transcript Summaries]
        CN_USER[User-Selected Clinical Notes]
        CN_AUTO[Auto-Included Clinical Notes]
        FILES[User-Selected Files from S3]
    end

    subgraph ConversationHistory["Thread History Variable"]
        MERGED[Chronologically Merged]
    end

    subgraph ClinicalNotes["Selected Clinical Notes Variable"]
        NOTES[Formatted Markdown]
    end

    subgraph Attachments["Attachments to Bedrock"]
        PROC[Processed Files]
    end

    MSG --> MERGED
    ATT --> MERGED
    CALL --> MERGED

    CN_USER --> NOTES
    CN_AUTO --> NOTES

    FILES --> PROC
```

## Conversation History Gathering

```mermaid
flowchart TB
    subgraph Query["Database Queries"]
        Q1[Query Messages<br/>with date range filter]
        Q2[Query Attachments<br/>where summaryStatus = COMPLETED]
        Q3[Query Call Transcripts<br/>where summaryStatus = COMPLETED]
        Q4[Query ICD Codes<br/>for diagnosis resolution]
    end

    subgraph Merge["Chronological Merge"]
        M1[Sort all items by timestamp]
    end

    subgraph Format["Format Each Item"]
        F1["[timestamp] Sender: content"]
        F2["[timestamp] System: File uploaded<br/>Summary + diagnoses + medications..."]
        F3["[timestamp] System: Video call<br/>Duration + Summary + diagnoses..."]
    end

    Q1 --> M1
    Q2 --> M1
    Q3 --> M1
    Q4 --> M1

    M1 --> F1
    M1 --> F2
    M1 --> F3

    F1 --> OUT[thread_history variable]
    F2 --> OUT
    F3 --> OUT
```

## Auto-Included Clinical Notes Logic

```mermaid
flowchart TB
    START[getAutoIncludedNoteIds]

    subgraph Exclude["IDs to Exclude"]
        E1[Current Note ID]
        E2[User-Selected Note IDs]
    end

    subgraph Query["Single Query"]
        Q1[threadId matches]
        Q2[status = PUBLIC]
        Q3[isStruck = false]
        Q4[id NOT IN excludeIds]
        Q5[Optional: date range filter]
    end

    subgraph Filter["In-Memory Filter"]
        F1[Separate by noteCategory]
        F2[clinicianNotes: take first 2<br/>sorted DESC by createdAt]
        F3[nurseNotes: take ALL]
    end

    START --> Exclude
    Exclude --> Query
    Query --> Filter
    Filter --> RESULT[Return autoIncludedIds]
```

## Attachment Processing Flow

```mermaid
flowchart TB
    subgraph Input["User-Selected Attachments"]
        A1[Attachment IDs from request]
    end

    subgraph Validate["Validation"]
        V1{Attachment belongs<br/>to thread?}
        V2{Supported file type?}
    end

    subgraph Process["Processing"]
        P1[Fetch from S3]
        P2[Build summaryFallback<br/>from AI-generated summary]
        P3{Would require<br/>slow OCR?}
    end

    subgraph Output["Output"]
        O1[AttachmentFile array<br/>with summaryFallback]
        O2[Skip attachment]
    end

    A1 --> V1
    V1 -->|No| SECURITY_ERROR[Security Error]
    V1 -->|Yes| V2
    V2 -->|No| O2
    V2 -->|Yes| P1 --> P2 --> P3
    P3 -->|Yes, no summary| O2
    P3 -->|No| O1
```

## Bedrock Content Block Building

```mermaid
flowchart TB
    subgraph Input["Attachments Array"]
        ATT[AttachmentFile with<br/>bytes, mimeType, summaryFallback]
    end

    subgraph TypeCheck["Type Detection"]
        T1{Is Image?}
        T2{Is Document?}
        T3{Is Video?}
    end

    subgraph ImageProcess["Image Processing"]
        I1{Count < 20?}
        I2[processImage:<br/>resize/compress]
        I3{Size < 3.75MB?}
        I4{Has summaryFallback?}
        I5[Use as image block]
        I6[Use summary as text]
        I7[Skip]
    end

    subgraph DocProcess["Document Processing"]
        D1{Count < 5?}
        D2{Within limits?<br/>< 4.5MB, < 100 pages}
        D3{Has summaryFallback?}
        D4[Send natively]
        D5[Use summary as text]
        D6[Extract via OCR]
        D7[Skip]
    end

    subgraph VideoProcess["Video Processing"]
        V1{Has summaryFallback?}
        V2[Use summary as text<br/>Videos not supported inline]
        V3[Skip with warning]
    end

    ATT --> T1
    T1 -->|Yes| I1
    T1 -->|No| T2

    I1 -->|Yes| I2 --> I3
    I1 -->|No| I4
    I3 -->|Yes| I5
    I3 -->|No| I4
    I4 -->|Yes| I6
    I4 -->|No| I7

    T2 -->|Yes| D1
    T2 -->|No| T3

    D1 -->|Yes| D2
    D1 -->|No| D3
    D2 -->|Yes| D4
    D2 -->|No| D3
    D3 -->|Yes| D5
    D3 -->|No| D6

    T3 -->|Yes| V1
    T3 -->|No| SKIP[Skip unsupported]

    V1 -->|Yes| V2
    V1 -->|No| V3
```

## Bedrock Limits

| Resource | Limit | Fallback Strategy |
|----------|-------|-------------------|
| Images per request | 20 | Use summaryFallback as text, or skip |
| Image max size | 3.75MB | Compress (85% → 70% → 50% → 30%), then fallback |
| Image max dimension | 8000px | Resize proportionally |
| Documents per request | 5 | Use summaryFallback as text, or skip |
| Document max size | 4.5MB | Use summaryFallback, or extract via OCR |
| PDF max pages | 100 | Use summaryFallback, or extract via OCR |
| Videos per request | 1 | Always use summaryFallback (not supported inline) |
| Video max size (S3) | 1GB | N/A (use summary) |
| Video max duration | 30s | N/A (use summary) |

## Document OCR Extraction Flow

```mermaid
flowchart TB
    subgraph Trigger["When Triggered"]
        T1[Document > 4.5MB<br/>AND no summaryFallback]
        T2[PDF > 100 pages<br/>AND no summaryFallback]
    end

    subgraph Strategy["Extraction Strategy"]
        S1{PDF?}
        S2[AWS Textract OCR]
        S3{DOCX/DOC?}
        S4[mammoth library]
        S5{XLSX/XLS?}
        S6[exceljs library]
        S7[Direct UTF-8 decode]
    end

    subgraph Textract["Textract OCR"]
        TX1{Single page?}
        TX2[Sync: DetectDocumentText]
        TX3[Async: StartDocumentTextDetection]
        TX4[Upload to S3 temp]
        TX5[Poll every 15s<br/>max 10 min timeout]
        TX6[Cap at 400 pages]
    end

    subgraph Budget["Text Budget"]
        B1[Total: 600K chars<br/>~150K tokens]
        B2[Per-doc budget =<br/>Total / attachment count]
        B3[Truncate if exceeds budget]
    end

    T1 --> S1
    T2 --> S1

    S1 -->|Yes| TX1
    S1 -->|No| S3
    S3 -->|Yes| S4
    S3 -->|No| S5
    S5 -->|Yes| S6
    S5 -->|No| S7

    TX1 -->|Yes| TX2
    TX1 -->|No| TX3 --> TX4 --> TX5 --> TX6

    TX2 --> Budget
    TX6 --> Budget
    S4 --> Budget
    S6 --> Budget
    S7 --> Budget
```

## Retry Strategy

```mermaid
flowchart TB
    subgraph Config["Configuration"]
        C1[Max Retries: 3]
        C2[Delays: 1s, 2s, 4s<br/>exponential backoff]
    end

    subgraph Errors["Error Types"]
        E1{Throttling?<br/>ThrottlingException<br/>429}
        E2{Timeout?<br/>TimeoutError}
        E3{Server Error?<br/>502, 503}
        E4{Validation Error?<br/>ValidationException}
    end

    subgraph Action["Action"]
        A1[Retry with backoff]
        A2[Fail immediately<br/>no retry]
    end

    E1 -->|Yes| A1
    E2 -->|Yes| A1
    E3 -->|Yes| A1
    E4 -->|Yes| A2
```

## Response Streaming & Parsing

```mermaid
flowchart TB
    subgraph Stream["Stream from Bedrock"]
        S1[Chunk events with text]
        S2[Metadata event with usage]
    end

    subgraph Buffer["InlineTagStreamBuffer"]
        B1[Buffer incoming chunks]
        B2[Detect XML tags]
        B3[Extract title from<br/>title tags]
        B4[Extract ICD-10 codes from<br/>icd-10 tags]
        B5[Extract CPT codes from<br/>cpt tags]
        B6[Strip tags from output]
    end

    subgraph Match["Code Matching"]
        M1[Query ICD codes table<br/>by code string]
        M2[Query CPT codes table<br/>by code string]
    end

    subgraph Result["Final Result"]
        R1[title: string]
        R2[noteText: string<br/>tags stripped]
        R3[icdCodeIds: number array]
        R4[cptCodeIds: number array]
        R5[usage: token counts]
    end

    S1 --> B1 --> B2
    B2 --> B3
    B2 --> B4
    B2 --> B5
    B2 --> B6

    B4 --> M1
    B5 --> M2

    B3 --> R1
    B6 --> R2
    M1 --> R3
    M2 --> R4
    S2 --> R5
```

## Prompt Variables Summary

```mermaid
flowchart LR
    subgraph Patient["Patient Context"]
        P1[patient_name]
        P2[patient_dob]
        P3[patient_age]
        P4[patient_facility]
    end

    subgraph User["User Context"]
        U1[user_name]
        U2[user_role]
        U3[today_date]
    end

    subgraph Content["Content"]
        C1[thread_history<br/>Messages + Summaries merged]
        C2[selected_clinical_notes<br/>User + Auto-included]
        C3[user_instructions<br/>Highest priority guidance]
    end

    subgraph Edit["Edit Mode Optional"]
        E1[current_draft_content]
        E2[current_title]
    end

    subgraph Format["Format"]
        F1[output_format<br/>XML tag instructions]
    end

    Patient --> PROMPT[AWS Bedrock<br/>Managed Prompt]
    User --> PROMPT
    Content --> PROMPT
    Edit --> PROMPT
    Format --> PROMPT
```

## Fallback Strategy Summary

| Scenario | Primary Strategy | Fallback | Last Resort |
|----------|------------------|----------|-------------|
| Image > 3.75MB | Compress progressively | Use summaryFallback | Skip |
| Image count > 20 | - | Use summaryFallback | Skip |
| Document > 4.5MB | Use summaryFallback | OCR extraction | Skip |
| Document > 100 pages | Use summaryFallback | OCR extraction | Skip |
| Document count > 5 | - | Use summaryFallback | Skip |
| Video attachment | Use summaryFallback | - | Skip with warning |
| Unsupported type | - | - | Skip |
| OCR timeout (10min) | - | - | Fail gracefully |
| S3 fetch fails | - | - | Skip that attachment |
| Bedrock throttling | Retry 3x with backoff | - | Return error |
| Bedrock validation | - | - | Return error immediately |
