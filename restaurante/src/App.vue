<template>
  <div id="app">
    <!-- Header -->
    <header>
      <h1>bristo y brasa</h1>
      <button class="btn-pedido" v-on:click="abrirModal">
        <span>{{ carrito.length > 0 ? 'Mi Carrito ' : 'CARRITO' }}</span>
        <span class="badge-carrito">{{ totalUnidades }}</span>
      </button>
    </header>

    <!-- Barra de Categorias -->
    <nav class="barra-categorias">
      <template v-for="categoria in categorias">
        <button v-if="categoriaSeleccionada === categoria" class="btn-categoria active"
          v-on:click="filtrarPlatos(categoria)">
          {{ categoria }}
        </button>
        <button v-else class="btn-categoria" v-on:click="filtrarPlatos(categoria)">
          {{ categoria }}
        </button>
      </template>
    </nav>

    <!-- Grid de Productos -->
    <main>
      <div class="grid-productos">
        <div v-for="producto in platosFiltrados" class="card">

          <div class="card-en-pedido" v-if="cantidadEnCarrito(producto.nombre) > 0">
            {{ cantidadEnCarrito(producto.nombre) }} en pedido
          </div>

          <img :src="producto.imagen">
          <div class="card-content">
            <div class="card-title">{{ producto.nombre }}</div>
            <div class="card-price-tag">${{ producto.precio.toLocaleString() }}</div>

            <div v-if="producto.stock > 0" class="stock-tag">
              Disponibles: {{ producto.stock }}
            </div>
            <div v-else class="stock-tag agotado">
              AGOTADO
            </div>

            <p class="card-desc">{{ producto.descripcion }}</p>

            <button v-if="producto.stock > 0" class="btn-pedido btn-full" v-on:click="agregarAlCarrito(producto)">
              {{ cantidadEnCarrito(producto.nombre) > 0 ? 'AGREGAR OTRO' : 'AGREGAR' }}
            </button>
            <button v-else class="btn-pedido btn-full btn-desactivado">
              SIN STOCK
            </button>
          </div>
        </div>
      </div>
    </main>

    <!-- Modal Factura -->
    <div v-if="mostrarModal" class="modal-overlay" v-on:click="cerrarSiEsFuera">
      <div class="factura-box">
        <div class="factura-header">
          <h2>SU CUENTA</h2>
          <p>Bistro & Brasa - Mesa #{{ cliente.mesa || ' ' }}</p>
          <button v-on:click="cerrarModal" class="btn-cerrar-x">x</button>
        </div>

        <div class="factura-body">

          <!-- Carrito vacio -->
          <div v-if="carrito.length === 0" class="factura-vacia">
            <p class="vacia-sub">Su carrito esta vacio.</p>
            <p class="vacia-sub">Agregue platos desde el menu.</p>
            <button class="btn-pedido btn-oscuro" v-on:click="cerrarModal">VER MENU</button>
          </div>

          <!-- Encabezado de items -->
          <div class="items-header" v-if="carrito.length > 0">
            <span class="items-count">{{ totalUnidades }} articulo(s)</span>
            <button class="btn-vaciar" v-on:click="vaciarCarrito">Vaciar todo</button>
          </div>

          <!-- Items -->
          <div v-for="(item, index) in carrito" class="item-control">
            <div class="item-info">
              <div class="item-nombre">{{ item.nombre }}</div>
              <div class="item-precio-unit">${{ item.precio.toLocaleString() }} c/u</div>
            </div>
            <div class="item-qty-selector">
              <button class="qty-btn" v-on:click="cambiarCantidad(index, -1)">-</button>
              <span class="item-nombre">{{ item.cantidad }}</span>
              <button class="qty-btn" v-on:click="cambiarCantidad(index, 1)">+</button>
            </div>
            <div class="item-total-line">
              ${{ (item.precio * item.cantidad).toLocaleString() }}
            </div>
          </div>

          <!-- Formulario -->
          <div v-if="carrito.length > 0" class="form-datos">

            <div class="form-field">
              <label>NOMBRE DEL CLIENTE</label>
              <input type="text" v-model="cliente.nombre" placeholder="Ej: Juan Perez">
            </div>

            <div class="form-grid-2">
              <div class="form-field">
                <label>NUMERO DE MESA</label>
                <input type="number" v-model="cliente.mesa" placeholder="01" min="1">
              </div>
              <div class="form-field">
                <label>METODO DE PAGO</label>
                <select v-model="cliente.metodoPago">
                  <option value="">Seleccionar</option>
                  <option value="Efectivo">Efectivo</option>
                  <option value="Tarjeta">Tarjeta</option>
                </select>
              </div>
            </div>

            <div class="form-field">
              <label>INSTRUCCIONES ESPECIALES</label>
              <textarea v-model="cliente.notas" rows="2" placeholder="Ej: Sin cebolla, termino medio"></textarea>
            </div>

            <!-- Propina -->
            <label class="label-propina">DESEA DEJAR PROPINA?</label>
            <div class="propina-grid">
              <button v-if="propinaPorcentaje === 0" class="btn-propina active" v-on:click="seleccionarPropina(0)">Sin
                propina</button>
              <button v-else class="btn-propina" v-on:click="seleccionarPropina(0)">Sin propina</button>

              <button v-if="propinaPorcentaje === 10" class="btn-propina active"
                v-on:click="seleccionarPropina(10)">10%</button>
              <button v-else class="btn-propina" v-on:click="seleccionarPropina(10)">10%</button>

              <button v-if="propinaPorcentaje === 15" class="btn-propina active"
                v-on:click="seleccionarPropina(15)">15%</button>
              <button v-else class="btn-propina" v-on:click="seleccionarPropina(15)">15%</button>
            </div>

            <!-- Totales -->
            <div class="resumen-final">
              <div class="total-line">
                <span>Subtotal:</span>
                <span>${{ totales.subtotal.toLocaleString() }}</span>
              </div>
              <div class="total-line">
                <span>IVA (19%):</span>
                <span>${{ totales.iva.toLocaleString() }}</span>
              </div>
              <div class="total-line total-propina-line" v-if="propinaPorcentaje > 0">
                <span>Propina ({{ propinaPorcentaje }}%):</span>
                <span>${{ totales.propina.toLocaleString() }}</span>
              </div>
              <div class="total-line total-grande">
                <span>TOTAL:</span>
                <span>${{ totales.total.toLocaleString() }}</span>
              </div>

              <div class="botones-pago">
                <button class="btn-pedido btn-gris" v-on:click="cerrarModal">CERRAR</button>
                <button class="btn-pedido" v-on:click="finalizarPedido">PAGAR AHORA</button>
              </div>
            </div>

          </div>
        </div>
      </div>
    </div>

    <!-- Footer -->
    <footer>
      <h2 class="footer-logo">BISTRO & BRASA</h2>
      <p class="footer-info">Calle de los Sabores #123 - Abierto de 12pm a 10pm</p>
    </footer>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import Swal from 'sweetalert2';


