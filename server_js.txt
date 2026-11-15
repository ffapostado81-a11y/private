// ============================================================================
//  LUNAR REMOTE SERVER
//  WebSocket relay: painel web ←→ celular com hack
//  Deploy: Oracle Cloud, Render, qualquer VPS com Node.js
// ============================================================================

const http = require('http');
const fs   = require('fs');
const path = require('path');
const crypto = require('crypto');
const { WebSocketServer, WebSocket } = require('ws');

const PORT = process.env.PORT || 3000;

// ── Armazenamento de chaves e dispositivos ──────────────────────────────────
// Em produção, trocar por um banco (SQLite, Postgres etc.)
// Formato: keys[key] = { created, label, lastSeen, deviceWs, panelWs[], ip, config }
const keys = new Map();

// Gera chave de acesso única
function generateKey() {
  return crypto.randomBytes(12).toString('hex').toUpperCase()
    .match(/.{4}/g).join('-'); // XXXX-XXXX-XXXX-XXXX-XXXX-XXXX
}

// Config padrão (espelha lunar::Config)
function defaultConfig() {
  return {
    aim_mode: 0, aim_fov: 1, aim_smooth: 1, aim_bone: 0,
    chest_first_n: 1, head_switch_delay: 0,
    silent_aim: false, silent_fov: 1,
    esp: false, esp_box: false, esp_hp: false, esp_dist: false,
    esp_snap: false, esp_max_dist: 50,
    norecoil: false, nospread: false, rapidfire: false,
    recoil_mode: 0, recoil_scale: 0, spread_scale: 0, fire_delay: 0,
    bypass: false, speedhack: false, speed_scale: 0.5
  };
}

// ── Servir arquivos estáticos ────────────────────────────────────────────────
const MIME = {
  '.html': 'text/html', '.css': 'text/css', '.js': 'application/javascript',
  '.json': 'application/json', '.png': 'image/png', '.svg': 'image/svg+xml',
  '.ico': 'image/x-icon'
};

function serveStatic(req, res) {
  // API: gerar chave nova
  if (req.method === 'POST' && req.url === '/api/key') {
    const key = generateKey();
    keys.set(key, {
      created: Date.now(), label: '', lastSeen: 0,
      deviceWs: null, panelWs: new Set(), ip: null,
      config: defaultConfig()
    });
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ key }));
    console.log(`[KEY] Nova chave criada: ${key}`);
    return;
  }

  // API: listar chaves (admin simples)
  if (req.method === 'GET' && req.url === '/api/keys') {
    const list = [];
    for (const [k, v] of keys) {
      list.push({
        key: k, label: v.label, ip: v.ip,
        online: v.deviceWs?.readyState === WebSocket.OPEN,
        lastSeen: v.lastSeen, panels: v.panelWs.size
      });
    }
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(list));
    return;
  }

  // Servir arquivos estáticos
  let filePath = req.url === '/' ? '/index.html' : req.url;
  filePath = path.join(__dirname, 'public', filePath);

  const ext = path.extname(filePath);
  const mime = MIME[ext] || 'application/octet-stream';

  fs.readFile(filePath, (err, data) => {
    if (err) {
      res.writeHead(404);
      res.end('404 Not Found');
      return;
    }
    res.writeHead(200, { 'Content-Type': mime });
    res.end(data);
  });
}

// ── HTTP Server ─────────────────────────────────────────────────────────────
const server = http.createServer(serveStatic);

// ── WebSocket Server ────────────────────────────────────────────────────────
const wss = new WebSocketServer({ server });

