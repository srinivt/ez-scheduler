# WhatsApp Community Integration Architecture

## Overview

Integration of WhatsApp Business API to enable real-time notifications and community coordination for event registrations.

## Architecture

### WhatsApp Community Structure

```
WhatsApp Community: "Event Name"
├── 📢 Submissions Channel (Private)
│   └── Members: SignupPro Bot + Event Creator
│       Purpose: Private registration notifications
│
└── 💬 Attendees Channel (Optional, Opt-in)
    └── Members: Event Creator + SignupPro Bot + Opted-in Registrants
        Purpose: Event coordination and updates
```

### User Flows

#### Flow 1: Form Creator Creates Event
```
1. User creates form via MCP conversation
2. Form is published with unique URL
3. System creates WhatsApp Community for the event
   - Community name: Event title
   - Submissions Channel created (private)
   - Attendees Channel created (visible if user opts in)
4. Creator is added to both channels
5. Creator receives WhatsApp message with community invite link
```

#### Flow 2: Registrant Submits Form (Opts into Community)
```
1. User fills registration form
2. User checks "Join WhatsApp Community" checkbox
3. Form submitted successfully
4. Bot posts to Submissions Channel:
   "🎉 New registration
   Name: John Doe
   Email: john@example.com
   Phone: +1234567890
   WhatsApp: ✅ Opted in"
5. User receives WhatsApp message:
   "Thanks for registering! You've been invited to the event community."
6. User is added to Attendees Channel
7. User receives confirmation message in community
```

#### Flow 3: Registrant Submits Form (No Community)
```
1. User fills registration form
2. User leaves "Join WhatsApp Community" unchecked
3. Form submitted successfully
4. Bot posts to Submissions Channel:
   "🎉 New registration
   Name: Jane Smith
   Email: jane@example.com
   Phone: +0987654321
   WhatsApp: ❌ Not opted in"
5. User receives email/SMS confirmation (existing flow)
6. Creator sees registration privately
```

## Technical Implementation

### Prerequisites

1. **WhatsApp Business Account**
   - Meta Business verification required
   - WhatsApp Business API access
   - Phone number for bot (+1 number recommended)

2. **Meta Developer Account**
   - App created with WhatsApp Business API enabled
   - Access tokens generated
   - Webhook configured for message responses

3. **Infrastructure**
   - Webhook endpoint for WhatsApp callbacks
   - Redis for message queue/deduplication
   - Background worker for async messaging

### Database Schema Changes

```sql
-- Add WhatsApp Community tracking to signup_forms table
ALTER TABLE signup_forms ADD COLUMN whatsapp_community_id VARCHAR(255);
ALTER TABLE signup_forms ADD COLUMN whatsapp_submissions_channel_id VARCHAR(255);
ALTER TABLE signup_forms ADD COLUMN whatsapp_attendees_channel_id VARCHAR(255);

-- Add WhatsApp opt-in to registrations table
ALTER TABLE registrations ADD COLUMN whatsapp_optin BOOLEAN DEFAULT FALSE;
ALTER TABLE registrations ADD COLUMN whatsapp_phone VARCHAR(20);
ALTER TABLE registrations ADD COLUMN whatsapp_added_to_community BOOLEAN DEFAULT FALSE;

-- Create WhatsApp integration settings table
CREATE TABLE whatsapp_settings (
    id SERIAL PRIMARY KEY,
    access_token TEXT NOT NULL,
    phone_number_id VARCHAR(255) NOT NULL,
    business_account_id VARCHAR(255) NOT NULL,
    webhook_verify_token VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Create message queue table for reliability
CREATE TABLE whatsapp_message_queue (
    id SERIAL PRIMARY KEY,
    registration_id INTEGER REFERENCES registrations(id),
    message_type VARCHAR(50) NOT NULL, -- 'notification', 'invite', 'confirmation'
    recipient_phone VARCHAR(20) NOT NULL,
    message_body TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'pending', -- 'pending', 'sent', 'failed'
    attempts INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    sent_at TIMESTAMP
);

CREATE INDEX idx_whatsapp_queue_status ON whatsapp_message_queue(status);
CREATE INDEX idx_whatsapp_queue_created ON whatsapp_message_queue(created_at);
```

### API Integration

#### WhatsApp Business Cloud API Endpoints

```python
# endpoints to use:
BASE_URL = "https://graph.facebook.com/v18.0"

# Send message
POST {BASE_URL}/{phone_number_id}/messages

# Create community
POST {BASE_URL}/{business_account_id}/communities

# Create channel in community
POST {BASE_URL}/{community_id}/channels

# Add user to channel
POST {BASE_URL}/{channel_id}/participants

# Remove user from channel
DELETE {BASE_URL}/{channel_id}/participants/{user_id}
```