// ========================================
// 1. ESTADO REACTIVO
// ========================================
const platos = ref([
  { nombre: "Hamburguesa Especial", descripcion: "Carne angus 200g, queso cheddar, tocino y vegetales frescos.", precio: 25000, imagen: "src/assets/hamburguesaE.jpg", categoria: "Rapidas", stock: 8 },
  { nombre: "Club Sandwich", descripcion: "Triple piso with pollo, jamon, huevo, queso y tocino.", precio: 21000, imagen: "src/assets/clubS.jpg", categoria: "Rapidas", stock: 12 },
  { nombre: "Hot-dog Picante", descripcion: "Salchicha premium, jalapenos, cebolla caramelizada y salsa brava.", precio: 18000, imagen: "src/assets/hotdogP.jpg", categoria: "Rapidas", stock: 15 },
  { nombre: "Nuggets de Pollo", descripcion: "8 piezas de pechuga apanada with papas fritas pequenas.", precio: 16000, imagen: "src/assets/nuggetsP.jpg", categoria: "Snacks", stock: 20 },
  { nombre: "Salchipapa Monstruosa", descripcion: "Cama de papas fritas, salchicha suiza, pollo desmechado y mucho queso.", precio: 32000, imagen: "src/assets/salchipapaM.jpg", categoria: "Snacks", stock: 6 },
  { nombre: "Empanadas (x3)", descripcion: "Tradicionales de carne y papa, acompanadas de aji casero.", precio: 12000, imagen: "src/assets/empanadasT.jpg", categoria: "Snacks", stock: 30 },
  { nombre: "Nachos Locos", descripcion: "Totopos crujientes with chili, queso fundido y pico de gallo.", precio: 19000, imagen: "src/assets/nachosL.jpg", categoria: "Snacks", stock: 10 },
  { nombre: "Papas Bravas", descripcion: "Cubos de papa frita with salsa picante y alioli.", precio: 14000, imagen: "src/assets/papasB.jpg", categoria: "Snacks", stock: 18 },
  { nombre: "Alitas BBQ (x10)", descripcion: "Alitas crujientes banadas en salsa a eleccion.", precio: 29000, imagen: "src/assets/alitasB.jpg", categoria: "Snacks", stock: 12 },
  { nombre: "Tacos al Pastor", descripcion: "3 tacos de cerdo marinado with pina, cilantro y cebolla.", precio: 20000, imagen: "src/assets/tacosP.jpg", categoria: "Tacos y Wraps", stock: 25 },
  { nombre: "Burrito Supremo", descripcion: "Tortilla gigante de harina rellena de carne, frijol y guacamole.", precio: 24000, imagen: "src/assets/burritoP.jpg", categoria: "Tacos y Wraps", stock: 10 },
  { nombre: "Wrap de Vegetales", descripcion: "Tortilla integral, hummus, vegetales asados y queso feta.", precio: 17000, imagen: "src/assets/wrapV.jpg", categoria: "Tacos y Wraps", stock: 14 },
  { nombre: "Quesadilla de Pollo", descripcion: "Tortilla de trigo with queso fundido y pollo desmechado.", precio: 18500, imagen: "src/assets/quesadillas.jpg", categoria: "Tacos y Wraps", stock: 12 },
  { nombre: "Arepa con Todo", descripcion: "Arepa de maiz rellena de carne, pollo, queso y huevo de codorniz.", precio: 15500, imagen: "src/assets/arepaT.jpg", categoria: "Tacos y Wraps", stock: 20 },
  { nombre: "Costillas BBQ", descripcion: "Costillitas de cerdo banadas en salsa BBQ ahumada with papas.", precio: 45000, imagen: "src/assets/costillasB.jpg", categoria: "Platos Fuertes", stock: 8 },
  { nombre: "Churrasco 300g", descripcion: "Corte de res a la parrilla with chimichurri artesanal.", precio: 42000, imagen: "src/assets/churrasco.jpg", categoria: "Platos Fuertes", stock: 7 },
  { nombre: "Parrillada Mixta", descripcion: "Res, pollo, cerdo, chorizo y acompanamientos.", precio: 55000, imagen: "src/assets/parrilladaM.jpg", categoria: "Platos Fuertes", stock: 6 },
  { nombre: "Arroz Atollado", descripcion: "Arroz humedo tipico with pollo, cerdo y longaniza.", precio: 28000, imagen: "src/assets/arrozA.jpg", categoria: "Platos Fuertes", stock: 10 },
  { nombre: "Ceviche de Camaron", descripcion: "Camarones frescos marinados en limon citrico y cebolla roja.", precio: 35000, imagen: "src/assets/cevicheC.jpg", categoria: "Platos Fuertes", stock: 15 },
  { nombre: "Sushi Roll Filadelfia", descripcion: "Salmon, queso crema y aguacate, envuelto en sesamo.", precio: 30000, imagen: "src/assets/sushiR.jpg", categoria: "Platos Fuertes", stock: 20 },
  { nombre: "Poke de Atun", descripcion: "Atun marinado, mango, rabano y arroz de sushi.", precio: 31000, imagen: "src/assets/pokeA.jpg", categoria: "Platos Fuertes", stock: 12 },
  { nombre: "Bowl de Salmon", descripcion: "Base de quinoa, salmon fresco, edamame y aderezo ginger.", precio: 38000, imagen: "src/assets/bowlS.jpg", categoria: "Platos Fuertes", stock: 10 },
  { nombre: "Filete de Pescado", descripcion: "Pescado blanco al ajillo with ensalada verde.", precio: 29000, imagen: "src/assets/fileteP.jpg", categoria: "Platos Fuertes", stock: 9 },
  { nombre: "Pizza Pepperoni", descripcion: "Masa artesanal, salsa de tomate italiana y doble pepperoni.", precio: 28000, imagen: "src/assets/pizzaP.jpg", categoria: "Platos Fuertes", stock: 15 },
  { nombre: "Pasta Carbonara", descripcion: "Espagueti with salsa de huevo, queso pecorino y guanciale.", precio: 27000, imagen: "src/assets/pastaC.jpg", categoria: "Platos Fuertes", stock: 12 },
  { nombre: "Lasana de Carne", descripcion: "Capas de pasta with bolonesa de la casa y salsa bechamel.", precio: 26000, imagen: "src/assets/lasañaC.jpg", categoria: "Platos Fuertes", stock: 10 },
  { nombre: "Ramen Tonkotsu", descripcion: "Caldo de cerdo, fideos frescos, huevo marinado y cerdo chashu.", precio: 33000, imagen: "src/assets/ramenT.jpg", categoria: "Platos Fuertes", stock: 8 },
  { nombre: "Sopa de Tortilla", descripcion: "Caldo de tomate with tiras de tortilla, aguacate y crema.", precio: 15000, imagen: "src/assets/sopaT.jpg", categoria: "Platos Fuertes", stock: 20 },
  { nombre: "Ensalada de Pollo", descripcion: "Mix de lechugas, pechuga a la plancha, croutons y aderezo cesar.", precio: 22000, imagen: "src/assets/ensaladaP.jpg", categoria: "Platos Fuertes", stock: 15 },
  { nombre: "Brownie con Helado", descripcion: "Postre de chocolate melcochudo with helado de vainilla.", precio: 12500, imagen: "src/assets/brownieH.jpg", categoria: "Postres", stock: 25 },
  { nombre: "Limonada Cerezada", descripcion: "Refrescante mezcla de limon y cerezas maceradas.", precio: 12000, imagen: "src/assets/limonadaC.jpg", categoria: "Bebidas", stock: 50 },
  { nombre: "Jugo de Mango", descripcion: "Jugo natural de mango maduro, 100% fruta.", precio: 9500, imagen: "src/assets/jugoM.jpg", categoria: "Bebidas", stock: 40 },
  { nombre: "Soda de Frutos Rojos", descripcion: "Soda artesanal with infusion de fresa, mora y arandanos.", precio: 13900, imagen: "src/assets/sodaF.jpg", categoria: "Bebidas", stock: 35 },
  { nombre: "Malteada de Vainilla", descripcion: "Cremosa malteada with crema batida y chispas.", precio: 16500, imagen: "src/assets/malteadaV.jpg", categoria: "Bebidas", stock: 20 },
  { nombre: "Vino Tinto (Copa)", descripcion: "Seleccion de la casa, ideal para carnes rojas.", precio: 18000, imagen: "src/assets/copaV.jpg", categoria: "Bebidas", stock: 15 },
  { nombre: "Cheesecake de Oreo", descripcion: "Postre cremoso de queso with trozos de galleta negra.", precio: 15000, imagen: "src/assets/oreoC.jpg", categoria: "Postres", stock: 12 },
  { nombre: "Tarta de Manzana", descripcion: "Tarta horneada with canela y helado de vainilla.", precio: 14000, imagen: "src/assets/tartaM.jpg", categoria: "Postres", stock: 10 },
  { nombre: "Salmon Teriyaki", descripcion: "Salmon glaseado en salsa dulce with vegetales salteados.", precio: 49000, imagen: "src/assets/salmonT.jpg", categoria: "Platos Fuertes", stock: 15 },
  { nombre: "Lomo a la Pimienta", descripcion: "Corte premium de res with salsa cremosa de pimienta.", precio: 54000, imagen: "src/assets/lomoP.jpg", categoria: "Platos Fuertes", stock: 10 },
  { nombre: "Risotto de Hongos", descripcion: "Arroz cremoso with variedad de setas y queso parmesano.", precio: 39500, imagen: "src/assets/risottoH.jpg", categoria: "Platos Fuertes", stock: 8 },
]);

