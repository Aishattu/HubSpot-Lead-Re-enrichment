# n8n node contents

## 1. Compare Latest Data to Prev (Claude)

```
Today: {{ $now.toFormat('dd LLL yyyy') }}

HubSpot contact:
Name: {{ $('Webhook').item.json.query['First Name'] }} {{ $('Webhook').item.json.query['last name'] }}
LinkedIn URL: {{ $('Webhook').item.json.query['LinkedIn URL'] }}
Company: {{ $('Webhook').item.json.query['company name'] }}
Job title: {{ $('Webhook').item.json.query['Job title'] }}

LinkedIn profile:
Name: {{ $json.fullname }}
Current company: {{ $json.current_company_name }}
Roles (newest first): {{ JSON.stringify(($json.experience || []).slice(0, 3).map(e => ({ company: e.company, title: e.title, current: e.is_current, start: e.start_date }))) }}

Check:
1. Same person: Does the LinkedIn name match the HubSpot name?
2. Still at company: Is their current role at the HubSpot company? Treat name variations (Inc, Ltd, International) as the same.
3. Title changed: Answer "No" if both titles describe the same job, even if the words are reordered, punctuated, abbreviated or slightly reworded (e.g. "Purchasing Manager" = "Manager, Purchasing", "VP Sales" = "Vice President of Sales"). Answer "Yes" only if it is a different role.
4. Promoted: Answer "Yes" only if the title changed to a more senior role AND the new role started in the last 12 months. In every other case answer "No".

Use only the data given. If missing, write "Not found". No em dashes.

Reply with JSON only, no other text, in exactly this shape:
{
  "profile_match": "Yes" or "No",
  "name": "[name]",
  "still_at_company": "Yes" or "No",
  "current_company": "[company from LinkedIn]",
  "current_title": "[LinkedIn title]",
  "hubspot_title": "[HubSpot title]",
  "title_changed": "Yes" or "No",
  "promoted": "Yes" or "No",
  "promoted_to": "[new title, or empty]",
  "promotion_date": "[month year, or empty]"
}
```

## 2. Parse Result (Code, JavaScript)

```javascript
const content = $input.first().json.content;
const text = content.find(c => c.type === 'text').text;
const r = JSON.parse(text.replace(/```json|```/g, '').trim());

const today = new Date().toLocaleDateString('en-GB', { day: 'numeric', month: 'short', year: 'numeric' });

const lines = [`Re-enrichment check (${today})`, `Name: ${r.name}`];
if (r.profile_match === 'No') {
  lines.push('Profile does not match contact');
} else if (r.still_at_company === 'No') {
  lines.push(`Still at company: No (now at ${r.current_company})`);
} else {
  lines.push('Still at company: Yes');
  lines.push(`Current title: ${r.current_title} (HubSpot had: ${r.hubspot_title})`);
  lines.push(`Title changed: ${r.title_changed}`);
  lines.push(r.promoted === 'Yes'
    ? `Promoted: Yes (to ${r.promoted_to}, ${r.promotion_date})`
    : 'Promoted: No');
}

return [{ json: { ...r, note: lines.join('<br>') } }];
```

## 3. Add Notes to HubSpot (HTTP request)

`POST https://api.hubapi.com/crm/v3/objects/notes` with the HubSpot OAuth2 credential.

```
{
  "properties": {
    "hs_timestamp": "{{ $now.toISO() }}",
    "hs_note_body": {{ JSON.stringify($('Parse Result').item.json.note) }}
  },
  "associations": [
    {
      "to": { "id": "{{ $('Webhook').item.json.query['contact id'] }}" },
      "types": [{ "associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 202 }]
    }
  ]
}
```

## 4. Research New Company (HTTP request to the Claude API)

`POST https://api.anthropic.com/v1/messages`, headers `anthropic-version: 2023-06-01` plus the API key as a Header Auth credential (`x-api-key`).

```
{
  "model": "claude-sonnet-5-5",
  "max_tokens": 2000,
  "output_config": { "effort": "low" },
  "tools": [
    { "type": "web_search_20260209", "name": "web_search", "max_uses": 2 },
    { "type": "web_fetch_20260209", "name": "web_fetch", "max_uses": 3 }
  ],
  "messages": [{
    "role": "user",
    "content": "Research {{ new company }} ({{ website }}). Reply in 2 sentences: what they do and who they sell to. Facts only from their website or reliable sources."
  }]
}
```

The final answer is the text after the last tool result:

```
{{ $('Research New Company').item.json.content.slice($('Research New Company').item.json.content.findLastIndex(b => b.type.endsWith('_tool_result')) + 1).filter(b => b.type === 'text').map(b => b.text).join('') }}
```

## 5. Slack message

```
:rotating_light: *SQL has changed companies*

*{{ $('Parse Result').item.json.name }}* just became a Sales Qualified Lead, but no longer at *{{ $('Webhook (Data from HubSpot)').item.json.query['company name'] }}*.

*Now at:* {{ $('Parse Result').item.json.current_company }}
*New title:* {{ $('Parse Result').item.json.current_title }}
*Website:* {{ $('Get Lead\'s Latest Data').item.json.current_company_website }}

*What they do:*
{{ research answer }}

*Next step:* Find the new contact for this role at {{ $('Webhook (Data from HubSpot)').item.json.query['company name'] }}, and decide whether {{ $('Parse Result').item.json.name }} is worth pursuing at {{ $('Parse Result').item.json.current_company }}.
```