wss.on('connection', (ws, req) => {
  const ip = req.headers['x-forwarded-for']?.split(',')[0]?.trim()
          || req.socket.remoteAddress;

  let role = null;    // 'device' ou 'panel'
  let userKey = null; // chave associada
  let entry = null;   // referência no Map

  console.log(`[WS] Nova conexão de ${ip}`);

  ws.on('message', (raw) => {
    let msg;
    try { msg = JSON.parse(raw); } catch { return; }

    // ── Autenticação ────────────────────────────────────
    if (msg.type === 'auth') {
      const key = msg.key;

      // Auto-criar chave se não existe (para facilitar teste)
      if (!keys.has(key)) {
        keys.set(key, {
          created: Date.now(), label: msg.label || '', lastSeen: 0,
          deviceWs: null, panelWs: new Set(), ip: null,
          config: defaultConfig()
        });
        console.log(`[KEY] Chave auto-criada: ${key}`);
      }

      entry = keys.get(key);
      userKey = key;

      if (msg.role === 'device') {
        role = 'device';
        entry.deviceWs = ws;
        entry.ip = ip;
        entry.lastSeen = Date.now();

        // Notifica painéis que o dispositivo conectou
        for (const pw of entry.panelWs) {
          safeSend(pw, { type: 'device_status', online: true, ip });
        }

        // Manda config atual pro dispositivo
        safeSend(ws, { type: 'config', config: entry.config });
        console.log(`[AUTH] Device conectado: ${key} (${ip})`);

      } else if (msg.role === 'panel') {
        role = 'panel';
        entry.panelWs.add(ws);

        // Manda estado atual pro painel
        safeSend(ws, {
          type: 'init',
          config: entry.config,
          online: entry.deviceWs?.readyState === WebSocket.OPEN,
          ip: entry.ip
        });
        console.log(`[AUTH] Painel conectado: ${key} (${ip})`);
      }
      return;
    }

    if (!entry || !role) {
      safeSend(ws, { type: 'error', msg: 'Não autenticado. Envie {type:"auth", key:"...", role:"device"|"panel"}' });
      return;
    }

    // ── Painel → Dispositivo: mudança de config ─────────
    if (role === 'panel' && msg.type === 'config_update') {
      // Atualiza config no servidor
      Object.assign(entry.config, msg.changes);
      entry.lastSeen = Date.now();

      // Repassa pro dispositivo
      if (entry.deviceWs?.readyState === WebSocket.OPEN) {
        safeSend(entry.deviceWs, { type: 'config_update', changes: msg.changes });
      }

      // Repassa pros outros painéis (sync multi-painel)
      for (const pw of entry.panelWs) {
        if (pw !== ws) safeSend(pw, { type: 'config_sync', changes: msg.changes });
      }
      return;
    }

    // ── Painel → Dispositivo: desativar tudo ────────────
    if (role === 'panel' && msg.type === 'reset_all') {
      entry.config = defaultConfig();
      if (entry.deviceWs?.readyState === WebSocket.OPEN) {
        safeSend(entry.deviceWs, { type: 'config', config: entry.config });
      }
      for (const pw of entry.panelWs) {
        safeSend(pw, { type: 'config_sync', changes: entry.config });
      }
      return;
    }

    // ── Dispositivo → heartbeat ─────────────────────────
    if (role === 'device' && msg.type === 'heartbeat') {
      entry.lastSeen = Date.now();
      entry.ip = ip;
      return;
    }

    // ── Dispositivo → sync IP ───────────────────────────
    if (role === 'device' && msg.type === 'sync_ip') {
      entry.ip = ip;
      entry.lastSeen = Date.now();
      for (const pw of entry.panelWs) {
        safeSend(pw, { type: 'ip_update', ip });
      }
      console.log(`[SYNC] IP atualizado: ${key} → ${ip}`);
      return;
    }
  });

  ws.on('close', () => {
    if (!entry) return;

    if (role === 'device') {
      entry.deviceWs = null;
      for (const pw of entry.panelWs) {
        safeSend(pw, { type: 'device_status', online: false });
      }
      console.log(`[DC] Device desconectou: ${userKey}`);
    } else if (role === 'panel') {
      entry.panelWs.delete(ws);
      console.log(`[DC] Painel desconectou: ${userKey}`);
    }
  });

  ws.on('error', () => {});
});

function safeSend(ws, obj) {
  try {
    if (ws.readyState === WebSocket.OPEN)
      ws.send(JSON.stringify(obj));
  } catch {}
}

// ── Heartbeat check (limpa conexões mortas) ─────────────────────────────────
setInterval(() => {
  for (const [, entry] of keys) {
    if (entry.deviceWs?.readyState !== WebSocket.OPEN) {
      entry.deviceWs = null;
    }
    const dead = [];
    for (const pw of entry.panelWs) {
      if (pw.readyState !== WebSocket.OPEN) dead.push(pw);
    }
    dead.forEach(pw => entry.panelWs.delete(pw));
  }
}, 30000);

// ── Start ───────────────────────────────────────────────────────────────────
server.listen(PORT, () => {
  console.log(`\n=== LUNAR REMOTE SERVER ===`);
  console.log(`Porta: ${PORT}`);
  console.log(`Painel: http://localhost:${PORT}`);
  console.log(`WebSocket: ws://localhost:${PORT}\n`);
});