const categorias = ref(["Todas", "Rapidas", "Snacks", "Tacos y Wraps", "Platos Fuertes", "Bebidas", "Postres"]);
const categoriaSeleccionada = ref("Todas");
const platosFiltrados = ref([...platos.value]);
const carrito = ref([]);
const mostrarModal = ref(false);
const totalUnidades = ref(0);
const propinaPorcentaje = ref(0);
const cliente = ref({ nombre: '', mesa: '', metodoPago: '', notas: '' });
const totales = ref({ subtotal: 0, iva: 0, propina: 0, total: 0 });

// ========================================
// 2. PERSISTENCIA - Cargar datos guardados
// ========================================
const datosGuardados = localStorage.getItem('carrito_bistro');
if (datosGuardados) {
  const guardado = JSON.parse(datosGuardados);
  carrito.value = guardado;

  // Ajustar stock inicial basado en lo guardado
  guardado.forEach(itemCarrito => {
    const plato = platos.value.find(producto => producto.nombre === itemCarrito.nombre);
    if (plato) {
      plato.stock -= itemCarrito.cantidad;
    }
  });
}

// ========================================
// 3. FUNCIONES DE CÁLCULO MANUAL
// ========================================

function actualizarTotales() {
  let subtotal = 0;
  let unidades = 0;

  for (let i = 0; i < carrito.value.length; i++) {
    subtotal += carrito.value[i].precio * carrito.value[i].cantidad;
    unidades += carrito.value[i].cantidad;
  }

  totalUnidades.value = unidades;
  totales.value.subtotal = subtotal;
  totales.value.iva = subtotal * 0.19;
  totales.value.propina = subtotal * (propinaPorcentaje.value / 100);
  totales.value.total = totales.value.subtotal + totales.value.iva + totales.value.propina;

  // Guardar en localStorage
  localStorage.setItem('carrito_bistro', JSON.stringify(carrito.value));
}

