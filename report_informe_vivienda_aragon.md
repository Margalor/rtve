// app.js - carga datos locales y crea visualizaciones
async function loadCSV(path){
  const res = await fetch(path);
  const txt = await res.text();
  return txt;
}

function parseCSVtoRows(csv){
  const lines = csv.trim().split(/\r?\n/);
  const headers = lines.shift().split(';').map(h=>h.replace(/"/g,'').trim());
  return lines.map(l=>{
    const cols = l.split(';').map(c=>c.replace(/"/g,'').trim());
    const obj = {};
    headers.forEach((h,i)=> obj[h]=cols[i]);
    return obj;
  });
}

async function init(){
  // Cargar datos
  const preciosCSV = await loadCSV('data/precios_trimestrales.csv');
  const preciosRows = parseCSVtoRows(preciosCSV);

  // Preparar series (ejemplo: extraer Trimestre 4 por año)
  const years = [];
  const aragon = [];
  const esp = [];
  preciosRows.forEach(r=>{
    if(r['Periodo'] && r['Periodo'].includes('Trimestre 4')){
      years.push(r['Año']);
      aragon.push(parseFloat(r['Aragón'].replace(',','.')));
      esp.push(parseFloat(r['España'].replace(',','.')));
    }
  });

  // Line chart (precios Trimestre 4 por año)
  const ctx = document.getElementById('lineChart').getContext('2d');
  window.lineChart = new Chart(ctx, {
    type:'line',
    data:{
      labels: years,
      datasets:[
        {label:'Aragón', data:aragon, borderColor:'#ff7a59', backgroundColor:'rgba(255,122,89,0.12)', tension:0.2},
        {label:'España', data:esp, borderColor:'#4cc9f0', backgroundColor:'rgba(76,201,240,0.08)', tension:0.2}
      ]
    },
    options:{responsive:true,plugins:{legend:{position:'bottom'}}}
  });

  // Bar chart: últimos 8 trimestres (ejemplo)
  const barCtx = document.getElementById('barChart').getContext('2d');
  window.barChart = new Chart(barCtx, {
    type:'bar',
    data:{
      labels: years.slice(-8),
      datasets:[
        {label:'Aragón', data:aragon.slice(-8), backgroundColor:'#ff7a59'},
        {label:'España', data:esp.slice(-8), backgroundColor:'#4cc9f0'}
      ]
    },
    options:{responsive:true,plugins:{legend:{position:'bottom'}}}
  });

  // Area chart: compraventa (ejemplo simplificado)
  const comprCSV = await loadCSV('data/compraventa_mensual.csv');
  const comprRows = parseCSVtoRows(comprCSV);
  const months = comprRows.slice(-24).map(r=>`${r['Año']}-${r['Periodo']}`);
  const ventas = comprRows.slice(-24).map(r=>parseInt(r['Viviendas']||r['Compraventa de viviendas']||0));
  const areaCtx = document.getElementById('areaChart').getContext('2d');
  window.areaChart = new Chart(areaCtx, {
    type:'line',
    data:{labels:months,datasets:[{label:'Compraventa (últimos 24)',data:ventas,fill:true,backgroundColor:'rgba(76,201,240,0.12)',borderColor:'#4cc9f0'}]},
    options:{responsive:true}
  });

  // Mapa Leaflet (capa base y ejemplo GeoJSON)
  const map = L.map('map').setView([41.65, -0.89], 7);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',{attribution:'© OpenStreetMap'}).addTo(map);

  // Ejemplo: capa GeoJSON mínima (sustituir por fichero real en data/geo.json)
  const sampleGeo = {
    "type":"FeatureCollection",
    "features":[
      {"type":"Feature","properties":{"name":"Zaragoza","precio":1200},"geometry":{"type":"Point","coordinates":[-0.8891,41.6488]}},
      {"type":"Feature","properties":{"name":"Huesca","precio":980},"geometry":{"type":"Point","coordinates":[-0.4089,42.1401]}},
      {"type":"Feature","properties":{"name":"Teruel","precio":860},"geometry":{"type":"Point","coordinates":[-1.1064,40.3449]}}
    ]
  };
  L.geoJSON(sampleGeo, {
    pointToLayer: (f,latlng)=> L.circleMarker(latlng,{radius:8,fillColor:'#ff7a59',color:'#fff',weight:1,fillOpacity:0.9}),
    onEachFeature: (f,layer)=> layer.bindPopup(`<strong>${f.properties.name}</strong><br>Precio medio: ${f.properties.precio} €/m²`)
  }).addTo(map);

  // Controles UI
  document.getElementById('yearRange').addEventListener('input', e=>{
    document.getElementById('yearLabel').textContent = e.target.value;
  });
  document.getElementById('resetBtn').addEventListener('click', ()=>{
    map.setView([41.65,-0.89],7);
    lineChart.reset();
    barChart.reset();
    areaChart.reset();
  });

  // Conclusiones dinámicas (ejemplo simple)
  document.getElementById('conclusions').innerHTML = `
    En los últimos años Aragón ha mostrado una recuperación sostenida del precio de la vivienda.
    El último dato trimestral disponible muestra un aumento interanual notable en 2024.
  `;
}

init().catch(err=>console.error(err));