### New Service: `whatsapp_service.py`

```python
"""
WhatsApp Business API integration service
"""

from dataclasses import dataclass
from typing import Optional
import httpx
from sqlmodel import Session

@dataclass
class WhatsAppCommunity:
    community_id: str
    submissions_channel_id: str
    attendees_channel_id: str
    invite_link: str


class WhatsAppService:
    """Handles WhatsApp Business API operations"""

    def __init__(self, db: Session):
        self.db = db
        self.access_token = self._get_access_token()
        self.phone_number_id = self._get_phone_number_id()
        self.base_url = "https://graph.facebook.com/v18.0"

    async def create_event_community(
        self,
        form_title: str,
        creator_phone: str
    ) -> WhatsAppCommunity:
        """
        Create WhatsApp Community for an event with two channels

        Returns community details with channel IDs
        """
        # 1. Create community
        # 2. Create Submissions channel (private)
        # 3. Create Attendees channel (public within community)
        # 4. Add creator to both channels
        # 5. Return community details
        pass

    async def send_registration_notification(
        self,
        registration_id: int,
        form_id: int
    ):
        """
        Send registration details to Submissions channel

        Posts formatted message with registrant details
        """
        pass

    async def add_registrant_to_community(
        self,
        registration_id: int,
        phone: str
    ):
        """
        Add registrant to Attendees channel if they opted in

        Sends invite and adds to channel
        """
        pass

    async def send_template_message(
        self,
        to_phone: str,
        template_name: str,
        parameters: dict
    ):
        """
        Send template message via WhatsApp Business API

        Templates must be pre-approved by Meta
        """
        pass

    def _format_registration_message(self, registration: dict) -> str:
        """Format registration as WhatsApp message"""
        return f"""
🎉 **New Registration**

**Name:** {registration['name']}
**Email:** {registration.get('email', 'Not provided')}
**Phone:** {registration.get('phone', 'Not provided')}
**WhatsApp:** {'✅ Opted in' if registration['whatsapp_optin'] else '❌ Not opted in'}

Registered at: {registration['created_at']}
"""
```

### Background Worker

```python
"""
Background worker for WhatsApp message processing
"""

import asyncio
from sqlmodel import select
from ez_scheduler.models.whatsapp import WhatsAppMessageQueue
from ez_scheduler.services.whatsapp_service import WhatsAppService


async def process_whatsapp_queue():
    """
    Process pending WhatsApp messages

    Runs continuously, processing queue every 5 seconds
    """
    while True:
        # Get pending messages
        # Send via WhatsApp API
        # Update status
        # Handle retries for failures
        await asyncio.sleep(5)


# Add to docker-compose.yml:
# whatsapp-worker:
#   build: .
#   command: uv run python -m ez_scheduler.workers.whatsapp_worker
#   depends_on:
#     - db
#     - redis
```

### Router Changes

Update `routers/forms.py` to handle WhatsApp opt-in:

```python
@router.post("/form/{url_slug}")
async def submit_registration(
    url_slug: str,
    request: Request,
    db: Session = Depends(get_session)
):
    # ... existing validation ...

    # Get WhatsApp opt-in status
    form_data = await request.form()
    whatsapp_optin = form_data.get("whatsapp_optin") == "true"

    # Create registration
    registration = Registration(
        # ... existing fields ...
        whatsapp_optin=whatsapp_optin,
        whatsapp_phone=phone if whatsapp_optin else None
    )
    db.add(registration)
    db.commit()

    # Queue WhatsApp notification (async)
    await queue_whatsapp_notification(registration.id, form.id)

    if whatsapp_optin and phone:
        await queue_community_invite(registration.id, phone)

    return {"success": True, "message": "Registration successful!"}
```

## Message Templates

WhatsApp requires pre-approved templates for business-initiated messages.

### Template 1: Registration Confirmation (Opted In)
```
Name: registration_confirmation_with_community
Category: TRANSACTIONAL

Message:
Thanks for registering for {{1}}! 🎉

You've been invited to join the event's WhatsApp Community where you can:
• Get updates from the organizer
• Coordinate with other attendees
• Ask questions about the event

Click here to join: {{2}}

Event Details:
📅 {{3}}
📍 {{4}}
```

### Template 2: Registration Confirmation (No Community)
```
Name: registration_confirmation_simple
Category: TRANSACTIONAL

Message:
Thanks for registering for {{1}}! 🎉

Event Details:
📅 {{2}}
📍 {{3}}

You'll receive updates via email/SMS.
```

### Template 3: New Registration Notification (to Creator)
```
Posted directly to Submissions Channel (doesn't need template)
```

## Security & Privacy

