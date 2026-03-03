# Medhaya Interview Preparation Guide
## For UPTIME Crew Recruiter Interview

---

## 1. Project Overview (Non-Technical Explanation)

**What is Medhaya?**
> "Medhaya is a healthcare platform I built that helps doctors conduct patient consultations more efficiently. Think of it as a smart assistant for doctors - it listens to the conversation between a doctor and patient, transcribes it in real-time, and provides AI-powered insights to help with diagnosis."

**The Problem It Solves:**
> "Doctors spend a lot of time taking notes during consultations instead of focusing on the patient. Medhaya automates this by transcribing the conversation live and generating medical documents automatically. It also has an AI assistant called 'Aila' that analyzes the conversation and suggests possible conditions based on symptoms mentioned."

**Key Value Proposition:**
- Saves doctors 10-15 minutes per consultation on documentation
- Supports multiple languages for diverse patient populations  
- Provides AI-powered diagnostic insights based on NHS/NICE guidelines
- Generates professional medical documents automatically

---

## 2. Key Features to Demonstrate (Screen Share Order)

### A. Landing Page (1-2 minutes)
**Path:** Home page  
**Show:**
- Clean, professional healthcare UI
- Feature cards explaining capabilities
- "Meet Aila" section - the AI assistant

**What to say:**
> "This is the landing page. I designed it to clearly communicate the value to healthcare professionals. The main features are patient-centered care, intelligent triage, and streamlined consultation workflow."

---

### B. Doctor Login/Registration (1 minute)
**Path:** `/doctor-login` or `/doctor-registration`  
**Show:**
- Secure authentication flow
- NHS ID integration

**What to say:**
> "Doctors can register using their NHS credentials. The authentication system validates their identity and stores session data securely using HTTP-only cookies."

---

### C. Consultation Dashboard (Main Demo - 3-4 minutes)
**Path:** After login → Start a consultation  
**Show:**
1. **Recording Controls** - Start/pause/stop recording
2. **Live Transcription** - Watch text appear in real-time
3. **Language Selector** - Support for multiple languages
4. **Microphone Selection** - Choose audio device
5. **Audio Upload** - Import existing recordings
6. **Aila Insights Panel** - AI diagnostic suggestions

**What to say:**
> "This is the core of the application. When a doctor starts a consultation, they click record, and the system transcribes the conversation in real-time using WebSocket streaming. On the right, you can see Aila - our AI assistant - analyzing the conversation and suggesting possible conditions with confidence levels."

---

### D. Document Generation (1-2 minutes)
**Path:** Document Editor tab  
**Show:**
- Template selection
- Auto-generated patient notes
- Export options (PDF, email)

**What to say:**
> "After the consultation, the system can automatically generate medical documents like referral letters, prescriptions, or consultation summaries. Doctors can choose templates or create custom ones."

---

## 3. Code Walkthrough - Key Files to Explain

### If asked about Architecture:
| Area | File | What to Explain |
|------|------|-----------------|
| Frontend Framework | `package.json` | Next.js 16, React, TypeScript, Tailwind CSS |
| Core Consultation | `src/components/consultation/UnifiedConsultation.tsx` | Main consultation component managing recording state |
| Voice Recording | `src/hooks/useVoiceRecording.ts` | Custom React hook for microphone access and transcription |
| Real-time Transcription | `src/api/transcriptionService.ts` | WebSocket connection for live speech-to-text |
| AI Diagnostics | `src/api/ailaDiagnosticService.ts` | API service for AI-powered condition analysis |
| AI UI Component | `src/components/ui/AilaDiagnosticSidebar.tsx` | Displays AI suggestions with confidence levels |
| Authentication | `src/api/authService.ts` | Doctor/patient login with secure cookie management |
| Document Service | `src/api/documentService.ts` | Template management and document generation |

---

### Sample Code Explanations:

**Q: "How does the real-time transcription work?"**
> "I use WebSocket connections for real-time streaming. The browser captures audio through the Web Audio API, processes it with an AudioWorklet for better performance, and streams 16kHz audio chunks to the backend. The TranscriptionService class (`transcriptionService.ts`) manages the WebSocket connection, handles reconnection logic, and buffers the incoming transcript text."

**Q: "How does the AI diagnostic feature work?"**
> "The Aila service (`ailaDiagnosticService.ts`) takes the conversation transcript and sends it to our RAG (Retrieval Augmented Generation) endpoint. The backend analyzes the symptoms mentioned and returns possible conditions with confidence scores, plus recommended actions. I display these in a sidebar component with color-coded confidence levels - green for high, yellow for medium, red for low."

**Q: "How did you handle state management?"**
> "I use React Context for global state like AI chat visibility, and custom hooks for reusable logic. For example, `useVoiceRecording` encapsulates all the recording logic - microphone access, state management, timer, and transcription callbacks. This keeps components clean and makes the code more testable."

---

## 4. Where and How You Used AI (Critical for Interview)

### AI Tools Used in Development:

#### 1. **GitHub Copilot / AI Assistant - Code Generation**
**Example:** Creating the transcription WebSocket service

> "When building the real-time transcription feature, I used AI to help generate the WebSocket boilerplate and audio processing code. I provided context about needing 16kHz audio streaming, and the AI helped scaffold the AudioWorklet implementation. However, I had to modify the error handling significantly - the AI's initial version didn't handle connection drops gracefully, so I added reconnection logic with exponential backoff."

