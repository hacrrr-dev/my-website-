<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1.0">
  <title>Location Tracker – Share Your Position</title>
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css"/>
  <style>
    * { box-sizing: border-box;}
    body {
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      min-height: 100vh;
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      margin: 0;
      padding: 0;
      display: flex;
      flex-direction: column;
      align-items: center;
    }
    .container {
      background: #fff;
      border-radius: 18px;
      box-shadow: 0 10px 32px rgba(0,0,0,.12);
      padding: 32px 18px 26px 18px;
      max-width: 430px;
      margin-top: 32px;
      width: 98%;
      text-align: center;
    }
    h1 { font-size: 2em; color: #2c2e35; margin-bottom: 10px;}
    .status {
      font-size: 1.08em;
      margin: 12px auto;
      border-radius: 6px;
      padding: 9px 10px;
      font-weight: bold;
      display: inline-block;
    }
    .status.ok { background: #e3f9e6; color: #20633b;}
    .status.error { background: #f8d7da; color: #9a232c;}
    .status.wait { background: #e7e7fa; color: #6959a7;}
    .coords {
      background: #f8f9fa;
      display: inline-block;
      padding: 10px;
      margin-top: 10px;
      margin-bottom: 10px;
      border-radius: 9px;
      font-family: 'Courier New', Courier, monospace;
      font-size: 1.05em;
      text-align: left;
    }
    button {
      background: linear-gradient(45deg, #667eea, #764ba2);
      color: white;
      border: none;
      padding: 13px 25px;
      border-radius: 22px;
      font-size: 1.04em;
      cursor: pointer;
      margin: 7px;
      letter-spacing: 0.03em;
      transition: background .15s, box-shadow .15s, transform .15s;
      box-shadow: 0 0.5px 10px rgba(0,0,0,0.11);
    }
    button:hover { filter: brightness(1.08); }
    button:disabled { opacity: 0.56; cursor: not-allowed;}
    .map-container {
      margin: 14px auto 0 auto;
      width: 100%;
      max-width: 390px;
      height: 250px;
      border-radius: 12px;
      overflow: hidden;
      border: 1px solid #d8d8ef;
      display: none;
    }
    #map { width: 100%; height: 100%; }
    .share-box {
      padding: 11px 10px;
      margin: 14px 0 6px 0;
      background: #e7f3ff;
      border-radius: 8px;
      word-break: break-all;
      text-align: left;
      font-size: 0.98em;
    }
    .footer {
      margin: 31px 0 10px 0;
      color: #888;
      font-size: 0.93em;
    }
    @media(max-width:560px){.container{padding:18px 5px 14px 5px;max-width:99vw;} .map-container{max-width:100vw;}}
  </style>
</head>
<body>
  <div class="container">
    <h1>📍 Location Tracker</h1>
    <div id="status" class="status wait">Press "Get Location" to begin.</div>
    <div id="coords" class="coords" style="display:none;">
      <div><b>Latitude:</b> <span id="lat">---</span></div>
      <div><b>Longitude:</b> <span id="lon">---</span></div>
      <div><b>Accuracy:</b> <span id="acc">--</span> m</div>
    </div>
    <button id="getBtn">Get Location</button>
    <button id="trackBtn" disabled>Start Tracking</button>
    <button id="stopBtn" disabled>Stop Tracking</button>

    <div id="shareBox" class="share-box" style="display:none;"></div>
    <div class="map-container" id="mapCtn">
      <div id="map"></div>
    </div>
    <div class="footer">
      Powered by browser geolocation &amp; <a href="https://www.openstreetmap.org/" target="_blank">OSM</a> | For privacy: location only sent if you share link
    </div>
  </div>
  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
  <script>
    let watchId = null, map = null, marker = null;

    const status = document.getElementById('status');
    const coords = document.getElementById('coords');
    const getBtn = document.getElementById('getBtn');
    const latSpan = document.getElementById('lat');
    const lonSpan = document.getElementById('lon');
    const accSpan = document.getElementById('acc');
    const trackBtn = document.getElementById('trackBtn');
    const stopBtn = document.getElementById('stopBtn');
    const shareBox = document.getElementById('shareBox');
    const mapCtn = document.getElementById('mapCtn');

    function updateStatus(txt, type = 'wait') {
      status.textContent = txt;
      status.className = 'status ' + type;
    }
    function showCoords(lat, lon, acc) {
      latSpan.textContent = Number(lat).toFixed(6);
      lonSpan.textContent = Number(lon).toFixed(6);
      accSpan.textContent = acc ? Math.round(acc) : '--';
      coords.style.display = 'block';
    }
    function shareLink(lat, lon) {
      const href = window.location.origin + window.location.pathname + `?lat=${lat}&lon=${lon}`;
      shareBox.style.display = 'block';
      shareBox.innerHTML = `<b>Share this location:</b><br><a href="${href}">${href}</a> 
        <button onclick="navigator.clipboard.writeText('${href}')">Copy</button>`;
    }
    function showMap(lat, lon) {
      mapCtn.style.display = 'block';
      if (!map) {
        map = L.map('map').setView([lat, lon], 16);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
          attribution: '&copy; OSM contributors'
        }).addTo(map);
        marker = L.marker([lat, lon]).addTo(map);
      } else {
        map.setView([lat, lon], 16);
        marker.setLatLng([lat, lon]);
      }
    }
    function showError(msg) {
      updateStatus(msg, 'error');
    }
    getBtn.onclick = function() {
      if (!navigator.geolocation) return showError('Geolocation not supported.');
      updateStatus('Getting location...', 'wait');
      navigator.geolocation.getCurrentPosition(
        function(pos){
          const {latitude, longitude, accuracy} = pos.coords;
          showCoords(latitude, longitude, accuracy);
          updateStatus('Location found!', 'ok');
          shareLink(latitude, longitude);
          showMap(latitude, longitude);
          trackBtn.disabled = false;
        },
        function(err){
          if(err.code === 1) showError('Permission denied.');
          else if(err.code===2) showError('Location unavailable.');
          else if(err.code===3) showError('Timed out.');
          else showError('Unknown error.');
        },
        {enableHighAccuracy:true,timeout:10000}
      );
    };
    trackBtn.onclick = function() {
      if (!navigator.geolocation) return showError('Geolocation not supported.');
      updateStatus('Tracking your movement…', 'ok');
      trackBtn.disabled = true;
      stopBtn.disabled = false;
      watchId = navigator.geolocation.watchPosition(
        function(pos){
          const {latitude, longitude, accuracy} = pos.coords;
          showCoords(latitude, longitude, accuracy);
          updateStatus('Tracking! Moving? Map will update.', 'ok');
          shareLink(latitude, longitude);
          showMap(latitude, longitude);
        },
        function(err){
          showError('Error: ' + (err.message || 'problem getting tracking.'));
        },
        {enableHighAccuracy:true,timeout:6500}
      );
    };
    stopBtn.onclick = function() {
      if (watchId !== null) {
        navigator.geolocation.clearWatch(watchId);
        watchId = null;
        updateStatus('Tracking stopped. Try Start to track again.','wait');
        trackBtn.disabled = false;
        stopBtn.disabled = true;
      }
    };

    // If arrived via shared link
    window.addEventListener('DOMContentLoaded', ()=>{
      const q = new URLSearchParams(location.search);
      const lat = q.get('lat'), lon = q.get('lon');
      if (lat && lon) {
        showCoords(lat, lon, null);
        updateStatus('Location loaded from link.', 'ok');
        shareLink(lat, lon);
        showMap(lat, lon);
      }
    });
  </script>
</body>
</html># my-website-
