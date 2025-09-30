---
title: "launchctl: Bootstrap failed: 5: Input/output error"
date: 2025-09-26
categories:
- Programming
tags:
- macOS
- launchd
---

#### Initial symtoms

We have an internal service that needs to start a boot. The responsible team wrote a simple launchd script, like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.watchdog</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/watchdog</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>SessionCreate</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/var/log/watchdog.out</string>

    <key>StandardErrorPath</key>
    <string>/var/log/watchdog.err</string>
</dict>
</plist>
```

However, when trying to load it, we ran into this error:

```sh
sudo launchctl bootstrap system /Library/LaunchDaemons/com.example.watchdog.plist
Password:
Bootstrap failed: 5: Input/output error
```

Interestingly, the same error also appears if the service is already running. But in this case, it wasn't:

```sh
sudo launchctl bootout system /Library/LaunchDaemons/com.example.watchdog.plist
Boot-out failed: 5: Input/output error
```

To rule out issues with the binary itself, I tried starting it manually:

```sh
/usr/local/bin/watchdog
```

It ran without problems, so the error cleary happens before the binary is executed.

Unfortunately, `man launchctl` shows no option to enable debugging for `bootstrap`, which makes troubleshooting trickier.

#### Narrowing down the cause

I then tried swapping the binary path in a known working launchd script with `/usr/local/bin/watchdog`. The modified script looked like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
        <key>Crashed</key>
        <true/>
        <key>KeepAlive</key>
        <true/>
        <key>Label</key>
        <string>org.openvpn.client</string>
        <key>ProgramArguments</key>
        <array>
                <string>/usr/local/bin/watchdog</string>
        </array>
        <key>RunAtLoad</key>
        <true/>
        <key>SessionCreate</key>
        <true/>
        <key>StandardOutPath</key>
        <string>/var/log/ovpnagent.log</string>
</dict>
</plist>
```

By removing keys one by one, I eventually found the culprit: the label value. If the label is set to `com.example.watchdog`, the bootstrap failed. If I changed it to something else, it worked.

#### Root cause

It turned out the service had been accidentally disabled during earlier experiments with the `.plist`.
Running:

```sh
sudo launchctl print system
```

revealed this:

```sh
disabled services = {
        "com.example.watchdog" => disabled
}
```

#### Fix

The solution is to re-enable the service and then bootstrap it again:

```sh
sudo launchctl enable system/com.example.watchdog
sudo launchctl bootstrap system /Library/LaunchDaemons/com.example.watchdog.plist
```
