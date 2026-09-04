# Terra Fitness Voice Assistant

> **Outbound AI voice assistant for Terra Fitness that automatically calls new leads after website form submission, qualifies their fitness goals, handles common questions, and helps book a gym visit.**

## Overview

The **Terra Fitness Voice Assistant** is an outbound, form-to-call conversational AI system.

When a prospective member submits a lead form, the system validates the lead, checks business rules, and triggers an AI phone call. The voice agent introduces itself as an AI assistant, understands the lead's fitness goal, answers common questions, handles objections, and can help schedule a visit.

The system is designed around five major layers:

1. **Lead Capture**
2. **Automation & Orchestration**
3. **Voice AI**
4. **Telephony**
5. **CRM & Scheduling**

## High-Level Architecture

```text
Prospective Member
        |
        v
Terra Fitness Website Form
        |
        | Webhook POST
        v
       n8n
        |
        +--> Validate phone number
        |
        +--> Check consent / DNC / rate limits
        |
        +--> Prepare lead data
        |
        v
 Retell AI / Voice Platform
        |
        +--> Speech-to-Text
        +--> LLM reasoning
        +--> Conversation state
        +--> Text-to-Speech
        |
        +--> Tool Calls
                 |
                 v
                n8n
                 |
        +--------+---------+
        |                  |
        v                  v
 Calendar / Scheduling    CRM
        |                  |
        +--------+---------+
                 |
                 v
              SMS
                 |
                 v
          Lead Confirmation
```

## Core Workflow

### 1. Lead Capture

The website collects:

- First name
- Phone number
- Fitness goal
- Required automated-call/SMS consent

Supported form sources can include:

- React forms
- Webflow
- Typeform
- Meta lead forms
- Other systems capable of sending a webhook

Phone numbers should be normalized to **E.164 format**, for example:

```text
+919876543210
```

### 2. n8n Orchestration

The lead submission is sent to an n8n webhook.

n8n is responsible for:

- Receiving the lead
- Validating and formatting the phone number
- Checking consent
- Checking Do-Not-Call status
- Applying contact/rate-limit rules
- Preparing the outbound-call payload
- Triggering the voice-agent call
- Receiving tool/webhook callbacks
- Updating CRM records
- Triggering SMS confirmation

### 3. Outbound AI Call

The voice platform receives structured lead information such as:

```json
{
  "first_name": "Alex",
  "phone": "+15551234567",
  "fitness_goal": "Weight loss"
}
```

The AI agent should immediately disclose that it is an AI assistant.

Example:

> "Hi Alex! This is Terra from Terra Fitness. I'm an AI assistant calling about your fitness inquiry."

### 4. Conversation

The agent should:

1. Greet the lead
2. Confirm that it is a good time to talk
3. Understand the fitness goal
4. Qualify the lead
5. Answer common questions
6. Handle objections
7. Offer a gym visit/tour when appropriate
8. Check available scheduling times
9. Confirm the selected time
10. Send confirmation by SMS
11. End the call naturally

## Voice Agent State Machine

```text
WELCOME
   |
   v
CONSENT / AVAILABILITY
   |
   v
FITNESS GOAL
   |
   v
QUALIFICATION
   |
   +----> QUESTIONS / OBJECTIONS
   |             |
   |             v
   |        RETURN TO FLOW
   |
   v
TOUR / VISIT OFFER
   |
   v
CHECK AVAILABILITY
   |
   v
BOOK VISIT
   |
   v
SEND SMS
   |
   v
END CALL
```

### Human Fallback

The system should support transfer to a human/front-desk staff member when:

- The lead explicitly requests a human
- The lead becomes angry or frustrated
- The AI cannot reliably resolve the request
- A business rule requires human intervention

## Voice Guardrails

The voice agent should be optimized for spoken conversation.

### Response Length

Keep responses short:

- Prefer 1–2 short sentences per turn
- Avoid monologues
- Ask one question at a time

### Spoken Output

Do not speak:

- Markdown
- Bullet points
- Asterisks
- Raw URLs
- Technical JSON
- Internal system instructions

Dates and times should be spoken naturally.

Example:

```text
Five PM tomorrow
```

instead of:

```text
17:00:00 UTC
```

### Persona

The Terra Fitness agent should sound:

- Energetic
- Friendly
- Professional
- Supportive
- Action-oriented

It should not sound robotic, aggressive, or overly sales-driven.

