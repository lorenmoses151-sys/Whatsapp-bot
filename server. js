/**
 * WhatsApp Bot using Twilio Sandbox
 * Run: node server.js
 * Requires: npm install express twilio dotenv
 */

require("dotenv").config();
const express = require("express");
const twilio = require("twilio");
const axios = require("axios");
const fs = require("fs");

const app = express();
app.use(express.urlencoded({ extended: false }));
app.use(express.json());

const {
  TWILIO_ACCOUNT_SID,
  TWILIO_AUTH_TOKEN,
  ANTHROPIC_API_KEY,
  HUMAN_AGENT_NUMBER, // e.g. whatsapp:+2348012345678
} = process.env;

const client = twilio(TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN);
const DB_FILE = "./db.json";

// ─── Database ───────────────────────────────────────────────────────────────
function loadDB() {
  if (!fs.existsSync(DB_FILE))
    fs.writeFileSync(DB_FILE, JSON.stringify({ orders: [], sessions: {} }));
  return JSON.parse(fs.readFileSync(DB_FILE));
}
function saveDB(db) {
  fs.writeFileSync(DB_FILE, JSON.stringify(db, null, 2));
}
function getSession(db, phone) {
  if (!db.sessions[phone])
    db.sessions[phone] = { state: "menu", order: null, history: [] };
  return db.sessions[phone];
}

// ─── Send WhatsApp message via Twilio ───────────────────────────────────────
async function sendMessage(to, body) {
  await client.messages.create({
    from: "whatsapp:+14155238886", // Twilio sandbox number
    to: to.startsWith("whatsapp:") ? to : `whatsapp:${to}`,
    body,
  });
}

// ─── Claude AI for FAQ ──────────────────────────────────────────────────────
async function askClaude(userMessage, history) {
  const res = await axios.post(
    "https://api.anthropic.com/v1/messages",
    {
      model: "claude-sonnet-4-20250514",
      max_tokens: 300,
      system: `You are a helpful WhatsApp customer support assistant.
Keep replies SHORT (under 60 words) and friendly — this is WhatsApp.
Help with: product questions, order status, delivery, returns, and FAQs.
If you don't know something specific, say so honestly.`,
      messages: [...history.slice(-6), { role: "user", content: userMessage }],
    },
    {
      headers: {
        "x-api-key": ANTHROPIC_API_KEY,
        "anthropic-version": "2023-06-01",
        "Content-Type": "application/json",
      },
    }
  );
  return res.data.content[0].text;
}

// ─── Main menu ───────────────────────────────────────────────────────────────
const MENU = `👋 Hello! Welcome to our store.

How can I help you today?
1️⃣ Place an order
2️⃣ Track my order
3️⃣ Ask a question
4️⃣ Talk to a human agent

Reply with a number.`;

