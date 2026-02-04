# OpenClaw: Connect to Remote Ollama Service

This guide explains how to configure OpenClaw to use an existing Ollama service running on another local IP address.

## Overview

OpenClaw can connect to a remote Ollama instance to use its models for AI operations. This allows you to centralize your Ollama deployment while using OpenClaw from multiple devices.

---

## Configuration Methods

### Method 1: Configuration File (Recommended)

Edit your OpenClaw configuration file at `~/.openclaw/openclaw.json`:

```json
{
  "models": {
    "providers": {
      "ollama": {
        "apiKey": "ollama-local",
        "baseUrl": "http://192.168.1.100:11434/v1",
        "api": "openai-responses"
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "ollama/llama3.3"
      }
    }
  }
}
```

**Replace:**
- `192.168.1.100` with your Ollama server's IP address
- `llama3.3` with the model you have installed on your Ollama server

### Method 2: CLI Commands

```bash
# Set the Ollama base URL
openclaw config set models.providers.ollama.baseUrl "http://192.168.1.100:11434/v1"

# Set the API key
openclaw config set models.providers.ollama.apiKey "ollama-local"

# Set the primary model
openclaw config set agents.defaults.model.primary "ollama/llama3.3"
```

### Method 3: Environment Variables

```bash
export OLLAMA_HOST="http://192.168.1.100:11434"
```

---

## Full Configuration Example

Here's a complete configuration for a remote Ollama setup with model definitions:

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "ollama": {
        "baseUrl": "http://192.168.1.100:11434/v1",
        "apiKey": "ollama-local",
        "api": "openai-responses",
        "models": [
          {
            "id": "llama3.3",
            "name": "Llama 3.3 70B",
            "reasoning": false,
            "input": ["text"],
            "cost": {
              "input": 0,
              "output": 0,
              "cacheRead": 0,
              "cacheWrite": 0
            },
            "contextWindow": 128000,
            "maxTokens": 4096
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "ollama/llama3.3",
        "fallback": ["ollama/qwen2.5-coder:32b"]
      }
    }
  }
}
```

---

## Server-Side Configuration (Ollama Host)

On your **Ollama server** (e.g., 192.168.1.100), you need to expose it to the network:

### Option 1: Environment Variable

```bash
# Set environment variable to listen on all interfaces
export OLLAMA_HOST=0.0.0.0:11434
```

### Option 2: Systemd Service (Linux)

```bash
# Edit the systemd service
sudo systemctl edit ollama.service
```

Add the following:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

Then restart the service:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

---

## Testing the Connection

### 1. Test Ollama Server Accessibility

```bash
# Test if Ollama is accessible from your OpenClaw machine
curl http://192.168.1.100:11434/api/tags
```

### 2. Verify OpenClaw Configuration

```bash
# List available models
openclaw models list

# Test the setup
openclaw models status --probe

# Check overall health
openclaw health
```

### 3. Check Configuration Values

```bash
# Verify Ollama configuration
openclaw config get models.providers.ollama
```

---

## Restart OpenClaw Gateway

After making configuration changes, restart the gateway:

```bash
# If using systemd daemon
openclaw gateway restart

# Or manually stop and start
openclaw gateway stop
openclaw gateway start --port 18789 --verbose
```

---

## Recommended Models for OpenClaw

Models with good tool-calling and general capability support:

- **`qwen2.5-coder:32b`** - Best for coding tasks and tool use
- **`qwen3`** - Good all-around performance
- **`deepseek-r1:32b`** - Strong reasoning capabilities
- **`llama3.3:70b`** - Excellent quality (requires powerful hardware)

### Pulling Models on Ollama Server

```bash
# On your Ollama server, pull the models you want to use
ollama pull llama3.3
ollama pull qwen2.5-coder:32b
ollama pull deepseek-r1:32b

# List installed models
ollama list
```

---

## Troubleshooting

### Connection Refused

**Problem:** Cannot connect to Ollama server

**Solutions:**
1. Check firewall on Ollama server allows port 11434
   ```bash
   sudo ufw allow 11434/tcp
   ```
2. Verify `OLLAMA_HOST` is set to `0.0.0.0:11434`
3. Test connection manually:
   ```bash
   curl http://192.168.1.100:11434/api/tags
   ```

### Model Not Found

**Problem:** OpenClaw reports model is not available

**Solutions:**
1. Verify the model is installed on Ollama server:
   ```bash
   ollama list
   ```
2. Pull the model if missing:
   ```bash
   ollama pull llama3.3
   ```
3. Check model name matches exactly in OpenClaw config

### No Response from Bot

**Problem:** Bot doesn't respond to messages

**Solutions:**
1. Check gateway logs:
   ```bash
   openclaw gateway logs
   ```
2. Verify configuration:
   ```bash
   openclaw config get models.providers.ollama
   openclaw status --all
   ```
3. Test model connectivity:
   ```bash
   openclaw models status --probe
   ```

### Performance Issues

**Problem:** Slow responses or timeouts

**Solutions:**
1. Use smaller models on less powerful hardware
2. Check network latency between OpenClaw and Ollama server
3. Consider using quantized models (e.g., `:q4_K_M` variants)
4. Monitor Ollama server resources (CPU, RAM, GPU)

---

## Quick Reference Commands

```bash
# Installation
curl -fsSL https://openclaw.ai/install.sh | bash

# Onboarding
openclaw onboard --install-daemon

# Gateway management
openclaw gateway start --port 18789 --verbose
openclaw gateway stop
openclaw gateway restart
openclaw gateway status

# Health checks
openclaw health
openclaw status --all
openclaw security audit --deep

# Model management
openclaw models list
openclaw models status --probe

# Configuration
openclaw config get models.providers.ollama
openclaw config set models.providers.ollama.baseUrl "http://192.168.1.100:11434/v1"

# Logs
openclaw gateway logs
```

---

## Network Security Considerations

When exposing Ollama to your local network:

1. **Firewall Rules:** Only allow connections from trusted IPs
2. **Network Segmentation:** Keep on a trusted internal network
3. **VPN/Tunnel:** Consider using Tailscale or WireGuard for secure access
4. **No Public Exposure:** Never expose Ollama directly to the internet

---

## Additional Resources

- OpenClaw Documentation: https://docs.openclaw.ai/
- Ollama Documentation: https://ollama.ai/
- OpenClaw Models Guide: https://docs.openclaw.ai/providers
- OpenClaw GitHub: https://github.com/openclaw/openclaw

---

## Support

If you encounter issues:

1. Check the OpenClaw documentation
2. Review gateway logs: `openclaw gateway logs`
3. Run diagnostics: `openclaw status --all`
4. Visit the OpenClaw GitHub issues page

---

*Document created: February 4, 2026*
*For: cengel@gmail.com*