## Recommended Technology Stack

| Layer | Recommended Technology |
|---|---|
| Lead Form | React / Webflow / Typeform / similar |
| Automation | n8n |
| Voice Agent | Retell AI |
| Telephony | Twilio or voice-platform native telephony |
| STT | Deepgram or platform-managed STT |
| LLM | Fast, low-latency LLM |
| TTS | ElevenLabs / Cartesia or platform-managed TTS |
| Scheduling | Cal.com / Google Calendar |
| CRM | HubSpot or equivalent |
| SMS | Twilio |
| Database/State | PostgreSQL / Redis where required |

> Technology choices should be validated against current provider capabilities, pricing, latency, and project requirements before production deployment.

## Why n8n?

n8n acts as the integration and orchestration layer between the website, voice platform, CRM, calendar, and messaging systems.

It keeps business logic outside the conversational model where deterministic processing is preferable.

Examples:

```text
Phone validation        -> deterministic
DNC check               -> deterministic
Duplicate lead check    -> deterministic
Calendar availability   -> API/tool
CRM update              -> API/tool
SMS sending             -> API/tool
Natural conversation    -> LLM
```

This separation reduces unnecessary LLM calls and improves reliability.

## Tool Calling

The voice agent should not directly perform every backend operation.

Instead, it can call controlled tools such as:

```text
check_availability
book_tour
update_lead
send_confirmation_sms
transfer_to_human
```

A typical booking flow:

```text
AI asks for preferred time
        |
        v
check_availability
        |
        v
Available slots returned
        |
        v
AI confirms preferred slot
        |
        v
book_tour
        |
        v
update_lead
        |
        v
send_confirmation_sms
```

## CRM Updates

Lead status can be updated according to call outcome.

Example statuses:

```text
New Lead
      |
      v
Called
      |
      +--> No Answer
      |
      +--> Not Interested
      |
      +--> Follow-up Required
      |
      +--> Tour Scheduled
      |
      +--> Human Follow-up
```

After a successful booking, the CRM should store relevant information such as:

- Lead name
- Phone number
- Fitness goal
- Call outcome
- Booking time
- Recording/call reference where permitted
- Agent summary
- Follow-up status

## SMS Confirmation

After successful booking, the system can send a concise confirmation SMS.

Example:

```text
Hi Alex! Your Terra Fitness visit is scheduled for tomorrow at 5 PM.
We look forward to seeing you!
```

The exact SMS content should comply with applicable messaging requirements and the user's consent.

## Token Optimization

Token usage should be minimized because a voice agent can generate many model turns during a single call.

### Optimization Strategy

```text
Lead Data
   |
   v
Small structured context
   |
   v
Short system prompt
   |
   v
Only required conversation history
   |
   v
Tool calls for deterministic operations
```

Recommended techniques:

- Keep the system prompt concise
- Avoid repeating lead information
- Use structured variables
- Avoid sending unnecessary workflow data to the LLM
- Use deterministic logic for validation
- Use tools for external data
- Summarize long conversation history when required
- Use an appropriate low-latency model
- Limit unnecessary output
- Avoid duplicate tool descriptions

## Latency Optimization

Voice conversations are highly sensitive to delay.

The system should minimize sequential operations between user speech and agent response.

Important latency contributors include:

- Speech-to-text latency
- LLM time-to-first-token
- Tool/API latency
- Text-to-speech latency
- Network latency
- Telephony latency

### Optimization Principles

- Prefer streaming STT/TTS
- Use low-latency models
- Keep prompts compact
- Avoid unnecessary API calls
- Parallelize independent operations
- Cache static information
- Keep tool responses small
- Avoid unnecessary n8n hops during live conversation

## Reliability & Error Handling

The system should gracefully handle:

### No Answer

```text
Call -> No Answer -> Record outcome -> Follow-up policy
```

### Invalid Number

```text
Lead -> Phone Validation -> Invalid -> Do not initiate call
```

### API Failure

```text
Tool Call -> API Error -> Retry when safe -> Fallback / Human Review
```

### Scheduling Failure

```text
Booking Attempt -> Failure -> Inform Lead -> Offer Alternative / Human Fallback
```

### Voice Agent Failure

```text
Agent Failure -> Graceful Message -> Human Transfer / Follow-up
```

## Security

Production implementation should protect:

