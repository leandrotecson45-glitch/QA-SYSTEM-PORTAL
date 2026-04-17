
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>QA Portal - Approval System</title>

  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>

  <link rel="stylesheet" href="https://unpkg.com/leaflet-routing-machine/dist/leaflet-routing-machine.css" />
  <script src="https://unpkg.com/leaflet-routing-machine/dist/leaflet-routing-machine.js"></script>

  <style>
    body { font-family: Arial; background:WHITE; color:BLACK; text-align:center; }
    .box { margin:20px auto; padding:20px; background:SKYBLUE; border-radius:12px; width:95%; max-width:1100px; }
    input { padding:10px; margin:10px; width:80%; }
    button { padding:8px 12px; margin:2px; border:none; border-radius:6px; cursor:pointer; }
    #map { height:450px; border-radius:10px; margin-top:15px; }
    table { width:100%; border-collapse: collapse; margin-top:15px; }
    th, td { padding:8px; border:1px solid #334155; font-size:12px; }
    th { background:#0ea5e9; }

    .approved { background:#22c55e; color:black; }
    .rejected { background:#ef4444; color:white; }
    .pending { background:#facc15; color:black; }
    .export { background:#38bdf8; color:black; font-weight:bold; }
  </style>
</head>
<body>

<h1>QA Portal - Approval System</h1>

<div class="box">
  <input type="file" id="jsonFile" accept=".json"><br>
  <button onclick="loadJSON()">Load Data</button>
  <button class="export" onclick="downloadReport()">Download QA Report</button>
</div>

<div class="box">
  <h3>Map View</h3>
  <div id="map"></div>
</div>

<div class="box">
  <h3>Summary</h3>
  <p id="fs"></p>
  <p id="distance"></p>
  <p id="amount"></p>
  <p id="statusSummary"></p>
</div>

<div class="box">
  <h3>Photo Details (QA Review)</h3>
  <table>
    <thead>
      <tr>
        <th>#</th>
        <th>File</th>
        <th>Date</th>
        <th>Time</th>
        <th>Latitude</th>
        <th>Longitude</th>
        <th>Status</th>
        <th>Action</th>
      </tr>
    </thead>
    <tbody id="tableBody"></tbody>
  </table>
</div>

<script>
let map = L.map('map').setView([15.5,120.9],10);

L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

let routingControl;
let markers=[];
let globalData=null;

function loadJSON(){
  let file=document.getElementById('jsonFile').files[0];
  if(!file) return alert('Upload JSON first');

  let reader=new FileReader();
  reader.onload=function(e){
    globalData=JSON.parse(e.target.result);

    globalData.photos.forEach(p=>{
      if(!p.status) p.status="PENDING";
    });

    render(globalData);
  };
  reader.readAsText(file);
}

function getActivePhotos(data){
  // REJECTED photos are removed from computation (no KM, no rate)
  return data.photos.filter(p => p.status !== "REJECTED");
}

function render(data){
  if(routingControl) map.removeControl(routingControl);
  markers.forEach(m=>map.removeLayer(m));
  markers=[];

  let coords=[];
  let activePhotos = getActivePhotos(data);

  data.photos.forEach((p,i)=>{
    let marker=L.marker([p.latitude,p.longitude]).addTo(map)
      .bindPopup(`<b>Point ${i+1}</b><br>${p.name}<br>${p.date} ${p.time}<br>Status: ${p.status}`);
    markers.push(marker);
  });

  activePhotos.forEach(p=>{
    coords.push(L.latLng(p.latitude,p.longitude));
  });

  if(coords.length >= 2){
    routingControl=L.Routing.control({
      waypoints: coords,
      addWaypoints:false,
      draggableWaypoints:false,
      routeWhileDragging:false,
      router:L.Routing.osrmv1({
        serviceUrl:'https://router.project-osrm.org/route/v1'
      })
    }).addTo(map);

    routingControl.on('routesfound',function(e){
      let route=e.routes[0];
      let totalKm=route.summary.totalDistance/1000;

      let rate=10.4;
      let amount=totalKm*rate;

      document.getElementById('distance').innerText="Total KM (Excluding Rejected): "+totalKm.toFixed(2);
      document.getElementById('amount').innerText="Amount: ₱"+amount.toFixed(2);
    });
  } else {
    document.getElementById('distance').innerText="Not enough valid points";
    document.getElementById('amount').innerText="₱0.00";
  }

  document.getElementById('fs').innerText="Field Supervisor: "+data.field_supervisor;

  let approved=data.photos.filter(p=>p.status=="APPROVED").length;
  let rejected=data.photos.filter(p=>p.status=="REJECTED").length;
  let pending=data.photos.filter(p=>p.status=="PENDING").length;

  document.getElementById('statusSummary').innerHTML=
    `Approved: ${approved} | Rejected: ${rejected} | Pending: ${pending}`;

  let tbody=document.getElementById('tableBody');
  tbody.innerHTML="";

  data.photos.forEach((p,i)=>{
    tbody.innerHTML+=`
      <tr>
        <td>${p.index}</td>
        <td>${p.name}</td>
        <td>${p.date}</td>
        <td>${p.time}</td>
        <td>${p.latitude.toFixed(6)}</td>
        <td>${p.longitude.toFixed(6)}</td>
        <td class="${p.status.toLowerCase()}">${p.status}</td>
        <td>
          <button class="approved" onclick="setStatus(${i},'APPROVED')">Approve</button>
          <button class="rejected" onclick="setStatus(${i},'REJECTED')">Reject</button>
        </td>
      </tr>
    `;
  });
}

function setStatus(index,status){
  globalData.photos[index].status=status;
  render(globalData);
}

function downloadReport(){
  if(!globalData) return alert("No data loaded");

  let report={
    field_supervisor: globalData.field_supervisor,
    summary:{
      approved:globalData.photos.filter(p=>p.status=="APPROVED").length,
      rejected:globalData.photos.filter(p=>p.status=="REJECTED").length,
      pending:globalData.photos.filter(p=>p.status=="PENDING").length
    },
    photos:globalData.photos
  };

  let blob=new Blob([JSON.stringify(report,null,2)],{type:"application/json"});
  let url=URL.createObjectURL(blob);

  let a=document.createElement("a");
  a.href=url;
  a.download="QA_REPORT.json";
  a.click();

  URL.revokeObjectURL(url);
}
</script>

</body>
</html>
