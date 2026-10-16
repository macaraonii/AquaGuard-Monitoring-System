# AquaGuard Monitoring System

**Caribbean Waves Safety Operations**

AquaGuard is an IoT enabled wearable arm device with AI camera-based image processing for real-time drowning detection, emergency alert management, and guest armband tracking. The wearable armbands and AI camera detections stream alerts into Firebase, and the dashboard turns them into a prioritized, always-live picture of what needs attention right now.

## Tech Stack

- **Frontend:** React, Vite, Tailwind CSS
- **Data Visualization:** Recharts, with dynamic time-range filtering
- **Date Handling:** date-fns
- **Backend / Database:** Firebase Realtime Database (live `/alerts`, `/assignments` and `/inventory` streams)
- **Hardware Integration:** ESP32 wearable armbands (heartbeat / status tracking) and AI camera feed integration
- **Access:** Direct dashboard access. The app opens straight into `AdminDashboard.jsx` with no login screen. `firebase.js` signs each browser session in anonymously so the database security rules can require `auth != null`.

## Key Features

### Live Emergency Dashboard
- Real-time 3-tier severity tracking: **Minor**, **Major** and **Critical**.
- **5-minute priority pin:** an open Critical alert stays pinned to the top of the list while it is under 5 minutes old (based on `readAtMs`). After that it falls back to a strict newest-first order. The list re-evaluates every 30 seconds, so the pin expires live.
- **Continuous blinking** on every open Critical alert (row and badge) until the operator clicks **Resolve**. Users with reduced-motion enabled get a static highlight instead.
- **Clickable status badges** on the dashboard preview jump straight to the Alert Logs, pre-filtered to that alert's severity.
- **Audible critical alarm:** browsers block autoplay audio, so the operator clicks **Start Monitoring** once to unlock it. After that, every newly arrived open Critical alert plays the alarm. A **Mute Alerts** toggle is available in the header.
- **Guest names on alerts:** alerts only carry an `armbandID`, so the dashboard looks up the matching guest in `/assignments`. Open alerts show whoever currently holds the armband. Resolved alerts show the guest who had it at the time.

### Incident Trend Analysis
- Interactive Recharts line graph of Minor, Major and Critical alert volume.
- Time-range selector: **Today** (by hour), **This Week** (by day), **This Month** (by day of month) and **This Year** (by month).
- Axis labels and tooltips adapt to the selected range.

### Armband & Guest Management
- Manual guest registration and armband assignment (first name, last name, age, stay dates, armband).
- **Strict duplicate validation:** an armband that is held by an Active or Offline guest cannot be assigned again until that guest is marked **Returned**. The check reads fresh data from Firebase right before writing.
- Real-time inventory counter showing active, offline and available armbands. Inventory size is set by `DEFAULT_ARMBAND_INVENTORY` in `AdminDashboard.jsx` (set it to `10` for a 10-armband deployment). It can also be overridden live by writing a number to `/inventory/totalArmbands`.
- **ESP32 status indicator** in the header (online / offline), based on a periodic health check.
- **Automated 10-minute inactivity sweep:** an Active armband whose `lastSeen` is older than 10 minutes is flipped to **Offline** in Firebase, and flips back to Active when it reports again. Offline guests stay in the monitored list with a highlighted amber card until they are marked Returned.
- The "Guests currently monitored" list shows the newest assignments first.

### Alert Logs & Reporting
- Filterable historical alert table: **All**, **Minor**, **Major**, **Critical**, **Camera** and **Armband**.
- Camera alerts are identified by the word "camera" in the alert's `cause`.
- One-click **CSV export** of the currently filtered log.

## Project Structure

```
.
├── client/                     # React + Vite frontend
│   ├── public/
│   │   └── critical-alarm.mp3  # Audible critical alarm
│   ├── src/
│   │   ├── api.js              # ESP32 health-check helper
│   │   ├── config/
│   │   │   └── firebase.js     # Firebase initialisation (Auth + Realtime Database)
│   │   ├── pages/
│   │   │   └── AdminDashboard.jsx
│   │   ├── App.jsx
│   │   ├── main.jsx
│   │   └── index.css
│   ├── .env                    # Firebase keys (not committed)
│   ├── index.html
│   ├── package.json
│   ├── tailwind.config.cjs
│   ├── postcss.config.cjs
│   └── vite.config.js
└── res/                        # Assets (includes the Caribbean Waves logo)
```

## Getting Started

### 1) Install dependencies

```bash
cd client
npm install
npm install recharts
npm install date-fns
```

### 2) Configure environment variables

Create a `.env` file in the `client/` directory with your Firebase project keys (Firebase Console → Project settings → Your apps → SDK setup and configuration):

```bash
VITE_FIREBASE_apiKey=your-api-key
VITE_FIREBASE_authDomain=your-project-id.firebaseapp.com
VITE_FIREBASE_databaseURL=https://your-project-id-default-rtdb.firebaseio.com
VITE_FIREBASE_projectId=your-project-id
VITE_FIREBASE_storageBucket=your-project-id.appspot.com
VITE_FIREBASE_messagingSenderId=your-messaging-sender-id
VITE_FIREBASE_appId=your-app-id
VITE_FIREBASE_measurementId=your-measurement-id   # optional
```

