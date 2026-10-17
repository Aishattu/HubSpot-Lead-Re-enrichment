# HubSpot Lead Re-enrichment

## The problem

A lead can sit in a CRM for months before it becomes sales-ready. By the time it reaches a rep, the data is often out of date:

- **The person has left.** The rep calls or emails someone who no longer works there and burns the first touch.
- **The title is wrong.** They have been promoted or moved roles, so the pitch is aimed at the wrong level.
- **Nobody checks.** Reps trust what HubSpot says, and re-checking LinkedIn by hand for every lead doesn't happen.

## The solution

Every time a contact becomes a **Sales Qualified Lead**, their LinkedIn profile is pulled fresh and compared with the HubSpot record. The result is written to the contact as a note **before** the lead is assigned to a rep, so the rep opens a record that has already been verified.

| Problem | What the workflow does |
|---|---|
| The person has left | Checks whether their current role is still at the HubSpot company, and names where they work now if not |
| The title is wrong | Compares the HubSpot job title with the current LinkedIn title, ignoring reworded titles ("Purchasing Manager" = "Manager, Purchasing") |
| Promotions are missed | Flags a promotion only if the new role is more senior and started in the last 12 months |
| Nobody checks | Runs automatically on every new SQL, no rep action needed |

[**See the animated walkthrough**](https://claude.ai/artifact/CQ1tvV2CqrjXgGuoz2WTyy)

![How it works](docs/how-it-works.svg)

## How it works

**HubSpot workflow**
1. **Trigger:** lifecycle stage becomes Sales Qualified Lead.
2. **Send webhook** to n8n with the contact ID, name, company, job title and LinkedIn URL.
3. **Delay 5 minutes** while n8n does the check.
4. **Rotate record to owner** so the rep gets the lead with the note already on it.

**n8n workflow**
1. **Webhook** receives the contact from HubSpot.
2. **Apify actor** scrapes the person's live LinkedIn profile.
3. **Claude** compares HubSpot and LinkedIn: same person, still at the company, title changed, promoted. Only the 3 most recent roles are sent, to keep token use low.
4. **HTTP request** creates a note on the contact through the HubSpot API.

## Proof of work

**Still at the company.** Title matches, no promotion.

![Note on a contact who is still at the company](images/note-still-at-company.png)

**Left the company.** The note says where they are now, so the rep knows before reaching out.

![Note on a contact who has left](images/note-left-company.png)

**The n8n workflow**

![n8n workflow](images/n8n-workflow.png)

**The HubSpot workflow**

![HubSpot workflow](images/hubspot-workflow.png)

## Files

| File | Purpose |
|---|---|
| `prompt.md` | The comparison prompt and the HubSpot note request used in n8n |
| `docs/how-it-works.svg` | Animated diagram of the two workflows |
| `docs/workflow-animation.html` | Source of the interactive walkthrough |
| `images/` | Screenshots from the live HubSpot and n8n workflows |

## Built with

[HubSpot Workflows](https://knowledge.hubspot.com/workflows) · [n8n](https://n8n.io) · [Apify](https://apify.com) · [Claude](https://www.anthropic.com/claude) · [HubSpot CRM API v3](https://developers.hubspot.com/docs/api/crm/notes)
