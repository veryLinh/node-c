# <div align='center'>Wileys</div>

<div align='center'>

[![npm version](https://img.shields.io/npm/v/wileys.svg)](https://www.npmjs.com/package/wileys)
[![License](https://img.shields.io/badge/license-GPL%203-blue.svg)](LICENSE)
[![Downloads](https://img.shields.io/npm/dm/wileys.svg)](https://www.npmjs.com/package/wileys)

</div>

## 💝 Donation

Support the development of this project:

**Solana Address:**
```
8xN639anSq5q64793tseCjPaXNgXEPaKxr91CKEuggKd
```

---

**Wileys** is a modern WebSocket-based TypeScript library for interacting with the WhatsApp Web API. This library has been enhanced to address issues with WhatsApp group `@lid` and `@jid` handling, ensuring robust performance.

## ✨ Key Features

- 🚀 **Modern & Fast** - Built with TypeScript and cutting-edge technologies
- 🔧 **Fixed @lid & @jid** - Resolved WhatsApp group `@lid` to @pn` issues
- 📱 **Multi-Device Support** - Supports WhatsApp multi-device connections
- 🔐 **End-to-End Encryption** - Fully encrypted communications
- 📨 **All Message Types** - Supports all message types (text, media, polls, etc.)
- 🎯 **Easy to Use** - Simple and intuitive API

## ⚠️ Disclaimer

This project is not affiliated, associated, authorized, endorsed by, or in any way officially connected with WhatsApp or any of its subsidiaries or affiliates. The official WhatsApp website can be found at [whatsapp.com](https://whatsapp.com).

The maintainers of `wileys` do not condone the use of this application in practices that violate WhatsApp's Terms of Service. We urge users to exercise personal responsibility and use this application fairly and responsibly.

**Use wisely. Avoid spamming. Refrain from excessive automated usage.**

## 📦 Installation

### Stable Version (Recommended)
```bash
npm i wileys
```

### Edge Version (Latest Features)
```bash
npm i wileys@latest
# or
yarn add wileys@latest
```

### Import in Code
```javascript
const { default: makeWASocket } = require("wileys")
// or ES6
import makeWASocket from "wileys"
```

## 🚀 Quick Start

### Example
Here is an example you can use: [example.ts](Example/example.ts) or follow this tutorial to run the Wileys WhatsApp API code:
1. ```bash
   cd path/to/wileys
   ```
2. ```bash
   npm install
   ```
3. ```bash
   node example.js
   ```

### Basic Example
```javascript
const { default: makeWASocket, DisconnectReason, useMultiFileAuthState } = require('wileys')
const { Boom } = require('@hapi/boom')

async function connectToWhatsApp() {
    const { state, saveCreds } = await useMultiFileAuthState('auth_info_baileys')
    
    const sock = makeWASocket({
        auth: state,
        printQRInTerminal: true,
        browser: ['Wileys', 'Desktop', '3.0']
    })

    sock.ev.on('connection.update', (update) => {
        const { connection, lastDisconnect } = update
        if(connection === 'close') {
            const shouldReconnect = (lastDisconnect?.error as Boom)?.output?.statusCode !== DisconnectReason.loggedOut
            console.log('Connection closed due to ', lastDisconnect.error, ', reconnecting ', shouldReconnect)
            if(shouldReconnect) {
                connectToWhatsApp()
            }
        } else if(connection === 'open') {
            console.log('✅ Successfully connected to WhatsApp!')
        }
    })

    sock.ev.on('messages.upsert', async ({ messages }) => {
        for (const m of messages) {
            if (!m.message) continue
            
            console.log('📱 New message:', JSON.stringify(m, undefined, 2))
            
            // Auto reply
            await sock.sendMessage(m.key.remoteJid!, { 
                text: 'Hello! I am a WhatsApp bot using Wileys 🤖' 
            })
        }
    })

    sock.ev.on('creds.update', saveCreds)
}

connectToWhatsApp()
```

# Index

- [Connecting Account](#connecting-account)
    - [Connect with QR Code](#starting-socket-with-qr-code)
    - [Connect with Pairing Code](#starting-socket-with-pairing-code)
    - [Receive Full History](#receive-full-history)
- [Important Notes About Socket Config](#important-notes-about-socket-config)
    - [Caching Group Metadata (Recommended)](#caching-group-metadata-recommended)
    - [Improve Retry System & Decrypt Poll Votes](#improve-retry-system--decrypt-poll-votes)
    - [Receive Notifications in WhatsApp App](#receive-notifications-in-whatsapp-app)
- [Save Auth Info](#saving--restoring-sessions)
- [Handling Events](#handling-events)
    - [Example to Start](#example-to-start)
    - [Decrypt Poll Votes](#decrypt-poll-votes)
    - [Summary of Events on First Connection](#summary-of-events-on-first-connection)
- [Implementing a Data Store](#implementing-a-data-store)
- [WhatsApp IDs Explained](#whatsapp-ids-explain)
- [Utility Functions](#utility-functions)
- [Sending Messages](#sending-messages)
    - [Non-Media Messages](#non-media-messages)
    - [Sending with Link Preview](#sending-messages-with-link-previews)
    - [Media Messages](#media-messages)
- [Modify Messages](#modify-messages)
- [Manipulating Media Messages](#manipulating-media-messages)
- [Reject Call](#reject-call)
- [Send States in Chat](#send-states-in-chat)
- [Modifying Chats](#modifying-chats)
- [User Queries](#user-querys)
- [Change Profile](#change-profile)
- [Groups](#groups)
- [Privacy](#privacy)
- [Broadcast Lists & Stories](#broadcast-lists--stories)
- [Writing Custom Functionality](#writing-custom-functionality)
- [Tips & Best Practices](#tips--best-practices)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)

## Connecting Account

WhatsApp provides a multi-device API that allows `wileys` to authenticate as a secondary WhatsApp client via QR code or pairing code.

### Starting Socket with **QR Code**

> [!TIP]
> Customize the browser name using the `Browsers` constant. See available configurations [here](https://baileys.whiskeysockets.io/types/BrowsersMap.html).

```javascript
const { default: makeWASocket, Browsers } = require("wileys")

const sock = makeWASocket({
    browser: Browsers.ubuntu('My App'),
    printQRInTerminal: true
})
```

Upon successful connection, a QR code will be printed in the terminal. Scan it with WhatsApp on your phone to log in.

### Starting Socket with **Pairing Code**

> [!IMPORTANT]
> Pairing codes are not part of the Mobile API. They allow connecting WhatsApp Web without a QR code, but only one device can be connected. See [WhatsApp FAQ](https://faq.whatsapp.com/1324084875126592/?cms_platform=web).

Phone numbers must exclude `+`, `()`, or `-`, and include the country code.

```javascript
const { default: makeWASocket } = require("wileys")

const sock = makeWASocket({
    printQRInTerminal: false
})

// Normal Pairing
if (!sock.authState.creds.registered) {
    const number = '6285134816783'
    const code = await sock.requestPairingCode(number)
    console.log('🔑 Pairing Code:', code)
}

// Custom Pairing
if (!sock.authState.creds.registered) {
    const pair = "YP240125" // 8 characters
    const number = '6285134816783'
    const code = await sock.requestPairingCode(number, pair)
    console.log('🔑 Custom Pairing Code:', code)
}
```

### Receive Full History

1. Set `syncFullHistory` to `true`.
2. By default, Wileys uses Chrome browser config. For desktop-like connections (to receive more message history), use:

```javascript
const { default: makeWASocket, Browsers } = require("wileys")

const sock = makeWASocket({
    browser: Browsers.macOS('Desktop'),
    syncFullHistory: true
})
```

## Important Notes About Socket Config

### Caching Group Metadata (Recommended)

For group usage, implement caching for group metadata:

```javascript
const { default: makeWASocket } = require("wileys")
const NodeCache = require('node-cache')

const groupCache = new NodeCache({ stdTTL: 5 * 60, useClones: false })

const sock = makeWASocket({
    cachedGroupMetadata: async (jid) => groupCache.get(jid)
})

sock.ev.on('groups.update', async ([event]) => {
    const metadata = await sock.groupMetadata(event.id)
    groupCache.set(event.id, metadata)
})

sock.ev.on('group-participants.update', async (event) => {
    const metadata = await sock.groupMetadata(event.id)
    groupCache.set(event.id, metadata)
})
```

### Improve Retry System & Decrypt Poll Votes

Enhance message sending and poll vote decryption with a store:

```javascript
const sock = makeWASocket({
    getMessage: async (key) => await getMessageFromStore(key)
})
```

### Receive Notifications in WhatsApp App

Disable online status to receive notifications:

```javascript
const sock = makeWASocket({
    markOnlineOnConnect: false
})
```

## Saving & Restoring Sessions

Avoid rescanning QR codes by saving credentials:

```javascript
const makeWASocket = require("wileys").default
const { useMultiFileAuthState } = require("wileys")

async function connect() {
    const { state, saveCreds } = await useMultiFileAuthState('auth_info_baileys')
    const sock = makeWASocket({ auth: state })
    sock.ev.on('creds.update', saveCreds)
}

connect()
```

> [!IMPORTANT]
> `useMultiFileAuthState` saves auth state in a folder. For production, use SQL/no-SQL databases and carefully manage key updates.

## Handling Events

Wileys uses EventEmitter for events, fully typed for IDE support.

> [!IMPORTANT]
> See all events [here](https://baileys.whiskeysockets.io/types/BaileysEventMap.html).

```javascript
const sock = makeWASocket()
sock.ev.on('messages.upsert', ({ messages }) => {
    console.log('Got messages:', messages)
})
```

### Example to Start

```javascript
const makeWASocket = require("wileys").default
const { DisconnectReason, useMultiFileAuthState } = require("wileys")
const { Boom } = require('@hapi/boom')

async function connectToWhatsApp() {
    const { state, saveCreds } = await useMultiFileAuthState('auth_info_baileys')
    const sock = makeWASocket({
        auth: state,
        printQRInTerminal: true
    })

    sock.ev.on('connection.update', (update) => {
        const { connection, lastDisconnect } = update
        if(connection === 'close') {
            const shouldReconnect = (lastDisconnect.error as Boom)?.output?.statusCode !== DisconnectReason.loggedOut
            console.log('Connection closed due to ', lastDisconnect.error, ', reconnecting ', shouldReconnect)
            if(shouldReconnect) {
                connectToWhatsApp()
            }
        } else if(connection === 'open') {
            console.log('Opened connection')
        }
    })

    sock.ev.on('messages.upsert', async ({ messages }) => {
        for (const m of messages) {
            console.log(JSON.stringify(m, undefined, 2))
            console.log('Replying to', m.key.remoteJid)
            await sock.sendMessage(m.key.remoteJid!, { text: 'Hello World' })
        }
    })

    sock.ev.on('creds.update', saveCreds)
}

connectToWhatsApp()
```

## License

Distributed under the GPL-3.0 License. See `LICENSE` for more information.

## Acknowledgments

- [Group](https://chat.whatsapp.com/DUWddjCXWv4CCoeS8uP2lP) - Comunity

---

<div align='center'>

**Fork from baileys modified by wileys**

*Wileys - Modern WhatsApp Web API with Fixed @lid To @pn*

</div>