// Inicializar totales al cargar
actualizarTotales();

function filtrarPlatos(categoria) {
  categoriaSeleccionada.value = categoria;

  if (categoria === "Todas") {
    platosFiltrados.value = [...platos.value];
  } else {
    const nuevosPlatos = [];
    for (let i = 0; i < platos.value.length; i++) {
      if (platos.value[i].categoria === categoria) {
        nuevosPlatos.push(platos.value[i]);
      }
    }
    platosFiltrados.value = nuevosPlatos;
  }
}

function cantidadEnCarrito(nombre) {
  for (let i = 0; i < carrito.value.length; i++) {
    if (carrito.value[i].nombre === nombre) {
      return carrito.value[i].cantidad;
    }
  }
  return 0;
}

// ========================================
// 4. ACCIONES DEL USUARIO
// ========================================

function agregarAlCarrito(producto) {
  // Validar stock disponible
  if (producto.stock <= 0) {
    Swal.fire({
      icon: 'error',
      title: 'Sin stock',
      text: 'Lo sentimos, este producto se ha agotado.'
    });
    return;
  }

  // Buscar si el producto ya existe en el carrito
  let item = null;
  for (let i = 0; i < carrito.value.length; i++) {
    if (carrito.value[i].nombre === producto.nombre) {
      item = carrito.value[i];
      break;
    }
  }

  // Si ya existe, aumentar cantidad (máximo 3)
  if (item) {
    if (item.cantidad >= 3) {
      Swal.fire({
        icon: 'warning',
        title: 'Límite alcanzado',
        text: 'Solo se permiten máximo 3 unidades de cada producto por cliente.',
        confirmButtonColor: '#E76F51'
      });
      return;
    }
    item.cantidad++;
  } else {
    // Si no existe, agregarlo con cantidad 1
    carrito.value.push({
      nombre: producto.nombre,
      precio: producto.precio,
      cantidad: 1,
      categoria: producto.categoria
    });
  }

  // Reducir stock del producto original
  producto.stock--;

  // Notificación de éxito
  Swal.fire({
    toast: true,
    position: 'top-end',
    icon: 'success',
    title: '¡Agregado!',
    showConfirmButton: false,
    timer: 1000,
    background: '#2A9D8F',
    color: '#fff'
  });

  actualizarTotales();
}

