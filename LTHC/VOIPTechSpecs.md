# Platinum Line VOIP Feature - Technical Overview

## Summary

Add phone and SMS capabilities to Platinum Line through managed phone number pools. Clinicians communicate with external contacts (family members, pharmacies, facilities) via **external threads** that operate alongside existing patient threads. External threads can either link to 0 or 1 facilities, or to 0, 1, or many patients. Clinicians make outbound calls using call bridging to mask personal phone numbers.

**Critical Architectural Requirement**: The system is designed with a robust provider abstraction layer to enable frictionless VOIP provider swapping. Given past reliability issues with telephony providers, all provider-specific code is isolated behind a standard `TelephonyService` interface, ensuring we can switch providers with minimal code changes if reliability issues arise.

---

## Core Concepts

### External Threads

- **Purpose**: Handle communications with external phone numbers/emails that don't fit the 1-to-1 patient thread model
- **Flexibility**: Can link to 0, 1, or many patients via PostgreSQL array (e.g., family member calling about multiple patients)
- **Coexistence**: Patient threads remain unchanged; external threads are separate but unified in inbox UI
- **Use Cases**: Family member texts, pharmacy calls, facility inquiries, telehealth follow-ups

### Phone Number Pools

- **Unified Numbers**: All numbers can receive inbound calls/SMS AND serve as outbound caller ID
- **Admin Managed**: Admins provision numbers manually (provider dashboard → add to Platinum Line UI)
- **Labeling**: Admin assigns caller id labels for organization ("Main Line", "Clinician 1")
- **Selection**: Clinicians choose caller ID from pool when making outbound calls

### Architecture Principles

- **Provider Abstraction**: TelephonyService interface completely decouples our application from provider-specific implementations
- **Async Processing**: Webhooks queue messages for processing via Bull/Redis queues
- **Real-time Updates**: Leverage existing Socket.IO (ChatGateway) for thread/message broadcasts
- **Minimal Schema Changes**: Reuse existing patterns (soft deletes, JSONB metadata, TypeORM conventions)


### Mock-ups