### Privacy Considerations

1. **Phone Number Visibility**
   - Attendees Channel: Phone numbers visible to all members
   - Submissions Channel: Only creator + bot see all details
   - Clear disclosure in opt-in checkbox

2. **Data Minimization**
   - Store only necessary WhatsApp identifiers
   - Don't duplicate full registration in WhatsApp messages
   - Allow users to leave community anytime

3. **Consent Management**
   - Explicit opt-in required for community
   - Terms updated to include WhatsApp usage
   - Users can opt-out via WhatsApp settings

### Security Measures

1. **Webhook Verification**
   - Verify Meta webhook signatures
   - Use verify token for initial setup
   - Validate all incoming payloads

2. **Token Management**
   - Store access tokens encrypted
   - Rotate tokens regularly
   - Use environment variables, never hardcode

3. **Rate Limiting**
   - Respect WhatsApp API rate limits (80 msg/sec)
   - Queue messages to avoid throttling
   - Implement exponential backoff for retries

## Cost Estimation

### WhatsApp Business API Pricing (as of 2024)

- **Service conversations** (business-initiated): $0.0047 per conversation
- **Free tier**: 1,000 conversations/month
- **Utility conversations** (user-initiated): $0.0033 per conversation

### Cost per Event (100 registrations)

```
Scenario: Event with 100 registrations, 60% community opt-in

1. Community creation: Free
2. Creator notification setup: 1 conversation = $0.0047
3. Registration notifications to Submissions Channel: 100 messages (free within community)
4. Community invites to opted-in users: 60 conversations = 60 × $0.0047 = $0.282
5. Ongoing updates in channels: Free (unlimited messages within community)

Total cost per event: ~$0.29 for 100 registrations with 60% opt-in
```

### Monthly Cost Projection

```
Assuming:
- 50 events/month
- 100 registrations per event average
- 60% community opt-in rate

Monthly WhatsApp costs: 50 × $0.29 = $14.50/month

Free tier covers: 1,000 conversations = ~34 events
Paid costs for remaining 16 events: ~$4.64/month
```

**Conclusion: Very affordable, especially with free tier**

## Implementation Timeline

### Phase 1: Foundation (3-4 days)
- [ ] Set up WhatsApp Business Account
- [ ] Get Meta Business verification
- [ ] Create and configure WhatsApp app
- [ ] Database migrations
- [ ] Basic `WhatsAppService` implementation

### Phase 2: Core Integration (4-5 days)
- [ ] Community creation on form publish
- [ ] Submissions Channel notifications
- [ ] Message queue system
- [ ] Background worker setup

### Phase 3: Community Features (3-4 days)
- [ ] Attendees Channel invites
- [ ] Opt-in handling in forms
- [ ] Template message approvals
- [ ] Webhook endpoint for responses

### Phase 4: Testing & Deployment (2-3 days)
- [ ] End-to-end testing
- [ ] Error handling and retries
- [ ] Monitoring and logging
- [ ] Production deployment

**Total: 12-16 days**

## Rollout Strategy

### Phase 1: Internal Testing
- Test with 2-3 internal events
- Verify community creation works
- Test notifications and invites

### Phase 2: Limited Beta
- Offer to 10-20 trusted users
- Gather feedback
- Monitor costs and performance

### Phase 3: Opt-in Feature
- Make WhatsApp integration optional
- Form creators can enable/disable
- Default: disabled

### Phase 4: General Availability
- Enable for all users
- Monitor adoption rate
- Optimize based on usage patterns

## Success Metrics

- **Adoption Rate**: % of forms with WhatsApp enabled
- **Opt-in Rate**: % of registrants joining community
- **Engagement**: Messages sent per community
- **Reliability**: Message delivery success rate (target: >99%)
- **Cost per Event**: Stay within budget projections
- **User Satisfaction**: NPS from creators and registrants

## Alternative Approaches Considered

### 1. Simple WhatsApp Notifications (No Communities)
- Simpler implementation
- No coordination features
- Less engaging for attendees

### 2. Telegram Groups
- More developer-friendly API
- Free unlimited messages
- Smaller user base than WhatsApp

### 3. Discord Servers
- Great for tech-savvy users
- Less familiar for general public
- More features than needed

**Decision: WhatsApp Communities** - Best balance of familiarity, features, and reach

## Open Questions

1. Should creators be able to customize community/channel names?
2. How to handle users who block the bot?
3. Should we support multiple languages for templates?
4. What happens to communities after event date passes?
5. Should we archive communities or delete them after X days?

---

**Next Steps:**
1. Get approval for WhatsApp Business Account setup
2. Begin Phase 1 implementation
3. Create message templates and submit for approval
4. Set up development webhook endpoint
