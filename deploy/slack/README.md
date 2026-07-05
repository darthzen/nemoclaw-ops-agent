# Slack app (T-006)

`nemoclaw-slack-app.manifest.json` — Slack app manifest for the **NemoClaw** bot
identity in the `vibecoding-wu42136` workspace (separate app from Hermes).

Socket Mode (on-prem / outbound-only). OpenClaw "Recommended" scope set incl. the
full `mpim:*` group-DM scopes, plus `chat:write.customize` for the NemoClaw identity.

## Create
api.slack.com/apps → Create New App → From a manifest → select `vibecoding-wu42136`
→ paste this manifest → Next → Create.

## Then (two tokens for Socket Mode)
1. Basic Information → App-Level Tokens → Generate Token and Scopes → add
   `connections:write` → copy the **App-Level Token** (`xapp-…`).
2. Install App → Install to Workspace → copy the **Bot User OAuth Token** (`xoxb-…`).

## Hand off to T-014
Store both tokens as a k8s Secret in the `nemoclaw` ns (never in git); the agent
references the secret name. Example:
`kubectl create secret generic nemoclaw-slack -n nemoclaw \
  --from-literal=SLACK_APP_TOKEN=xapp-... --from-literal=SLACK_BOT_TOKEN=xoxb-...`
