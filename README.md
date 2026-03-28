const { default: makeWASocket, useMultiFileAuthState, DisconnectReason } = require('@whiskeysockets/baileys')
const pino = require('pino')
const axios = require('axios')

// ===== CONFIG =====
const OWNER = "628xxxx@s.whatsapp.net" // ganti nomor lu
let antiGroup = false
let users = {}

const OPENAI_API_KEY = "ISI_API_KEY_KAMU"

// ===== USER =====
function getUser(sender) {
  if (!users[sender]) {
    users[sender] = {
      limit: 10,
      premium: false
    }
  }
  return users[sender]
}

// ===== AI =====
async function aiResponse(text) {
  try {
    const res = await axios.post(
      "https://api.openai.com/v1/chat/completions",
      {
        model: "gpt-4o-mini",
        messages: [{ role: "user", content: text }]
      },
      {
        headers: {
          Authorization: `Bearer ${OPENAI_API_KEY}`,
          "Content-Type": "application/json"
        }
      }
    )
    return res.data.choices[0].message.content
  } catch (e) {
    console.log(e.response?.data || e.message)
    return "AI error"
  }
}

// ===== TRANSLATE =====
async function translate(text) {
  try {
    const res = await axios.get(
      `https://api.mymemory.translated.net/get?q=${encodeURIComponent(text)}&langpair=auto|id`
    )
    return res.data.responseData.translatedText
  } catch {
    return text
  }
}

// ===== MENU =====
const menu = `
╭━━━〔 BOT PUBLIC PREMIUM 〕━━━⬣
┃ .menu
┃ .limit
┃ .buy premium
┃ .antigroup on/off
┃ chat bebas = AI
╰━━━━━━━━━━━━━━━━━━⬣
`

async function startBot() {
  const { state, saveCreds } = await useMultiFileAuthState('session')

  const sock = makeWASocket({
    logger: pino({ level: 'silent' }),
    auth: state,
    printQRInTerminal: true
  })

  sock.ev.on('creds.update', saveCreds)

  sock.ev.on('connection.update', (update) => {
    const { connection, lastDisconnect } = update
    if (connection === 'close') {
      const shouldReconnect =
        lastDisconnect?.error?.output?.statusCode !== DisconnectReason.loggedOut
      if (shouldReconnect) startBot()
    }
  })

  sock.ev.on('messages.upsert', async ({ messages }) => {
    try {
      const msg = messages[0]
      if (!msg.message) return

      const from = msg.key.remoteJid
      const sender = msg.key.participant || from
      const isGroup = from.endsWith('@g.us')

      const body =
        msg.message.conversation ||
        msg.message.extendedTextMessage?.text ||
        ''

      const user = getUser(sender)

      // anti group
      if (antiGroup && isGroup) return

      // menu
      if (body === '.menu') {
        return sock.sendMessage(from, { text: menu })
      }

      // limit
      if (body === '.limit') {
        return sock.sendMessage(from, { text: `Limit: ${user.limit}` })
      }

      // buy
      if (body === '.buy premium') {
        return sock.sendMessage(from, { text: 'Chat owner buat premium' })
      }

      // add premium (owner only)
      if (body.startsWith('.addprem') && sender === OWNER) {
        const number = body.split(' ')[1]
        const target = number + '@s.whatsapp.net'
        getUser(target).premium = true
        return sock.sendMessage(from, { text: 'User jadi premium' })
      }

      // anti group control (owner)
      if (body === '.antigroup on' && sender === OWNER) {
        antiGroup = true
        return sock.sendMessage(from, { text: 'Anti group ON' })
      }

      if (body === '.antigroup off' && sender === OWNER) {
        antiGroup = false
        return sock.sendMessage(from, { text: 'Anti group OFF' })
      }

      // AI CHAT
      if (!body.startsWith('.')) {
        if (!user.premium) {
          if (user.limit <= 0) {
            return sock.sendMessage(from, { text: 'Limit habis, beli premium' })
          }
          user.limit -= 1
        }

        const translated = await translate(body)
        const ai = await aiResponse(translated)

        return sock.sendMessage(from, { text: ai })
      }

    } catch (err) {
      console.log(err)
    }
  })

  console.log('🔥 BOT AKTIF')
}

startBot()
