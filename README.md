<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Contador editable</title>
<style>
body{font-family:Arial, sans-serif;background:#f2f2f2;padding:15px;margin:0}
h1{text-align:center;color:#333}
.top{display:flex;gap:10px;margin-bottom:20px}
input,button,select{padding:12px;font-size:16px;border-radius:10px;box-sizing:border-box}
input,select{flex:1;border:1px solid #ccc;background:#fff}
button{border:none;cursor:pointer;font-weight:bold}
.add{background:#2563eb;color:#fff}
.card{background:#fff;padding:15px;border-radius:15px;margin-bottom:20px;box-shadow:0 2px 6px rgba(0,0,0,.1)}
.title{font-size:22px;font-weight:bold;margin-bottom:10px}
.total{font-size:48px;text-align:center;margin:10px 0;font-weight:bold;color:#1e293b}

/* Botonera de toques más pequeña y estilizada */
.controls{display:flex;gap:10px;justify-content:center;margin-bottom:15px}
.controls button{width:60px;height:45px;padding:0;color:#fff;font-size:24px;border-radius:8px;display:flex;align-items:center;justify-content:center}
.minus{background:#dc2626}
.plus{background:#16a34a}

/* Desplegable de Historial */
.history-toggle{width:100%;background:#e2e8f0;color:#334155;padding:10px;margin-top:10px;text-align:left;display:flex;justify-content:space-between;align-items:center;border-radius:8px}
.history-toggle::after{content:'▼';font-size:12px}
.history-toggle.open::after{content:'▲'}
.history-content{display:none;margin-top:10px;background:#fafafa;padding:12px;border-radius:10px;border:1px solid #e2e8f0}
.history-content.show{display:block}

/* Filtros de agrupación */
.group-filter{display:flex;align-items:center;gap:10px;margin-bottom:12px;font-size:14px;color:#64748b}
.group-filter select{padding:5px;font-size:14px;border-radius:6px}

/* Filas del historial */
.day{display:flex;justify-content:space-between;align-items:center;padding:8px 0;border-bottom:1px solid #eee;gap:10px}
.day:last-child{border-bottom:none}
.day input[type="date"], .day input[type="month"], .day input[type="number"]{padding:6px;font-size:14px;border-radius:6px;border:1px solid #cbd5e1}
.day input[type="date"], .day input[type="month"]{flex:2;min-width:110px}
.day input[type="number"]{width:60px;text-align:center}
.day-actions{display:flex;gap:5px;align-items:center}
.btn-del-log{background:#ef4444;color:white;padding:6px 10px;font-size:12px;border-radius:6px}

.delete{width:100%;background:#6b7280;color:#fff;margin-top:15px;padding:8px}
</style>
</head>
<body>

<h1>📦 Contador editable</h1>

<div class="top">
<input id="nombre" placeholder="Nombre del lote">
<button class="add" onclick="crear()">Añadir</button>
</div>

<div id="lista"></div>

<script>
let datos = JSON.parse(localStorage.getItem('contador_editable')) || [];
// Guardamos el estado de qué desplegables están abiertos y qué agrupación tienen (en memoria)
let estadoUI = {}; 

function guardar(){
localStorage.setItem('contador_editable', JSON.stringify(datos));
}

function fechaHoy(){
return new Date().toISOString().split('T')[0];
}

// Inicializa o mantiene el estado visual de cada lote
function inicializarEstadoLote(i){
if(!estadoUI[i]){
estadoUI[i] = { abierto: false, agrupacion: 'diario' };
}
}

function toggleHistorial(i){
estadoUI[i].abierto = !estadoUI[i].abierto;
render();
}

function cambiarAgrupacion(i, valor){
estadoUI[i].agrupacion = valor;
render();
}

function render(){
const lista = document.getElementById('lista');
lista.innerHTML = '';

datos.forEach((item, i)=>{
inicializarEstadoLote(i);
const vistaActual = estadoUI[i].agrupacion;

// Agrupación de datos según el filtro
let agrupado = {};
item.historial.forEach((h, idxOriginal)=>{
let clave = h.fecha; // 'YYYY-MM-DD'

if(vistaActual === 'mensual'){
clave = h.fecha.substring(0,7); // 'YYYY-MM'
} else if(vistaActual === 'anual'){
clave = h.fecha.substring(0,4); // 'YYYY'
}

if(!agrupado[clave]){
agrupado[clave] = { valor: 0, indices: [] };
}
agrupado[clave].valor += h.valor;
agrupado[clave].indices.push(idxOriginal); 
});

let historialHTML = '';
const clavesOrdenadas = Object.keys(agrupado).sort().reverse();

if(clavesOrdenadas.length === 0){
historialHTML = '<div style="color:#94a3b8; font-size:14px; text-align:center; padding:10px;">Sin registros aún</div>';
}

clavesOrdenadas.forEach(clave => {
let inputFechaHTML = '';

// Ajustamos el tipo de input según la agrupación para que sea cómodo editar
if(vistaActual === 'diario'){
inputFechaHTML = `<input type="date" value="${clave}" onchange="editarFecha(${i}, [${agrupado[clave].indices}], this.value, 'diario')">`;
} else if(vistaActual === 'mensual'){
inputFechaHTML = `<input type="month" value="${clave}" onchange="editarFecha(${i}, [${agrupado[clave].indices}], this.value, 'mensual')">`;
} else {
// Anual: usamos un input numérico simple para el año
inputFechaHTML = `<input type="number" min="2000" max="2100" value="${clave}" onchange="editarFecha(${i}, [${agrupado[clave].indices}], this.value, 'anual')">`;
}

historialHTML += `
<div class="day">
${inputFechaHTML}
<div class="day-actions">
<input type="number" value="${agrupado[clave].valor}" onchange="editarValorAgrupado(${i}, [${agrupado[clave].indices}], this.value)">
<button class="btn-del-log" onclick="eliminarRegistros(${i}, [${agrupado[clave].indices}])">✕</button>
</div>
</div>
`;
});

const card = document.createElement('div');
card.className = 'card';
card.innerHTML = `
<div class="title">${item.nombre}</div>
<div class="total">${item.total}</div>

<div class="controls">
<button class="minus" onclick="menos(${i})">−</button>
<button class="plus" onclick="mas(${i})">+</button>
</div>

<button class="history-toggle ${estadoUI[i].abierto ? 'open' : ''}" onclick="toggleHistorial(${i})">
Historial de toques
</button>

<div class="history-content ${estadoUI[i].abierto ? 'show' : ''}">
<div class="group-filter">
<span>Agrupar por:</span>
<select onchange="cambiarAgrupacion(${i}, this.value)">
<option value="diario" ${vistaActual === 'diario' ? 'selected' : ''}>Días</option>
<option value="mensual" ${vistaActual === 'mensual' ? 'selected' : ''}>Meses</option>
<option value="anual" ${vistaActual === 'anual' ? 'selected' : ''}>Años</option>
</select>
</div>
<div class="history-list">
${historialHTML}
</div>
</div>

<button class="delete" onclick="eliminar(${i})">Eliminar lote</button>
`;

lista.appendChild(card);
});
}

function crear(){
const texto = document.getElementById('nombre').value.trim();
if(texto === ''){
alert('Escribe un nombre');
return;
}
datos.push({
nombre: texto,
total: 0,
historial: []
});
document.getElementById('nombre').value = '';
guardar();
render();
}

function mas(i){
datos[i].total++;
datos[i].historial.push({
fecha: fechaHoy(),
valor: 1
});
guardar();
render();
}

function menos(i){
if(datos[i].total > 0){
datos[i].total--;
datos[i].historial.push({
fecha: fechaHoy(),
valor: -1
});
guardar();
render();
}
}

// Permite cambiar la fecha a todos los elementos contenidos en esa agrupación
function editarFecha(indexLote, indicesHistorial, nuevaFecha, tipo){
indicesHistorial.forEach(idx => {
if(tipo === 'diario'){
datos[indexLote].historial[idx].fecha = nuevaFecha;
} else if(tipo === 'mensual'){
// Si cambia el mes, conserva el día original (ej: cambia 2026-03-15 a 2026-05-15)
let diaOriginal = datos[indexLote].historial[idx].fecha.substring(7);
datos[indexLote].historial[idx].fecha = nuevaFecha + diaOriginal;
} else if(tipo === 'anual'){
// Si cambia el año, conserva el mes y día original
let mesDiaOriginal = datos[indexLote].historial[idx].fecha.substring(4);
datos[indexLote].historial[idx].fecha = nuevaFecha + mesDiaOriginal;
}
});
guardar();
render();
}

// Permite editar el número total de toques directamente desde el input del historial
function editarValorAgrupado(indexLote, indicesHistorial, nuevoValor){
nuevoValor = parseInt(nuevoValor) || 0;
let totalActualGrupo = 0;

indicesHistorial.forEach(idx => {
totalActualGrupo += datos[indexLote].historial[idx].valor;
});

let diferencia = nuevoValor - totalActualGrupo;

if(indicesHistorial.length > 0){
// Aplicamos la diferencia al último registro de ese grupo para ajustar el total del lote
let ultimoIdx = indicesHistorial[indicesHistorial.length - 1];
datos[indexLote].historial[ultimoIdx].valor += diferencia;
} else {
// Si por alguna razón no hubiera registros, creamos uno nuevo
datos[indexLote].historial.push({ fecha: fechaHoy(), valor: nuevoValor });
}

// Recalcular el total absoluto del lote basado en su historial
recargarTotalLote(indexLote);
guardar();
render();
}

// Elimina el grupo de toques seleccionado
function eliminarRegistros(indexLote, indicesHistorial){
if(confirm('¿Eliminar estos registros de toques?')){
// Eliminamos de atrás hacia adelante para no romper los índices correlativos
indicesHistorial.sort((a,b) => b - a).forEach(idx => {
datos[indexLote].historial.splice(idx, 1);
});
recargarTotalLote(indexLote);
guardar();
render();
}
}

function recargarTotalLote(indexLote){
let nuevoTotal = datos[indexLote].historial.reduce((acc, h) => acc + h.valor, 0);
datos[indexLote].total = nuevoTotal < 0 ? 0 : nuevoTotal; // Evita totales negativos si se desea
}

function eliminar(i){
if(confirm('¿Eliminar lote entero?')){
datos.splice(i,1);
if(estadoUI[i]) delete estadoUI[i];
guardar();
render();
}
}

render();
</script>

</body>
</html>
