# Claude VR Proxy — Cloudflare Worker Setup

## What This Does
A tiny cloud function that sits between your VR headset and the Claude API.
Your Quest browser sends questions to this Worker, the Worker forwards them
to Claude with your API key, and sends the response back. No server on
the K-State network. IT never sees anything unusual.

## Setup (about 15 minutes, one time)

### Step 1: Get a Claude API Key
1. Go to https://console.anthropic.com
2. Sign up or log in
3. Go to API Keys → Create Key
4. Copy the key (starts with "sk-ant-...")
5. Keep this secret — it's like a password

### Step 2: Create the Cloudflare Worker
1. Go to https://workers.cloudflare.com
2. Sign up (free plan is fine — 100,000 requests/day)
3. Click "Create a Worker"
4. Name it something like "claude-vr-proxy"
5. Delete all the default code and paste the code below
6. Click "Save and Deploy"

### Step 3: Add Your API Key as a Secret
1. In the Worker dashboard, go to Settings → Variables
2. Under "Environment Variables", click "Add variable"
3. Name: ANTHROPIC_API_KEY
4. Value: paste your sk-ant-... key
5. Click "Encrypt" (important!)
6. Click "Save"

### Step 4: Note Your Worker URL
Your Worker URL will be something like:
  https://claude-vr-proxy.YOUR_SUBDOMAIN.workers.dev

You'll paste this URL into the VR page settings.

---

## Worker Code (paste this into the Worker editor)

```javascript
export default {
  async fetch(request, env) {
    // Handle CORS preflight
    if (request.method === "OPTIONS") {
      return new Response(null, {
        headers: {
          "Access-Control-Allow-Origin": "*",
          "Access-Control-Allow-Methods": "POST, OPTIONS",
          "Access-Control-Allow-Headers": "Content-Type",
        },
      });
    }

    if (request.method !== "POST") {
      return new Response("POST only", { status: 405 });
    }

    try {
      const { message, system_context } = await request.json();

      const systemPrompt = `You are a control theory tutor working inside a VR state-space visualization. The student is standing inside a 3D representation of a dynamical system, with eigenvector arrows, trajectory trails, and the A matrix visible around them.

Current system context:
${system_context || "No system loaded yet."}

IMPORTANT: You can modify the VR scene by including a JSON command block in your response. Wrap it in <scene_cmd> tags like this:

<scene_cmd>
{
  "action": "update_A",
  "system": "thermostat",
  "A": [[-0.15, 0.10], [0.05, -0.12]],
  "explanation": "Added window cooling term to A(1,1)"
}
</scene_cmd>

Available actions:
- "update_A": Change the A matrix. Include "system" name and new "A" matrix.
- "set_system": Switch to a different system. Include "system": "thermostat"|"wedding"|"swerve".
- "add_system": Create a brand new system. Include "name", "states" (array of names), "A" (matrix), "scale" (number), "description".
- "highlight_entry": Highlight a specific A matrix entry. Include "row" and "col" (0-indexed).
- "add_label": Add a floating text label in the scene. Include "text", "position" [x,y,z], "color" (hex).

Only include scene commands when the student asks you to change or demonstrate something. For pure Q&A, just respond with text. Keep explanations concise but insightful — the student can see the 3D visualization, so reference what they're looking at.

Use plain language. Reference eigenvectors as "the arrows", trajectories as "the trails", the origin as "the center point". The student is a BAE professor at Kansas State learning control theory.`;

      const response = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "x-api-key": env.ANTHROPIC_API_KEY,
          "anthropic-version": "2023-06-01",
        },
        body: JSON.stringify({
          model: "claude-sonnet-4-6",
          max_tokens: 1024,
          system: systemPrompt,
          messages: [{ role: "user", content: message }],
        }),
      });

      const data = await response.json();

      return new Response(JSON.stringify(data), {
        headers: {
          "Content-Type": "application/json",
          "Access-Control-Allow-Origin": "*",
        },
      });
    } catch (err) {
      return new Response(JSON.stringify({ error: err.message }), {
        status: 500,
        headers: {
          "Content-Type": "application/json",
          "Access-Control-Allow-Origin": "*",
        },
      });
    }
  },
};
```

---

## Testing
Once deployed, test from your browser console:

```javascript
fetch("https://claude-vr-proxy.YOUR_SUBDOMAIN.workers.dev", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    message: "What is an eigenvalue?",
    system_context: "Thermostat system with A = [[-0.10, 0.10], [0.05, -0.08]]"
  })
}).then(r => r.json()).then(d => console.log(d.content[0].text));
```

If you see Claude's response, you're good. Paste the Worker URL into the VR page and you're live.
