# GRASP: Opportunity Discovery System

## Overview

**GRASP** is a purposed built suite of tools for finding, filtering, and sharing opportunities from online sources. Tools are built from free and open-sourced repositories and services and are designed to be automated using workflows and dashboards.
It can combine workflow automation, AI-assisted screening, and purpose-built collectors to turn scattered listings into a useful, deduplicated feed.
## Suite of tools
1) Luma to Telegram alert system
2) Eventbrite to Telegram alert system

## GRASP concept

GRASP can grow to include independent source connectors and processing tools:

- **n8n** orchestrates scheduled collection, filtering, and delivery. [Disclaimer: N8N is a paid service with the option to locally host on a container for free, I choose to host N8N on a Oracle Cloud linux environment with Nginx using a cloudflare domain]
- **OpenClaw** may serve as an additional agent or automation component for future sources and tasks. It is not connected in the current Luma workflow export.
- **Source connectors** retrieve opportunities from services such as Luma, subject to each service's access method and terms.
- **Screening** applies configurable rules and, where useful, AI classification to rank or reject candidates.
- **Destinations** deliver selected opportunities to Telegram or other configured channels.
- **A registry** tracks handled items so repeated runs do not process the same opportunity again.

The intended pattern is: **collect → normalize → deduplicate → screen → deliver → record**.

## Project 1: Luma Event Curator to Telegram

### Purpose

Periodically retrieve upcoming events from Luma using the Apify scraper, the events are then filtered using an AI through N8N. 

### Workflow summary

1. **Schedule:** Starts every Sunday
2. **Configuration:** Configure the Apify Luma Scraper with the regional coordinates (longitude and latitude)
3. **Retrieve events:** Calls `GET https://public-api.luma.com/v1/organizations/events/list` with an `x-luma-api-key` header. The request asks for events from now through the look-ahead date, up to 100 results, sorted in ascending order.
4. **Normalize candidates:** Extracts an event ID, name, description, URL, start/end time, timezone, location, and raw event payload. It supports several common response shapes and drops records without an ID or duplicate IDs within the same run.
5. **Load the registry:** Reads previously processed event IDs from the n8n Data Table `luma_event_registry`.
6. **Deduplicate:** Keeps only candidate events whose IDs are not already in the registry.
7. **AI screening:** Sends each new event and the group's requirements to an OpenRouter chat model (`google/gemini-2.5-flash`). The prompt requests a JSON decision, brief reason, confidence score, and neutral Telegram summary. The model is configured for JSON output and low temperature.
8. **Apply threshold:** Parses the result. An event is approved only when `approved` is true and confidence is at least 70. Invalid JSON is treated as a rejection.
9. **Deliver and record:** Approved events are sent to Telegram with title, summary, date, location, and an event link, then recorded as `published`. Rejected events are recorded as `rejected` with their reason. Both decisions enter the registry to avoid repeated screening.

### Flow diagram

```text
Every 6 hours
   ├─ Configuration ─> Luma organization events API ─> Normalize candidates ─┐
   └─ Read processed-event registry ────────────────────────────────────────┤
                                                                            v
                                                              Combine and deduplicate
                                                                            |
                                                                  AI event screening
                                                                            |
                                                              Parse decision / threshold
                                                               /                    \
                                                 Approved: Telegram post       Rejected
                                                        |                         |
                                                Record as published       Record as rejected
                                                               \                    /
                                                                Registry for next run
```

### Configuration and prerequisites

The exported workflow contains placeholders; configure these before activation:

1. Create and configure your local instance of N8N
2. Set up the hypebridge luma event scraper, add regional coordinates, luma discover link and set the default number of events pulled per request (Link to Apify: https://apify.com/hypebridge/luma-com-event-scraper
3. Create an n8n Data Table named `luma_event_registry` with text columns:
   - `event_name`
   - `event_id`
   - `event_date`
4. Install the Apify N8N node
5. Setup the credentials using their respective API keys for Apify, Telegram and whichever AI model you prefer (In the workflow, I used Groq AI). Add the Chat ID of the telegram chat (as well as the Message Thread ID if channels are used)
6. Ensure that the data table schema is being adhered to throughout the workflow

Credentials belong in n8n's credential store, not in this documentation or a shared workflow export.

### Data and behavior notes

- N8N can be a good alternative to something like Grok Bot as it can be more thorough in combing through events as the LLM may not be pull all the available events
- The event listing is scoped to the authenticated Luma organization endpoint. It should not be assumed to search all public Luma events
- Apify's free tier provides $5 worth of tokens which should be plenty to run the workflow several times per month
- Rejected events are not stored or outputted anywhere, events that are past their end-date will be deleted on the same day at 2400 hrs. Duplicate events are filtered out using the event id
- Due to the limitations of the Telegram API, the description of each Luma event will need to be shortened so that it can fit under the luma character limit

## Suggested next steps for GRASP

1. Using a Code Node, regex can be applied to search through events via keywords which can be more effective for finding events

## Source artifact

The workflow described here is based on the exported file `luma-event-curator-telegram.json` from the earlier project workspace. Keep the export alongside the project when maintaining this documentation, and update this file when node behavior or configuration changes.
