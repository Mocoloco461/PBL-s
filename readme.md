# Nginx Load Balancer & Async Client Verification

**המטרה:** הקמת סביבת Microservices מבוססת Docker, יישום Load Balancing באמצעות Nginx, וכתיבת כלי בדיקה אסינכרוניים לאימות חלוקת העומסים.

### 🛠️ רכיבים נדרשים
* **Docker & Docker Compose**
* **Python 3.9+**
* **Libraries:** `fastapi`, `uvicorn`, `aiohttp`, `pandas` (אופציונלי לניתוח)

---

## שלב 1: הקמת תשתית ה-API
צור אימג' (Image) המבוסס על קוד ה-Python הבא. הקוד חייב להחזיר את מזהה השרת (Hostname) בכל בקשה.

**דרישות:**
1. השתמש ב-`FastAPI` ובשרת `Uvicorn`.
2. ה-Endpoint `/` יחזיר JSON עם ה-Hostname של הקונטיינר.
3. הרץ **שני קונטיינרים** זהים מאותו אימג' (למשל: `app_1`, `app_2`).

```python
# main.py
import os
import socket
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {
        "message": "Hello", 
        "server": os.getenv("HOSTNAME", socket.gethostname())
    }
```

## שלב 2: שכבת ה-Reverse Proxy
הוסף קונטיינר Nginx שישמש כ-Load Balancer מול שני שרתי ה-API.

**דרישות:**
1. הגדרת `upstream` ב-`nginx.conf` המכיל את שני הקונטיינרים משלב 1.
2. ה-Nginx יאזין בפורט `80` ויעביר בקשות ל-API בשיטת **Round Robin** (ברירת מחדל).
3. כל הבקשות מבחוץ חייבות לעבור דרך ה-Nginx בלבד.

## שלב 3: מחולל עומס אסינכרוני (Client)
כתוב סקריפט Python (`generate_load.py`) המבצע בדיקת עומסים מול ה-Nginx.

**דרישות:**
1. שימוש ב-`asyncio` ו-`aiohttp` בלבד (ללא `requests` סינכרוני).
2. שליחת **100 בקשות** במקביל (Concurrent) לכתובת ה-Nginx.
3. חילוץ שדה ה-`server` מה-Response JSON.
4. שמירת התוצאות לקובץ `results.csv` בפורמט הבא:
   ```csv
   request_id, status_code, server_handled
   1, 200, 1a2b3c4d5e
   2, 200, 6f7g8h9i0j
   ...
   ```

## שלב 4: בקרת איכות ואימות (Validation)
כתוב סקריפט נוסף (`verify_distribution.py`) לניתוח הלוגים.

**דרישות:**
1. קריאת קובץ ה-`results.csv`.
2. ספירת מספר הבקשות שטופלו על ידי כל קונטיינר (`unique server_handled`).
3. **תנאי מעבר (Pass Condition):** הוכחה שהיחס בין השרתים הוא בקירוב 50/50 (סטייה מותרת: עד 10%).
4. הדפסת דוח מסכם לטרמינל:
   ```text
   Total Requests: 100
   Server A Count: 51
   Server B Count: 49
   Status: BALANCED (PASS)
   ```