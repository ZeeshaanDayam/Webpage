# Medhaya Technical Features Deep Dive
## End-to-End Feature Explanations for Interview

---

## Table of Contents
1. [Authentication & Login Flow](#1-authentication--login-flow)
2. [Cookie Storage System](#2-cookie-storage-system)
3. [Real-Time Transcription with WebSockets](#3-real-time-transcription-with-websockets)
4. [AI Diagnostic Service (Aila)](#4-ai-diagnostic-service-aila)
5. [Document Generation System](#5-document-generation-system)
6. [Session Management](#6-session-management)
7. [Route Protection (Middleware)](#7-route-protection-middleware)
8. [API Architecture](#8-api-architecture)

---

## 1. Authentication & Login Flow

### Overview
The authentication system supports two user types: **Doctors** and **Patients**. Each has a separate login flow with secure credential handling.

### Flow Diagram
```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Login Form    │────▶│  authService.ts │────▶│  Backend API    │
│ (email/password)│     │  (encode creds) │     │ (/api/auth/...) │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                │                        │
                                ▼                        ▼
                        ┌─────────────────┐     ┌─────────────────┐
                        │  Set Cookies    │◀────│  Return Session │
                        │  (cookieUtils)  │     │  Data + Tokens  │
                        └─────────────────┘     └─────────────────┘
                                │
                                ▼
                        ┌─────────────────┐
                        │ Redirect to     │
                        │ Dashboard       │
                        └─────────────────┘
```

### Key Code Locations
| Step | File | Function |
|------|------|----------|
| Login Form | `src/app/doctor-login/page.tsx` | Form submission |
| API Call | `src/api/authService.ts` | `requestDoctorLoginOtp()` |
| Cookie Storage | `src/utils/cookieUtils.ts` | `setCookie()` |
| Route Protection | `src/middleware.ts` | `middleware()` |

### Doctor Login Process (authService.ts)

```typescript
// Step 1: Encode credentials for secure transmission
const originalPayload = { email, password };
const jsonString = JSON.stringify(originalPayload);
const encodedPayloadString = btoa(jsonString);  // Base64 encode

// Step 2: Send to backend
const response = await fetchWithAuth('/api/auth/patient/login', {
  method: 'POST',
  body: JSON.stringify({ encodedData: encodedPayloadString }),
});

// Step 3: Store session in cookies
setCookie(DOCTOR_AUTH_COOKIE, 'true');
setCookie(DOCTOR_UUID_COOKIE, doctorData.doctorid);
setCookie(DOCTOR_MID_COOKIE, doctorData.medhayaId);
setCookie(DOCTOR_DISPLAY_NAME_COOKIE, `${firstname} ${lastname}`);
setCookie(DOCTOR_SESSION_ID_COOKIE, sessionId);
```

### What to Explain in Interview
> "When a doctor logs in, the credentials are Base64 encoded on the client side before being sent to the backend. This isn't encryption - it's just obfuscation to prevent casual inspection. The real security comes from HTTPS in production. Once authenticated, the backend returns a session ID and doctor profile data, which I store in HTTP cookies for subsequent requests. The cookies are set with security flags like `sameSite: 'strict'` in production to prevent CSRF attacks."

---

## 2. Cookie Storage System

### Overview
Cookies are used to persist authentication state and user profile information across page reloads and browser sessions.

### Cookie Architecture
```
┌──────────────────────────────────────────────────────────────────┐
│                        COOKIE LAYER                               │
├──────────────────────────────────────────────────────────────────┤
│  Authentication Cookies          │  Profile Cookies                │
│  ─────────────────────────────   │  ─────────────────────────────  │
│  • medhaya-doctor-auth (bool)    │  • medhaya-doctor-uuid          │
│  • medhaya-patient-auth (bool)   │  • medhaya-doctor-display-name  │
│  • medhaya-csrf-token            │  • medhaya-doctor-clientid      │
│                                  │  • medhaya-doctor-email         │
│                                  │  • medhaya-doctor-organization  │
└──────────────────────────────────────────────────────────────────┘
```

### Cookie Constants (cookieUtils.ts)
```typescript
// Authentication state cookies
export const PATIENT_AUTH_COOKIE = 'medhaya-patient-auth';
export const DOCTOR_AUTH_COOKIE = 'medhaya-doctor-auth';
export const CSRF_TOKEN_COOKIE = 'medhaya-csrf-token';

// Doctor profile cookies
export const DOCTOR_UUID_COOKIE = 'medhaya-doctor-uuid';      // UUID from backend
export const DOCTOR_MID_COOKIE = 'medhaya-doctor-clientid';   // MedhayaId (MH00086)
export const DOCTOR_DISPLAY_NAME_COOKIE = 'medhaya-doctor-display-name';
export const DOCTOR_SESSION_ID_COOKIE = 'medhaya-doctor-session-id';
```

### Security Configuration
```typescript
const getCookieOptions = () => {
  const isProduction = process.env.NODE_ENV === 'production';

  return {
    expires: 7,  // 7 days
    secure: isProduction,                    // HTTPS only in production
    sameSite: isProduction ? 'strict' : 'lax', // Prevent CSRF
    path: '/',
    // Note: httpOnly must be set server-side for true security
  };
};
```

### Key Functions
| Function | Purpose |
|----------|---------|
| `setCookie(name, value)` | Store cookie with security options |
| `getCookie(name)` | Retrieve cookie value |
| `removeCookie(name)` | Delete cookie (for logout) |
| `isAuthenticated(type)` | Check if user is logged in |
| `setCSRFToken()` | Generate and store CSRF protection token |

### What to Explain in Interview
> "I use cookies instead of localStorage for authentication because cookies can be sent automatically with every HTTP request and have built-in security features. Each cookie is set with `sameSite: strict` in production to prevent cross-site request forgery. I also generate a CSRF token that's stored in a cookie and sent as a header with every API request. The backend validates that these match."

---

## 3. Real-Time Transcription with WebSockets

### Overview
The transcription feature uses WebSockets for bidirectional, real-time communication. Audio is captured from the browser, streamed to the backend, and transcription results are pushed back in real-time.

### Architecture
```
┌─────────────────────────────────────────────────────────────────────────┐
│                           BROWSER                                        │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────┐   │
│  │ Microphone   │───▶│ AudioContext │───▶│ AudioWorkletProcessor    │   │
│  │ (getUserMedia)│    │ (16kHz)      │    │ (process audio chunks)   │   │
│  └──────────────┘    └──────────────┘    └──────────────────────────┘   │
│                                                      │                   │
│                                                      ▼                   │
│                                          ┌──────────────────────┐       │
│                                          │ Convert to PCM Int16 │       │
│                                          └──────────────────────┘       │
│                                                      │                   │
└──────────────────────────────────────────────────────│───────────────────┘
                                                       │
                                          WebSocket (wss://)
                                               Binary Data
                                                       │
┌──────────────────────────────────────────────────────│───────────────────┐
│                          BACKEND                     │                   │
│                                                      ▼                   │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐   │
│  │ WebSocket Server │───▶│ Speech-to-Text   │───▶│ Send JSON Result │   │
│  │ (receive audio)  │    │ (ASR engine)     │    │ {transcript: ".."}│   │
│  └──────────────────┘    └──────────────────┘    └──────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### WebSocket Connection Flow (transcriptionService.ts)

#### Step 1: Initialize WebSocket
```typescript
private initializeWebSocket(options: TranscriptionOptions): Promise<void> {
  const newSocket = new WebSocket(this.WEBSOCKET_URL);
  newSocket.binaryType = "arraybuffer";  // For binary audio data
  
  newSocket.onopen = () => {
    // Send session initialization with doctor ID and language
    const initialMessage = JSON.stringify({ 
      type: "session_init", 
      medhayaId: getCookie(DOCTOR_MID_COOKIE),
      languageCode: options.languageCode  // e.g., 'en-GB', 'hi-IN'
    });
    newSocket.send(initialMessage);
  };
}
```

#### Step 2: Capture Audio with AudioWorklet
```typescript
private async initializeAudioWorklet(): Promise<void> {
  // Create inline worklet processor
  const workletCode = `
    class TranscriptionWorkletProcessor extends AudioWorkletProcessor {
      process(inputs, outputs) {
        const input = inputs[0];
        if (input && input.length > 0) {
          const channelData = input[0];
          // Send audio samples to main thread
          this.port.postMessage({
            type: 'audioData',
            data: Array.from(channelData)
          });
        }
        return true;  // Keep processor running
      }
    }
    registerProcessor('transcription-worklet', TranscriptionWorkletProcessor);
  `;
  
  await this.audioContext.audioWorklet.addModule(workletUrl);
  this.workletNode = new AudioWorkletNode(this.audioContext, 'transcription-worklet');
}
```

#### Step 3: Convert and Send Audio
```typescript
private handleAudioData = (inputData: number[]): void => {
  // Convert float samples (-1 to 1) to 16-bit PCM integers
  const pcmData = new Int16Array(inputData.length);
  for (let i = 0; i < inputData.length; i++) {
    const s = Math.max(-1, Math.min(1, inputData[i])); 
    pcmData[i] = s < 0 ? s * 0x8000 : s * 0x7FFF; 
  }
  
  // Send binary data over WebSocket
  this.socket.send(pcmData.buffer);
};
```

#### Step 4: Receive Transcription Results
```typescript
newSocket.onmessage = (event: MessageEvent) => {
  const messageData = JSON.parse(event.data);
  
  switch (messageData.type) {
    case 'server_ready':
      // Connection established, start sending audio
      break;
    case 'transcription_result':
      // Pass transcript to callback
      options.onTranscriptionResult(messageData.transcript);
      break;
    case 'error':
      options.onError(new Error(messageData.message));
      break;
  }
};
```

### Message Protocol
| Direction | Type | Payload | Purpose |
|-----------|------|---------|---------|
| Client → Server | `session_init` | `{medhayaId, languageCode}` | Initialize session |
| Client → Server | Binary | ArrayBuffer (Int16) | Audio samples |
| Server → Client | `server_ready` | `{}` | Confirm session started |
| Server → Client | `transcription_result` | `{transcript: string}` | Real-time text |
| Server → Client | `error` | `{message: string}` | Error notification |

### What to Explain in Interview
> "The transcription uses WebSockets because we need real-time bidirectional communication. REST APIs would be too slow - the user would have to wait for the entire audio file to upload before getting any text. With WebSockets, I stream audio chunks continuously and receive partial transcriptions as they're processed.

> I use the Web Audio API with an AudioWorklet for audio processing. This runs in a separate thread so it doesn't block the UI. The audio is captured at 16kHz (optimal for speech recognition), converted to 16-bit PCM format, and sent as binary data over the WebSocket.

> One challenge I solved was handling connection drops. Mobile browsers especially can disconnect when the screen locks. I implemented reconnection logic with exponential backoff - first retry after 1 second, then 2, then 4, up to a maximum of 3 attempts."

---

## 4. AI Diagnostic Service (Aila)

### Overview
Aila is the AI assistant that analyzes consultation transcripts and provides diagnostic suggestions with confidence scores.

### Architecture
```
┌──────────────────────────────────────────────────────────────────┐
│                    FRONTEND                                       │
│  ┌─────────────────┐    ┌────────────────────────────────────┐   │
│  │ Consultation    │───▶│ AilaDiagnosticSidebar.tsx          │   │
│  │ Transcript      │    │ - Manages state & caching          │   │
│  │                 │    │ - Displays conditions + confidence │   │
│  └─────────────────┘    └────────────────────────────────────┘   │
│                                        │                          │
└────────────────────────────────────────│──────────────────────────┘
                                         │
                              POST /api/diagnosis/aila
                              {userMessage, doctorid, medhayaId}
                                         │
┌────────────────────────────────────────│──────────────────────────┐
│                    BACKEND             │                          │
│                                        ▼                          │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │                 RAG Pipeline                              │    │
│  │  ┌──────────┐   ┌──────────────┐   ┌─────────────────┐   │    │
│  │  │ Extract  │──▶│ Vector DB    │──▶│ LLM Generation  │   │    │
│  │  │ Symptoms │   │ (NHS/NICE)   │   │ (Summarize)     │   │    │
│  │  └──────────┘   └──────────────┘   └─────────────────┘   │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                        │                          │
└────────────────────────────────────────│──────────────────────────┘
                                         │
                                         ▼
                         ┌───────────────────────────────┐
                         │ DiagnosticResponse            │
                         │ {                             │
                         │   summary: "...",             │
                         │   possibleConditions: [...],  │
                         │   recommendedActions: [...]   │
                         │ }                             │
                         └───────────────────────────────┘
```

### API Service (ailaDiagnosticService.ts)
```typescript
class AilaDiagnosticService {
  private baseUrl = '/api/diagnosis/';

  async getDiagnosticSuggestions(
    userMessage: string,
    doctorId?: string,
    medhayaId?: string
  ): Promise<DiagnosticResponse> {
    
    const requestBody: DiagnosticRequest = {
      userMessage,  // The consultation transcript
      doctorid: doctorId || getCookie(DOCTOR_MID_COOKIE),
      medhayaId
    };

    const response = await fetch(`${this.baseUrl}aila`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(requestBody),
    });

    const data = await response.json();
    
    return {
      summary: data.summary || 'No summary available',
      possibleConditions: data.possibleConditions || [],
      recommendedActions: data.recommendedActions || []
    };
  }
}
```

### Response Types
```typescript
interface PossibleCondition {
  name: string;           // e.g., "Type 2 Diabetes"
  confidence: number;      // 0.0 - 1.0 (0% - 100%)
  description: string;     // Brief explanation
  citation: string;        // Source (NHS/NICE guideline)
}

interface DiagnosticResponse {
  summary: string;                    // Overview of findings
  possibleConditions: PossibleCondition[];  // Ranked by confidence
  recommendedActions: string[];       // Suggested next steps
}
```

### Smart Caching (AilaDiagnosticSidebar.tsx)
```typescript
// Cache key combines context, session, and document
const cachedData = ailaDiagnosticState.getValidDiagnosticData(
  context,     // 'consultation' or 'document'
  userMessage, // The transcript text
  sessionId,   // Current session
  documentId   // Specific document being viewed
);

// Only fetch if no valid cache exists
if (!cachedData) {
  await fetchDiagnosticSuggestions();
}
```

### What to Explain in Interview
> "Aila uses a RAG (Retrieval Augmented Generation) architecture. When I send the consultation transcript, the backend first extracts relevant symptoms and medical terms. It then searches a vector database containing NHS and NICE clinical guidelines to find relevant context. Finally, an LLM generates a summary and ranks possible conditions with confidence scores.

> On the frontend, I implemented smart caching so we don't re-fetch the same analysis when users toggle the sidebar open and close. The cache is keyed by session ID and document ID, so different consultations get fresh analysis."

---

## 5. Document Generation System

### Overview
Automatically generates medical documents (referral letters, prescriptions, SOAP notes) from consultation transcripts using templates.

### Flow
```
┌────────────────┐    ┌────────────────┐    ┌────────────────┐
│ Consultation   │───▶│ Select Template│───▶│ Generate Doc   │
│ Transcript     │    │ (saved/default)│    │ POST /generate │
└────────────────┘    └────────────────┘    └────────────────┘
                                                    │
                                                    ▼
                                            ┌────────────────┐
                                            │ LLM Fills      │
                                            │ Template       │
                                            └────────────────┘
                                                    │
                                                    ▼
                                            ┌────────────────┐
                                            │ Document Editor│
                                            │ (Edit/Export)  │
                                            └────────────────┘
```

### Document Service API (documentService.ts)
```typescript
// Generate document from template + transcript
export const generateDocument = async (
  doctorId: string,
  templateId: string,
  consultationId: string,
  patientInfo: PatientInfo,
  transcript: string,
  documentType: string,         // 'referralLetter', 'soapNotes', etc.
  isDefaultTemplate: boolean,   // System template vs custom
  medhayaId: string
) => {
  const response = await fetchWithAuth('/api/documents/generate', {
    method: 'POST',
    body: JSON.stringify({
      doctorId, templateId, consultationId, 
      patientInfo, transcript, documentType,
      isDefaultTemplate, medhayaId
    }),
  }, 'doctor');
  
  return response;  // Generated document content
};

// Save custom template
export const saveDocumentTemplate = async (
  doctorId: string,
  templateName: string,
  template: string,      // Template content with placeholders
  doctype: string        // Document type category
) => {
  return await fetchWithAuth('/api/documents/templates/', {
    method: 'PUT',
    body: JSON.stringify({ template, doctorId, docType, templateName }),
  }, 'doctor');
};
```

### SNOMED Integration
```typescript
// Get standardized medical codes for terms
export const getSnowMedCodesForTerm = async (term: string) => {
  return await fetchWithAuth('/api/documents/snomed', {
    method: 'POST',
    body: JSON.stringify({ data: term }),
  }, 'doctor');
};
```

### What to Explain in Interview
> "After a consultation, doctors can generate standardized medical documents. They select a template - either a system default or one they've customized - and the backend uses the transcript plus patient info to fill in the template. The system is integrated with SNOMED CT codes, so diagnoses and procedures use standardized medical terminology that can be shared between healthcare systems."

---

## 6. Session Management

### Overview
The Patient Session Context manages multiple active patient consultations, storing transcripts, documents, and metadata.

### Session Storage Architecture
```
┌───────────────────────────────────────────────────────────────┐
│                  PatientSessionContext                         │
├───────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ PatientSession                                          │  │
│  │ ├── id: string (UUID)                                   │  │
│  │ ├── patientName: string                                 │  │
│  │ ├── hospitalId: string                                  │  │
│  │ ├── status: 'active' | 'completed' | 'pending'          │  │
│  │ ├── transcripts: SessionTranscript[]                    │  │
│  │ │   ├── content: string                                 │  │
│  │ │   ├── type: 'voice' | 'text' | 'summary'              │  │
│  │ │   └── duration?: number                               │  │
│  │ ├── documents: SessionDocument[]                        │  │
│  │ │   ├── name: string                                    │  │
│  │ │   ├── type: 'soapNotes' | 'referralLetter' | ...     │  │
│  │ │   └── content: string                                 │  │
│  │ └── tabs: SessionTab[]                                  │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

### Context API (PatientSessionContext.tsx)
```typescript
interface PatientSessionContextType {
  // Session management
  sessions: PatientSession[];
  currentSession: PatientSession | null;
  
  // Operations
  createSession: (hospitalId, patientName, age?, gender?) => PatientSession;
  selectSession: (sessionId: string) => void;
  updateSession: (sessionId, updates) => PatientSession | null;
  deleteSession: (sessionId: string) => boolean;
  
  // Content operations
  addTranscript: (content, type, duration?) => SessionTranscript | null;
  addDocument: (name, type, content) => SessionDocument | null;
  updateDocument: (documentId, updates) => SessionDocument | null;
  
  // Persistence
  exportSessions: () => string;           // JSON export
  importSessions: (json) => { success, imported };
  backupCurrentSession: () => boolean;
}
```

### What to Explain in Interview
> "I use React Context for session management because multiple components need access to the current patient session - the recording controls, document editor, and AI sidebar all need to know which patient we're working with. The context provides a centralized state with operations like `addTranscript` and `addDocument`. I also implemented export/import functionality so doctors can back up their session data or transfer to another device."

---

## 7. Route Protection (Middleware)

### Overview
Next.js middleware protects routes based on authentication status.

### Implementation (middleware.ts)
```typescript
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

const PATIENT_AUTH_COOKIE = 'medhaya-patient-auth';
const DOCTOR_AUTH_COOKIE = 'medhaya-doctor-auth';

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Skip auth in development mode
  if (shouldBypassAuth()) {
    return NextResponse.next();
  }

  // Protect doctor dashboard routes
  if (pathname.startsWith('/doctor')) {
    const doctorAuth = request.cookies.get(DOCTOR_AUTH_COOKIE);
    
    if (!doctorAuth) {
      // Redirect to login, preserving intended destination
      const loginUrl = new URL('/doctor-login', request.url);
      loginUrl.searchParams.set('callbackUrl', pathname);
      return NextResponse.redirect(loginUrl);
    }
  }

  return NextResponse.next();
}

// Only run middleware on these routes
export const config = {
  matcher: ['/patient/:path*', '/doctor/:path*']
};
```

### What to Explain in Interview
> "I use Next.js middleware for route protection because it runs on the server before the page loads. This prevents any flash of protected content. When an unauthenticated user tries to access `/doctor/dashboard`, the middleware checks for the auth cookie. If it's missing, they're redirected to `/doctor-login` with a callback URL so they return to their intended page after logging in."

---

## 8. API Architecture

### Overview
The API layer uses a centralized `fetchWithAuth` wrapper that handles authentication, timeouts, retries, and error handling for all HTTP requests.

### Architecture Diagram
```
┌─────────────────────────────────────────────────────────────────────────┐
│                         API SERVICE LAYER                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │
│  │ authService  │  │ documentSvc  │  │ transcriptSvc│  │ ailaDiagSvc │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬──────┘  │
│         │                 │                 │                  │         │
│         └─────────────────┴─────────────────┴──────────────────┘         │
│                                    │                                     │
│                                    ▼                                     │
│                        ┌────────────────────┐                            │
│                        │   fetchWithAuth    │ ◀── Central fetch wrapper  │
│                        └─────────┬──────────┘                            │
└──────────────────────────────────│───────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        fetchWithAuth INTERNALS                           │
│                                                                          │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐  │
│  │ getHeaders  │   │ Timeout     │   │ Retry Logic │   │ Error       │  │
│  │ (Auth+CSRF) │   │ Controller  │   │ (Backoff)   │   │ Handling    │  │
│  └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
                          ┌─────────────────┐
                          │  Backend API    │
                          │  (REST Server)  │
                          └─────────────────┘
```

### Complete fetchWithAuth Implementation (config.ts)

#### Step 1: Header Configuration
```typescript
// Set up common headers with authorization
export const getHeaders = (userType: 'patient' | 'doctor' = 'patient') => {
  const token = getCookie(userType === 'patient' ? PATIENT_AUTH_COOKIE : DOCTOR_AUTH_COOKIE);
  const csrfToken = getCookie(CSRF_TOKEN_COOKIE);
  
  return {
    'Content-Type': 'application/json',
    'Authorization': token ? `Bearer ${token}` : '',
    'X-CSRF-Token': csrfToken || '',
    'X-Requested-With': 'XMLHttpRequest', // Helps protect against CSRF
  };
};
```

#### Step 2: Timeout Controller
```typescript
// Request timeout in milliseconds (150 seconds for slow connections)
const REQUEST_TIMEOUT = 30000 * 5;

// Create AbortController for request timeout
const createTimeoutController = (ms: number = REQUEST_TIMEOUT): { 
  controller: AbortController, 
  timeoutId: number 
} => {
  const controller = new AbortController();
  const timeoutId = window.setTimeout(() => controller.abort(), ms);
  return { controller, timeoutId };
};
```

#### Step 3: Main fetchWithAuth Function
```typescript
// Maximum retry attempts for failed requests
const MAX_RETRY_ATTEMPTS = 3;

export const fetchWithAuth = async (
  endpoint: string, 
  options: RequestInit = {}, 
  userType: 'patient' | 'doctor' = 'patient',
  retryCount = 0
): Promise<unknown> => {
  
  // Set up timeout - cancels request if it takes too long
  const { controller, timeoutId } = createTimeoutController();

  try {
    // Make the actual fetch request
    const response = await fetch(`${API_BASE_URL}${endpoint}`, {
      ...options,
      headers: {
        ...getHeaders(userType),  // Add auth headers
        ...options.headers,       // Allow header overrides
      },
      signal: controller.signal,  // Attach timeout controller
    });

    // Clear timeout since request completed successfully
    clearTimeout(timeoutId);

    // ═══════════════════════════════════════════════════════════
    // RATE LIMITING HANDLING (HTTP 429)
    // ═══════════════════════════════════════════════════════════
    if (response.status === 429 && retryCount < MAX_RETRY_ATTEMPTS) {
      // Server says "slow down" - wait the specified time then retry
      const retryAfter = response.headers.get('Retry-After') || '1';
      const waitTime = parseInt(retryAfter, 10) * 1000;
      
      await new Promise(resolve => setTimeout(resolve, waitTime));
      return fetchWithAuth(endpoint, options, userType, retryCount + 1);
    }

    // ═══════════════════════════════════════════════════════════
    // ERROR RESPONSE HANDLING
    // ═══════════════════════════════════════════════════════════
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}));
      
      // Authentication failure - redirect to login
      if (response.status === 401 || response.status === 403) {
        window.location.href = userType === 'patient' ? '/' : '/doctor-login';
      }
      
      throw new Error(errorData.message || `API error: ${response.status}`);
    }

    // Parse and return JSON response
    const responseData = await response.json();
    return responseData;

  } catch (error) {
    // Always clear timeout on error
    clearTimeout(timeoutId);
    
    // ═══════════════════════════════════════════════════════════
    // TIMEOUT HANDLING
    // ═══════════════════════════════════════════════════════════
    if (error instanceof DOMException && error.name === 'AbortError') {
      throw new Error('Request timeout exceeded');
    }
    
    // ═══════════════════════════════════════════════════════════
    // NETWORK ERROR WITH EXPONENTIAL BACKOFF RETRY
    // ═══════════════════════════════════════════════════════════
    if (error instanceof TypeError && 
        error.message === 'Failed to fetch' && 
        retryCount < MAX_RETRY_ATTEMPTS) {
      // Exponential backoff: 1s, 2s, 4s, etc.
      const backoffTime = Math.pow(2, retryCount) * 1000;
      await new Promise(resolve => setTimeout(resolve, backoffTime));
      return fetchWithAuth(endpoint, options, userType, retryCount + 1);
    }

    // Re-throw unhandled errors
    throw error;
  }
};
```

### How Each Feature Works

| Feature | How It Works | Why It's Important |
|---------|--------------|-------------------|
| **Auto Headers** | `getHeaders()` reads auth cookies and adds them to every request | Services don't need to manually add tokens |
| **Request Timeout** | `AbortController` cancels request after 150 seconds | Prevents UI from hanging indefinitely |
| **Rate Limit Handling** | On 429, reads `Retry-After` header and waits | Respects server's backpressure signals |
| **Exponential Backoff** | Network errors retry after 1s, 2s, 4s... | Gracefully handles temporary outages |
| **Auth Redirect** | 401/403 automatically redirects to login | User doesn't see broken authenticated pages |
| **CSRF Protection** | `X-Requested-With` + `X-CSRF-Token` headers | Prevents cross-site request forgery |

### Request Flow Diagram
```
┌────────────────────────────────────────────────────────────────────────┐
│                         REQUEST LIFECYCLE                               │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  START ──▶ Add Headers ──▶ Start Timer ──▶ fetch() ──────────────────┐ │
│                                               │                       │ │
│                                               ▼                       │ │
│  ┌─────────────────────────────────────────────────────────────────┐ │ │
│  │                    RESPONSE HANDLING                            │ │ │
│  ├─────────────────────────────────────────────────────────────────┤ │ │
│  │                                                                 │ │ │
│  │  Status 200-299 ──▶ Return JSON ──▶ ✅ SUCCESS                  │ │ │
│  │                                                                 │ │ │
│  │  Status 401/403 ──▶ Redirect to Login ──▶ 🔐 AUTH REQUIRED      │ │ │
│  │                                                                 │ │ │
│  │  Status 429 ──▶ Wait (Retry-After) ──▶ 🔄 RETRY (up to 3x)      │ │ │
│  │                                                                 │ │ │
│  │  Status 4xx/5xx ──▶ Throw Error ──▶ ❌ FAIL                     │ │ │
│  │                                                                 │ │ │
│  └─────────────────────────────────────────────────────────────────┘ │ │
│                                                                       │ │
│  ┌─────────────────────────────────────────────────────────────────┐ │ │
│  │                     ERROR HANDLING                              │ │ │
│  ├─────────────────────────────────────────────────────────────────┤ │ │
│  │                                                                 │ │ │
│  │  AbortError (timeout) ──▶ Throw "timeout exceeded"              │ │ │
│  │                                                                 │ │ │
│  │  Network Error ──▶ Backoff ──▶ 🔄 RETRY (1s, 2s, 4s...)         │ │ │
│  │                                                                 │ │ │
│  │  Other Error ──▶ Re-throw ──▶ ❌ FAIL                           │ │ │
│  │                                                                 │ │ │
│  └─────────────────────────────────────────────────────────────────┘ │ │
│                                                                       │ │
└───────────────────────────────────────────────────────────────────────┘ │
                                                                          │
    ◀──────────────────────────────────────────────────────────────────────┘
```

### Usage Examples

```typescript
// Simple GET request
const templates = await fetchWithAuth('/api/documents/templates', {}, 'doctor');

// POST with body
const result = await fetchWithAuth('/api/auth/login', {
  method: 'POST',
  body: JSON.stringify({ email, password }),
}, 'doctor');

// With custom headers
const response = await fetchWithAuth('/api/custom', {
  method: 'POST',
  headers: { 'X-Custom-Header': 'value' },
  body: JSON.stringify(data),
}, 'patient');
```

### Enhanced Variant: fetchWithSanitization

For POST/PUT requests with user input, there's a sanitized version:

```typescript
export const fetchWithSanitization = async (
  endpoint: string,
  method: 'POST' | 'PUT',
  body: object,
  userType: 'patient' | 'doctor' = 'patient'
) => {
  // Sanitize all string values to prevent XSS
  const sanitizeObject = (obj: unknown): unknown => {
    if (typeof obj === 'string') {
      // HTML-encode the string
      const div = document.createElement('div');
      div.textContent = obj;
      return div.innerHTML;
    }
    if (Array.isArray(obj)) {
      return obj.map(item => sanitizeObject(item));
    }
    if (typeof obj === 'object' && obj !== null) {
      const sanitized: Record<string, unknown> = {};
      for (const [key, value] of Object.entries(obj)) {
        sanitized[key] = sanitizeObject(value);
      }
      return sanitized;
    }
    return obj;
  };
  
  const sanitizedBody = sanitizeObject(body);
  return fetchWithAuth(endpoint, {
    method,
    body: JSON.stringify(sanitizedBody),
  }, userType);
};
```

### What to Explain in Interview

> "All API calls go through a centralized `fetchWithAuth` wrapper that handles cross-cutting concerns:

> **1. Authentication:** It automatically reads the auth token from cookies and adds it to every request header. This means individual API services don't need to know about authentication.

> **2. Timeouts:** I use an `AbortController` with a 150-second timeout. If the backend doesn't respond, the request is cancelled and an error is thrown. This prevents the UI from appearing frozen.

> **3. Automatic Retries:** For network failures like 'Failed to fetch', it retries with exponential backoff - first wait 1 second, then 2, then 4. This handles temporary network issues gracefully.

> **4. Rate Limiting:** If the server returns a 429 'Too Many Requests', it reads the `Retry-After` header and waits that long before retrying. This respects the server's backpressure.

> **5. Security:** Every request includes a CSRF token and the `X-Requested-With` header. This protects against cross-site request forgery attacks.

> **6. Auth Failures:** If the backend returns 401 or 403, it automatically redirects to the login page. The user never sees a broken authenticated page.

> The benefit is that services like `authService.ts` or `documentService.ts` just call `fetchWithAuth('/api/endpoint', options)` and all this behavior happens automatically."

---

## Quick Reference: Key Files

| Feature | Primary File | Supporting Files |
|---------|--------------|------------------|
| Authentication | `src/api/authService.ts` | `src/utils/cookieUtils.ts` |
| WebSocket Transcription | `src/api/transcriptionService.ts` | `src/hooks/useVoiceRecording.ts` |
| AI Diagnostics | `src/api/ailaDiagnosticService.ts` | `src/components/ui/AilaDiagnosticSidebar.tsx` |
| Document Generation | `src/api/documentService.ts` | `src/components/documentEditor/*` |
| Session Management | `src/context/PatientSessionContext.tsx` | `src/hooks/usePatientSessions.ts` |
| Route Protection | `src/middleware.ts` | `src/config/environment.ts` |
| API Config | `src/api/config.ts` | - |

---

## Interview Tips

1. **Start with the flow, then dive into code** - Draw the architecture first, then show specific code when asked.

2. **Explain trade-offs** - "I used WebSockets instead of REST because..." or "I chose cookies over localStorage because..."

3. **Mention challenges you solved** - Mobile permission handling, reconnection logic, caching strategies.

4. **Be honest about AI assistance** - "The AI helped me scaffold the AudioWorklet, but I had to customize the error handling."