**What I accepted:**
- Basic WebSocket setup structure
- Audio constraint configuration
- TypeScript interfaces

**What I changed/improved:**
- Error handling and reconnection logic
- Permission management for mobile browsers
- Custom worklet processor implementation

---

#### 2. **AI-Assisted Component Development**
**Example:** Building the AilaDiagnosticSidebar component

> "I used AI to help structure the diagnostic sidebar component. I described wanting a collapsible panel that shows conditions with confidence bars. The AI generated the initial JSX structure, but I customized the animations using Framer Motion and designed the caching strategy myself to prevent redundant API calls when users toggle the sidebar."

**What I accepted:**
- Basic component structure
- Prop interface definitions
- Conditional rendering patterns

**What I changed:**
- Added sessionId and documentId for state isolation
- Implemented smart caching to avoid re-fetching
- Custom styling to match the Medhaya design system

---

#### 3. **AI for Debugging and Optimization**
**Example:** Cookie handling in authService

> "I had a complex bug where login state wasn't persisting correctly across page reloads. I shared the code with an AI assistant and asked for debugging help. It helped me identify that I was setting cookies without the `httpOnly` flag and explained the difference between client and server-side cookies in Next.js. I then rewrote the cookie utility to handle both cases properly."

---

#### 4. **AI for Test Writing**
**Example:** Jest test configuration and test cases

> "I used AI to help generate initial test cases for my components. The `jest.setup.js` and test files in `__tests__` folders were scaffolded with AI assistance. However, I refined the mocks for browser APIs like MediaDevices and WebSocket since the AI-generated mocks weren't accurate enough for my specific use cases."

---

### Summary Table of AI Usage:

| Task | AI Helped With | What I Accepted | What I Changed |
|------|---------------|-----------------|----------------|
| WebSocket Transcription | Boilerplate, audio config | Structure, interfaces | Error handling, reconnection |
| Diagnostic Sidebar | Component structure | JSX layout, props | Caching, animation, state isolation |
| Authentication | Cookie handling patterns | Utility functions | Server-side vs client handling |
| API Services | REST request patterns | Fetch wrappers | Auth headers, error responses |
| Testing | Test scaffolding | Basic test cases | Browser API mocks |

---

## 5. Technical Stack Summary

| Category | Technology |
|----------|------------|
| Framework | Next.js 16 (App Router) |
| Language | TypeScript |
| Styling | Tailwind CSS + shadcn/ui |
| State | React Context + Custom Hooks |
| Real-time | WebSocket for transcription |
| Audio | Web Audio API + AudioWorklet |
| AI Integration | Custom RAG service (Aila) |
| Testing | Jest + React Testing Library |
| Animations | Framer Motion |

---

## 6. Potential Interview Questions & Answers

**Q: "What was the most challenging part of this project?"**
> "Real-time audio transcription across different devices and browsers. Mobile browsers handle microphone permissions differently, and I had to implement a permission manager that caches permission state to avoid repeatedly prompting users. The AudioWorklet implementation was also tricky - I had to downsample audio to 16kHz for the transcription API while maintaining quality."

**Q: "What would you do differently if you started over?"**
> "I would implement a more robust state management solution earlier, possibly using Zustand or Jotai instead of multiple React contexts. As the app grew, coordinating state between the consultation, documents, and AI panels became complex. I'd also set up end-to-end testing earlier with Playwright."

**Q: "How did you ensure code quality?"**
> "I used TypeScript for type safety, ESLint for code style, and Jest for unit testing. I also created custom hooks to encapsulate complex logic, which made the codebase more maintainable and testable. The test coverage reports in the `coverage/` directory show which areas need more testing."

**Q: "What's next for this project?"**
> "I'm planning to add video consultation support, improve the AI model with specialty-specific training, and implement better offline support for areas with poor connectivity."

---

## 7. Demo Checklist

Before the interview:
- [ ] Run `npm run dev` and verify the app starts
- [ ] Test microphone permissions work
- [ ] Have a sample audio file ready for the import feature
- [ ] Open VS Code with these files ready:
  - `src/components/consultation/UnifiedConsultation.tsx`
  - `src/api/transcriptionService.ts`
  - `src/api/ailaDiagnosticService.ts`
  - `src/hooks/useVoiceRecording.ts`
- [ ] Clear browser console/cache for a clean demo
- [ ] Prepare one code snippet to walk through in detail

---

## 8. Quick Talking Points

**Opening (30 seconds):**
> "I'd like to show you Medhaya, a healthcare consultation platform I've been building. It helps doctors save time by automatically transcribing patient conversations and providing AI-powered diagnostic insights."

**AI Usage Summary (30 seconds):**
> "Throughout development, I used AI assistants to help with code generation and debugging. For example, the AI helped me scaffold the WebSocket transcription service, but I rewrote the error handling and added features like device selection and permission management. I found AI most useful for boilerplate code, but I always reviewed and customized the output to fit the project's architecture."

**Closing (15 seconds):**
> "This project demonstrates my ability to build full-stack applications with modern technologies, work with real-time data streaming, and integrate AI services into practical applications."

---

Good luck with your interview! 🎯
