<!doctype html>
<html lang="es">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Calendario de Eventos Zurich</title>
<style>
:root{--blue:#0b4f8a;--light:#f4f7fb;--line:#dbe3ec;--text:#203040;--accent:#e6f0fa}
*{box-sizing:border-box}body{margin:0;font-family:Arial,Helvetica,sans-serif;background:var(--light);color:var(--text)}
.wrap{max-width:1100px;margin:30px auto;padding:0 16px}.header{background:#fff;border-radius:18px;padding:22px;box-shadow:0 8px 25px #0001;margin-bottom:18px}
h1{margin:0 0 6px;color:var(--blue);font-size:28px}.sub{color:#667788}.controls{display:flex;gap:9px;flex-wrap:wrap;margin-top:18px}
button,select{border:1px solid var(--line);background:#fff;border-radius:10px;padding:10px 13px;font-size:15px;cursor:pointer}
button.primary{background:var(--blue);color:#fff;border-color:var(--blue)}
.status{margin-top:12px;font-size:13px;color:#667788}.error{color:#b42318}
.calendar{background:#fff;border-radius:18px;overflow:hidden;box-shadow:0 8px 25px #0001}
.title{padding:18px 22px;font-size:21px;font-weight:700;text-transform:capitalize}.grid{display:grid;grid-template-columns:repeat(7,1fr)}
.dow{background:var(--blue);color:#fff;padding:12px 8px;text-align:center;font-weight:700}
.day{min-height:125px;border-top:1px solid var(--line);border-right:1px solid var(--line);padding:8px}
.muted{background:#fafbfd;color:#a2acb7}.num{font-weight:700;font-size:14px;margin-bottom:7px}
.event{background:var(--accent);border-left:4px solid var(--blue);border-radius:7px;padding:7px;margin-top:5px;font-size:12px;line-height:1.35}
.event strong{display:block;color:var(--blue);margin-bottom:2px}.legend{padding:14px 20px;color:#667788;font-size:13px;border-top:1px solid var(--line)}
@media(max-width:700px){.wrap{margin:12px auto}.day{min-height:95px;padding:5px}.event{font-size:10px}.dow{font-size:11px}h1{font-size:22px}}
</style>
</head>
<body>
<div class="wrap">
<section class="header">
<h1>Calendario de Eventos Zurich</h1>
<div class="sub">Los datos se cargan directamente desde Google Sheets.</div>
<div class="controls">
<button onclick="prevMonth()">‹ Mes anterior</button>
<button class="primary" onclick="today()">Hoy</button>
<button onclick="nextMonth()">Mes siguiente ›</button>
<select id="month" onchange="jump()"></select>
<select id="year" onchange="jump()"></select>
</div>
<div id="status" class="status">Cargando información…</div>
</section>
<section class="calendar">
<div id="title" class="title"></div>
<div class="grid" id="calendar"></div>
<div class="legend">Actualización: los datos se vuelven a consultar al recargar la página.</div>
</section>
</div>

<script>
const CSV_URL = "https://docs.google.com/spreadsheets/d/e/2PACX-1vRD40pbilCXxwy-3JrpWt9LjCy4Y7AvLO19tieZNqUfGf6V4WUc7bQK0hNCAhLaOg/pub?gid=280662515&single=true&output=csv";
const monthNames=["enero","febrero","marzo","abril","mayo","junio","julio","agosto","septiembre","octubre","noviembre","diciembre"];
const dow=["Lun","Mar","Mié","Jue","Vie","Sáb","Dom"];
let events=[], view=new Date(new Date().getFullYear(),new Date().getMonth(),1);

function escapeHtml(s){return String(s??"").replace(/[&<>"']/g,m=>({"&":"&amp;","<":"&lt;",">":"&gt;","\"":"&quot;","'":"&#39;"}[m]))}
function normalize(s){return String(s??"").normalize("NFD").replace(/[\u0300-\u036f]/g,"").trim().toLowerCase()}
function parseCSV(text){
  const rows=[];let row=[],field="",quoted=false;
  for(let i=0;i<text.length;i++){const c=text[i],n=text[i+1];
    if(c=='"'){if(quoted&&n=='"'){field+='"';i++}else quoted=!quoted}
    else if(c==','&&!quoted){row.push(field);field=""}
    else if((c=='\n'||c=='\r')&&!quoted){if(c=='\r'&&n=='\n')i++;row.push(field);field="";if(row.some(x=>x.trim()!=""))rows.push(row);row=[]}
    else field+=c;
  }
  if(field.length||row.length){row.push(field);rows.push(row)}
  return rows;
}
function dateFromValue(v){
  v=String(v??"").trim();
  if(!v)return null;
  let m=v.match(/^(\d{4})[-\/](\d{1,2})[-\/](\d{1,2})$/);
  if(m)return new Date(+m[1],+m[2]-1,+m[3]);
  m=v.match(/^(\d{1,2})[\/\-](\d{1,2})[\/\-](\d{4})$/);
  if(m)return new Date(+m[3],+m[2]-1,+m[1]);
  const d=new Date(v);return isNaN(d)?null:d;
}
function iso(d){return d.getFullYear()+"-"+String(d.getMonth()+1).padStart(2,"0")+"-"+String(d.getDate()).padStart(2,"0")}
function mondayIndex(d){return(d.getDay()+6)%7}

function render(){
  document.getElementById("title").textContent=monthNames[view.getMonth()]+" "+view.getFullYear();
  const cal=document.getElementById("calendar");cal.innerHTML="";
  dow.forEach(x=>{const e=document.createElement("div");e.className="dow";e.textContent=x;cal.appendChild(e)});
  const first=new Date(view.getFullYear(),view.getMonth(),1),start=new Date(first);
  start.setDate(1-mondayIndex(first));
  for(let i=0;i<42;i++){
    const d=new Date(start);d.setDate(start.getDate()+i);
    const cell=document.createElement("div");cell.className="day"+(d.getMonth()!=view.getMonth()?" muted":"");
    const n=document.createElement("div");n.className="num";n.textContent=d.getDate();cell.appendChild(n);
    events.filter(e=>e.fecha===iso(d)).forEach(e=>{
      const ev=document.createElement("div");ev.className="event";
      ev.innerHTML="<strong>"+escapeHtml(e.responsable||"Evento")+"</strong>"+
        (e.direccion?"📍 "+escapeHtml(e.direccion)+"<br>":"")+
        (e.estatus?"Estado: "+escapeHtml(e.estatus):"");
      cell.appendChild(ev);
    });
    cal.appendChild(cell);
  }
  fillSelectors();
}
function fillSelectors(){
  const m=document.getElementById("month"),y=document.getElementById("year");
  m.innerHTML=monthNames.map((x,i)=>`<option value="${i}">${x}</option>`).join("");
  const years=new Set();
  events.forEach(e=>{const d=e.fecha.slice(0,4);if(d)years.add(+d)});
  years.add(view.getFullYear());
  y.innerHTML=[...years].sort((a,b)=>a-b).map(v=>`<option value="${v}">${v}</option>`).join("");
  m.value=view.getMonth();y.value=view.getFullYear();
}
function jump(){view=new Date(+year.value,+month.value,1);render()}
function prevMonth(){view.setMonth(view.getMonth()-1);render()}
function nextMonth(){view.setMonth(view.getMonth()+1);render()}
function today(){const d=new Date();view=new Date(d.getFullYear(),d.getMonth(),1);render()}

async function loadData(){
  const status=document.getElementById("status");
  try{
    const response=await fetch(CSV_URL+"&_="+Date.now(),{cache:"no-store"});
    if(!response.ok)throw new Error("No se pudo leer Google Sheets.");
    const text=await response.text();
    const rows=parseCSV(text);
    if(rows.length<2)throw new Error("La hoja publicada no contiene filas suficientes.");

    // Busca encabezados aunque estén en español y con acentos.
    const header=rows[0].map(normalize);
    const findCol=(names)=>{
      for(const name of names){const i=header.indexOf(normalize(name));if(i>=0)return i}
      return -1;
    };
    const fechaCol=findCol(["fecha","date"]);
    const respCol=findCol(["responsable","responsable de evento","responsable del evento"]);
    const dirCol=findCol(["direccion","dirección","domicilio"]);
    const estCol=findCol(["estatus","estado","status"]);

    if(fechaCol<0)throw new Error("No encontré una columna llamada Fecha en la hoja publicada.");

    events=rows.slice(1).map(r=>{
      const d=dateFromValue(r[fechaCol]);
      return d?{
        fecha:iso(d),
        responsable:respCol>=0?r[respCol]:"",
        direccion:dirCol>=0?r[dirCol]:"",
        estatus:estCol>=0?r[estCol]:""
      }:null;
    }).filter(Boolean);

    status.textContent="✓ Datos actualizados desde Google Sheets · "+events.length+" evento(s)";
    status.classList.remove("error");
    render();
  }catch(err){
    status.textContent="No se pudo cargar Google Sheets: "+err.message;
    status.classList.add("error");
    render();
  }
}
render();
loadData();
</script>
</body>
</html>
