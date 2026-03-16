# ADS-B Live Flight Tracker — Self-Hosted Setup (Cloudflare Tunnel)                                  
                                                                                                       
  This version runs entirely on the Raspberry Pi. The Node.js server reads `dump1090-mutability`'s     
  output directly and serves the web map, exposed to the internet via a Cloudflare Tunnel. No external 
  server or paid hosting required.                                                                     
                                                                                                       
  ## Architecture 

  dump1090-mutability → aircraft.json (on Pi)
                             ↓
                    Node.js server (on Pi)
                             ↓
                    Cloudflare Tunnel (free)
                             ↓
                          Browser

  ## Prerequisites

  - Raspberry Pi running FlightRadar24 OS with SDR antenna attached
  - `dump1090-mutability` running (standard on FR24 OS)
  - A Cloudflare account with a domain whose DNS is managed by Cloudflare

  ---

  ## 1. Verify dump1090 is writing data

  ```bash
  ls /run/dump1090-mutability/
  cat /run/dump1090-mutability/aircraft.json

  You should see a JSON object with an aircraft array. Confirm dump1090-mutability is running:

  ps aux | grep dump1090

  ---
  2. Install Node.js

  curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
  sudo apt install -y nodejs
  node -v

  ---
  3. Set up the server

  sudo mkdir -p /opt/adsb-server
  cd /opt/adsb-server
  npm init -y
  npm install express ws

  Create server.js:

  nano /opt/adsb-server/server.js

  const express = require('express');
  const { WebSocketServer } = require('ws');
  const http = require('http');
  const fs = require('fs');
  const path = require('path');

  const AIRCRAFT_FILE = '/run/dump1090-mutability/aircraft.json';
  const PORT = 3000;

  const app = express();
  const server = http.createServer(app);
  const wss = new WebSocketServer({ server });

  app.use(express.static(path.join(__dirname, 'public')));

  let latestAircraft = [];

  setInterval(() => {
      try {
          const raw = fs.readFileSync(AIRCRAFT_FILE, 'utf8');
          const data = JSON.parse(raw);
          latestAircraft = (data.aircraft || []).filter(a => a.lat && a.lon);
          const payload = JSON.stringify({ aircraft: latestAircraft, ts: Date.now() });
          wss.clients.forEach(client => {
              if (client.readyState === 1) client.send(payload);
          });
      } catch (e) {}
  }, 3000);

  wss.on('connection', ws => {
      ws.send(JSON.stringify({ aircraft: latestAircraft, ts: Date.now() }));
  });

  setInterval(() => {
      wss.clients.forEach(client => {
          if (client.readyState === 1) client.ping();
      });
  }, 25000);

  server.listen(PORT, () => console.log(`Listening on ${PORT}`));

  Create a public/ directory and put your index.html inside it:

  mkdir -p /opt/adsb-server/public

  ---
  4. Run the server as a systemd service

  sudo nano /etc/systemd/system/adsb-server.service

  [Unit]
  Description=ADS-B Web Server
  After=network.target

  [Service]
  ExecStart=/usr/bin/node /opt/adsb-server/server.js
  Restart=always
  User=pi
  WorkingDirectory=/opt/adsb-server

  [Install]
  WantedBy=multi-user.target

  sudo systemctl enable adsb-server
  sudo systemctl start adsb-server
  sudo systemctl status adsb-server

  ---
  5. Install cloudflared

  curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64 -o
   cloudflared
  sudo mv cloudflared /usr/local/bin/
  sudo chmod +x /usr/local/bin/cloudflared

  ---
  6. Authenticate and create a tunnel

  Authenticate (run this, then open the printed URL in a browser on another device):

  cloudflared tunnel login

  Create the tunnel:

  cloudflared tunnel create adsb

  Note the tunnel ID printed — you\'ll need it in the next step.

  Add a DNS record pointing your subdomain to the tunnel:

  cloudflared tunnel route dns adsb flights.yourdomain.com

  ---
  7. Create the tunnel config file

  Use Python to write the config — avoids shell escaping issues that break the YAML:

  python3 -c "
  with open('/home/pi/.cloudflared/config.yml', 'w') as f:
      f.write('tunnel: YOUR-TUNNEL-ID\n')
      f.write('credentials-file: /home/pi/.cloudflared/YOUR-TUNNEL-ID.json\n')
      f.write('ingress:\n')
      f.write('  - hostname: flights.yourdomain.com\n')
      f.write('    service: http://localhost:3000\n')
      f.write('  - service: http_status:404\n')
  "

  Replace YOUR-TUNNEL-ID and flights.yourdomain.com with your actual values. Verify it looks right:

  cat ~/.cloudflared/config.yml

  Expected output:
  tunnel: YOUR-TUNNEL-ID
  credentials-file: /home/pi/.cloudflared/YOUR-TUNNEL-ID.json
  ingress:
    - hostname: flights.yourdomain.com
      service: http://localhost:3000
    - service: http_status:404

  ---
  8. Install cloudflared as a system service

  sudo cp ~/.cloudflared/config.yml /etc/cloudflared/config.yml
  sudo cloudflared service install
  sudo systemctl enable cloudflared
  sudo systemctl start cloudflared
  sudo systemctl status cloudflared

  ---
  9. Update index.html with your public URL

  In public/index.html, update the WebSocket connection and map center:

  const ws = new WebSocket(`wss://flights.yourdomain.com`);
  const map = L.map('map').setView([YOUR_LAT, YOUR_LON], 9);

  Then restart the server to pick up the change:

  sudo systemctl restart adsb-server

  ---
  Verify everything is running

  sudo systemctl status adsb-server
  sudo systemctl status cloudflared

  Both should show active (running). Visit your subdomain in a browser — you should see the map. Planes
   will appear as soon as any are in range of your antenna.

  ---
  Notes

  - The server polls aircraft.json every 3 seconds and pushes updates to all connected browsers via
  WebSocket
  - Only aircraft with a known position (lat/lon) are displayed — not all aircraft broadcast
  coordinates
  - The WebSocket keepalive ping runs every 25 seconds to prevent Cloudflare from closing idle
  connections
  - dump978-fr24 also runs on FR24 OS for UAT (978 MHz) GA aircraft — this project only uses 1090 MHz
  ADS-B data
  - If you ever rotate the Cloudflare tunnel, re-run steps 6–8