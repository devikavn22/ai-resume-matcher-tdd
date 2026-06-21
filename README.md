# ai-resume-matcher-tdd
Technical Design Document (TDD) for a scalable, multi-tenant AI Resume Screening system featuring asynchronous task processing and structured LLM validation.

# AI Resume Matcher — Technical Design Document (TDD)

This repository contains the Technical Design Document (TDD) for the **AI Resume Screening** feature. It outlines an asynchronous, highly scalable, and multi-tenant ready architecture designed for an AI-powered CRM startup.

---

## 1. System Architecture & End-to-End Data Flow

The diagram below outlines the asynchronous processing pipeline engineered to handle variable LLM latencies without blocking client-facing web servers.

```mermaid
sequenceDiagram
    autonumber
    actor Recruiter as Recruiter Dashboard (React)
    participant API as Backend API Gateway
    participant Queue as Task Queue (Redis / BullMQ)
    participant Worker as Background Worker Service
    participant LLM as LLM Provider Gateway (OpenAI/Anthropic)
    participant DB as Database (PostgreSQL)

    Recruiter->>API: POST /v1/screenings (Payload: Job Description & CV Text)
    activate API
    API->>DB: Initialize Screening Record (Status: PENDING)
    API->>Queue: Enqueue Screening Job (Job ID)
    API-->>Recruiter: HTTP 202 Accepted (Payload: { job_id })
    deactivate API

    Note over Recruiter: UI switches to polling or SSE listener state.

    activate Worker
    Queue->>Worker: Consume Screening Job
    Worker->>LLM: Request Analysis (Structured Schema Enforced)
    activate LLM
    LLM-->>Worker: Return Validated JSON Object
    deactivate LLM
    
    alt Validation Success
        Worker->>DB: Update Record (Status: COMPLETED, Payload: JSON results)
    else Validation / Model Failure
        Worker->>Worker: Trigger Exponential Backoff Retry Policy
        Note over Worker: Max retries exhausted -> Status: FAILED
        Worker->>DB: Update Record (Status: FAILED, Error details)
    end
    deactivate Worker

    loop Poll / Stream Status
        Recruiter->>API: GET /v1/screenings/{job_id}
        API->>DB: Fetch Current Status
        DB-->>API: Return DB State
        API-->>Recruiter: Response (PENDING, COMPLETED, or FAILED)
    end
    Note over Recruiter: UI renders results on dashboard upon COMPLETED status.
