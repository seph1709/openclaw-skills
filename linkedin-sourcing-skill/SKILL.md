---
name: linkedin-sourcing-agent
description: Find, open, and extract candidate profiles from LinkedIn using the agent-browser-clawdbot skill with a headless browser. Detects login state automatically — notifies the user to log in and switches to visible mode only when needed, then returns to headless for all browsing.
version: 1.1.0
user-invocable: true
metadata: {"openclaw":{"emoji":"🔍","homepage":"https://clawhub.ai"}}
---

# LinkedIn Sourcing Agent

## Purpose

Automates LinkedIn candidate sourcing using the agent-browser-clawdbot `browser` skill. The browser runs **headless at all times** except during the login step. The agent detects whether the user is already logged in and acts accordingly — no unnecessary prompts, no unnecessary visible-mode switches.

Use this skill when the user asks to:
- Search for candidates or professionals on LinkedIn
- Find profiles by role, skill, or location
- Extract profile data (name, title, location, company, skills)
- Run Google X-ray searches to discover LinkedIn profiles

---

## Required Tool

This skill requires the `agent-browser-clawdbot` skill.  

---

## Browser Mode Rules

| Situation | Browser Mode |
|---|---|
| Session valid (already logged in) | **Headless** — proceed directly to search |
| Not logged in / session expired | Headless → detect → notify user → switch to **Visible** → login → switch back to **Headless** |
| CAPTCHA or checkpoint mid-run | Pause → switch to **Visible** → wait for user → switch back to **Headless** |
| All searching and profile browsing | Always **Headless** |

The browser must **never** open in visible mode unless a login or human verification step is required. Once that step is resolved, the browser must immediately return to headless for the remainder of the run.

---

## Execution Workflow

### Step 1 — Launch Browser in Headless Mode
- Start the built-in browser in **headless mode**
- Navigate to: `https://www.linkedin.com/feed/`
- Use persistent session storage to preserve cookies across runs

---

### Step 2 — Detect Login State

After the page loads, inspect the resulting URL and page content:

**CASE A — Already logged in:**
- Current URL contains `/feed/` and the feed content is visible
- Action: Proceed directly to Step 4 (search) — remain in **headless mode**
- No message to the user is needed

**CASE B — Not logged in (redirected to login page):**
- Current URL contains `/login`, `/authwall`, or `/checkpoint`
- Action:
  1. Notify the user with this exact message:
     > "You are not logged in to LinkedIn. I'm opening the browser so you can log in. Please complete the login, then let me know and I'll continue."
  2. Switch the browser to **visible mode**
  3. Navigate to: `https://www.linkedin.com/login`
  4. Wait — do not proceed until the user confirms they have logged in
  5. Once the user confirms: save the session, switch back to **headless mode**
  6. Verify the session by navigating to `https://www.linkedin.com/feed/` in headless mode
  7. Confirm feed is accessible, then proceed to Step 4

---

### Step 3 — Session Verification (after login)

After switching back to headless:
- Navigate to `https://www.linkedin.com/feed/`
- If URL is still `/login` or `/checkpoint` → repeat Step 2B (login not complete)
- If feed loads → session confirmed, continue in headless mode

---

### Step 4 — Run Search (Headless)

**Option B — Google X-ray (broader discovery)**
```
https://www.google.com/search?q=site:linkedin.com/in+{role}+{skill}+{location}
```

---

### Step 5 — Open and Extract Profiles (Headless)

For each valid result (limit: **5–20 profiles per run**):
1. Navigate to `https://www.linkedin.com/in/{profile}` in headless mode
2. Extract visible fields:

```
Name:
Title:
Location:
Company:
Skills:
URL:
```

---

### Step 6 — Mid-Run Risk Detection

If at any point during Steps 4–5 the following is detected:
- CAPTCHA challenge
- Suspicious activity warning
- Unexpected redirect to `/checkpoint` or `/login`

Then:
1. Pause execution immediately
2. Notify the user:
   > "LinkedIn has triggered a verification. I'm switching to visible mode so you can complete it. Let me know when it's done."
3. Switch to **visible mode**
4. Wait for user confirmation
5. Switch back to **headless mode**
6. Resume from where execution paused

---

### Step 7 — Filter Results

Retain only profiles matching all applicable criteria:
- Role relevance
- Skill match (if provided)
- Location match (if provided)
- Seniority level (if specified)

---

## Search Templates

### General

### Google X-ray — Base
```
https://www.google.com/search?q=site:linkedin.com/in+{role}+{skill}+{location}
```

### Google X-ray — Senior Filter
```
https://www.google.com/search?q=site:linkedin.com/in+("senior+engineer"+OR+"lead")+python+manila
```

---

## Output Format

Return results as a structured list. For each candidate:

```
- Name:     <full name>
- Title:    <current job title>
- Location: <city, country>
- Company:  <current employer>
- Skills:   <comma-separated, if visible>
- URL:      https://www.linkedin.com/in/<profile-slug>
```

---

## Behavior Rules

### DO
- Always start and stay in headless mode unless login or CAPTCHA is required
- Detect login state silently before doing anything else
- Notify the user clearly before switching to visible mode
- Switch back to headless immediately after the user resolves the login or verification
- Save session after login so future runs skip the login step entirely
- Use slow, human-like navigation pacing between page loads

### DO NOT
- Open the browser in visible mode unless a login or human verification step requires it
- Prompt the user to log in if the session is already valid
- Spam or rapid-fire searches or profile opens
- Automatically bypass CAPTCHA — always hand off to the user
- Reset the browser session unless it is genuinely broken

---

## Example Prompts

> "Find me 10 Python developers in Manila on LinkedIn."

> "Search LinkedIn for senior backend engineers with Kubernetes experience."

> "Use Google X-ray to find talent acquisition managers in the Philippines."

> "Source 5 React developers from LinkedIn and give me their names and current titles."

---

## Notes

- Credentials are never stored by this skill. Login is always performed manually by the user in visible mode.
- Profile data extracted is limited to what is publicly visible without a LinkedIn connection.
- Do not exceed 20 profile opens per session to avoid triggering LinkedIn rate limits.
- Session cookies are persisted by the browser tool's built-in storage and reused on the next run automatically.
