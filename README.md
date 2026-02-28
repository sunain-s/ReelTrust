# ReelTrust

ReelTrust is a mobile-first deepfake verification project that combines:
- on-device video verification using the Presage SmartSpectra SDK (Android),
- backend verification + deduplication (Node.js + MongoDB Atlas), and
- blockchain anchoring (Solana Devnet Memo transactions).

The backend accepts signed media-hash submissions, checks whether the hash already exists, optionally anchors/validates on Solana, and returns a normalized verification response.

Core SDK documentation: [Presage SmartSpectra SDK for Android](https://docs.physiology.presagetech.com/android/index.html)

This project was developed during HackLondon 2026 and won “Best Use of Presage” from [Major League Hacking](https://mlh.io/).

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Backend Setup (WSL)](#backend-setup-wsl)
- [Run the Full Demo (Android + WSL)](#run-the-full-demo-android--wsl)
- [API Contract](#api-contract)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Notes](#notes)
- [Contributors](#contributors)



## Project Overview

ReelTrust is built around the **Presage SmartSpectra SDK** as the primary verification technology. The Android experience and core detection capability come from SmartSpectra, while the backend and blockchain layers add persistence, deduplication, and auditability.

ReelTrust verifies media authenticity using a hash-first workflow:

1. Android app computes/collects media verification context.
2. Backend receives a signed request containing media hash and metadata.
3. Backend checks MongoDB Atlas for prior submissions.
4. Backend anchors to Solana Devnet (Memo) for immutable proof when needed.
5. Backend returns:

```json
{
	"status": "verified",
	"alreadyExists": false,
	"txId": "<solana_tx_signature_or_null>"
}
```



## Architecture

### Backend (`backend/`)

- Express API with request validation and HMAC signature verification.
- MongoDB Atlas persistence using Mongoose.
- Solana service for:
	- writing memo-based anchor transactions,
	- verify-on-read validation of stored `txId`.
- Replay-protection primitives: timestamp skew checks + nonce.

### Android (`android/`)

- Android app + Presage SmartSpectra SDK modules for video verification and UI flows.
- Demo path connects device to WSL backend via `adb reverse`.



## Tech Stack

### Core Technology

- [Presage SmartSpectra SDK for Android](https://docs.physiology.presagetech.com/android/index.html) (primary media verification engine)

### Backend

- [Node.js](https://nodejs.org/)
- [Express](https://expressjs.com/)
- [Mongoose](https://mongoosejs.com/)
- [MongoDB Atlas](https://www.mongodb.com/atlas)
- [Solana Web3.js](https://solana-labs.github.io/solana-web3.js/)
- [dotenv](https://github.com/motdotla/dotenv)

### Mobile / Platform

- [Android Studio](https://developer.android.com/studio)
- [Kotlin](https://kotlinlang.org/)
- [ADB / Platform Tools](https://developer.android.com/tools/adb)

### Blockchain

- [Solana Devnet](https://solana.com/docs/references/clusters#devnet)
- [Memo Program](https://spl.solana.com/memo)



## Repository Structure

```text
ReelTrust/
├── android/
│   ├── app/                  # Main Android application module
│   └── sdk/                  # Presage SmartSpectra SDK module and assets
├── backend/                  # Node/Express verification backend
│   ├── package.json
│   ├── .env.example
│   ├── src/
│   │   ├── config.js
│   │   ├── server.js
│   │   ├── models/
│   │   │   └── mediaSubmission.js
│   │   ├── solana/
│   │   │   └── service.js
│   │   └── utils/
│   │       └── signing.js
│   └── tests/
│       └── api.e2e.js
└── README.md
```



## Prerequisites

### Required

- Node.js 18+
- npm 9+
- MongoDB Atlas connection string
- Solana keypair for Devnet anchoring

### For Android physical device demo

- Android Studio installed on Windows
- Presage SmartSpectra SDK integrated in `android/sdk`
- USB debugging enabled on device
- Android Platform Tools (`adb`) installed and available

### Android Presage API key (required)

Add your Presage key to `android/local.properties`:

```properties
PRESAGE_API_KEY=<your_presage_api_key>
```

Notes:
- Use the exact key name `PRESAGE_API_KEY` (this is what `android/app/build.gradle.kts` reads).
- Keep `local.properties` local to your machine; do not commit your real API key.



## Backend Setup (WSL)

From WSL:

```bash
git clone <your-reeltrust-repo-url>
cd ReelTrust/backend
npm install
```

Create/update `.env` using `backend/.env.example` keys.

Minimum recommended values:

```dotenv
PORT=3000
REQUEST_BODY_LIMIT=10mb
MONGODB_URI=<your_atlas_uri>
CLIENT_ID=android-app
CLIENT_SECRET=<your_shared_secret>
MAX_CLOCK_SKEW_MS=300000
SOLANA_RPC_URL=https://api.devnet.solana.com
SOLANA_KEYPAIR_PATH=/home/<your-user>/.config/solana/devnet.json
SOLANA_PRIVATE_KEY_JSON=
SOLANA_REQUIRED=false
SOLANA_VERIFY_ON_READ=true
```

Run backend:

```bash
npm run start
```

Health check:

```bash
curl http://127.0.0.1:3000/health
```

Expected:

```json
{"status":"ok"}
```



## Run the Full Demo (Android + WSL)

### 1) Start backend in WSL

```bash
cd ReelTrust/backend
npm run start
```

### 2) On Windows, bridge device traffic to backend

```powershell
cmd /c "C:\Users\<your-user>\AppData\Local\Android\Sdk\platform-tools\adb.exe reverse tcp:3000 tcp:3000"
cmd /c "C:\Users\<your-user>\AppData\Local\Android\Sdk\platform-tools\adb.exe reverse --list"
```

You should see `tcp:3000 tcp:3000`.

### 3) (If needed) ensure Windows can reach WSL backend

If `curl.exe http://127.0.0.1:3000/health` fails in Windows but WSL curl works, use portproxy (Admin PowerShell):

```powershell
netsh interface portproxy add v4tov4 listenaddress=127.0.0.1 listenport=3000 connectaddress=<WSL_IP> connectport=3000
netsh interface portproxy show all
```

### 4) Validate from device context

- Open phone browser: `http://127.0.0.1:3000/health`
- Expected: `{"status":"ok"}`

### 5) Run Android app from Android Studio

- Connect phone
- Ensure app uses backend base URL: `http://127.0.0.1:3000`
- Run upload/analysis flow



## API Contract

### Endpoint

`POST /api/v1/videos/submit`

### Request body

```json
{
	"videoHash": "<64-char sha256 hex>",
	"mediaType": "video",
	"metadata": {
		"source": "android"
	},
	"auth": {
		"clientId": "android-app",
		"timestamp": 1700000000000,
		"nonce": "random_nonce",
		"requestSignature": "hmac_sha256_signature"
	}
}
```

### Success response

```json
{
	"status": "verified",
	"alreadyExists": false,
	"txId": "<solana_signature_or_null>"
}
```

### Common error responses

- `400` invalid payload / invalid JSON / aborted request
- `401` invalid signature or client
- `409` stored txId failed on-chain verification
- `413` request body too large
- `500` database error
- `502` Solana anchor or verify error



## Testing

From `backend/`:

```bash
npm run test:api
```

This E2E suite validates:

- health endpoint
- valid submit
- duplicate hash behavior (`alreadyExists`)
- signature/hash/timestamp/mediaType validation
- verify-on-read tamper detection for stored `txId`



## Troubleshooting

### `npm run test:api` fails with `ENOENT package.json`

Run from backend folder:

```bash
cd ReelTrust/backend
npm run test:api
```

### Phone cannot reach `127.0.0.1:3000`

- Re-run `adb reverse tcp:3000 tcp:3000`
- Confirm with `adb reverse --list`
- If needed, configure Windows `portproxy` to WSL IP

### Android upload shows `unexpected end of stream`

- Check backend logs for explicit JSON error (`413`, invalid JSON, aborted request)
- Increase `REQUEST_BODY_LIMIT` if needed
- Ensure app sends expected JSON contract for `/api/v1/videos/submit`

### Android shows missing API key / SDK init error

- Ensure `android/local.properties` contains:

```properties
PRESAGE_API_KEY=<your_presage_api_key>
```
- Re-sync Gradle and rebuild the app.

### Solana failures (`502`)

- Confirm keypair path or inline key JSON is valid
- Ensure funded Devnet wallet
- Verify `SOLANA_RPC_URL` points to Devnet



## Notes

- This repo includes both Android and backend code; backend is the primary verification API runtime.
- This project was developed on Windows x64 systems using WSL Ubuntu 24.04.
- The SmartSpectra SDK is typically used for live camera health-related applications rather than offline deepfake video analysis, so the video upload flow may have some accuracy limitations.


## Contributors

- [Yik Hui Ng](https://github.com/yhCannotThink)
- [Sunain Syed](https://github.com/sunain-s)