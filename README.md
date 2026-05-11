# Social Post Studio — Next.js

A one-page app that takes an image + description and kicks off an n8n workflow that:
1. **Edits the image** (e.g. via OpenAI DALL-E / Stability AI / Cloudinary)
2. **Writes social media copy** (via an LLM node in n8n)
3. **Sends to Slack for approval** (Slack bot posts the draft; team member clicks Approve/Reject)
4. **Publishes via GoHighLevel (GHL)** once approved

---

## Quick start

```bash
npm install
cp .env.local.example .env.local
# fill in N8N_WEBHOOK_URL in .env.local
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

---

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `N8N_WEBHOOK_URL` | ✅ | Webhook URL from the n8n "Webhook" trigger node |
| `N8N_STATUS_URL` | optional | Base URL for polling job status (if your workflow is async) |

---

## n8n Workflow Setup

### 1 — Webhook trigger node
- Method: **POST**
- Response mode: **"Respond to Webhook"** → *"When last node finishes"* (synchronous)
  - OR set to *"Immediately"*, return a `jobId`, and use the polling route `/api/status/:jobId`
- The node receives two fields: `image` (binary file) and `description` (string).

### 2 — AI Image Editing node
Options:
- **OpenAI** node → Images → Edit endpoint (send the binary + a prompt derived from `description`)
- **HTTP Request** node → Stability AI / Replicate API
- **Cloudinary** node → run an AI transformation

Output: save the edited image URL to a variable (e.g. `editedImageUrl`).

### 3 — Write Social Copy node
- Use the **OpenAI** (or Anthropic / any LLM) node
- Prompt template:
  ```
  Write a compelling social media caption for the following image context:
  "{{$json.description}}"
  Include relevant hashtags. Keep it under 280 characters for Twitter,
  and provide a longer version for Instagram/Facebook.
  ```
- Save output to `generatedCopy`.

### 4 — Slack Approval node
- Use the **Slack** node → Post message to your `#social-approvals` channel
- Include the `editedImageUrl` and `generatedCopy` in the message
- Add two interactive buttons: **✅ Approve** and **❌ Reject**
- Use n8n's **Wait** node set to *"On webhook call"* to pause until Slack sends the interaction callback

### 5 — Conditional node
- If Slack response = `approved` → continue to GHL publish
- If `rejected` → send failure response back to the frontend

### 6 — GoHighLevel (GHL) Publish node
- Use the **HTTP Request** node to call the GHL API
- Endpoint: `POST https://services.leadconnectorhq.com/social-media-posting/`
- Auth: Bearer token from GHL → Settings → API Keys
- Body:
  ```json
  {
    "locationId": "YOUR_LOCATION_ID",
    "content": "{{generatedCopy}}",
    "mediaUrls": ["{{editedImageUrl}}"],
    "platforms": ["facebook", "instagram"],
    "scheduledAt": "now"
  }
  ```

### 7 — Respond to Webhook node (final)
Return this JSON to the Next.js app:
```json
{
  "stage": "done",
  "message": "Published!",
  "editedImageUrl": "{{editedImageUrl}}",
  "generatedCopy": "{{generatedCopy}}",
  "postUrl": "{{ghlPostUrl}}"
}
```

---

## How the frontend handles responses

The `/api/submit` route in Next.js proxies the form to n8n and returns the response.

**Synchronous mode** (n8n waits and returns final JSON):
- n8n processes everything and returns `{ stage: "done", ... }`
- Frontend displays success immediately

**Async / polling mode** (n8n returns a `jobId` immediately):
- Frontend polls `/api/status/:jobId` every 3 seconds
- Each poll returns the current stage and message
- You update n8n's job status via a separate webhook or n8n's internal state

---

## Project structure

```
src/
  app/
    page.tsx              ← One-page UI
    layout.tsx            ← Root layout
    globals.css           ← Design tokens + fonts
    api/
      submit/route.ts     ← Proxies form → n8n webhook
      status/[jobId]/     ← Optional polling endpoint
        route.ts
```
"# Carl-social-media-post" 
