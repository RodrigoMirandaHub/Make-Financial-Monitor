# Make Financial Monitor

An automation built in [Make](https://www.make.com) (formerly Integromat) that monitors the USD/BRL exchange rate on a schedule, branches on real business logic, and takes different actions depending on the result — including generating an AI-written recommendation with an LLM. 

Built in preparation for an Automation Manager (No-code/Low-code) interview, as a way to get hands-on with Make after primarily working in N8N.

## What it does

<img width="1687" height="877" alt="Screenshot_43" src="https://github.com/user-attachments/assets/583939cc-91b9-47c7-bc11-e25eea5bae57" />


1. **Schedule** — runs every 10h30 minutes
2. **HTTP** — fetches the live USD/BRL rate from a public exchange rate API
3. **Router** — branches on whether the rate is above or below a 5.10 threshold (a real either/or, not two parallel paths)
   - **Rate > 5.10**: calls the Groq API (LLaMA 3.3) to generate a short business recommendation, then emails it via Gmail
   - **Rate ≤ 5.10**: creates a Salesforce Lead with the rate embedded in the description
4. **Error handling** — both external API calls (exchange rate + Groq) have automatic retry logic, since these are the steps most likely to hit transient failures like rate limits or timeouts

<img width="1528" height="374" alt="Screenshot_41" src="https://github.com/user-attachments/assets/b23d3f4f-706e-4b2f-b0a1-cb08b99a88ef" />

<img width="1835" height="864" alt="Screenshot_45" src="https://github.com/user-attachments/assets/beadf689-ef5c-40c8-b923-4ce8c928a59d" />


## Stack

- Make (Scenarios, Router, HTTP, Error Handlers, Data Stores)
- Groq API (LLaMA 3.3 70B) for generating natural-language recommendations
- Gmail module
- Salesforce (Developer Edition) — Lead creation

## Make vs N8N: notes from switching tools

Coming from N8N, a few real differences stood out while building this:

- **Parsing responses**: Make's HTTP module has a built-in "Parse response" toggle that structures JSON automatically. In N8N I sometimes needed a separate node for this.
- **Branching**: Make's Router isn't a single if/else — each branch has its own independent filter. If a branch has no filter, it runs unconditionally. This is different from N8N's IF node and took some getting used to.
- **Mapping nested data**: pulling a value out of a deeply nested JSON response (like an LLM's `choices[0].message.content`) was less straightforward in Make's field picker than writing the path directly in N8N. I ended up typing paths manually rather than relying on the auto-detected field list.
- **Overall**: once past the initial learning curve, Make felt simpler for straightforward automations. N8N's flexibility comes with more manual JSON/API handling.

## Error handling

Both HTTP calls (exchange rate API and Groq) use Make's **Retry** error handler, configured with a limited number of attempts and a delay between retries. This was tested against a real rate-limit error from the exchange rate API during development, not just configured in theory.

## Deduplication with Data Stores

Since the scenario runs every 30 minutes, sending an alert every single run while the rate stays above the threshold would spam the same information repeatedly. To avoid that, the "high rate" branch now includes:

1. **Data store — Get a record**: checks a `currency-alert-state` Data Store for the timestamp of the last alert sent
2. **Filter**: only continues if the last alert was sent more than 6 hours ago, OR no alert has ever been sent (first run)
3. **Data store — Update a record**: after the alert is sent, writes the current timestamp back to the Data Store, so the next run within the 6-hour window is correctly skipped

<img width="1919" height="868" alt="Screenshot_50" src="https://github.com/user-attachments/assets/fd90df9e-6a15-499c-8747-651a6711d64b" />


This was tested live: the first run sent the alert and wrote the timestamp; running the scenario again immediately afterward correctly stopped at the Data Store check and skipped sending, without touching Groq or Gmail.

This mirrors a real production concern: with many automations running on a schedule, you need a way to track state between runs, not just react to each run in isolation.


https://www.linkedin.com/in/rodrigo-h-miranda/
https://github.com/RodrigoMirandaHub