function cambiarCantidad(index, valor) {
  const itemCarrito = carrito.value[index];

  // Buscar el producto original para devolver o quitar stock
  let productoOriginal = null;
  for (let i = 0; i < platos.value.length; i++) {
    if (platos.value[i].nombre === itemCarrito.nombre) {
      productoOriginal = platos.value[i];
      break;
    }
  }

  if (valor > 0) {
    // Aumentar cantidad
    if (itemCarrito.cantidad >= 3) {
      Swal.fire({
        icon: 'warning',
        title: 'Límite alcanzado',
        text: 'Solo se permiten máximo 3 unidades de cada producto por cliente.',
        confirmButtonColor: '#E76F51'
      });
      return;
    }
    if (productoOriginal.stock <= 0) {
      Swal.fire({
        icon: 'error',
        title: 'Sin stock',
        text: 'No hay más unidades disponibles.'
      });
      return;
    }
    itemCarrito.cantidad++;
    productoOriginal.stock--;
  } else {
    // Disminuir cantidad
    itemCarrito.cantidad--;
    productoOriginal.stock++;

    // Si la cantidad llega a 0, eliminar del carrito
    if (itemCarrito.cantidad <= 0) {
      carrito.value.splice(index, 1);
    }
  }

  actualizarTotales();
}

