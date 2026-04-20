# Email Setup Guide

Email notifications in Zeyaroo are powered by [Resend](https://resend.com) and disabled by default. This guide explains how to enable email for production.

## Overview

The application sends transactional emails for critical events:
- New inquiries, proposals, and counter-proposals
- Milestone funding and completion
- Payment releases

If email is not configured, notifications are silently skipped (no errors). However, users will miss important updates.

## Resend Account Setup

1. **Sign up for Resend**
   Go to [resend.com](https://resend.com) and create an account. The free tier (100 emails/day) is sufficient for beta.

2. **Add and verify your domain**
   - Navigate to **Domains** in the Resend dashboard
   - Add your sending domain (e.g., `phto.app`)
   - Add the required DNS records (SPF, DKIM, DMARC) to your domain's DNS settings
   - Wait for verification (usually takes a few minutes)

3. **Create an API key**
   - Go to **API Keys** in the Resend dashboard
   - Create a new API key with **Send access**
   - Copy the key immediately (it won't be shown again)

## Environment Variables

Add these variables to your `.env` file:

```bash
# Resend email service (optional — emails disabled if not set)
RESEND_API_KEY=re_xxxxxxxxxxxxxxxxxxxxxxxxxxxx
RESEND_FROM_EMAIL=noreply@yourdomain.com
```

**Important:**
- `RESEND_API_KEY` — Your Resend API key from step 3 above
- `RESEND_FROM_EMAIL` — Must match your verified domain (e.g., `noreply@phto.app`)

If `RESEND_API_KEY` is empty or missing, emails will be disabled and the app will log:
```
Email notifications disabled: RESEND_API_KEY not set
```

## Email Templates

The following email notifications are currently implemented:

| Event | Recipient | Subject | Trigger |
|-------|-----------|---------|---------|
| **Inquiry Received** | Creator | "New Inquiry Received - {title}" | Client sends an inquiry to creator |
| **Proposal Sent** | Client | "New Proposal from {creator}" | Creator sends a proposal in response to inquiry |
| **Proposal Accepted** | Creator | "Proposal Accepted - {title}" | Client accepts a proposal (project created) |
| **Counter-Proposal** | Other party | "Counter-Proposal Received - {title}" | Either party sends a counter-proposal |
| **Milestone Funded** | Creator | "Milestone Funded - {title}" | Client funds a milestone (escrow locked) |
| **Milestone Completed** | Client | "Milestone Completed - {title}" | Creator marks milestone as complete |
| **Payment Released** | Creator | "Payment Released - {title}" | Client approves milestone and releases payment |

All templates use HTML formatting and include relevant details (names, amounts, project context).

## Troubleshooting

### Email not sending (no errors in logs)

**Cause:** `RESEND_API_KEY` is not set.
**Fix:** Add the API key to your `.env` file and restart the server.

### "API key invalid" error

**Cause:** The API key is incorrect or has been revoked.
**Fix:**
- Double-check the key in your `.env` file (no extra spaces)
- Generate a new API key in Resend dashboard if needed

### Emails sent but not received

**Cause:** Domain not verified or DNS records not propagated.
**Fix:**
1. Check domain verification status in Resend dashboard
2. Verify DNS records are correctly configured (use `dig` or online DNS checkers)
3. Check recipient spam folder
4. Review Resend logs for delivery status

### "From email does not match verified domain"

**Cause:** `RESEND_FROM_EMAIL` uses an unverified domain.
**Fix:** Update `RESEND_FROM_EMAIL` to match your verified domain in Resend.

### Rate limit errors (429)

**Cause:** Free tier limit (100 emails/day) exceeded.
**Fix:** Upgrade to a paid Resend plan or reduce email volume during testing.

## Testing Email Setup

To verify email is working:

1. **Check startup logs** — should see:
   ```
   Email notifications enabled via Resend (from: noreply@yourdomain.com)
   ```

2. **Trigger a test email** — create an inquiry as a client:
   ```bash
   curl -X POST http://localhost:8080/api/inquiries \
     -H "Authorization: Bearer <token>" \
     -H "Content-Type: application/json" \
     -d '{"creatorId": "...", "title": "Test inquiry", ...}'
   ```

3. **Check creator's inbox** — the inquiry email should arrive within seconds

4. **Review Resend logs** — check delivery status at [resend.com/logs](https://resend.com/logs)

## Production Deployment Checklist

- [ ] Resend account created
- [ ] Domain added and verified (DNS records propagated)
- [ ] API key generated with send access
- [ ] `RESEND_API_KEY` set in production `.env`
- [ ] `RESEND_FROM_EMAIL` matches verified domain
- [ ] Test email sent and received successfully
- [ ] Startup logs confirm email is enabled

## Alternative: Zoho Mail

Currently, the app only supports Resend. If you prefer Zoho Mail for support inboxes or general email, you can:
- Use Zoho for support@ and general business email
- Keep Resend for transactional notifications

Switching to Zoho for transactional email would require code changes (replacing the Resend client with an SMTP or Zoho API client).

## Need Help?

- **Resend docs:** [resend.com/docs](https://resend.com/docs)
- **Resend support:** [resend.com/support](https://resend.com/support)
- **Codebase reference:** `phto-api/internal/notifications/email.go`
