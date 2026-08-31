# Payment & Subscription System

## Overview

The system uses PayPal's subscription API to handle recurring billing for memberships.

## Membership Tiers

### Trial (Free)
- **Duration:** 7 days
- **Cost:** Free
- **Automatic:** Starts when user registers
- **After Trial:** User prompted to upgrade

### Monthly Plan
- **Cost:** $5/month
- **Billing:** Every 30 days
- **Cancellation:** Anytime

### Yearly Plan
- **Cost:** $30/year
- **Billing:** Every 365 days
- **Cancellation:** Anytime

### Standard Plan (Legacy)
- **Cost:** $15/year
- **Billing:** Every 365 days
- **For:** Existing members

## Subscription Flow

### New User Registration
1. User creates account via `/api/auth/register`
2. Free 7-day trial automatically starts
3. Trial expires after 7 days
4. User receives reminder to choose membership

### Upgrading After Trial
1. User visits membership page
2. Selects Monthly ($5), Yearly ($30), or Standard ($15) plan
3. Clicks "Subscribe"
4. Redirected to PayPal approval page
5. After approval, subscription becomes active

### PayPal Webhook Events

The system listens for these PayPal events:

- **BILLING.SUBSCRIPTION.ACTIVATED** → Update subscription status to `active`
- **BILLING.SUBSCRIPTION.CANCELLED** → Update subscription status to `cancelled`
- **BILLING.SUBSCRIPTION.EXPIRED** → Update subscription status to `expired`
- **BILLING.SUBSCRIPTION.PAYMENT.FAILED** → Log payment failure

## Database Schema

### subscriptions table
```
id                  - Unique subscription ID
user_id             - User who owns subscription
membership_plan_id  - Reference to membership_plans
start_date          - When subscription started
end_date            - When subscription ends (NULL if active)
status              - 'trial', 'active', 'cancelled', 'expired'
paypal_subscription_id - PayPal subscription reference
created_at, updated_at
```

### membership_plans table
```
id              - Unique plan ID
name            - Plan name (Trial, Monthly, Yearly, Standard)
price           - Cost in USD
billing_cycle   - 'trial', 'monthly', 'yearly'
description     - User-facing description
```

### payments table
```
id                      - Unique transaction ID
user_id                 - User who made payment
subscription_id         - Related subscription
amount                  - Amount charged
status                  - 'pending', 'completed', 'failed', 'refunded'
paypal_transaction_id   - PayPal reference
created_at, updated_at
```

## API Usage Examples

### Check Current Subscription
```bash
curl -X GET http://localhost:5000/api/subscriptions/current \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

### Get Available Plans
```bash
curl -X GET http://localhost:5000/api/subscriptions/plans
```

### Subscribe to a Plan
```bash
curl -X POST http://localhost:5000/api/payments/subscribe \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"planId": 2}'
```

Response includes PayPal approval link for user to click.

## Testing in Sandbox

1. Use PayPal sandbox credentials
2. All charges are test transactions
3. No real money is involved
4. Test different payment scenarios

## Going to Production

When ready for real payments:

1. Get PayPal Production credentials
2. Change `PAYPAL_MODE=production` in `.env`
3. Update return/cancel URLs to production domain
4. Register production webhook in PayPal dashboard
5. Update any hardcoded URLs
6. Test with small real transactions first

## Security Notes

- **Never** expose PayPal Client Secret in frontend
- Always validate payments on backend
- Verify webhook signatures from PayPal
- Use HTTPS in production
- Store sensitive data in environment variables

## Troubleshooting

**"Plan not found" error?**
- Ensure membership_plans were seeded in database
- Check that plan has PayPal Plan ID stored

**Subscription not activating after payment?**
- Verify webhook is registered in PayPal dashboard
- Check webhook logs in PayPal dashboard
- Ensure webhook handler is working (see logs)

**User stuck in trial?**
- Manually update subscription status if needed
- Add reminder email to convert trial to paid
