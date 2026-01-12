# Docker-based Burp Suite Project Parser

This document explains how to use the pre-configured Docker image `burp-parser:latest` to parse Burp Suite project files from any directory on vader (192.168.2.8 and 192.168.0.8)

## Overview

The `burp-parser` Docker image contains:
- Burp Suite Professional (with EULA pre-accepted)
- The burpsuite-project-file-parser plugin (with timestamp, comment, and highlight export)
- X11 libraries for optional GUI mode
- Pre-configured user settings

## Quick Start

Export proxy history (with requests, responses, timestamps, comments, and highlights) to JSON:

```bash
docker run --rm -v "/path/to/your/project/dir:/projects" burp-parser:latest \
  bash -c "java -jar -Djava.awt.headless=true \
    --add-opens=java.desktop/javax.swing=ALL-UNNAMED \
    --add-opens=java.base/java.lang=ALL-UNNAMED \
    /opt/burp/burpsuite_pro.jar \
    --project-file=/projects/yourfile.burp \
    --user-config-file=/opt/burp-parser/user-config.json \
    proxyHistory.both"
```

**Note:** Use `proxyHistory.both` to get both requests AND responses. Plain `proxyHistory` only exports requests.

## Usage Examples

### Export Proxy History (Full - Recommended)

```bash
docker run --rm -v "$(pwd):/projects" burp-parser:latest \
  bash -c "java -jar -Djava.awt.headless=true \
    --add-opens=java.desktop/javax.swing=ALL-UNNAMED \
    --add-opens=java.base/java.lang=ALL-UNNAMED \
    /opt/burp/burpsuite_pro.jar \
    --project-file=/projects/myproject.burp \
    --user-config-file=/opt/burp-parser/user-config.json \
    proxyHistory.both"
```

### Export Proxy History (Requests Only - Faster)

```bash
docker run --rm -v "$(pwd):/projects" burp-parser:latest \
  bash -c "java -jar -Djava.awt.headless=true \
    --add-opens=java.desktop/javax.swing=ALL-UNNAMED \
    --add-opens=java.base/java.lang=ALL-UNNAMED \
    /opt/burp/burpsuite_pro.jar \
    --project-file=/projects/myproject.burp \
    --user-config-file=/opt/burp-parser/user-config.json \
    proxyHistory"
```

### Export Site Map

```bash
docker run --rm -v "$(pwd):/projects" burp-parser:latest \
  bash -c "java -jar -Djava.awt.headless=true \
    --add-opens=java.desktop/javax.swing=ALL-UNNAMED \
    --add-opens=java.base/java.lang=ALL-UNNAMED \
    /opt/burp/burpsuite_pro.jar \
    --project-file=/projects/myproject.burp \
    --user-config-file=/opt/burp-parser/user-config.json \
    siteMap"
```

### Export Audit Items (Vulnerabilities)

```bash
docker run --rm -v "$(pwd):/projects" burp-parser:latest \
  bash -c "java -jar -Djava.awt.headless=true \
    --add-opens=java.desktop/javax.swing=ALL-UNNAMED \
    --add-opens=java.base/java.lang=ALL-UNNAMED \
    /opt/burp/burpsuite_pro.jar \
    --project-file=/projects/myproject.burp \
    --user-config-file=/opt/burp-parser/user-config.json \
    auditItems"
```

### Search Response Headers with Regex

```bash
docker run --rm -v "$(pwd):/projects" burp-parser:latest \
  bash -c "java -jar -Djava.awt.headless=true \
    --add-opens=java.desktop/javax.swing=ALL-UNNAMED \
    --add-opens=java.base/java.lang=ALL-UNNAMED \
    /opt/burp/burpsuite_pro.jar \
    --project-file=/projects/myproject.burp \
    --user-config-file=/opt/burp-parser/user-config.json \
    responseHeader='.*(Servlet|nginx).*'"
```

### Export Only Request Headers (Faster)

```bash
docker run --rm -v "$(pwd):/projects" burp-parser:latest \
  bash -c "java -jar -Djava.awt.headless=true \
    --add-opens=java.desktop/javax.swing=ALL-UNNAMED \
    --add-opens=java.base/java.lang=ALL-UNNAMED \
    /opt/burp/burpsuite_pro.jar \
    --project-file=/projects/myproject.burp \
    --user-config-file=/opt/burp-parser/user-config.json \
    proxyHistory.request.headers"
```

## Output Format

The parser outputs one JSON object per line. For proxy history with `proxyHistory.both`:

```json
{
  "request": {
    "url": "https://example.com/path?query=value",
    "headers": ["Host: example.com", "User-Agent: ...", ...],
    "body": "request body if present"
  },
  "response": {
    "response-headers": ["HTTP/1.1 200 OK", "Content-Type: application/json", ...],
    "response-body": "response body content"
  },
  "timestamp": "2025-11-05T17:06:48.933Z[Etc/UTC]",
  "comment": "User annotation from Burp (if present)",
  "highlight": "RED|GREEN|YELLOW|BLUE|etc (if present)"
}
```

- `request` - always present
- `response` - only present when using `.both`, `.response.headers`, or `.response.body` flags
- `timestamp` - ISO 8601 format with timezone, always present
- `comment` and `highlight` - only included when the proxy history item has annotations set in Burp