[CC Platinum Line Figma](https://www.figma.com/design/rVORtmqIMrMbf5tk7Btpbb/LTHC-Mockup?node-id=0-1&t=NgSuAK66aGqHEiqp-1)

- New SMS threads and unified SMS inbox for unread/unassigned external threads
<img width="349" height="737" alt="Screenshot 2025-10-22 at 19 55 40" src="https://github.com/user-attachments/assets/4248d0cd-2e27-4a4a-9a48-8dd0f65bc1f7" />

---

## Database Schema

### Schema Changes to Existing Tables

#### messages (add single column)

```sql
ALTER TABLE public.messages ADD COLUMN external_thread_id int8;
ALTER TABLE public.messages ADD CONSTRAINT fk_messages_external_thread
  FOREIGN KEY (external_thread_id) REFERENCES public.external_threads(id);

-- Constraint: message belongs to EITHER thread_id OR external_thread_id
ALTER TABLE public.messages ADD CONSTRAINT chk_thread_xor_external_thread
  CHECK ((thread_id IS NOT NULL AND external_thread_id IS NULL) OR
         (thread_id IS NULL AND external_thread_id IS NOT NULL));
```

**Reuse existing fields for VOIP:**

- `message_type`: extend enum to include `'sms'`, `'voicemail'`, `'call_log'`
- `content`: SMS body or voicemail transcription
- `sender_id`: user_id for outbound, null for inbound
- `meta_data`: JSONB stores provider message IDs, delivery status, transcription confidence, etc.
- `attachments`: MMS media URLs (existing pattern)

#### message_read_status (no changes needed)

Existing table works perfectly for tracking read status on external thread messages.

---

### New Tables

#### external_threads

```typescript
@Entity('external_threads')
export class ExternalThread {
  @PrimaryGeneratedColumn('increment')
  id: number;

  @Column({ type: 'varchar' })
  threadType: string; // 'voip', 'email'

  @Column({ type: 'varchar' })
  externalContact: string; // E.164 phone number or email

  @Column({ type: 'varchar', nullable: true })
  contactName?: string; // user-assigned label

  @Column({ type: 'varchar', nullable: true })
  fromContact?: string; // OUR number/email for outbound consistency

  @Column({ type: 'int8', nullable: true })
  @ManyToOne(() => Facility)
  facilityId?: number;

  @Column({ type: 'int8', array: true, default: [] })
  patientIds: number[]; // array of linked patient IDs

  @Column({ type: 'varchar', default: 'open' })
  status: string;

  @Column({ type: 'int8', nullable: true })
  @ManyToOne(() => User)
  assigneeId?: number;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @Column({ type: 'int8' })
  @ManyToOne(() => User)
  creatorId: number;

  @Column({ type: 'int8', nullable: true })
  @ManyToOne(() => User)
  updaterId?: number;

  @DeleteDateColumn()
  deletedAt?: Date; // soft delete
}
```

**Indexes:**

```sql
CREATE INDEX idx_external_threads_patient_ids ON external_threads USING GIN (patient_ids);
CREATE INDEX idx_external_threads_type ON external_threads (thread_type);
CREATE INDEX idx_external_threads_contact ON external_threads (external_contact, thread_type);
```

---

#### phone_numbers

```typescript
@Entity('phone_numbers')
export class PhoneNumber {
  @PrimaryGeneratedColumn('increment')
  id: number;

  @Column({ type: 'varchar', unique: true })
  number: string; // E.164 format

  @Column({ type: 'varchar', nullable: true })
  label?: string; // "Main Line", "After Hours"

  @Column({ type: 'varchar', nullable: true })
  providerSid?: string; // Provider's unique identifier for this number

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

---

#### call_logs

TBD if needed...

```typescript
@Entity('call_logs')
export class CallLog {
  @PrimaryGeneratedColumn('increment')
  id: number;

  @Column({ type: 'int8' })
  @ManyToOne(() => ExternalThread)
  externalThreadId: number;

  @Column({ type: 'varchar' })
  direction: string; // 'inbound', 'outbound_bridged'

  @Column({ type: 'varchar' })
  fromNumber: string; // E.164

  @Column({ type: 'varchar' })
  toNumber: string; // E.164

  @Column({ type: 'varchar', nullable: true })
  callerIdNumber?: string; // E.164, for bridged calls

  @Column({ type: 'int', nullable: true })
  durationSeconds?: number;

  @Column({ type: 'varchar' })
  callStatus: string; // 'completed', 'no_answer', 'busy', 'failed'

  @Column({ type: 'varchar', nullable: true })
  providerCallId?: string;

  @Column({ type: 'int8', nullable: true })
  @ManyToOne(() => User)
  initiatedByUserId?: number;

  @CreateDateColumn()
  createdAt: Date;
}
```

---

#### voip_settings

Might wrap into any existing system settings?  Or leave as constants not data driven...

```typescript
@Entity('voip_settings')
export class VoipSettings {
  @PrimaryGeneratedColumn('increment')
  id: number;

  @Column({ type: 'boolean', default: false })
  voipEnabled: boolean;

  @Column({ type: 'text', nullable: true })
  autoReplyMessage?: string;

  @Column({ type: 'int', default: 20 })
  escalationTimerMinutes: number;

  @Column({ type: 'varchar', array: true, default: [] })
  escalationPhoneNumbers: string[]; // E.164 numbers

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

---

## System Flows

### Inbound SMS Flow

1. **Webhook Reception**

   - Provider sends POST to `/api/webhooks/{provider}/sms`
   - Validate webhook signature (provider-specific HMAC/verification)
   - Queue to Bull/Redis (`inbound_sms_queue`)
   - Return 200 OK immediately

2. **Message Processing** (async worker)
   - Find or create external thread (by `from` number + thread type)
   - Lock in `from_contact` (our number they texted) for consistency
   - Check if auto-reply sent (query messages for existing auto-reply in thread)
   - If not sent: send auto-reply via TelephonyService
   - Create message record:
     - `external_thread_id` = thread ID
     - `message_type` = 'sms'
     - `sender_id` = null (external sender)
     - `content` = SMS body
     - `meta_data` = { provider_message_id, from/to numbers, delivery status }
   - Start escalation timer for this thread, if not already running (20 min)
   - Trigger push notifications to enabled clinicians
   - Broadcast via Socket.IO (ChatGateway)

---

### Inbound Call (Voicemail) Flow

1. **Call Webhook**

   - Provider sends POST to `/api/webhooks/{provider}/call`
   - System responds with provider-specific call control format (TwiML, NCCO, etc.) to play IVR message
   - IVR instructs caller to leave voicemail or hang up and text

2. **Recording Webhook**
   - After caller leaves voicemail, provider sends recording URL
   - Download audio file, upload to S3 for permanent storage
   - Call TelephonyService.getTranscription()
   - Find/create external thread (same logic as SMS)
   - Create message record:
     - `message_type` = 'voicemail'
     - `content` = transcription text (or empty if transcription fails)
     - `attachments` = [{ type: 'audio', url: s3_signed_url, duration }]
     - `meta_data` = { recording_url, duration, transcription_confidence, provider_call_id }
   - Send auto-reply if not already sent
   - Create entry in `call_logs`
   - Start escalation timer, notify clinicians

---

### Outbound SMS Flow

1. **From Thread UI**

   - Clinician types message, clicks "Send"
   - POST `/api/external-threads/{id}/messages`
   - Backend selects `from_number`:
     - Use `thread.from_contact` if set (consistency)
     - Otherwise use number recipient last contacted
   - Call TelephonyService.sendSms()
   - Create message record:
     - `message_type` = 'sms'
     - `sender_id` = current user
     - `meta_data` = { provider_message_id, delivery_status: 'pending' }
   - Provider webhook updates delivery status async
   - Broadcast to Socket.IO

2. **New Thread Creation**
   - "New SMS" button → modal with phone input
   - Create new external thread, set `external_contact`
   - Follow same send flow

---

### Call Bridging (Outbound) Flow

1. **Initiation**

   - Clinician clicks "Call" button in thread
   - Modal asks for:
     - Clinician's phone number (E.164)
     - Caller ID to display (dropdown from `phone_numbers`)
   - POST `/api/external-threads/{id}/bridge-call`

2. **Backend**
   - Call TelephonyService.initiateBridgeCall()
   - Provider initiates two-leg call:
     - **Leg A**: Call clinician (private/restricted caller ID)
     - **Leg B**: When clinician answers, dial destination (show selected caller ID)
   - Create entry in `call_logs` (status: 'initiated')
   - Webhooks update call status throughout lifecycle
   - On completion: create message record (type: 'call_log', content: "Call duration: Xs")

---

## Thread Management

### Thread Organization

- **Patient Linking**: UI allows searching/linking multiple patients → updates `patient_ids` array
- **Facility Linking**: Dropdown to set `facility_id`
- **Contact Naming**: Assign friendly name → updates `contact_name`
- **Assignment**: Use `assignee_id` (same as patient threads)
- **Status**: 'open', 'resolved', 'closed' (filters default view)

### Viewing Threads

- **From Patient**: Query `WHERE patient_ids @> ARRAY[patient_id]`
- **From Facility**: Query `WHERE facility_id = X`
- **Unified Inbox**: UNION query combining patient threads + external threads, sorted by last message

---

## Escalation System

### Timer Logic

- **Trigger**: Unread message arrives in external thread
- **Duration**: 20 minutes (configurable via `voip_settings`)
- **Reset Actions**: Clinician reads thread, sends reply, or resolves thread
- **Escalation Fired**: Send SMS to all `escalation_phone_numbers` with message preview + deep link

### Configuration

Admin UI to manage:

- `escalation_timer_minutes`
- `escalation_phone_numbers` (array)
- `auto_reply_message`
- `voip_enabled` (master on/off)

---

## Integration with Existing System

### Leveraging Current Infrastructure

- **Socket.IO**: Reuse ChatGateway for real-time broadcasts (thread events, message updates)
- **Bull/Redis**: Existing queue infrastructure for async webhook processing
- **TypeORM**: Follow existing entity patterns (snake_case columns, camelCase properties)
- **message_read_status**: Works for external threads without changes
- **Attachments**: MMS media and voicemail audio stored in existing `attachments` JSONB field
- **Soft Deletes**: Use `deleted_at` on external threads (same as patient threads)

### Differentiation in UI

- **Thread List**: Icon/badge to distinguish external vs patient threads
- **Thread Header**: Show phone/email, contact name, linked patients, facility
- **Message Bubbles**: Different styling for VOIP messages

---

## Technical Implementation Notes

### Queue Architecture

- **Queues**: `inbound_sms_queue`, `inbound_call_queue`, `transcription_queue`
- **Workers**: NestJS processors using Bull
- **Retries**: Exponential backoff, dead-letter queue for failures

### Phone Number Handling

- **Storage**: E.164 format (+15551234567)
- **Display**: Regional formatting for UI (555) 123-4567
- **Validation**: libphonenumber library for normalization

### Media Handling

**MMS (Images/Files)**

- **Inbound**: Download from provider webhook, upload to S3, store signed URLs in `attachments` JSONB
- **Outbound**: Upload to S3, pass URL to provider API

**Voice Messages (Voicemail Playback)**

- **Storage**: Download audio from provider, upload to S3 with expiration policy
- **Format**: Store as MP3 or WAV (provider-dependent)
- **UI**: Audio player in thread view alongside transcription text
- **Fallback**: If transcription fails or has low confidence, audio playback is primary interface
- **Attachments**: Store in `attachments` JSONB: `{ type: 'audio', url: 's3://...', duration: 45, mimeType: 'audio/mpeg' }`

### Idempotency

- Check `meta_data.provider_message_id` before inserting messages
- Prevents duplicates from webhook retries

### Security

- **Webhook Validation**: HMAC signature verification (reject invalid)
- **Encryption**: TLS for all API calls, S3 server-side encryption
- **HIPAA**: BAA with telephony provider, audit logging, access controls

---

## Migration Strategy

### Phase 1: Schema

- Create `external_threads`, `phone_numbers`, `call_logs`, `voip_settings` tables
- Add `external_thread_id` column to `messages`
- Run migrations in staging, verify integrity

### Phase 2: Core Features

- Build TelephonyService abstraction + provider implementation
- Webhook handlers for inbound SMS/calls
- External thread CRUD + message creation
- Basic UI (separate from patient threads initially)

### Phase 3: Integration

- Call bridging
- Escalation system
- Admin config UI
- Unified inbox view (patient + external threads)

### Phase 4: Rollout

- Soft launch with pilot customer
- Monitor metrics, gather feedback
- Full rollout

---

## Provider Abstraction Layer

### Why Provider Abstraction is Critical

**Given past reliability issues with VOIP providers, we MUST design the system to make provider swapping frictionless.** All provider-specific code must be isolated behind a standard interface to prevent vendor lock-in.

### TelephonyService Interface

All provider implementations must fulfill this contract:

```typescript
interface TelephonyService {
  // SMS Operations
  sendSms(
    from: string,
    to: string,
    body: string,
    mediaUrls?: string[],
  ): Promise<string>; // returns provider message_id
  getMessageStatus(messageId: string): Promise<DeliveryStatus>;

  // Call Operations
  initiateBridgeCall(
    clinicianNumber: string,
    destination: string,
    callerId: string,
  ): Promise<string>; // returns provider call_id
  getCallStatus(callId: string): Promise<CallStatus>;

  // Voicemail/Recording Operations
  getTranscription(
    recordingUrl: string,
  ): Promise<{ text: string; confidence: number }>;

  // Utility Operations
  validateNumber(phoneNumber: string): string; // returns normalized E.164 format

  // Webhook Operations
  verifyWebhookSignature(
    payload: any,
    signature: string,
    secret: string,
  ): boolean;
  normalizeInboundSmsWebhook(providerPayload: any): NormalizedSmsEvent;
  normalizeInboundCallWebhook(providerPayload: any): NormalizedCallEvent;
  generateIvrResponse(message: string): any; // returns provider-specific format (TwiML, NCCO, XML, etc.)
}
```

### Normalized Event DTOs

Decouple internal processing from provider-specific webhook formats:

```typescript
export class NormalizedSmsEvent {
  providerMessageId: string;
  from: string; // E.164
  to: string; // E.164
  body: string;
  mediaUrls?: string[];
  timestamp: Date;
}

export class NormalizedCallEvent {
  providerCallId: string;
  from: string; // E.164
  to: string; // E.164
  callStatus: string; // normalized: 'ringing', 'answered', 'completed', 'failed'
  timestamp: Date;
}
```

### Implementation Guidelines

1. **Provider-Specific Code Isolation**

   - All provider implementations live in separate modules (e.g., `src/voip/providers/twilio/`, `src/voip/providers/bandwidth/`)
   - Provider selected via environment variable (e.g., `VOIP_PROVIDER=twilio`)
   - Use NestJS dependency injection to inject the active provider seamlessly

2. **Application Code Must Be Provider-Agnostic**

   - Controllers, services, and workers should NEVER import provider-specific classes directly
   - Always inject `TelephonyService` interface
   - Store provider-agnostic data in database (use `meta_data` JSONB for provider-specific details)

3. **Configuration**
   - Provider credentials stored in environment variables
   - Webhook URLs follow pattern: `/api/webhooks/{provider}/{event_type}`
   - Single provider active at a time (no multi-provider support needed)

### Benefits of This Approach

- **Easy Swapping**: Change env var → redeploy → done
- **No Vendor Lock-in**: Provider-specific details isolated to small modules
- **Future-Proof**: Can evaluate new providers without refactoring core application logic
- **Testability**: Mock implementation for testing without provider dependencies

---

## Open Questions

1. **Cost Tracking**: Display phone costs in admin UI only, or integrate with billing?
2. **Clinician Phone Storage**: Allow saving clinician's phone number (encrypted), or always manual entry?
3. **Transcription Fallback**: If transcription fails, retry or notify "transcription unavailable"?
4. **Thread Merging**: Allow merging duplicate external threads, or just delete?
5. **International**: Support non-US numbers, or US-only for MVP?
6. **Rate Limiting**: Limit outbound SMS per clinician to prevent spam?

---

## Future Enhancements (Post-MVP)

- Scheduled messaging
- Message templates library
- Multi-language IVR support
- Analytics dashboard (response times, call volumes)

---

## Summary

**Key Design Principles:**

- **Separation of Concerns**: External threads separate from patient threads (no breaking changes)
- **Flexible Patient Linking**: PostgreSQL arrays for 0-to-many patient associations
- **Minimal Schema Impact**: Single column added to messages, reuse JSONB for metadata
- **Provider Agnostic**: Abstraction layer enables frictionless provider swapping (critical due to past reliability issues with Vonage for example)
- **Async & Scalable**: Queue-based processing, real-time Socket.IO updates
- **HIPAA Compliant**: Encryption, audit logging, BAA with provider
