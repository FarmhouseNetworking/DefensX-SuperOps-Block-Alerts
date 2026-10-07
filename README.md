# DefensX-SuperOps-Block-Alerts

An **n8n workflow** that watches one **DefensX** URL group across every customer and opens a **SuperOps** ticket when a machine tries to reach a site on it. It checks every 5 minutes.

It is built for a global block list, such as a list of known malware domains, where a hit is not routine web filtering. Something on that machine tried to reach a known bad destination, and a technician should look at it soon.

```
 DefensX Partner API                      n8n (every 5 min)                         SuperOps
 --------------------                     ------------------                        --------
 URL group entries   --->  build matchers
 URL logs (browser)  --->  per customer:  keep blocked rows that match the group
 DNS logs (agent)    --->                 roll up to one hit per user + site
 agents + children   --->  device names   drop hits already ticketed (24 h)   --->  createTicket
                                                                                    (one per client per run)
```

| File | Purpose |
|---|---|
| `defensx-url-group-block-alerts.json` | The workflow. Import it into n8n. |

---

## What the workflow does

1. Finds the URL group by name on your DefensX partner account and loads its entries.
2. Lists every enabled DefensX customer.
3. For each customer, pulls the **URL logs** (browser extension) and the **DNS logs** (agent) since the last run.
4. Keeps only blocked requests whose host or URL is in the group. Exact hostnames, `*.domain.com` wildcards and full URLs with a path are all supported.
5. Rolls the matches up to one hit per user and site, with a count and first/last seen times.
6. Drops any user and site pair that was already ticketed in the last 24 hours.
7. Looks up the device name for each hit.
8. Matches the DefensX customer to a SuperOps client by name and creates **one ticket per client per run**.
9. Records what was ticketed, so the next run does not repeat it.

Each ticket contains a table with the time, user ID, device, site, number of blocks, source (browser or agent) and category. Sites are written as `example[.]com` so nobody clicks one by accident, and query strings are removed so session tokens never end up in a ticket.

DefensX has no webhook for blocks, and its logs do not record which URL group caused a block. That is why the workflow polls and does the matching itself. A hit therefore means "blocked, and the site is in the group".

---

## Requirements

- **n8n**, self-hosted. Tested on **2.1.4**.
- A **DefensX Partner API** key (DefensX portal, API Keys page).
- A **SuperOps** API token that can list clients and create tickets, plus your SuperOps subdomain.
- The URL group must be visible on your DefensX partner account (a global/public group), or you need the ID of the customer that owns it.

---

## Step 1: Create the credentials in n8n

Create two credentials of type **Bearer Auth**:

| Credential | Token |
|---|---|
| DefensX API | Your DefensX Partner API key |
| SuperOps API | Your SuperOps API token |

Keys belong in n8n credentials only. Do not paste them into the workflow.

## Step 2: Import the workflow

1. In n8n, create a new workflow and choose **Import from File**.
2. Select `defensx-url-group-block-alerts.json`.
3. Attach the **DefensX API** credential to the 7 DefensX HTTP Request nodes and the **SuperOps API** credential to the 3 SuperOps nodes. n8n marks each node that is missing one.

## Step 3: Fill in the Config node

Everything you need to change is in the **Config** node at the start of the workflow.

