const WebSocket = require('ws');
const http = require('http');
const fs = require('fs');
const path = require('path');


const server = http.createServer((req, res) => {
  if (req.url === '/') {
    const filePath = path.join(__dirname, 'public', 'index.html');
    fs.readFile(filePath, (err, content) => {
      if (err) {
        res.writeHead(500);
        res.end('Ошибка сервера');
      } else {
        res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
        res.end(content);
      }
    });
  } else {
    res.writeHead(404);
    res.end('Not found');
  }
});

const wss = new WebSocket.Server({ server });


const clients = new Map();

wss.on('connection', (ws) => {
 
  clients.set(ws, { name: null, color: '#000000' });

  ws.on('message', (data) => {
    try {
      const msg = JSON.parse(data);
      const client = clients.get(ws);

      if (!client.name && msg.type === 'join') {
       
        const name = msg.name.trim();
        const color = msg.color || '#000000';

        if (!name) return;

        const names = Array.from(clients.values()).map(c => c.name).filter(Boolean);
        if (names.includes(name)) {
          ws.send(JSON.stringify({ type: 'error', text: 'Имя уже занято!' }));
          return;
        }

        client.name = name;
        client.color = color;

        
        if (names.length === 0) {
          ws.send(JSON.stringify({ type: 'system', text: 'Добро пожаловать. Вы первый в чате.' }));
        } else {
          const others = names.join(', ');
          ws.send(JSON.stringify({ type: 'system', text: `Добро пожаловать. В чате уже присутствуют: ${others}.` }));
        }

       
        broadcast({
          type: 'system',
          text: `${name} присоединился к чату.`
        }, ws);

        
        updateUsersList();
        return;
      }

      if (client.name) {
        if (msg.type === 'message') {
          // Часть III: личные сообщения
          if (msg.to) {
            // Отправка конкретному пользователю
            let targetWs = null;
            for (const [w, c] of clients.entries()) {
              if (c.name === msg.to) {
                targetWs = w;
                break;
              }
            }
            if (targetWs) {
              targetWs.send(JSON.stringify({
                type: 'private',
                from: client.name,
                text: msg.text,
                color: client.color
              }));
              
              ws.send(JSON.stringify({
                type: 'private',
                from: client.name,
                to: msg.to,
                text: msg.text,
                color: client.color
              }));
            } else {
              ws.send(JSON.stringify({ type: 'error', text: 'Пользователь не найден.' }));
            }
          } else {
           
            broadcast({
              type: 'message',
              name: client.name,
              text: msg.text,
              color: client.color
            });
          }
        }
      }
    } catch (e) {
      console.error('Ошибка обработки сообщения:', e);
    }
  });

  ws.on('close', () => {
    const client = clients.get(ws);
    if (client && client.name) {
      broadcast({ type: 'system', text: `${client.name} нас покинул.` });
      updateUsersList();
    }
    clients.delete(ws);
  });

  ws.on('error', (err) => {
    console.error('WebSocket error:', err);
  });
});

function broadcast(data, excludeWs = null) {
  const payload = JSON.stringify(data);
  wss.clients.forEach((client) => {
    if (client.readyState === WebSocket.OPEN && client !== excludeWs) {
      client.send(payload);
    }
  });
}

function updateUsersList() {
  const users = Array.from(clients.values())
    .filter(c => c.name)
    .map(c => ({ name: c.name, color: c.color }));
  broadcast({ type: 'users', list: users });
}

const PORT = process.env.PORT || 8080;
server.listen(PORT, () => {
  console.log(`Сервер запущен на http://localhost:${PORT}`);
});
