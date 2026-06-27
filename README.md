# CODIGO-FUENTE
programacion web
JSON
{
  "name": "crud-optimizacion-consultas",
  "version": "1.0.0",
  "description": "Aplicacion web CRUD (gestion de inventario de productos) con Node.js, http nativo y SQLite (node:sqlite), enfocada en la optimizacion de consultas a la base de datos. Producto Academico 4 - Unidad 3.",
  "main": "src/server.js",
  "type": "commonjs",
  "scripts": {
    "start": "node src/server.js",
    "seed": "node scripts/seed.js",
    "benchmark": "node scripts/benchmark.js"
  },
  "engines": {
    "node": ">=22.5.0"
  },
  "license": "MIT"
}
 JAVA SACRIPT
 const { DatabaseSync } = require('node:sqlite');
const path = require('node:path');

const DB_PATH = path.join(__dirname, '..', 'data', 'app.db');

const db = new DatabaseSync(DB_PATH);
db.exec(`
  PRAGMA journal_mode = WAL;        -- escrituras concurrentes más rápidas
  PRAGMA foreign_keys = ON;

  CREATE TABLE IF NOT EXISTS categorias (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    nombre        TEXT NOT NULL UNIQUE
  );

  CREATE TABLE IF NOT EXISTS productos (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    sku           TEXT NOT NULL UNIQUE,
    nombre        TEXT NOT NULL,
    descripcion   TEXT,
    precio        REAL NOT NULL,
    stock         INTEGER NOT NULL DEFAULT 0,
    categoria_id  INTEGER NOT NULL,
    creado_en     TEXT NOT NULL DEFAULT (datetime('now')),
    actualizado_en TEXT NOT NULL DEFAULT (datetime('now')),
    FOREIGN KEY (categoria_id) REFERENCES categorias(id)
  );

  CREATE TABLE IF NOT EXISTS movimientos (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    producto_id   INTEGER NOT NULL,
    tipo          TEXT NOT NULL CHECK (tipo IN ('ENTRADA','SALIDA')),
    cantidad      INTEGER NOT NULL,
    creado_en     TEXT NOT NULL DEFAULT (datetime('now')),
    FOREIGN KEY (producto_id) REFERENCES productos(id)
  );
`);

function crearIndicesOptimizacion() {
  db.exec(`
    CREATE INDEX IF NOT EXISTS idx_productos_categoria ON productos(categoria_id);
    CREATE INDEX IF NOT EXISTS idx_productos_nombre ON productos(nombre);
    CREATE INDEX IF NOT EXISTS idx_movimientos_producto_fecha
      ON movimientos(producto_id, creado_en);
    CREATE INDEX IF NOT EXISTS idx_productos_stock ON productos(stock);
  `);
}

function eliminarIndicesOptimizacion() {
  db.exec(`
    DROP INDEX IF EXISTS idx_productos_categoria;
    DROP INDEX IF EXISTS idx_productos_nombre;
    DROP INDEX IF EXISTS idx_movimientos_producto_fecha;
    DROP INDEX IF EXISTS idx_productos_stock;
  `);
}

