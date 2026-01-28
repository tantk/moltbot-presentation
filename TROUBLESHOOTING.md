# Troubleshooting Log

This document records the issues encountered and solutions applied for various configurations.

---

## 1. Tetris Game Setup

### What Was Done
- Created a complete Tetris game in `/home/clawd/kimitest/tetris.html`
- Features: HTML5 Canvas, CSS styling, JavaScript game logic
- Controls: Arrow keys (move/rotate), Down (soft drop), Space (hard drop), P (pause)

### Server Configuration
```bash
# Started HTTP server on port 8080
python3 -m http.server 8080 --bind 100.121.213.25
```

**Security Note**: Initially bound to all interfaces (`0.0.0.0`), later restricted to Tailscale IP only for security.

---

## 2. Tailscale Network Access

### Issue
- Windows PC couldn't ping Ubuntu server at `192.168.0.121`
- HTTP connection timed out

### Root Cause
- Firewall blocking port 8080
- Server listening on wrong interface

### Solution
1. **Installed Tailscale** on both devices
2. **Bound server to Tailscale IP only**:
   ```bash
   python3 -m http.server 8080 --bind 100.121.213.25
   ```
3. Access via: `http://100.121.213.25:8080/tetris.html`

---

## 3. Moltbot + Kimi Code Integration

### Overview
Configured moltbot (formerly clawdbot) to use Moonshot's **Kimi For Coding** API instead of OpenAI Codex.

---

### Issue 1: Invalid Config Format

**Error:**
```
Invalid config at /home/clawd/.clawdbot/moltbot.json:
- auth.profiles.moonshot:default.mode: Invalid input
- auth.profiles.moonshot:default: Unrecognized keys: "baseUrl", "apiKey"
```

**Root Cause:**
Moltbot expects auth credentials in a separate file (`auth-profiles.json`), not inline in `moltbot.json`.

**Solution:**
1. Config file (`~/.clawdbot/moltbot.json`) only contains profile references:
   ```json
   "auth": {
     "profiles": {
       "moonshot:default": {
         "provider": "moonshot",
         "mode": "api_key"
       }
     }
   }
   ```

2. Credentials stored separately in `~/.clawdbot/agents/main/agent/auth-profiles.json`:
   ```json
   {
     "version": 1,
     "profiles": {
       "moonshot:default": {
         "type": "api_key",
         "provider": "moonshot",
         "key": "sk-..."
       }
     }
   }
   ```

---

### Issue 2: HTTP 401 - Invalid Authentication

**Error:**
```
HTTP 401: Invalid Authentication
{"error":{"message":"Invalid Authentication","type":"invalid_authentication_error"}}
```

**Root Cause:**
API key was generated for `platform.moonshot.cn` but we were trying to use it with:
- `api.moonshot.ai/v1` (international) - ❌ Doesn't work
- `api.moonshot.cn/v1` (Chinese) - ❌ Doesn't work

**Solution:**
Use the correct **Kimi Code endpoint**:
```
https://api.kimi.com/coding/v1
```

This endpoint validates the API key correctly.

---

### Issue 3: HTTP 403 - Model Access Restricted

**Error:**
```
HTTP 403: Kimi For Coding is currently only available for Coding Agents such as 
Kimi CLI, Claude Code, Roo Code, Kilo Code, etc.
```

**Root Cause:**
Kimi Code API requires a specific `User-Agent` header to identify as an approved coding agent.

**Solution:**
Add the `User-Agent` header to the model configuration:

```json
"headers": {
  "User-Agent": "KimiCLI/0.77"
}
```

Full config in `~/.clawdbot/agents/main/agent/models.json`:
```json
{
  "providers": {
    "moonshot": {
      "baseUrl": "https://api.kimi.com/coding/v1",
      "apiKey": "sk-...",
      "api": "openai-completions",
      "headers": {
        "User-Agent": "KimiCLI/0.77"
      },
      "models": [...]
    }
  }
}
```

---

### Issue 4: ROLE_UNSPECIFIED Error

**Error:**
```
400 invalid request: unsupported role ROLE_UNSPECIFIED
```

**Root Cause:**
Kimi Code API has stricter message role requirements than standard OpenAI API. It doesn't support the `developer` role (used by some models).

**Solution:**
Add compatibility settings to disable `developer` role:

```json
"compat": {
  "supportsDeveloperRole": false,
  "supportsStore": false
}
```

---

## Final Working Configuration

### File: `~/.clawdbot/moltbot.json`
```json
{
  "auth": {
    "profiles": {
      "moonshot:default": {
        "provider": "moonshot",
        "mode": "api_key"
      }
    },
    "order": {
      "moonshot": ["moonshot:default"]
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "moonshot": {
        "baseUrl": "https://api.kimi.com/coding/v1",
        "api": "openai-completions",
        "headers": {
          "User-Agent": "KimiCLI/0.77"
        },
        "models": [
          {
            "id": "kimi-for-coding",
            "name": "Kimi For Coding",
            "reasoning": true,
            "input": ["text"],
            "contextWindow": 262144,
            "maxTokens": 32768,
            "compat": {
              "supportsDeveloperRole": false,
              "supportsStore": false
            }
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "moonshot/kimi-for-coding"
      }
    }
  }
}
```

### File: `~/.clawdbot/agents/main/agent/auth-profiles.json`
```json
{
  "version": 1,
  "profiles": {
    "moonshot:default": {
      "type": "api_key",
      "provider": "moonshot",
      "key": "sk-kimi-..."
    }
  }
}
```

### File: `~/.clawdbot/agents/main/agent/models.json`
```json
{
  "providers": {
    "moonshot": {
      "baseUrl": "https://api.kimi.com/coding/v1",
      "apiKey": "sk-kimi-...",
      "api": "openai-completions",
      "headers": {
        "User-Agent": "KimiCLI/0.77"
      },
      "models": [
        {
          "id": "kimi-for-coding",
          "name": "Kimi For Coding",
          "reasoning": true,
          "input": ["text"],
          "contextWindow": 262144,
          "maxTokens": 32768,
          "compat": {
            "supportsDeveloperRole": false,
            "supportsStore": false
          }
        }
      ]
    }
  }
}
```

---

## Key Learnings

1. **Kimi Code ≠ Moonshot API** - They are separate services with different endpoints
2. **User-Agent is required** - Kimi Code validates the client application
3. **Compat settings matter** - Different APIs have different role support
4. **Config hierarchy** - moltbot uses both global (`moltbot.json`) and agent-level (`agents/main/agent/*.json`) configs
5. **Auth separation** - Credentials stored separately from config for security

---

## Useful Commands

```bash
# Test API key manually
curl -s -H "Authorization: Bearer sk-..." \
  -H "Content-Type: application/json" \
  -H "User-Agent: KimiCLI/0.77" \
  https://api.kimi.com/coding/v1/models

# Verify moltbot config
moltbot config get models
moltbot config get agents.defaults.model.primary

# Restart gateway
pkill -f moltbot-gateway

# Check gateway status
moltbot doctor
```

---

*Document created: 2026-01-28*
