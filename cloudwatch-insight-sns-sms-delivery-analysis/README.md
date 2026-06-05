# SNS SMS Delivery Analysis

This CloudWatch Logs Insights query analyzes SMS delivery success rates, costs, and message types using SNS delivery status logs for comprehensive SMS monitoring and cost optimization.

## Prerequisites
- SNS SMS delivery status logging enabled
- CloudWatch Logs destination configured for SMS delivery logs
- SMS messages sent through SNS topics

## Important: Log Group Selection
**⚠️ Critical Setup Step**: When running this query in CloudWatch Logs Insights, you **MUST select BOTH** SMS log groups:

✅ **Required Log Groups:**
```
sns/REGION/ACCOUNT-ID/DirectPublishToPhoneNumber          (Success logs)
sns/REGION/ACCOUNT-ID/DirectPublishToPhoneNumber/Failure  (Failure logs)
```

**Why both groups?** SMS delivery logs are automatically separated by AWS:
- **Success deliveries** → `DirectPublishToPhoneNumber` 
- **Failed deliveries** → `DirectPublishToPhoneNumber/Failure`

To get complete SMS analytics, select both log groups in the CloudWatch Console.

## Use Cases
- Monitor SMS delivery success rates by message type
- Track SMS costs and optimize spending
- Identify delivery issues and patterns
- Compare performance across different SMS types (promotional vs transactional)
- Generate cost reports for SMS usage

## Query Logic
1. Extracts key fields: timestamp, destination, SMS type, status, and cost
2. Filters for records with delivery information
3. Groups results by SMS type (Promotional/Transactional)
4. Calculates message counts, success/failure counts, and total costs
5. Computes success rate percentage
6. Sorts by total cost to identify highest-cost message types

## SMS Types Analyzed
### Promotional Messages:
- **Lower cost** per message
- **Lower delivery priority** during high traffic
- **Marketing and promotional** content
- **Rate-limited** by carriers

### Transactional Messages:
- **Higher cost** per message  
- **Higher delivery priority** and reliability
- **Critical notifications** (OTP, alerts, confirmations)
- **Better deliverability** rates

## Sample Results
```
smsType          messageCount  successCount  failureCount  totalCostUSD  successRate
Transactional    1,250         1,198         52            18.75         95.84
Promotional      3,420         3,251         169           17.10         95.06
```

## Key Metrics
- **Message Count**: Total SMS messages sent by type
- **Success Count**: Successfully delivered messages
- **Failure Count**: Failed delivery attempts  
- **Total Cost**: Cumulative cost in USD by message type
- **Success Rate**: Percentage of successful deliveries

## Setup Requirements
To enable SMS delivery status logging:

### 1. Enable SMS Delivery Status Logging
Configure an IAM role and set SMS attributes:
```bash
# Set the IAM role for SMS delivery logging
aws sns set-sms-attributes \
    --attributes DeliveryStatusIAMRole=arn:aws:iam::ACCOUNT-ID:role/SNSDeliveryStatusRole
```

### 2. CloudWatch Logs Integration
SMS delivery logs automatically appear in these log groups:
```
sns/REGION/ACCOUNT-ID/DirectPublishToPhoneNumber          (Success logs)
sns/REGION/ACCOUNT-ID/DirectPublishToPhoneNumber/Failure  (Failure logs)
```

### 3. IAM Role Permissions
The IAM role needs CloudWatch Logs permissions:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream", 
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:log-group:sns/*"
    }
  ]
}
```

### 4. Running the Query
**🔑 Critical Step**: In CloudWatch Logs Insights, select **BOTH** log groups:
- ✅ `sns/REGION/ACCOUNT-ID/DirectPublishToPhoneNumber`
- ✅ `sns/REGION/ACCOUNT-ID/DirectPublishToPhoneNumber/Failure`

This ensures you capture both successful and failed SMS deliveries for complete analysis.

## Cost Optimization Insights
- **High promotional volume**: Consider message batching or timing optimization
- **Low transactional success rates**: Review phone number validation
- **Cost per message type**: Balance between promotional and transactional usage
- **Geographic patterns**: Identify regions with higher delivery costs

## Extended Analysis
For deeper insights, modify the query:

### Success Rate by SMS Type
```sql
-- Calculate success rates per SMS type (requires both log groups)
fields @timestamp, delivery.smsType, status
| filter ispresent(delivery.destination)
| stats count() as totalMessages by delivery.smsType, status
| sort delivery.smsType, status
```

### Cost Analysis by Destination Country
```sql
-- Analyze costs by destination country code
fields @timestamp, delivery.destination, status, delivery.priceInUSD
| filter ispresent(delivery.destination) and status = "SUCCESS"
| extend countryCode = substr(delivery.destination, 1, 3)
| stats count() as messages, sum(delivery.priceInUSD) as totalCost, 
        avg(delivery.priceInUSD) as avgCost by countryCode
| sort totalCost desc
```

### Delivery Performance Over Time
```sql
-- Track delivery patterns over time (5-minute intervals)
fields @timestamp, status, delivery.smsType
| filter ispresent(delivery.destination)
| stats count() as totalMessages by bin(5m), delivery.smsType, status
| sort @timestamp desc
```

**💡 Remember**: All extended queries also require selecting both success and failure log groups for complete data coverage.

This query provides essential visibility into SMS delivery performance and costs without requiring additional configuration beyond standard SNS SMS delivery logging.