crearIndicesOptimizacion();
const Productos = {
  crear({ sku, nombre, descripcion, precio, stock, categoria_id }) {
    const stmt = db.prepare(`
      INSERT INTO productos (sku, nombre, descripcion, precio, stock, categoria_id)
      VALUES (?, ?, ?, ?, ?, ?)
    `);
    const info = stmt.run(sku, nombre, descripcion ?? '', precio, stock ?? 0, categoria_id);
    return this.obtenerPorId(Number(info.lastInsertRowid));
  },

  obtenerPorId(id) {
    return db.prepare(`SELECT * FROM productos WHERE id = ?`).get(id);
  },

  actualizar(id, campos) {
    const permitidos = ['sku', 'nombre', 'descripcion', 'precio', 'stock', 'categoria_id'];
    const claves = Object.keys(campos).filter((k) => permitidos.includes(k));
    if (claves.length === 0) return this.obtenerPorId(id);
    const set = claves.map((k) => `${k} = ?`).join(', ');
    const valores = claves.map((k) => campos[k]);
    db.prepare(`UPDATE productos SET ${set}, actualizado_en = datetime('now') WHERE id = ?`)
      .run(...valores, id);
    return this.obtenerPorId(id);
  },

  eliminar(id) {
    return db.prepare(`DELETE FROM productos WHERE id = ?`).run(id);
  },

listar({ busqueda = '', categoriaId = null, pagina = 1, tamanioPagina = 10 } = {}) {
    const offset = (Math.max(1, pagina) - 1) * tamanioPagina;
    const condiciones = [];
    const params = [];

    if (busqueda) {
      condiciones.push(`nombre LIKE ?`);
      params.push(`${busqueda}%`);
    }
    if (categoriaId) {
      condiciones.push(`categoria_id = ?`);
      params.push(categoriaId);
    }

    const where = condiciones.length ? `WHERE ${condiciones.join(' AND ')}` : '';

    const filas = db.prepare(
      `SELECT id, sku, nombre, precio, stock, categoria_id
       FROM productos ${where}
       ORDER BY nombre LIMIT ? OFFSET ?`
    ).all(...params, tamanioPagina, offset);

    const total = db.prepare(
      `SELECT COUNT(*) AS total FROM productos ${where}`
    ).get(...params).total;

    return { datos: filas, total, pagina, tamanioPagina };
  },

  obtenerDetalle(id) {
    const producto = this.obtenerPorId(id);
    if (!producto) return null;
    const movimientos = db.prepare(
      `SELECT id, tipo, cantidad, creado_en
       FROM movimientos WHERE producto_id = ?
       ORDER BY creado_en DESC LIMIT 20`
    ).all(id);
    return { producto, movimientos };
  },

   reporteStockPorCategoria() {
    return db.prepare(
      `SELECT c.nombre AS categoria,
              COUNT(p.id) AS total_productos,
              SUM(p.stock) AS stock_total,
              ROUND(AVG(p.precio), 2) AS precio_promedio
       FROM categorias c
       LEFT JOIN productos p ON p.categoria_id = c.id
       GROUP BY c.id ORDER BY stock_total DESC`
    ).all();
  },

  productosConStockBajo(umbral = 5) {
    return db.prepare(
      `SELECT id, sku, nombre, stock FROM productos
       WHERE stock < ? ORDER BY stock ASC`
    ).all(umbral);
  },

  contarTodos() {
    return db.prepare(`SELECT COUNT(*) AS total FROM productos`).get().total;
  },
};

const Categorias = {
  listar() {
    return db.prepare(`SELECT * FROM categorias ORDER BY nombre`).all();
  },
  crear(nombre) {
    db.prepare(`INSERT OR IGNORE INTO categorias (nombre) VALUES (?)`).run(nombre);
    return db.prepare(`SELECT * FROM categorias WHERE nombre = ?`).get(nombre);
  },
};

const Movimientos = {
  registrar({ producto_id, tipo, cantidad }) {
    db.prepare(
      `INSERT INTO movimientos (producto_id, tipo, cantidad) VALUES (?, ?, ?)`
    ).run(producto_id, tipo, cantidad);
    const delta = tipo === 'ENTRADA' ? cantidad : -cantidad;
    db.prepare(
      `UPDATE productos SET stock = stock + ?, actualizado_en = datetime('now') WHERE id = ?`
    ).run(delta, producto_id);
    return Productos.obtenerDetalle(producto_id);
  },
};

function explicarPlan(sql, params = []) {
  return db.prepare(`EXPLAIN QUERY PLAN ${sql}`).all(...params);
}

module.exports = {
  db, Productos, Categorias, Movimientos,
  crearIndicesOptimizacion, eliminarIndicesOptimizacion, explicarPlan,
};

SRC SERVER

// src/server.js
// Servidor HTTP nativo (sin frameworks). Corre con: node src/server.js

const http = require('node:http');
const { URL } = require('node:url');
const fs   = require('node:fs');
const path = require('node:path');
const { Productos, Categorias, Movimientos, explicarPlan } = require('./db');

const PORT = process.env.PORT || 3000;

function enviarJSON(res, status, payload) {
  const body = JSON.stringify(payload, null, 2);
  res.writeHead(status, {
    'Content-Type': 'application/json; charset=utf-8',
    'Content-Length': Buffer.byteLength(body),
  });
  res.end(body);
}

function leerCuerpo(req) {
  return new Promise((resolve, reject) => {
    let data = '';
    req.on('data', (chunk) => (data += chunk));
    req.on('end', () => {
      if (!data) return resolve({});
      try { resolve(JSON.parse(data)); }
      catch (e) { reject(new Error('JSON inválido en el cuerpo de la petición')); }
    });
    req.on('error', reject);
  });
}

