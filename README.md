### OverviewOverview

A web hook is a simple event-notification system. When an event occurs in SAIVA, a payload of JSON data containing information representing a report is sent via POST to a specified endpoint URL over HTTPS.

Each delivery will include a Saiva-Event-ID header. This will contain an unique id associated with this delivery.

## Creating a web hook

Web hooks have three components. A name, an endpoint URL, and a secret key.

Name (user-defined): The name is simply a reference for you. It can contain any string value.

Endpoint URL (user defined): This is the URL to which we will send the POST data when the event occurs. This must be a valid URL and must be https. 

Secret Key (auto-generated): This secret key will be used to generate a signature header that you may use to verify the hook data was sent from Saiva. The secret key will be used in conjunction with the web hook’s payload to generate a digital signature. 

Enabled (yes/no): Allows the user to keep the web hook definition but disable calls to the endpoint. This switch will also let the user know if the web hook was disabled due to failure 

Web hook should return 200 Ok if authentication succeeded AND payload received ok.  

## Content settings 

Report Type (single selection): This setting is used to define which report type will be contained in the payload (in v1 - daily_risk_report) 

For Report Type = daily_risk_report:

- QM Type (single selection): This setting is used to define which QM report will be generated (in v1 - rth, falls or wounds)

- Facility (multi-selector): Facility selector. Selecting a facility

For other future report types (i.e. cross-facility_report) content settings may vary TBD

User may define multiple web hooks for daily_risk_report as long as settings do not overlap. For example, multiple web hooks can exist for the same Report Type daily_risk_report only if a different QM type is set or list of different facilities selected. Otherwise, duplicate web hook error is shown upon save. 

## Saving & Testing

When the web hook is saved or updated, Saiva will attempt to ping the web hook URL with a payload consisting of most recent day’s report. 

If this ping fails (200 Ok not returned), Saiva will create the web hook in a disabled state and notify  that the ping has failed. 

## Service Level Agreement

For daily_risk_report, Web hook will be fired once a day by 8:00a ET. SLA is successful only if Web hook call is successful.

## Authentication

SAIVA uses a digital signature which is generated using the secret key entered when creating a web hook and the body of the web hook’s request. This data is contained within the Signature header.

`HMAC-SHA256 = {Base64(HmacSHA256(body_bytes, CLIENT_SECRET))}`

The header contains: the SHA algorithm used to generate the signature, a space, and the signature. To verify the request came from Saiva, compute the HMAC digest using your secret key and the body and compare it to the signature portion (after the space) contained in the header. If they match, you can be sure the web hook was sent from Saiva

`Signature: sha256 HMAC-SHA256`

i.e. Signature: sha256 37d2725109df92747ffcee59833a1d1262d74b9703fa1234c789407218b4a4ef

On the receiving end, this is how you verify the received HMAC (JS example):

```javascript
const verifyHmac = (
  receivedHmacHeader: string,
  receivedBody: any,
  secret: string
): boolean => {
  const hmacParts = receivedHmacHeader.split(' ')
  const receivedHmac = hmacParts[1]
  const hash = crypto
    .createHmac('sha256', secret)
    .update(receivedBody)
    .digest('hex')
  return receivedHmac == hash
}
```

Payload Description (daily_risk_report)

The payload will contain an array of report objects FacilityRTHRiskReport (see SAIV-2280)  one per facility

```json
{
  "version": "v1",
  "web_hook_name": "Sample RTH Daily Report For 10 Facilities", // From user web hook setting
  "report_type": "daily_risk_report",
  "report_date": "02/13/2023",
  "client": "Sample",
  "payload_length": 10,
  "payload": [
      {
        // 1st facility risk report as FacilitRiskReport object
        // Payload identical to sFTP facility export see SAIV-2280
        // version, client and report_date should be identical to the above
      },
      ...
      {
        // 10th facility risk report FacilityRTHRiskReport object
      }
    ]
}
```

## JSON Schema (daily_risk_report)

[DailyRiskReport schema](https://github.com/saivaai/webhook-integration/blob/dev/schema/DailyRiskReport.json "DailyRiskReport schema")

## Retry policy

In the event of a failed web hook request (due to timeout, a non HTTP 200 response, or network issues), Saiva will attempt a maximum of 5 retries according to the formula:

```python
RETRY_DELAY_MINUTES = [1, 15, 60, 120, 240]

sidekiq_retry_in do |index, _exception|
  seconds_delay = (RETRY_DELAY_MINUTES[index] || RETRY_DELAY_MINUTES.last) * 60
  offset = rand(30) * (index + 1)
  seconds_delay + offset
end

```


This formula increases the amount of time between each retry, while assigning a random number of seconds to avoid consistent failures from overload or contention.

Saiva will attempt 5 retries over the course of 7 hours. When all attempts have been exhausted, the web hook will be disabled and an error message will be sent to the org administrators via email. 

The table below outlines the estimated wait time for each retry request, assuming that rand(30) always returns 0.

|  Retry number  | Next Retry in  |  Total waiting time |
| ------------ | ------------ | ------------ |
| 1 | 1m   | 0h 1m  |
|  2 | 15m  | 0h 16m  |
|  3 | 60m  | 1h 16m  |
|  4 | 120m  | 3h 16m  |
|  5 | 240m  | 7h 16m  |