| Setting | Default | What it does |
|---|---|---|
| `urlGroupName` | `CHANGE_ME` | Exact name of the DefensX URL group to watch. Not case-sensitive. |
| `superopsSubdomain` | `CHANGE_ME` | Your SuperOps subdomain, sent as the `CustomerSubDomain` header. |
| `ticketRequestType` | `CHANGE_ME` | One of **your** SuperOps ticket type names, spelled exactly as SuperOps shows it. See [Lessons learned](#lessons-learned). |
| `fallbackSuperOpsAccountId` | empty | SuperOps `accountId` of the client that receives a ticket when a DefensX customer name has no match. If this is empty and a customer does not match, the run stops with an error and creates no tickets. Keep the type as **String**: SuperOps account IDs are too large for a number and would be rounded. |
| `customerNameMap` | `{}` | JSON for customers named differently in the two systems, for example `{"Acme": "Acme Holdings Inc"}`. |
| `groupOwnerCustomerId` | empty | Leave empty when the group lives on your partner account. Otherwise the ID of the customer that owns it. |
| `suppressHours` | `24` | How long the same user and site stay quiet after a ticket is created. |
| `overlapMinutes` | `5` | How far each run reaches back past the previous one, to catch logs that arrive late. |
| `firstRunLookbackMinutes` | `20` | How far back the first run (and every manual run) looks. |
| `maxLookbackHours` | `24` | Upper limit on how far back a run looks after an outage. |
| `includeConsented` | `true` | Also report blocks the user chose to click through. |
| `defangUrls` | `true` | Write sites as `example[.]com` in the ticket. |
| `defensxBaseUrl` | `https://cloud.defensx.com/api/partner/v1` | DefensX Partner API base URL. |

Also set the `CustomerSubDomain` header on the **DIAG Introspect CreateTicketInput** node. That node is not connected to anything, so it cannot read Config.

Ticket times are shown in US Pacific time. To change that, edit the `TZ` constant and the word "Pacific" in the **Build Tickets** node.

## Step 4: Test it

1. Run the **Manual Test** trigger. If the group name is wrong, **Resolve URL Group** stops and lists the groups it did find.
2. From a test machine, visit a site in the group. Wait a minute and run **Manual Test** again.
3. Open **Match Blocked Hits** and check `hits` for that customer. `urlActionCounts` shows every action value DefensX returned.
4. Confirm the ticket landed on the right SuperOps client.
5. Activate the workflow.

> **Manual runs do not remember anything.** The time cursors and the list of ticketed hits are stored in n8n workflow static data, which is only saved for active, scheduled runs. Every manual run looks back `firstRunLookbackMinutes` and will ticket the same hit again.

---

## Users and devices in the ticket

- **User ID:** the ticket shows the DefensX user ID exactly as the log records it. The workflow does not look up user names. To find the person, start from the device.
- **Device:** the hostname comes from `/customers/{id}/agents_with_children`. Browser logs carry the ID of a per-user child agent, so the workflow matches both device IDs and child agent IDs to the device's hostname.
- If a log row has no user, the ticket shows the user ID of the device's child agent, or `No user recorded`.

## How customers are matched to SuperOps clients

Names are compared after lower-casing, removing punctuation and dropping suffixes such as Inc, LLC and Ltd. So `Acme, Inc.` matches `ACME Inc`.

- A name in `customerNameMap` overrides the automatic match.
- If two SuperOps clients normalise to the same name, the match is treated as ambiguous and not used.
- An unmatched customer goes to the fallback client, with `[UNMAPPED CLIENT]` in the subject.

---

## Lessons learned

1. **The logs do not say which group blocked a request.** You get the URL, the action and a category. The matching has to be done by the workflow.
2. **Browser logs are only half the picture.** A `curl` from a command prompt never appears in the URL logs, because the browser extension only sees browser traffic. Scripts, services and anything running as SYSTEM are caught by the agent and written to the DNS logs. The workflow reads both.
3. **DNS logs are large.** One customer returned more than 5,700 rows in a 20-minute window, which is over the 5,000-row page limit. Both log requests are paginated.
4. **SuperOps can require `requestType` even though the schema marks it optional.** Without it, `createTicket` fails with `mandatory_validation_failed`.
5. **The next error names a field you never sent.** With a `requestType` your tenant does not have, SuperOps returns `referred_value_does_not_exist` for `ticketType`. The field was renamed and the error still uses the old name. Ticket types are customisable per tenant, so the value must be one of your own.
6. **Ask the schema for descriptions, not just types.** This query explains every `createTicket` field, including where the valid `requestType` values come from:

   ```json
   { "query": "query { __type(name: \"CreateTicketInput\") { inputFields { name description } } }" }
   ```

7. **A token that can create tickets may not be able to read them.** With our token, the ticket list returned a total count and no rows once ticket type fields were selected, and single-ticket lookups returned `forbidden`. Do not plan on reading existing tickets to discover valid values.
8. **Browser logs carry a child agent ID, not the device's agent ID.** `/agents` lists devices only, so a browser hit never matched and the ticket showed a raw ID. `/agents_with_children` returns each device with its per-user child agents, and matching on both fixed it.
9. **We could not turn user IDs into names.** In our tenant, the user IDs on these log rows were not in the response from `/customers/{id}/users`. We did not find out why. The ticket shows the ID, and the device name identifies the machine.
10. **SuperOps account IDs do not fit in a JavaScript number.** A 19-digit ID stored as a number is silently rounded to a different value. Keep account IDs as strings everywhere.
11. **Keep state writes and failures in separate nodes.** **Record Notified** saves what was ticketed and never throws. **Fail If Any Ticket Failed** raises the error afterwards. The record of created tickets is written before the run is marked as failed, so one rejected ticket does not cause the others to be sent again.

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `Expected exactly 1 URL group named ...` | The name in `urlGroupName` does not match. The error lists the groups that exist. If the group belongs to a customer, set `groupOwnerCustomerId`. |
| The run ends at **Collect Results** with no output | Normal. No customer had a new hit, so there is nothing to ticket. |
| Blocks exist in DefensX but no hits are found | Check `urlActionCounts` in **Match Blocked Hits**. Blocked rows are recognised by an action containing "block". Also confirm the site is really in the watched group. |
| `mandatory_validation_failed` on `requestType` | `ticketRequestType` is empty. |
| `referred_value_does_not_exist` on `ticketType` | `ticketRequestType` is not one of your tenant's ticket type names. Copy the name from SuperOps exactly. |
| `No SuperOps client matched DefensX customer(s)` | Set `fallbackSuperOpsAccountId` or add the customer to `customerNameMap`. No tickets are created until this is fixed, and the hits are retried. |
| The Device column says `Not found in DefensX agent list (ID ...)` | The agent lookup failed or the agent has been removed from DefensX. Open **Build Tickets** in that execution and read the `lookup` object on the item. |
| `DefensX URL log fetch failed for every customer` | The DefensX key or base URL is wrong, or the API is unreachable. |
| Duplicate tickets for the same hit | Usually manual test runs. It can also happen if one run takes longer than the schedule interval and two runs overlap. |

---

## Security notes

- API keys are stored only in n8n credentials. The workflow file contains none.
- Tickets include user IDs, device names and blocked sites. Treat them as client data.
- Query strings and fragments are removed from URLs before they are stored or written to a ticket.
- The workflow only reads from DefensX, and in SuperOps it only lists clients and creates tickets. Scope both keys to that.

## License

MIT. See [LICENSE](LICENSE). This project is not affiliated with DefensX, SuperOps or n8n.

---

Built and maintained by [Farmhouse Networking](https://www.farmhousenetworking.com). We offer white-labeled Tier 3 support, RMM as a service and co-managed IT for MSPs. Questions: scott@farmhousenetworking.com
