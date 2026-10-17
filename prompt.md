# Comparison prompt (n8n "Compare Latest Data to Prev" node)

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
1. Same person: Does the LinkedIn name match the HubSpot name? If not, reply only "Profile does not match contact".
2. Still at company: Is their current role at the HubSpot company? Treat name variations (Inc, Ltd, International) as the same. If No, name their current company and stop.
3. Title changed: Answer "No" if both titles describe the same job, even if the words are reordered, punctuated, abbreviated or slightly reworded (e.g. "Purchasing Manager" = "Manager, Purchasing", "VP Sales" = "Vice President of Sales"). Answer "Yes" only if it is a different role.
4. Promoted: Answer "Yes" only if the title changed to a more senior role AND the new role started in the last 12 months. In every other case answer "No".

Use only the data given. If missing, write "Not found". No em dashes.

Reply in this format only:
Re-enrichment check ({{ $now.toFormat('dd LLL yyyy') }})<br>
Name: [name]<br>
Still at company: Yes / No (now at [company])<br>
Current title: [title] (HubSpot had: [title])<br>
Title changed: Yes / No<br>
Promoted: Yes (to [new title], [month year]) / No
```

# Note request (n8n "Add Notes to HubSpot" HTTP node)

`POST https://api.hubapi.com/crm/v3/objects/notes` with the HubSpot OAuth2 credential.

```
{
  "properties": {
    "hs_timestamp": "{{ $now.toISO() }}",
    "hs_note_body": {{ JSON.stringify($('Message a model').item.json.content[1].text.replace(/\n/g, '')) }}
  },
  "associations": [
    {
      "to": { "id": "{{ $('Webhook').item.json.query['contact id'] }}" },
      "types": [{ "associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 202 }]
    }
  ]
}
```
