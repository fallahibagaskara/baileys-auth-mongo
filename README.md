# baileys-auth-mongo 🚀💾

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![npm version](https://img.shields.io/npm/v/baileys-auth-mongo)](https://www.npmjs.com/package/baileys-auth-mongo)
[![npm downloads](https://img.shields.io/npm/dm/baileys-auth-mongo)](https://www.npmjs.com/package/baileys-auth-mongo)

✨ Elevate your WhatsApp bot's reliability with seamless MongoDB session persistence! ✨

`baileys-auth-mongo` is a **fork** of [mongo-baileys](https://www.npmjs.com/package/mongo-baileys) with a bug fix for binary data (`Buffer`) storage to ensure compatibility with the latest versions of MongoDB and Baileys.
All APIs and usage remain the same, but the storage implementation has been improved so it no longer triggers errors.

## ✨ Why Choose baileys-auth-mongo?

- **Bugfix MongoDB** – no more `$set` errors when storing `Buffer` or `BinData`.
- **Persistence** – bot sessions are stored in MongoDB, safe even after server restarts.
- **Reliability** – automatic reconnect keeps your bot online.
- **TypeScript-ready** – includes type definitions out of the box.
- **Effortless Integration** – can be used as a drop-in replacement for `mongo-baileys`.

## 📦 Installation

```bash
npm install github:fallahibagaskara/baileys-auth-mongo
```

## 🛠️ Usage

### JavaScript

```javascript
import { MongoClient } from "mongodb";
import { makeWASocket, AnyMessageContent } from "@whiskeysockets/baileys";
const { useMongoDBAuthState } = require("@fallahibagaskara/baileys-auth-mongo");
import Boom from "@hapi/boom";

const url = "YOUR_MONGODB_URL"; // Replace with your MongoDB connection string // When Obtaining Mongodb URL Choose NodeJS Driver Version 2 or Later but don't 3 or it higher
const dbName = "whatsapp";
const collectionName = "authState";

async function connectToMongoDB() {
  const client = new MongoClient(url);
  await client.connect();
  const db = client.db(dbName);
  const collection = db.collection(collectionName);
  return { client, collection };
}

async function startWhatsApp() {
  const { collection } = await connectToMongoDB();
  const { state, saveCreds } = await useMongoDBAuthState(collection);

  const sock = makeWASocket({
    auth: state,
    printQRInTerminal: true,
  });

  sock.ev.on("creds.update", saveCreds);

  sock.ev.on("connection.update", (update) => {
    const { connection, lastDisconnect } = update;
    if (connection === "close") {
      const shouldReconnect =
        lastDisconnect && lastDisconnect.error
          ? Boom.boomify(lastDisconnect.error).output.statusCode
          : 500;
      console.log(
        "Connection closed due to",
        lastDisconnect?.error,
        ", reconnecting in",
        shouldReconnect,
        "ms"
      );
      if (shouldReconnect) {
        setTimeout(() => startWhatsApp(), shouldReconnect);
      }
    } else if (connection === "open") {
      console.log("Opened connection");
    }
  });

  sock.ev.on("messages.upsert", async (m) => {
    console.log(JSON.stringify(m, null, 2));

    const message = m.messages[0];
    if (message && !message.key.fromMe && m.type === "notify") {
      console.log("Replying to", message.key.remoteJid);
      await sock.sendMessage(message.key.remoteJid, { text: "Hello there!" });
    }
  });

  // Graceful shutdown
  process.on("SIGINT", async () => {
    console.log("Received SIGINT. Closing connection...");
    await sock.close();
    process.exit();
  });

  process.on("SIGTERM", async () => {
    console.log("Received SIGTERM. Closing connection...");
    await sock.close();
    process.exit();
  });
}

startWhatsApp().catch((err) => console.error("Unexpected error:", err));
```

### TypeScript

```typescript
import { MongoClient, Collection, Document } from "mongodb";
import { makeWASocket, AnyMessageContent } from "@whiskeysockets/baileys";
import { useMongoDBAuthState } from "@fallahibagaskara/baileys-auth-mongo";
import * as Boom from "@hapi/boom";
import { AuthenticationCreds } from "@whiskeysockets/baileys";

const url = "YOUR_MONGODB_URL"; // When Obtaining Mongodb URL Choose NodeJS Driver Version 2 or Later but don't 3 or it higher
const dbName = "whatsapp";
const collectionName = "authState";

interface AuthDocument extends Document {
  _id: string;
  creds?: AuthenticationCreds;
}

async function connectToMongoDB() {
  const client = new MongoClient(url);
  await client.connect();
  const db = client.db(dbName);
  const collection = db.collection<AuthDocument>(collectionName);
  return { client, collection };
}

async function startWhatsApp() {
  const { collection } = await connectToMongoDB();
  const { state, saveCreds } = await useMongoDBAuthState(collection);

  const sock = makeWASocket({
    auth: state,
    printQRInTerminal: true,
  });

  sock.ev.on("creds.update", saveCreds);

  sock.ev.on("connection.update", (update) => {
    const { connection, lastDisconnect } = update;
    if (connection === "close") {
      const shouldReconnect =
        lastDisconnect && lastDisconnect.error
          ? Boom.boomify(lastDisconnect.error).output.statusCode
          : 500;
      console.log(
        "Connection closed due to",
        lastDisconnect?.error,
        ", reconnecting in",
        shouldReconnect,
        "ms"
      );
      if (shouldReconnect) {
        setTimeout(() => startWhatsApp(), shouldReconnect);
      }
    } else if (connection === "open") {
      console.log("Opened connection");
    }
  });

  sock.ev.on("messages.upsert", async (m) => {
    console.log(JSON.stringify(m, undefined, 2));

    const message = m.messages[0];
    if (message && !message.key.fromMe && m.type === "notify") {
      console.log("Replying to", message.key.remoteJid);
      await sock.sendMessage(message.key.remoteJid!, {
        text: "Hello there!",
      } as AnyMessageContent);
    }
  });
}

startWhatsApp().catch((err) => console.log("Unexpected error:", err));
```

## 🤝 Contributing

Contributions are welcome! Feel free to open issues and submit pull requests to enhance `@fallahibagaskara/baileys-auth-mongo`.

# baileys-auth-mongo