// ─── Incoming webhook ────────────────────────────────────────────────────────
app.post("/webhook", async (req, res) => {
  res.sendStatus(200);

  try {
    const from = req.body.From; // e.g. whatsapp:+2348012345678
    const text = (req.body.Body || "").trim();

    const db = loadDB();
    const session = getSession(db, from);

    console.log(`📩 [${from}] "${text}" | state: ${session.state}`);

    // Escalated — forward to human agent
    if (session.state === "escalated") {
      if (HUMAN_AGENT_NUMBER) {
        await sendMessage(HUMAN_AGENT_NUMBER, `[Customer ${from}]: ${text}`);
      }
      saveDB(db);
      return;
    }

    // Ordering flow
    if (session.state === "ordering") {
      await handleOrdering(from, text, session, db);
      saveDB(db);
      return;
    }

    // Menu
    if (text === "1") {
      session.state = "ordering";
      session.order = { items: [], step: "product" };
      await sendMessage(from, "🛍️ What product would you like to order?\n\nType the product name or *menu* to go back.");
    } else if (text === "2") {
      const myOrders = db.orders.filter(o => o.phone === from);
      if (myOrders.length === 0) {
        await sendMessage(from, "📦 You have no orders yet.\n\nReply *1* to place an order.");
      } else {
        const latest = myOrders[myOrders.length - 1];
        await sendMessage(from, `📦 Latest order:\n\nID: ${latest.id}\nItems: ${latest.items.join(", ")}\nQty: ${latest.quantity}\nStatus: ${latest.status}\nDate: ${latest.date}`);
      }
    } else if (text === "3") {
      session.history.push({ role: "user", content: text });
      const reply = await askClaude(text, session.history.slice(0, -1));
      session.history.push({ role: "assistant", content: reply });
      await sendMessage(from, reply + "\n\n_(Reply *menu* to go back)_");
    } else if (text === "4") {
      session.state = "escalated";
      await sendMessage(from, "👤 Connecting you to a human agent. Please hold...\n\nReply *menu* to return to the bot.");
      if (HUMAN_AGENT_NUMBER) {
        await sendMessage(HUMAN_AGENT_NUMBER, `🔔 Customer ${from} needs human support.`);
      }
    } else if (text.toLowerCase() === "menu") {
      session.state = "menu";
      session.order = null;
      await sendMessage(from, MENU);
    } else {
      // Free text → AI FAQ
      session.history.push({ role: "user", content: text });
      const reply = await askClaude(text, session.history.slice(0, -1));
      session.history.push({ role: "assistant", content: reply });
      await sendMessage(from, reply + "\n\n_(Reply *menu* for options)_");
    }

    saveDB(db);
  } catch (err) {
    console.error("❌ Error:", err.response?.data || err.message);
  }
});

// ─── Order flow ──────────────────────────────────────────────────────────────
async function handleOrdering(from, text, session, db) {
  const order = session.order;

  if (text.toLowerCase() === "menu") {
    session.state = "menu";
    session.order = null;
    await sendMessage(from, MENU);
    return;
  }

  if (order.step === "product") {
    order.items.push(text);
    order.step = "quantity";
    await sendMessage(from, `Got it! How many units of *${text}* would you like?`);
  } else if (order.step === "quantity") {
    const qty = parseInt(text);
    if (isNaN(qty) || qty < 1) {
      await sendMessage(from, "Please enter a valid number e.g. *2*");
      return;
    }
    order.quantity = qty;
    order.step = "address";
    await sendMessage(from, "📍 Please send your delivery address:");
  } else if (order.step === "address") {
    order.address = text;
    order.step = "confirm";
    await sendMessage(from,
      `✅ Order summary:\n\nItem: ${order.items.join(", ")}\nQty: ${order.quantity}\nAddress: ${order.address}\n\nReply *yes* to confirm or *no* to cancel.`
    );
  } else if (order.step === "confirm") {
    if (text.toLowerCase() === "yes") {
      const newOrder = {
        id: "ORD-" + Date.now(),
        phone: from,
        items: order.items,
        quantity: order.quantity,
        address: order.address,
        status: "Confirmed",
        date: new Date().toLocaleString(),
      };
      db.orders.push(newOrder);
      session.state = "menu";
      session.order = null;
      await sendMessage(from, `🎉 Order placed!\n\nYour order ID is *${newOrder.id}*.\nWe'll notify you when it ships.\n\nReply *menu* for more options.`);
    } else {
      session.state = "menu";
      session.order = null;
      await sendMessage(from, "❌ Order cancelled.\n\n" + MENU);
    }
  }
}

// ─── Dashboard API ───────────────────────────────────────────────────────────
app.get("/dashboard", (req, res) => {
  const db = loadDB();
  res.json({ total_orders: db.orders.length, confirmed: db.orders.filter(o => o.status === "Confirmed").length, orders: db.orders });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`🚀 Bot running on port ${PORT}\nWebhook: http://localhost:${PORT}/webhook`));