const servidor = http.createServer(async (req, res) => {
  const url = new URL(req.url, `http://localhost:${PORT}`);
  const partes = url.pathname.split('/').filter(Boolean);

  try {
    // Página web
    if (url.pathname === '/' || url.pathname === '/index.html') {
      const filePath = path.join(__dirname, '..', 'public', 'index.html');
      const html = fs.readFileSync(filePath, 'utf-8');
      res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
      return res.end(html);
    }

    // GET /api/categorias
    if (partes[0]==='api' && partes[1]==='categorias' && req.method==='GET')
      return enviarJSON(res, 200, Categorias.listar());

    // POST /api/categorias
    if (partes[0]==='api' && partes[1]==='categorias' && req.method==='POST') {
      const { nombre } = await leerCuerpo(req);
      if (!nombre) return enviarJSON(res, 400, { error: 'nombre es requerido' });
      return enviarJSON(res, 201, Categorias.crear(nombre));
    }

    // GET /api/productos (listado paginado)
    if (partes[0]==='api' && partes[1]==='productos' && partes.length===2 && req.method==='GET') {
      const busqueda = url.searchParams.get('q') || '';
      const categoriaId = url.searchParams.get('categoria_id') ? Number(url.searchParams.get('categoria_id')) : null;
      const pagina = Number(url.searchParams.get('pagina') || 1);
      const tamanioPagina = Number(url.searchParams.get('tamanio') || 10);
      return enviarJSON(res, 200, Productos.listar({ busqueda, categoriaId, pagina, tamanioPagina }));
    }

    // POST /api/productos (crear)
    if (partes[0]==='api' && partes[1]==='productos' && partes.length===2 && req.method==='POST') {
      const cuerpo = await leerCuerpo(req);
      const { sku, nombre, precio, categoria_id } = cuerpo;
      if (!sku || !nombre || precio==null || !categoria_id)
        return enviarJSON(res, 400, { error: 'sku, nombre, precio y categoria_id son requeridos' });
      return enviarJSON(res, 201, Productos.crear(cuerpo));
    }

    // GET /api/productos/:id (detalle)
    if (partes[0]==='api' && partes[1]==='productos' && partes.length===3 && req.method==='GET') {
      const detalle = Productos.obtenerDetalle(Number(partes[2]));
      if (!detalle) return enviarJSON(res, 404, { error: 'producto no encontrado' });
      return enviarJSON(res, 200, detalle);
    }

    // PUT /api/productos/:id (actualizar)
    if (partes[0]==='api' && partes[1]==='productos' && partes.length===3 && req.method==='PUT') {
      const cuerpo = await leerCuerpo(req);
      const actualizado = Productos.actualizar(Number(partes[2]), cuerpo);
      if (!actualizado) return enviarJSON(res, 404, { error: 'producto no encontrado' });
      return enviarJSON(res, 200, actualizado);
    }

    // DELETE /api/productos/:id (eliminar)
    if (partes[0]==='api' && partes[1]==='productos' && partes.length===3 && req.method==='DELETE') {
      const info = Productos.eliminar(Number(partes[2]));
      if (info.changes===0) return enviarJSON(res, 404, { error: 'producto no encontrado' });
      return enviarJSON(res, 204, null);
    }

    // POST /api/productos/:id/movimientos
    if (partes[0]==='api' && partes[1]==='productos' && partes[3]==='movimientos' && req.method==='POST') {
      const { tipo, cantidad } = await leerCuerpo(req);
      if (!['ENTRADA','SALIDA'].includes(tipo) || !cantidad)
        return enviarJSON(res, 400, { error: 'tipo (ENTRADA|SALIDA) y cantidad son requeridos' });
      return enviarJSON(res, 200, Movimientos.registrar({ producto_id: Number(partes[2]), tipo, cantidad }));
    }

    // GET /api/reportes/stock-por-categoria
    if (partes[0]==='api' && partes[1]==='reportes' && partes[2]==='stock-por-categoria')
      return enviarJSON(res, 200, Productos.reporteStockPorCategoria());

    // GET /api/reportes/stock-bajo
    if (partes[0]==='api' && partes[1]==='reportes' && partes[2]==='stock-bajo') {
      const umbral = Number(url.searchParams.get('umbral') || 5);
      return enviarJSON(res, 200, Productos.productosConStockBajo(umbral));
    }

    // GET /api/explain
    if (partes[0]==='api' && partes[1]==='explain') {
      const q = url.searchParams.get('q') || '';
      const plan = explicarPlan(
        `SELECT id, sku, nombre, precio, stock FROM productos WHERE nombre LIKE ? ORDER BY nombre LIMIT 10 OFFSET 0`,
        [`${q}%`]
      );
      return enviarJSON(res, 200, plan);
    }

    return enviarJSON(res, 404, { error: 'ruta no encontrada' });
  } catch (error) {
    return enviarJSON(res, 500, { error: error.message });
  }
});

