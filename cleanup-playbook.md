# Home Assistant Yellow - SSH & Disk Cleanup Playbook

## SSH Connection

### Prerequisites
- Advanced SSH & Web Terminal add-on installed
- Your public key added to the add-on's `authorized_keys` field
- Protection mode disabled (for Docker access)

### SSH Config (~/.ssh/config)
```
Host ha homeassistant 
    HostName homeassistant.becquerel
    User gart
    IdentityFile ~/.ssh/id_ed25519
```

### Connect
```bash
ssh ha
```

---

## Disk Space Analysis

### Check overall usage
```bash
du -sh /* 2>/dev/null | sort -hr | head -20
```

### Check Home Assistant data
```bash
du -sh /homeassistant/* 2>/dev/null | sort -hr | head -15
```

### Check disk usage via HA CLI
```bash
ha host disks usage
```

### Check Docker usage
```bash
docker system df
docker system df -v | head -40
```

---

## Cleanup Commands

### 1. Docker cleanup (biggest impact)
Removes unused images, build cache, stopped containers:
```bash
docker system prune -a
```

### 2. Remove old log files
```bash
rm /homeassistant/home-assistant.log.1 /homeassistant/home-assistant.log.old
```

### 3. Clear TTS cache
```bash
rm /homeassistant/tts/*
```

### 4. Check Zigbee2MQTT logs
```bash
du -sh /homeassistant/zigbee2mqtt/log/* | sort -hr
```
Delete old log folders if needed.

---

## Notes

- Journal logs (~478MB in /var/log/journal) require physical console access to clean
- Backups are managed via HA UI (Settings → System → Backups)
- Database can be purged via Settings → System → Recorder
- Docker images for active add-ons cannot be removed without uninstalling the add-on