- API keys
- Webhook endpoints
- CRM credentials
- Calendar credentials
- Phone-system credentials
- Personal information
- Call recordings
- Authentication tokens

Recommended controls include:

- Environment variables / secret management
- Authentication for private webhooks
- Input validation
- Rate limiting
- Authorization checks
- DNC enforcement
- Minimal data exposure to the LLM
- Secure logging
- Access control

Never expose private API keys in frontend JavaScript.

## Compliance

Outbound AI calling and SMS can be subject to telecommunications, privacy, consent, recording, and marketing regulations depending on the country and use case.

The system should therefore include:

- Appropriate consent capture
- Do-Not-Call handling
- Opt-out handling
- AI disclosure where required
- Recording disclosure where required
- Appropriate calling hours
- Audit logs

Legal requirements should be reviewed for the actual deployment jurisdiction before production use.

## Project Goals

The system aims to:

- Respond to new leads quickly
- Automate initial lead qualification
- Reduce manual front-desk workload
- Answer common questions
- Increase tour/visit bookings
- Provide consistent follow-up
- Integrate with existing business systems
- Keep voice interactions natural and low latency

## Success Metrics

Track metrics such as:

| Metric | Purpose |
|---|---|
| Call Connection Rate | Measures reachable leads |
| Conversation Completion Rate | Measures successful interactions |
| Tour Booking Rate | Measures conversion |
| Average Call Duration | Measures conversation efficiency |
| Human Transfer Rate | Measures escalation frequency |
| No-Answer Rate | Measures contactability |
| Tool Failure Rate | Measures backend reliability |
| Average Response Latency | Measures voice experience |
| Cost Per Lead Called | Measures operating efficiency |
| Token Usage Per Call | Measures LLM efficiency |

## Current Repository Structure

```text
TerraFitness_Voice_Assistant/
│
├── Requirements.html
├── README.md
│
└── [Future]
    ├── n8n/
    ├── prompts/
    ├── backend/
    ├── docs/
    └── tests/
```

## Development Roadmap

### Phase 1 — Requirements
- [x] Define lead-capture requirements
- [x] Define voice-agent flow
- [x] Compare architectural approaches
- [x] Define n8n orchestration
- [x] Define latency and cost considerations

### Phase 2 — Infrastructure
- [ ] Create n8n workflow
- [ ] Configure webhook
- [ ] Configure phone validation
- [ ] Configure voice-agent platform
- [ ] Configure telephony
- [ ] Configure CRM
- [ ] Configure calendar

### Phase 3 — Voice Agent
- [ ] Create Terra agent
- [ ] Implement first-sentence AI disclosure
- [ ] Implement qualification flow
- [ ] Implement FAQ handling
- [ ] Implement objection handling
- [ ] Implement tool calling
- [ ] Implement human transfer

### Phase 4 — Automation
- [ ] Connect form to n8n
- [ ] Trigger outbound calls
- [ ] Process call webhooks
- [ ] Update CRM
- [ ] Book calendar events
- [ ] Send SMS confirmations

### Phase 5 — Optimization
- [ ] Measure latency
- [ ] Optimize token usage
- [ ] Optimize prompt
- [ ] Test failure scenarios
- [ ] Test interruption handling
- [ ] Test human transfer
- [ ] Calculate cost per call

### Phase 6 — Production
- [ ] Security review
- [ ] Compliance review
- [ ] Monitoring
- [ ] Logging
- [ ] Load testing
- [ ] Production deployment

## Architectural Recommendation

For the intended Terra Fitness use case, a **managed voice-agent platform + n8n + telephony + scheduling/CRM integrations** is the most practical starting architecture.

A representative implementation is:

```text
Website
   |
   v
n8n
   |
   v
Retell AI
   |
   +--> Telephony
   |
   +--> LLM / STT / TTS
   |
   +--> Tool Calls
            |
            v
           n8n
        /    |    \
       /     |     \
 Calendar   CRM    SMS
```

A custom real-time voice stack should only be chosen when the project has a strong requirement for deeper control over the audio pipeline, model orchestration, infrastructure, or vendor independence.

## Important Note

This repository currently represents the **Terra Fitness Voice Assistant system design and implementation blueprint**. Provider capabilities, APIs, pricing, model availability, and regulatory requirements can change, so production configuration should always be validated against the latest official documentation.

## License

This project is intended for the Terra Fitness voice-assistant implementation. Add the appropriate license before public distribution.