servidor.listen(PORT, () => {
  console.log(`Servidor escuchando en http://localhost:${PORT}`);
});

module.exports = servidor; 
HTML
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>CRUD Productos — Sin Servidor</title>
  <style>
    body { font-family: Arial, sans-serif; margin: 30px; background-color: #f0f0f0; }
    h1 { text-align: center; color: #333; }
    h2 { color: #444; border-bottom: 2px solid #999; padding-bottom: 5px; }
    .seccion { background-color: white; padding: 20px; margin-bottom: 25px; border: 1px solid #ccc; border-radius: 5px; }
    label { display: inline-block; width: 130px; font-weight: bold; margin-bottom: 8px; }
    input[type="text"], input[type="number"] { width: 250px; padding: 5px; margin-bottom: 8px; border: 1px solid #aaa; border-radius: 3px; }
    button { padding: 7px 15px; background-color: #4CAF50; color: white; border: none; border-radius: 3px; cursor: pointer; margin-top: 5px; margin-right: 5px; }
    button:hover { background-color: #45a049; }
    button.btn-eliminar { background-color: #e74c3c; padding: 4px 10px; font-size: 12px; }
    button.btn-editar   { background-color: #3498db; padding: 4px 10px; font-size: 12px; }
    button.btn-stock    { background-color: #e67e22; padding: 4px 10px; font-size: 12px; }
    table { width: 100%; border-collapse: collapse; margin-top: 10px; }
    table th { background-color: #4CAF50; color: white; padding: 8px; text-align: left; }
    table td { padding: 8px; border-bottom: 1px solid #ddd; }
    table tr:hover { background-color: #f5f5f5; }
    #mensaje { padding: 10px; margin-bottom: 15px; border-radius: 4px; display: none; font-weight: bold; }
    .ok  { background-color: #d4edda; color: #155724; border: 1px solid #c3e6cb; }
    .err { background-color: #f8d7da; color: #721c24; border: 1px solid #f5c6cb; }
    .stock-bajo { background-color: #fff3cd; }
    .sin-stock  { background-color: #f8d7da; }
    #modal-fondo { display: none; position: fixed; inset: 0; background: rgba(0,0,0,0.5); z-index: 100; justify-content: center; align-items: center; }
    #modal-fondo.abierto { display: flex; }
    #modal { background: white; padding: 25px; border-radius: 6px; width: 420px; border: 2px solid #4CAF50; }
    #modal h3 { margin-top: 0; color: #333; }
    #modal-error { color: red; font-size: 13px; margin-bottom: 8px; display: none; }
    .badge-modo { display: inline-block; background: #27ae60; color: white; font-size: 12px; padding: 3px 10px; border-radius: 20px; margin-left: 8px; vertical-align: middle; }
    .info-modo { background: #eaf4fb; border: 1px solid #aed6f1; border-radius: 5px; padding: 10px 16px; color: #1a5276; font-size: 13px; margin-bottom: 18px; }
  </style>
</head>
<body>

<h1>Sistema de Gestión de Productos <span class="badge-modo">✓ Sin servidor</span></h1>

<div class="info-modo">
  ℹ️ Esta versión funciona <strong>completamente en el navegador</strong>, sin necesidad de servidor ni Node.js.
  Los datos se almacenan en memoria durante la sesión.
</div>

<div id="mensaje"></div>

<!-- AGREGAR -->
<div class="seccion">
  <h2>Agregar nuevo producto</h2>
  <div><label>Nombre:</label><input type="text" id="nombre" placeholder="Nombre del producto" /></div>
  <div><label>Descripción:</label><input type="text" id="descripcion" placeholder="Opcional" /></div>
  <div><label>Precio:</label><input type="number" id="precio" placeholder="0.00" step="0.01" min="0" /></div>
  <div><label>Stock:</label><input type="number" id="stock" placeholder="0" min="0" /></div>
  <br>
  <button onclick="agregarProducto()">Agregar Producto</button>
</div>

<!-- LISTAR -->
<div class="seccion">
  <h2>Lista de Productos</h2>
  <div style="margin-bottom:12px;">
    <input type="text" id="buscar-nombre" style="padding:5px;width:230px;border:1px solid #aaa;border-radius:3px;margin-right:8px;" placeholder="Buscar por nombre..." oninput="buscar()" />
    <button onclick="cargarProductos()">Ver todos</button>
  </div>
  <table>
    <thead><tr><th>ID</th><th>Nombre</th><th>Precio</th><th>Stock</th><th>Acciones</th></tr></thead>
    <tbody id="cuerpo-tabla">
      <tr><td colspan="5" style="text-align:center;color:#888;">No hay productos aún.</td></tr>
    </tbody>
  </table>
  <div style="margin-top:10px;color:#888;font-size:13px;" id="info-paginacion"></div>
  <div style="margin-top:8px;">
    <button onclick="paginaAnterior()">« Anterior</button>
    <button onclick="paginaSiguiente()">Siguiente »</button>
  </div>
</div>

<!-- MODAL EDITAR -->
<div id="modal-fondo">
  <div id="modal">
    <h3>Editar Producto</h3>
    <div id="modal-error"></div>
    <div><label>Nombre:</label><input type="text" id="edit-nombre" /></div>
    <div><label>Descripción:</label><input type="text" id="edit-descripcion" /></div>
    <div><label>Precio:</label><input type="number" id="edit-precio" step="0.01" min="0" /></div>
    <div><label>Stock:</label><input type="number" id="edit-stock" min="0" /></div>
    <br>
    <button onclick="guardarEdicion()">Guardar cambios</button>
    <button onclick="cerrarModal()" style="background-color:#888;">Cancelar</button>
  </div>
</div>

<!-- MODAL STOCK -->
<div id="modal-stock-fondo" style="display:none;position:fixed;inset:0;background:rgba(0,0,0,0.5);z-index:100;justify-content:center;align-items:center;">
  <div style="background:white;padding:25px;border-radius:6px;width:360px;border:2px solid #e67e22;">
    <h3 style="margin-top:0;">Registrar Movimiento de Stock</h3>
    <p id="stock-producto-nombre" style="color:#555;margin-bottom:12px;"></p>
    <div>
      <label>Tipo:</label>
      <select id="stock-tipo" style="padding:5px;border:1px solid #aaa;border-radius:3px;">
        <option value="ENTRADA">ENTRADA (suma stock)</option>
        <option value="SALIDA">SALIDA (resta stock)</option>
      </select>
    </div>
    <div style="margin-top:8px;">
      <label>Cantidad:</label>
      <input type="number" id="stock-cantidad" value="1" min="1" style="width:80px;padding:5px;border:1px solid #aaa;border-radius:3px;" />
    </div>
    <br>
    <button onclick="guardarMovimiento()" style="background-color:#e67e22;">Registrar</button>
    <button onclick="cerrarModalStock()" style="background-color:#888;">Cancelar</button>
  </div>
</div>

<script>
  let productos = [];
  let nextId = 1;

  const datosSeed = [
    { nombre: 'Laptop Lenovo IdeaPad', descripcion: 'Intel Core i5, 8GB RAM, 256GB SSD', precio: 2499.90, stock: 12 },
    { nombre: 'Monitor Samsung 24"',   descripcion: 'Full HD, 60Hz, IPS',                precio: 799.00,  stock: 8  },
    { nombre: 'Teclado Mecánico',      descripcion: 'Switch Blue, RGB, USB',              precio: 199.50,  stock: 25 },
    { nombre: 'Mouse Inalámbrico',     descripcion: 'Logitech M185, 1000 DPI',            precio: 69.90,   stock: 3  },
    { nombre: 'Auriculares Sony',      descripcion: 'WH-1000XM4, cancelación de ruido',   precio: 1199.00, stock: 0  },
    { nombre: 'Webcam Logitech C920',  descripcion: 'Full HD 1080p, micrófono integrado', precio: 349.00,  stock: 7  },
  ];
  datosSeed.forEach(p => { productos.push({ id: nextId++, ...p, creado_en: new Date().toISOString() }); });

  let paginaActual = 1;
  const tamPagina  = 10;
  let idEditando   = null;
  let idMovimiento = null;

  function mostrarMensaje(texto, tipo) {
    const el = document.getElementById('mensaje');
    el.textContent = texto; el.className = tipo; el.style.display = 'block';
    setTimeout(() => { el.style.display = 'none'; }, 4000);
  }

  function dbListar(busqueda, pagina, tam) {
    let lista = productos.slice();
    if (busqueda) { const q = busqueda.toLowerCase(); lista = lista.filter(p => p.nombre.toLowerCase().startsWith(q)); }
    lista.sort((a, b) => a.nombre.localeCompare(b.nombre));
    const total = lista.length, offset = (pagina - 1) * tam;
    return { productos: lista.slice(offset, offset + tam), total, pagina, tamanio: tam };
  }

  function dbCrear(datos)     { const p = { id: nextId++, ...datos, creado_en: new Date().toISOString() }; productos.push(p); return p; }
  function dbObtener(id)      { return productos.find(p => p.id === id) || null; }
  function dbEliminar(id)     { const idx = productos.findIndex(p => p.id === id); if (idx === -1) return false; productos.splice(idx, 1); return true; }
  function dbActualizar(id, campos) {
    const idx = productos.findIndex(p => p.id === id);
    if (idx === -1) return null;
    productos[idx] = { ...productos[idx], ...campos, actualizado_en: new Date().toISOString() };
    return productos[idx];
  }
  function dbMovimiento(id, tipo, cantidad) {
    const p = dbObtener(id); if (!p) return null;
    p.stock = Math.max(0, p.stock + (tipo === 'ENTRADA' ? cantidad : -cantidad));
    p.actualizado_en = new Date().toISOString(); return p;
  }

  function agregarProducto() {
    const nombre = document.getElementById('nombre').value.trim();
    const descripcion = document.getElementById('descripcion').value.trim();
    const precio = parseFloat(document.getElementById('precio').value);
    const stock  = parseInt(document.getElementById('stock').value || '0');
    if (!nombre || isNaN(precio) || precio < 0) { mostrarMensaje('Completa Nombre y Precio con valores válidos.', 'err'); return; }
    dbCrear({ nombre, descripcion, precio, stock });
    mostrarMensaje('Producto "' + nombre + '" agregado correctamente.', 'ok');
    ['nombre','descripcion','precio','stock'].forEach(id => { document.getElementById(id).value = ''; });
    paginaActual = 1; cargarProductos();
  }

  function cargarProductos() {
    const busqueda = document.getElementById('buscar-nombre').value.trim();
    const data = dbListar(busqueda, paginaActual, tamPagina);
    renderTabla(data.productos, data.total, data.pagina, data.tamanio);
  }

  function renderTabla(lista, total, pagina, tamanio) {
    const tbody = document.getElementById('cuerpo-tabla');
    if (!lista || lista.length === 0) {
      tbody.innerHTML = '<tr><td colspan="5" style="text-align:center;color:#888;">No se encontraron productos</td></tr>';
      document.getElementById('info-paginacion').textContent = ''; return;
    }
    tbody.innerHTML = lista.map(p => {
      let c = p.stock === 0 ? 'sin-stock' : p.stock < 5 ? 'stock-bajo' : '';
      return '<tr class="'+c+'"><td>'+p.id+'</td><td>'+p.nombre+'</td><td>S/ '+Number(p.precio).toFixed(2)+'</td><td>'+p.stock+'</td><td>'
        +'<button class="btn-editar" onclick="abrirModal('+p.id+')">Editar</button> '
        +'<button class="btn-stock" onclick="abrirModalStock('+p.id+',\''+p.nombre.replace(/'/g,"\\'")+"')\">"+'Stock</button> '
        +'<button class="btn-eliminar" onclick="eliminarProducto('+p.id+',\''+p.nombre.replace(/'/g,"\\'")+'\')">Eliminar</button>'
        +'</td></tr>';
    }).join('');
    document.getElementById('info-paginacion').textContent =
      'Página '+pagina+' de '+Math.max(1,Math.ceil(total/tamanio))+' — Total: '+total+' productos';
  }

  function buscar()          { paginaActual = 1; cargarProductos(); }
  function paginaAnterior()  { if (paginaActual > 1) { paginaActual--; cargarProductos(); } }
  function paginaSiguiente() { paginaActual++; cargarProductos(); }

  function eliminarProducto(id, nombre) {
    if (!confirm('¿Seguro que deseas eliminar "'+nombre+'"?')) return;
    if (dbEliminar(id)) { mostrarMensaje('Producto "'+nombre+'" eliminado.', 'ok'); cargarProductos(); }
    else mostrarMensaje('No se encontró el producto.', 'err');
  }

  function abrirModal(id) {
    const data = dbObtener(id); if (!data) { mostrarMensaje('No se pudo cargar el producto.', 'err'); return; }
    idEditando = id;
    document.getElementById('edit-nombre').value      = data.nombre;
    document.getElementById('edit-descripcion').value = data.descripcion || '';
    document.getElementById('edit-precio').value      = data.precio;
    document.getElementById('edit-stock').value       = data.stock;
    document.getElementById('modal-error').style.display = 'none';
    document.getElementById('modal-fondo').classList.add('abierto');
  }
  function cerrarModal() { document.getElementById('modal-fondo').classList.remove('abierto'); idEditando = null; }

  function guardarEdicion() {
    const nombre = document.getElementById('edit-nombre').value.trim();
    const descripcion = document.getElementById('edit-descripcion').value.trim();
    const precio = parseFloat(document.getElementById('edit-precio').value);
    const stock  = parseInt(document.getElementById('edit-stock').value);
    if (!nombre || isNaN(precio)) {
      document.getElementById('modal-error').textContent = 'Nombre y precio son obligatorios.';
      document.getElementById('modal-error').style.display = 'block'; return;
    }
    const actualizado = dbActualizar(idEditando, { nombre, descripcion, precio, stock });
    if (actualizado) { mostrarMensaje('Producto actualizado correctamente.', 'ok'); cerrarModal(); cargarProductos(); }
    else { document.getElementById('modal-error').textContent = 'No se encontró el producto.'; document.getElementById('modal-error').style.display = 'block'; }
  }

  function abrirModalStock(id, nombre) {
    idMovimiento = id;
    document.getElementById('stock-producto-nombre').textContent = 'Producto: ' + nombre;
    document.getElementById('stock-cantidad').value = 1;
    document.getElementById('modal-stock-fondo').style.display = 'flex';
  }
  function cerrarModalStock() { document.getElementById('modal-stock-fondo').style.display = 'none'; idMovimiento = null; }

  function guardarMovimiento() {
    const tipo = document.getElementById('stock-tipo').value;
    const cantidad = parseInt(document.getElementById('stock-cantidad').value);
    if (!cantidad || cantidad < 1) { alert('Ingresa una cantidad válida.'); return; }
    const resultado = dbMovimiento(idMovimiento, tipo, cantidad);
    if (resultado) { mostrarMensaje(tipo+' de '+cantidad+' unidades registrado.', 'ok'); cerrarModalStock(); cargarProductos(); }
    else mostrarMensaje('Error al registrar el movimiento.', 'err');
  }

  cargarProductos();
</script>
</body>
</html>
scripts/seed.js
// Genera datos de prueba masivos. Uso: node scripts/seed.js [cantidad]
const fs   = require('node:fs');
const path = require('node:path');

const dataDir = path.join(__dirname, '..', 'data');
if (!fs.existsSync(dataDir)) fs.mkdirSync(dataDir, { recursive: true });

const { db, Categorias } = require('../src/db');

const CANTIDAD = Number(process.argv[2] || 20000);

const NOMBRES_CATEGORIAS = ['Electrónica','Hogar','Deportes','Juguetería','Ropa','Alimentos','Librería','Ferretería','Belleza','Mascotas'];
const ADJETIVOS   = ['Pro','Max','Lite','Plus','Classic','Eco','Ultra','Mini'];
const SUSTANTIVOS = ['Audífono','Mochila','Lámpara','Cuaderno','Zapatilla','Reloj','Mouse','Teclado','Silla','Termo','Bicicleta','Cámara','Bolso'];

function main() {
  console.log(`Insertando ${CANTIDAD} productos de prueba...`);
  const categorias = NOMBRES_CATEGORIAS.map(n => Categorias.crear(n));
  const stmt = db.prepare(`INSERT INTO productos (sku, nombre, descripcion, precio, stock, categoria_id) VALUES (?, ?, ?, ?, ?, ?)`);
  const inicio = Date.now();
  db.exec('BEGIN');
  try {
    for (let i = 1; i <= CANTIDAD; i++) {
      stmt.run(
        `SKU-${String(i).padStart(6,'0')}`,
        `${SUSTANTIVOS[i % SUSTANTIVOS.length]} ${ADJETIVOS[i % ADJETIVOS.length]} ${i}`,
        `Descripción de prueba para el producto número ${i}.`,
        Number((5 + (i % 500) * 1.37).toFixed(2)),
        i % 50,
        categorias[i % categorias.length].id
      );
    }
    db.exec('COMMIT');
  } catch(e) { db.exec('ROLLBACK'); throw e; }
  console.log(`Listo. ${CANTIDAD} productos insertados en ${Date.now()-inicio} ms.`);
}

main();
scripts/benchmark.js
// Mide rendimiento ANTES y DESPUÉS de cada optimización.
// Uso: node scripts/benchmark.js

const { db, Productos, crearIndicesOptimizacion, eliminarIndicesOptimizacion, explicarPlan } = require('../src/db');

function medir(etiqueta, fn, repeticiones = 30) {
  fn(); // warmup
  const inicio = process.hrtime.bigint();
  for (let i = 0; i < repeticiones; i++) fn();
  const fin = process.hrtime.bigint();
  const ms = Number(fin - inicio) / 1e6 / repeticiones;
  console.log(`  ${etiqueta}: ${ms.toFixed(3)} ms (promedio de ${repeticiones} ejecuciones)`);
  return ms;
}

function planComoTexto(plan) { return plan.map(p => p.detail).join(' | '); }

function main() {
  const total = Productos.contarTodos();
  console.log(`\nBase de datos con ${total} productos.\n`);
  if (total < 1000) console.log('Sugerencia: ejecuta "npm run seed" primero.\n');

  console.log('========================================================');
  console.log(' BENCHMARK 1: búsqueda por nombre (LIKE "Mochila%")');
  console.log('========================================================');

  const consultaBusqueda = () =>
    db.prepare(`SELECT id, sku, nombre, precio, stock FROM productos WHERE nombre LIKE ? ORDER BY nombre LIMIT 10 OFFSET 0`).all('Mochila%');

  console.log('\n-- SIN índice --');
  eliminarIndicesOptimizacion();
  console.log('Plan:', planComoTexto(explicarPlan(`SELECT id, sku, nombre, precio, stock FROM productos WHERE nombre LIKE ? ORDER BY nombre LIMIT 10 OFFSET 0`, ['Mochila%'])));
  const msSin = medir('Tiempo SIN índice', consultaBusqueda);

  console.log('\n-- CON índice --');
  crearIndicesOptimizacion();
  console.log('Plan:', planComoTexto(explicarPlan(`SELECT id, sku, nombre, precio, stock FROM productos WHERE nombre LIKE ? ORDER BY nombre LIMIT 10 OFFSET 0`, ['Mochila%'])));
  const msCon = medir('Tiempo CON índice', consultaBusqueda);
  console.log(`\n  >> Mejora: ${(((msSin-msCon)/msSin)*100).toFixed(1)}% más rápido con el índice.\n`);

  console.log('========================================================');
  console.log(' BENCHMARK 2: reporte agregado (stock por categoría)');
  console.log('========================================================');
  const msSQL = medir('GROUP BY en SQL', () => Productos.reporteStockPorCategoria());
  const msJS  = medir('reduce() en Node.js', () => {
    const filas = db.prepare(`SELECT categoria_id, stock, precio FROM productos`).all();
    const acc = {};
    for (const f of filas) {
      acc[f.categoria_id] ??= { total:0, stock:0, precioSum:0 };
      acc[f.categoria_id].total++; acc[f.categoria_id].stock += f.stock; acc[f.categoria_id].precioSum += f.precio;
    }
    return acc;
  });
  console.log(`\n  >> GROUP BY en SQL es ${(((msJS-msSQL)/msJS)*100).toFixed(1)}% más rápido.\n`);

  console.log('========================================================');
  console.log(' BENCHMARK 3: filtro por categoría');
  console.log('========================================================');
  const consultaCat = () => db.prepare(`SELECT id, nombre, stock FROM productos WHERE categoria_id = ? LIMIT 50`).all(3);
  eliminarIndicesOptimizacion();
  const msCatSin = medir('SIN índice', consultaCat);
  crearIndicesOptimizacion();
  const msCatCon = medir('CON índice', consultaCat);
  console.log(`\n  >> Mejora: ${(((msCatSin-msCatCon)/msCatSin)*100).toFixed(1)}%\n`);
}

main();
 