function vaciarCarrito() {
  Swal.fire({
    icon: 'warning',
    title: '¿Vaciar carrito?',
    text: 'Perderás el progreso de la compra',
    showCancelButton: true,
    confirmButtonText: 'Vaciar carrito',
    confirmButtonColor: '#E76F51'
  }).then(function (resultado) {
    if (resultado.isConfirmed) {
      // Devolver stock a todos los productos antes de vaciar
      for (let i = 0; i < carrito.value.length; i++) {
        const itemCarrito = carrito.value[i];
        const platoOriginal = platos.value.find(function (producto) {
          return producto.nombre === itemCarrito.nombre;
        });

        if (platoOriginal) {
          platoOriginal.stock += itemCarrito.cantidad;
        }
      }

      carrito.value = [];
      actualizarTotales();
    }
  });
}

function seleccionarPropina(valor) {
  propinaPorcentaje.value = valor;
  actualizarTotales();
}

function abrirModal() {
  mostrarModal.value = true;
}

function cerrarModal() {
  mostrarModal.value = false;
}

function cerrarSiEsFuera(evento) {
  if (evento.target.classList.contains('modal-overlay')) {
    cerrarModal();
  }
}

function finalizarPedido() {
  const { nombre, mesa, metodoPago, notas } = cliente.value;

  if (!carrito.value.length) return Swal.fire('Error', 'Carrito vacío', 'error');
  if (!nombre || !mesa || !metodoPago) return Swal.fire('Atención', 'Completa tus datos', 'info');

  const filas = carrito.value.map(i => `
    <tr>
      <td style="padding:8px; border-bottom:2px solid #011627;">${i.nombre} x${i.cantidad}</td>
      <td style="padding:8px; border-bottom:2px solid #011627; text-align:right;">$${(i.precio * i.cantidad).toLocaleString()}</td>
    </tr>`).join('');

  const propinaHtml = propinaPorcentaje.value > 0 
    ? `<div style="display:flex; justify-content:space-between;"><span>Propina (${propinaPorcentaje.value}%):</span><span>$${totales.value.propina.toLocaleString()}</span></div>` 
    : '';

  // Bloque para las notas solo si el cliente escribió algo
  const notasHtml = notas 
    ? `<div style="margin-top:15px; padding:10px; border:2px dashed #011627; font-size:0.9em;">
        <strong>NOTAS:</strong> ${notas}
       </div>` 
    : '';

  const win = window.open('', '_blank');
  win.document.write(`
    <html>
      <head>
        <link href="https://fonts.googleapis.com/css2?family=Bungee&family=Lexend:wght@400;800&display=swap" rel="stylesheet">
        <style>
          body { font-family: 'Lexend', sans-serif; color: #011627; padding: 30px; line-height: 1.4; }
          .ticket { border: 4px solid #011627; padding: 25px; box-shadow: 8px 8px 0 #011627; max-width: 500px; margin: auto; }
          h1 { font-family: 'Bungee'; color: #FF4D4D; text-align: center; margin: 0 0 10px; font-size: 2em; }
          .info-cliente { margin-bottom: 20px; padding-bottom: 10px; border-bottom: 2px solid #011627; font-size: 0.95em; }
          .total { font-weight: 800; font-size: 1.5em; border-top: 4px solid #011627; padding-top: 10px; margin-top: 10px; display: flex; justify-content: space-between; }
          .metodo-pago { text-transform: uppercase; font-weight: 800; color: #2EC4B6; margin-top: 5px; display: block; }
        </style>
      </head>
      <body>
        <div class="ticket">
          <h1>BISTRO Y BRASA</h1>
          
          <div class="info-cliente">
            <p><strong>MESA:</strong> ${mesa} | <strong>CLIENTE:</strong> ${nombre}</p>
            <span class="metodo-pago">PAGO: ${metodoPago}</span>
          </div>

          <table style="width:100%; border-collapse:collapse;">${filas}</table>
          
          <div style="margin-top:15px;">
            <div style="display:flex; justify-content:space-between;"><span>Subtotal:</span><span>$${totales.value.subtotal.toLocaleString()}</span></div>
            <div style="display:flex; justify-content:space-between;"><span>IVA (19%):</span><span>$${totales.value.iva.toLocaleString()}</span></div>
            ${propinaHtml}
            <div class="total"><span>TOTAL:</span><span>$${totales.value.total.toLocaleString()}</span></div>
          </div>

          ${notasHtml}
          
          <p style="text-align:center; margin-top:20px; font-size:0.8em; font-weight:800;">¡GRACIAS POR TU COMPRA!</p>
        </div>
      </body>
    </html>
  `);
  win.document.close();

  setTimeout(() => {
    win.print();
    win.close();

    Swal.fire({
      title: '¡ÉXITO!',
      text: 'Pedido procesado correctamente',
      icon: 'success',
      confirmButtonColor: '#FF4D4D'
    }).then(() => {
      carrito.value = [];
      cliente.value = { nombre: '', mesa: '', metodoPago: '', notas: '' };
      propinaPorcentaje.value = 0;
      if (typeof actualizarTotales === 'function') actualizarTotales();
      if (typeof cerrarModal === 'function') cerrarModal();
    });
  }, 500);
}
</script>