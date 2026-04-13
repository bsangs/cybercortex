# 08 — launchd Plist Files

3개 plist를 `agent/launchd/`에 생성 후, `~/Library/LaunchAgents/`에 심볼릭 링크하고 `launchctl load`로 등록한다.

## com.{username}.ingest-10min.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.{username}.ingest-10min</string>

    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>{project-path}/agent/scripts/run-tier.sh</string>
        <string>10min</string>
    </array>

    <key>StartInterval</key>
    <integer>600</integer>

    <key>StandardOutPath</key>
    <string>{project-path}/agent/logs/launchd-10min.log</string>

    <key>StandardErrorPath</key>
    <string>{project-path}/agent/logs/launchd-10min.log</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>HOME</key>
        <string>/Users/{username}</string>
    </dict>
</dict>
</plist>
```

## com.{username}.ingest-3hr.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.{username}.ingest-3hr</string>

    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>{project-path}/agent/scripts/run-tier.sh</string>
        <string>3hr</string>
    </array>

    <key>StartInterval</key>
    <integer>10800</integer>

    <key>StandardOutPath</key>
    <string>{project-path}/agent/logs/launchd-3hr.log</string>

    <key>StandardErrorPath</key>
    <string>{project-path}/agent/logs/launchd-3hr.log</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>HOME</key>
        <string>/Users/{username}</string>
    </dict>
</dict>
</plist>
```

## com.{username}.ingest-24hr.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.{username}.ingest-24hr</string>

    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>{project-path}/agent/scripts/run-tier.sh</string>
        <string>24hr</string>
    </array>

    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>4</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>

    <key>StandardOutPath</key>
    <string>{project-path}/agent/logs/launchd-24hr.log</string>

    <key>StandardErrorPath</key>
    <string>{project-path}/agent/logs/launchd-24hr.log</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>HOME</key>
        <string>/Users/{username}</string>
    </dict>
</dict>
</plist>
```

## 등록 명령어

```bash
# 심볼릭 링크
ln -sf {project-path}/agent/launchd/com.{username}.ingest-10min.plist ~/Library/LaunchAgents/
ln -sf {project-path}/agent/launchd/com.{username}.ingest-3hr.plist ~/Library/LaunchAgents/
ln -sf {project-path}/agent/launchd/com.{username}.ingest-24hr.plist ~/Library/LaunchAgents/

# 등록
launchctl load ~/Library/LaunchAgents/com.{username}.ingest-10min.plist
launchctl load ~/Library/LaunchAgents/com.{username}.ingest-3hr.plist
launchctl load ~/Library/LaunchAgents/com.{username}.ingest-24hr.plist
```