## Available Flags

| Flag | Description |
|------|-------------|
| `proxyHistory` | Export proxy history (requests only) |
| `proxyHistory.both` | Export proxy history with requests AND responses (recommended) |
| `siteMap` | Export site map (requests only) |
| `siteMap.both` | Export site map with requests AND responses |
| `auditItems` | Export all audit/vulnerability findings |
| `responseHeader=<regex>` | Search response headers with regex |
| `responseBody=<regex>` | Search response bodies with regex |

### Sub-component Flags (for faster parsing)

Append these to `proxyHistory` or `siteMap` to limit output:

- `.both` - Include both request and response data (recommended for most use cases)
- `.request.headers` - Only request headers
- `.request.body` - Only request body
- `.response.headers` - Only response headers
- `.response.body` - Only response body

Example: `proxyHistory.request.headers` exports only request headers.

## Important Notes

### Project File Access

Burp Suite requires **read/write access** to project files, even when only reading data. Do not use `:ro` (read-only) mounts:

```bash
# WRONG - will fail
-v "$(pwd):/projects:ro"

# CORRECT
-v "$(pwd):/projects"
```

### Memory Usage

For large project files, increase Java heap size:

```bash
docker run --rm -v "$(pwd):/projects" burp-parser:latest \
  bash -c "java -jar -Xmx4G -Djava.awt.headless=true \
    --add-opens=java.desktop/javax.swing=ALL-UNNAMED \
    --add-opens=java.base/java.lang=ALL-UNNAMED \
    /opt/burp/burpsuite_pro.jar \
    --project-file=/projects/myproject.burp \
    --user-config-file=/opt/burp-parser/user-config.json \
    proxyHistory"
```

### GUI Mode (for EULA acceptance or debugging)

If you need to run Burp with a GUI (e.g., to re-accept EULA), set the DISPLAY environment variable:

```bash
docker run --rm -v "$(pwd):/projects" -e DISPLAY=192.168.1.100:0 burp-parser:latest \
  bash -c "java -jar \
    --add-opens=java.desktop/javax.swing=ALL-UNNAMED \
    --add-opens=java.base/java.lang=ALL-UNNAMED \
    /opt/burp/burpsuite_pro.jar \
    --project-file=/projects/myproject.burp \
    --user-config-file=/opt/burp-parser/user-config.json"
```

## Shell Alias

Add this to your `.bashrc` or `.zshrc` for convenient usage:

```bash
burp-parse() {
  local project_file="$1"
  shift
  local flags="${@:-proxyHistory}"
  local project_dir=$(dirname "$(realpath "$project_file")")
  local project_name=$(basename "$project_file")

  docker run --rm -v "$project_dir:/projects" burp-parser:latest \
    bash -c "java -jar -Djava.awt.headless=true \
      --add-opens=java.desktop/javax.swing=ALL-UNNAMED \
      --add-opens=java.base/java.lang=ALL-UNNAMED \
      /opt/burp/burpsuite_pro.jar \
      --project-file=/projects/$project_name \
      --user-config-file=/opt/burp-parser/user-config.json \
      $flags"
}
```

Then use it like:

```bash
# Export proxy history with requests and responses (recommended)
burp-parse /path/to/project.burp proxyHistory.both

# Export proxy history (requests only, faster)
burp-parse /path/to/project.burp proxyHistory

# Export audit items
burp-parse /path/to/project.burp auditItems

# Search response headers
burp-parse /path/to/project.burp "responseHeader='.*nginx.*'"
```

## Rebuilding the Image

If you need to rebuild the image (e.g., after updating the plugin):

1. Start the base container:
```bash
docker run -d --name burp-runner -v "$(pwd):/data" -w /data -e DISPLAY=<your-x-server>:0 eclipse-temurin:21-jdk \
  bash -c "apt-get update -qq && apt-get install -y -qq libxext6 libxrender1 libxtst6 libxi6 > /dev/null && tail -f /dev/null"
```

2. Copy files into container:
```bash
docker cp burpsuite_pro.jar burp-runner:/opt/burp/burpsuite_pro.jar
docker cp build/burpsuite-project-file-parser-all.jar burp-runner:/opt/burp-parser/plugin.jar
```

3. Create browser stub directory (required by Burp):
```bash
docker exec burp-runner mkdir -p /opt/burp/burpbrowser/142.0.7444.134
```

4. Run Burp GUI to accept EULA:
```bash
docker exec burp-runner java -jar \
  --add-opens=java.desktop/javax.swing=ALL-UNNAMED \
  --add-opens=java.base/java.lang=ALL-UNNAMED \
  /opt/burp/burpsuite_pro.jar
```

5. Commit the container to an image:
```bash
docker commit -m "Burp Pro with project parser plugin" burp-runner burp-parser:latest
```

6. Clean up:
```bash
docker rm -f burp-runner
```

## Container Paths

| Path | Contents |
|------|----------|
| `/opt/burp/burpsuite_pro.jar` | Burp Suite Professional |
| `/opt/burp-parser/plugin.jar` | Project parser plugin |
| `/opt/burp-parser/user-config.json` | Pre-configured user settings |
| `/opt/burp/burpbrowser/` | Browser stub directory |
