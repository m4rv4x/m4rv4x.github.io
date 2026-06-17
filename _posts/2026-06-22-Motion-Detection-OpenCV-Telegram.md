---
title: "Motion detection avec OpenCV : alertes Telegram en temps réel"
date: 2026-06-22 10:00 +0200
categories: ["Dev"]
tags: ["opencv", "python", "motion-detection", "telegram", "surveillance", "self-hosted", "linux", "webcam"]
author: marvax
---

> Un script Python, une webcam, et un bot Telegram : tu as un système de surveillance maison en 30 lignes de code. Pas de cloud, pas d'abonnement, tout tourne localement.

<!--more-->

## Le concept

```
Webcam (/dev/video0)
       │
       ▼
OpenCV (diff frame N vs N-1)
       │
       ▼
Seuil de détection dépassé ?
       │ OUI
       ▼
Capture + envoi via Telegram Bot API
       │
       ▼
Alerte sur @TonBot (photo + timestamp)
```

## Installation

```bash
pip install opencv-python-headless requests
```

## Le script

```python
#!/usr/bin/env python3
"""Motion detection with Telegram alerts."""
import cv2
import os
import time
import requests
from datetime import datetime

# Config
TELEGRAM_TOKEN = os.environ.get("MOTION_TELEGRAM_TOKEN", "")
TELEGRAM_CHAT_ID = os.environ.get("MOTION_TELEGRAM_CHAT_ID", "")
CAMERA = int(os.environ.get("MOTION_CAMERA", "0"))
THRESHOLD = int(os.environ.get("MOTION_THRESHOLD", "5000"))
COOLDOWN = int(os.environ.get("MOTION_COOLDOWN", "10"))

def send_alert(image_path):
    """Send photo to Telegram."""
    url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendPhoto"
    with open(image_path, "rb") as photo:
        requests.post(url, data={
            "chat_id": TELEGRAM_CHAT_ID,
            "caption": f"🚨 Motion detected at {datetime.now().strftime('%H:%M:%S')}"
        }, files={"photo": photo})

def main():
    cap = cv2.VideoCapture(CAMERA)
    cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
    cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)

    _, prev_frame = cap.read()
    prev_gray = cv2.cvtColor(prev_frame, cv2.COLOR_BGR2GRAY)
    prev_gray = cv2.GaussianBlur(prev_gray, (21, 21), 0)

    last_alert = 0
    print(f"[+] Motion detection started (camera {CAMERA}, threshold {THRESHOLD})")

    while True:
        ret, frame = cap.read()
        if not ret:
            time.sleep(1)
            continue

        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        gray = cv2.GaussianBlur(gray, (21, 21), 0)

        delta = cv2.absdiff(prev_gray, gray)
        thresh = cv2.threshold(delta, 25, 255, cv2.THRESH_BINARY)[1]
        thresh = cv2.dilate(thresh, None, iterations=2)

        score = thresh.sum()

        if score > THRESHOLD and time.time() - last_alert > COOLDOWN:
            ts = datetime.now().strftime("%Y%m%d_%H%M%S")
            path = f"/tmp/motion_{ts}.jpg"
            cv2.imwrite(path, frame)
            print(f"[!] Motion detected (score={score}) — sending alert")
            try:
                send_alert(path)
            except Exception as e:
                print(f"[!] Telegram error: {e}")
            last_alert = time.time()

        prev_gray = gray
        time.sleep(0.1)

if __name__ == "__main__":
    main()
```

## Service systemd

```ini
# ~/.config/systemd/user/motion-server.service
[Unit]
Description=Motion Detection Server
After=graphical-session.target

[Service]
Type=simple
Environment=MOTION_TELEGRAM_TOKEN=TON_TOKEN
Environment=MOTION_TELEGRAM_CHAT_ID=TON_CHAT_ID
Environment=MOTION_CAMERA=0
Environment=MOTION_THRESHOLD=5000
ExecStart=/home/user/venv/bin/python /home/user/motion-server.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now motion-server
journalctl --user -u motion-server -f
```

## Tuning

| Paramètre | Effet | Recommandé |
|-----------|-------|------------|
| `THRESHOLD` | Sensibilité (plus bas = plus sensible) | 3000-8000 |
| `COOLDOWN` | Secondes entre 2 alertes | 10-30 |
| `FRAME_SIZE` | Résolution (plus bas = plus rapide) | 640x480 |
| `GaussianBlur` | Réduit le bruit | (21, 21) |

## Améliorations

- **Masque de zone** : ignorer une partie de l'image (fenêtre, route)
- **Enregistrement vidéo** : sauvegarder les 10 secondes avant/déclenchement
- **Web UI** : Flask dashboard avec historique des détections
- **Multi-caméras** : un thread par caméra
- **LED webcam** : allumer une LED quand la détection est active (rassurant)

## Le piège du LED

Certaines webcams ont une LED qui clignote quand elles captent. OpenCV peut parfois ne pas libérer la caméra proprement → la LED reste allumée même après arrêt du script. Solution :

```python
# Toujours libérer proprement
cap.release()
cv2.destroyAllWindows()
```

Et dans le service systemd :
```ini
ExecStop=/bin/kill -TERM $MAINPID
```

---

*Tu veux un système de surveillance complet ? [Contacte-moi](mailto:m4rv4x@protonmail.com).*
