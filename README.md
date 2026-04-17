<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Travel Replenishment Portal</title>
  <script src="https://unpkg.com/exif-js"></script>
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>

  <!-- Routing -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet-routing-machine/dist/leaflet-routing-machine.css" />
  <script src="https://unpkg.com/leaflet-routing-machine/dist/leaflet-routing-machine.js"></script>

  <style>
    body { font-family: Arial; background:WHITE; color:BLACK; text-align:center; }
    .box { margin:20px auto; padding:20px; background:SKYBLUE; border-radius:12px; width:95%; max-width:900px; }
    input, select { padding:10px; margin:10px; width:80%; }
    button { padding:10px 20px; background:#22c55e; border:none; border-radius:8px; color:white; margin:5px; }
    #map { height:400px; border-radius:10px; margin-top:15px; }
    table { width:100%; border-collapse: collapse; margin-top:15px; }
    th, td { padding:8px; border:1px solid #334155; font-size:12px; }
    th { background:#0ea5e9; }
  </style>
</head>
<body>

<h1>Travel Replenishment Portal</h1>

<div class="box">
  <input type="text" id="fsName" placeholder="Field Supervisor Name"><br>
  <input type="file" id="photos" multiple accept="image/*"><br>

  <select id="vehicle">
    <option value="10.4">Motorcycle (10.4/km)</option>
    <option value="24.5">4 Wheels (24.5/km)</option>
  </select><br>

  <button onclick="processPhotos()">Compute</button>
  <button onclick="downloadJSON()">Download JSON</button>
</div>

<div class="box">
  <h3>Map Route</h3>
  <div id="map"></div>
</div>

<div class="box">
  <h3>Result</h3>
  <p id="fsDisplay"></p>
  <p id="distance">Total Distance: 0 km</p>
  <p id="amount">Total Amount: ₱0</p>
</div>

<div class="box">
  <h3>Photo Details</h3>
  <table>
    <thead>
      <tr>
        <th>#</th>
        <th>File</th>
        <th>Date</th>
        <th>Time</th>
        <th>Latitude</th>
        <th>Longitude</th>
      </tr>
    </thead>
    <tbody id="tableBody"></tbody>
  </table>
</div>

<script>
let map = L.map('map').setView([15.5, 120.9], 10);
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

let routingControl;
let globalData = {};

function processPhotos() {
  let files = document.getElementById('photos').files;
  let coords = [];
  let tableData = [];
  let loaded = 0;

  for (let i = 0; i < files.length; i++) {
    EXIF.getData(files[i], function() {
      let lat = EXIF.getTag(this, "GPSLatitude");
      let lon = EXIF.getTag(this, "GPSLongitude");
      let dateTime = EXIF.getTag(this, "DateTimeOriginal") || "N/A";

      if (lat && lon) {
        let latitude = lat[0] + lat[1]/60 + lat[2]/3600;
        let longitude = lon[0] + lon[1]/60 + lon[2]/3600;

        let date = dateTime.split(" ")[0] || "N/A";
        let time = dateTime.split(" ")[1] || "N/A";

        coords.push(L.latLng(latitude, longitude));

        tableData.push({
          index: i+1,
          name: this.name,
          date: date,
          time: time,
          latitude: latitude,
          longitude: longitude
        });
      }

      loaded++;
      if (loaded === files.length) {
        compute(coords, tableData);
      }
    });
  }
}

function compute(coords, tableData) {
  if (routingControl) map.removeControl(routingControl);

  // STRICT: only connect uploaded points (no extra waypoints)
  routingControl = L.Routing.control({
    waypoints: coords,
    addWaypoints: false,
    draggableWaypoints: false,
    routeWhileDragging: false,
    createMarker: function(i, wp) {
      return L.marker(wp.latLng).bindPopup("Point " + (i+1));
    }
  }).addTo(map);

  routingControl.on('routesfound', function(e) {
    let route = e.routes[0];
    let totalKm = route.summary.totalDistance / 1000;

    let rate = document.getElementById('vehicle').value;
    let amount = totalKm * rate;

    document.getElementById('distance').innerText = "Total Distance: " + totalKm.toFixed(2) + " km";
    document.getElementById('amount').innerText = "Total Amount: ₱" + amount.toFixed(2);

    globalData.total_km = totalKm;
    globalData.total_amount = amount;
  });

  let fsName = document.getElementById('fsName').value || "N/A";
  document.getElementById('fsDisplay').innerText = "Field Supervisor: " + fsName;

  globalData = {
    field_supervisor: fsName,
    vehicle_rate: document.getElementById('vehicle').value,
    photos: tableData
  };

  // Table
  let tbody = document.getElementById('tableBody');
  tbody.innerHTML = "";

  tableData.forEach(row => {
    let tr = `<tr>
      <td>${row.index}</td>
      <td>${row.name}</td>
      <td>${row.date}</td>
      <td>${row.time}</td>
      <td>${row.latitude.toFixed(6)}</td>
      <td>${row.longitude.toFixed(6)}</td>
    </tr>`;
    tbody.innerHTML += tr;
  });
}

function downloadJSON() {
  let dataStr = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(globalData, null, 2));
  let dlAnchor = document.createElement('a');
  dlAnchor.setAttribute("href", dataStr);
  dlAnchor.setAttribute("download", "travel_data.json");
  dlAnchor.click();
}
</script>

</body>
</html>