| Variable | Firebase value |
| --- | --- |
| `VITE_FIREBASE_apiKey` | API Key |
| `VITE_FIREBASE_authDomain` | Auth Domain |
| `VITE_FIREBASE_databaseURL` | Realtime Database URL |
| `VITE_FIREBASE_projectId` | Project ID |
| `VITE_FIREBASE_storageBucket` | Storage Bucket |
| `VITE_FIREBASE_messagingSenderId` | Messaging Sender ID |
| `VITE_FIREBASE_appId` | App ID |
| `VITE_FIREBASE_measurementId` | Measurement ID (optional) |

Notes:
- The variable names are **case-sensitive** and must match `client/src/config/firebase.js` exactly (note the camelCase suffixes).
- Restart the dev server after changing `.env`.
- Never commit `.env`. Make sure it is listed in `.gitignore`.
- Copy the Realtime Database URL exactly as shown in the Firebase Console, because it differs by region.

### 3) Enable anonymous sign-in

`firebase.js` signs each browser session in anonymously so the database rules can require authentication. In the Firebase Console, go to **Authentication → Sign-in method** and enable **Anonymous**.

### 4) Run the app

```bash
npm run dev
```

Visit [http://localhost:5173](http://localhost:5173). The app opens directly into the AquaGuard dashboard.

For a production build, run `npm run build`.

## Firebase Realtime Database

### Data model

The dashboard listens live to three paths:

| Path | Purpose |
| --- | --- |
| `/alerts` | Incoming alerts from ESP32 armbands and the AI camera |
| `/assignments` | Guest-to-armband assignment records (history and current) |
| `/inventory/totalArmbands` | Optional number that overrides the armband inventory size |

**Alert** (`/alerts/{id}`)

| Field | Description |
| --- | --- |
| `alertID` | Optional display id |
| `alertlevel` | `Minor`, `Major` or `Critical` |
| `cause` | Human-readable cause (include "camera" for camera-sourced alerts) |
| `armbandID` | Armband that triggered the alert, for example `CW-001` |
| `readAt` | ISO string or epoch milliseconds. If missing, the dashboard stamps it on arrival and writes it back |
| `status` | `Open` or `Resolved` |

**Assignment** (`/assignments/{id}`, for example `GA1`, `GA2`)

| Field | Description |
| --- | --- |
| `assignmentsID` | Same as the record id |
| `fName`, `lName`, `age` | Guest details |
| `armbandID` | Assigned armband, for example `CW-001` |
| `assignedAt` | ISO timestamp |
| `lastSeen` | Last time the armband was heard from. The ESP32 must keep this updated (ISO string, epoch milliseconds or epoch seconds). The 10-minute sweep depends on it |
| `stayStart`, `stayEnd` | Stay dates (`YYYY-MM-DD`) |
| `status` | `Active`, `Offline` or `Returned` |

Armband ids are compared in a normalized form, so `CW-001`, `cw001` and `CW-1` all refer to the same armband.

**Example**

```json
{
  "alerts": {
    "-NxAbc123": {
      "alertID": 135,
      "alertlevel": "Critical",
      "cause": "Armband: irregular heartbeat detected",
      "armbandID": "CW-001",
      "readAt": "2026-10-11T09:15:00.000Z",
      "status": "Open"
    },
    "-NxAbc124": {
      "alertID": 136,
      "alertlevel": "Major",
      "cause": "Camera: swimmer motionless underwater",
      "armbandID": "CW-001",
      "readAt": "2026-10-11T09:16:30.000Z",
      "status": "Open"
    }
  },
  "assignments": {
    "GA8": {
      "assignmentsID": "GA8",
      "fName": "Ana",
      "lName": "Reyes",
      "age": 9,
      "armbandID": "CW-001",
      "assignedAt": "2026-10-11T08:00:00.000Z",
      "lastSeen": "2026-10-11T09:14:50.000Z",
      "stayStart": "2026-10-10",
      "stayEnd": "2026-10-12",
      "status": "Active"
    }
  },
  "inventory": {
    "totalArmbands": 10
  }
}
```

### Security rules

Open the Firebase Console → **Realtime Database → Rules**.

**Development only** (open access, so anyone with your database URL can read and write):

```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```

**Production** (requires an authenticated session):

```json
{
  "rules": {
    ".read": "auth != null",
    ".write": "auth != null"
  }
}
```

The app signs in anonymously on load, so `auth != null` is satisfied by the dashboard itself. Because there is no login screen, anyone who can reach the deployed app also gets an anonymous session and full dashboard access. Before a real deployment, restrict who can open the app (for example with hosting-level access control, or Firebase email/password authentication with rules based on the signed-in user), and make sure the ESP32 devices and AI camera service write to the database with credentials your rules allow.