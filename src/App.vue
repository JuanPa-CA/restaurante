<template>
  <div id="app">
    <!-- Header -->
    <header>
      <h1>RESTAURANTE BRISTO</h1>
      <div class="header-actions">
        <button class="btn-agregar-plus" v-on:click="abrirFormPlato()">+</button>
        <button class="btn-pedido" v-on:click="abrirModal">
          <span>{{ carrito.length > 0 ? 'Mi Carrito ' : 'CARRITO' }}</span>
          <span class="badge-carrito">{{ totalUnidades }}</span>
        </button>
      </div>
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

    <!--Barra de busqueda-->
    <div class="contenedor-busqueda">
      <div class="busqueda-wrapper">
        <span class="icon-busqueda">🔍</span>
        <input type="text" placeholder="¿Que plato buscas?" v-model="textoBusqueda" v-on:input="aplicarFiltros"
          class="input-busqueda">
      </div>
    </div>

    <!-- Grid de Productos -->
    <main>
      <div class="grid-productos">
        <div v-for="producto in platosFiltrados" class="card" :key="producto.nombre">

          <div class="card-options">
            <button class="btn-dots" v-on:click.stop="toggleMenu(producto.nombre)">...</button>
            <div v-if="menuAbierto === producto.nombre" class="dropdown-menu">
              <button v-on:click="abrirFormPlato(producto)">Editar</button>
              <button class="btn-eliminar" v-on:click="eliminarPlato(producto.nombre)">Eliminar</button>
            </div>
          </div>

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
            <button v-else class="btn-pedido btn-full btn-desactivado" v-on:click="agregarAlCarrito(producto)">
              SIN STOCK
            </button>
          </div>
        </div>
      </div>
    </main>

    <!-- Modal Formulario Producto -->
    <div v-if="mostrarFormPlato" class="modal-overlay" v-on:click.self="cerrarFormPlato">
      <div class="factura-box formulario-producto">
        <div class="factura-header">
          <h2>{{ editandoPlatoNombre ? 'EDITAR' : 'NUEVO' }} PLATO</h2>
          <button v-on:click="cerrarFormPlato" class="btn-cerrar-x">x</button>
        </div>
        <div class="factura-body">
          <div class="form-datos">
            <div class="form-field">
              <label>NOMBRE DEL PLATO</label>
              <input type="text" v-model="formPlato.nombre" placeholder="Ej: Pizza Hawayana">
            </div>
            <div class="form-field">
              <label>DESCRIPCIÓN</label>
              <textarea v-model="formPlato.descripcion" rows="2" placeholder="Ingredientes y detalles..."></textarea>
            </div>
            <div class="form-grid-2">
              <div class="form-field">
                <label>PRECIO ($)</label>
                <input type="number" v-model="formPlato.precio">
              </div>
              <div class="form-field">
                <label>STOCK</label>
                <input type="number" v-model="formPlato.stock">
              </div>
            </div>
            <div class="form-field">
              <label>CATEGORÍA</label>
              <select v-model="formPlato.categoria">
                <option v-for="categoria in categorias.filter(c => c !== 'Todas')" :value="categoria">{{ categoria }}</option>
              </select>
            </div>
            <div class="form-field">
              <label>IMAGEN DEL PLATO</label>
              <div class="upload-wrapper">
                <input type="file" v-on:change="manejarArchivo" accept="image/*" class="file-input">
                <div v-if="formPlato.imagen" class="preview-mini">
                  <img :src="formPlato.imagen" alt="Vista previa">
                </div>
              </div>
            </div>
            <div class="botones-pago">
              <button class="btn-pedido btn-gris" v-on:click="cerrarFormPlato">CANCELAR</button>
              <button class="btn-pedido" v-on:click="guardarPlato">GUARDAR PRODUCTO</button>
            </div>
          </div>
        </div>
      </div>
    </div>


    <!-- Modal Factura -->
    <div v-if="mostrarModal" class="modal-overlay" v-on:click="cerrarSiEsFuera">
      <div class="factura-box">
        <div class="factura-header">
          <h2>SU CUENTA</h2>
          <p>Restaurante Bistro - Mesa #{{ cliente.mesa || ' ' }}</p>
          <button v-on:click="cerrarModal" class="btn-cerrar-x">x</button>
        </div>

        <div class="factura-body">

          <!-- Carrito vacio -->
          <div v-if="carrito.length === 0" class="factura-vacia">
            <p class="vacia-sub">¡Tu carrito está vacío!</p>
            <p class="vacia-sub">Parece que aún no has elegido nada.</p>
            <button class="btn-pedido btn-oscuro" v-on:click="cerrarModal">EXPLORAR MENÚ</button>
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
              <input type="text" v-model="cliente.nombre" placeholder="Juan Perez">
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
              <textarea v-model="cliente.notas" rows="2" placeholder="Sin cebolla, termino medio"></textarea>
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
      <h2 class="footer-logo">RESTAURANTE BISTRO</h2>
      <p class="footer-info">Calle de los Sabores #123 - Abierto de 8am a 7pm</p>
    </footer>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import Swal from 'sweetalert2';


// ========================================
// 1. ESTADO REACTIVO
// ========================================
const PLATOSINICIO = [
  { nombre: "Hamburguesa Especial", descripcion: "Carne angus 200g, queso cheddar, tocino y vegetales frescos.", precio: 25000, imagen: "src/assets/hamburguesaE.jpg", categoria: "Rapidas", stock: 8 },
  { nombre: "Club Sandwich", descripcion: "Triple piso con pollo, jamon, huevo, queso y tocino.", precio: 21000, imagen: "src/assets/clubS.jpg", categoria: "Rapidas", stock: 12 },
  { nombre: "Hot-dog Picante", descripcion: "Salchicha premium, jalapeños, cebolla caramelizada y salsa brava.", precio: 18000, imagen: "src/assets/hotdogP.jpg", categoria: "Rapidas", stock: 15 },
  { nombre: "Nuggets de Pollo", descripcion: "8 piezas de pechuga apanada con papas fritas pequeñas.", precio: 16000, imagen: "src/assets/nuggetsP.jpg", categoria: "Snacks", stock: 20 },
  { nombre: "Salchipapa Monstruosa", descripcion: "Cama de papas fritas, salchicha suiza, pollo desmechado y mucho queso.", precio: 32000, imagen: "src/assets/salchipapaM.jpg", categoria: "Snacks", stock: 6 },
  { nombre: "Empanadas (x3)", descripcion: "Tradicionales de carne y papa, acompañadas de aji casero.", precio: 12000, imagen: "src/assets/empanadasT.jpg", categoria: "Snacks", stock: 30 },
  { nombre: "Nachos Locos", descripcion: "Totopos crujientes con chili, queso fundido y pico de gallo.", precio: 19000, imagen: "src/assets/nachosL.jpg", categoria: "Snacks", stock: 10 },
  { nombre: "Papas Bravas", descripcion: "Cubos de papa frita con salsa picante y alioli.", precio: 14000, imagen: "src/assets/papasB.jpg", categoria: "Snacks", stock: 18 },
  { nombre: "Alitas BBQ (x10)", descripcion: "Alitas crujientes bañadas en salsa a eleccion.", precio: 29000, imagen: "src/assets/alitasB.jpg", categoria: "Snacks", stock: 12 },
  { nombre: "Tacos al Pastor", descripcion: "3 tacos de cerdo marinado con piña, cilantro y cebolla.", precio: 20000, imagen: "src/assets/tacosP.jpg", categoria: "Tacos y Wraps", stock: 25 },
  { nombre: "Burrito Supremo", descripcion: "Tortilla gigante de harina rellena de carne, frijol y guacamole.", precio: 24000, imagen: "src/assets/burritoP.jpg", categoria: "Tacos y Wraps", stock: 10 },
  { nombre: "Wrap de Vegetales", descripcion: "Tortilla integral, hummus, vegetales asados y queso feta.", precio: 17000, imagen: "src/assets/wrapV.jpg", categoria: "Tacos y Wraps", stock: 14 },
  { nombre: "Quesadilla de Pollo", descripcion: "Tortilla de trigo con queso fundido y pollo desmechado.", precio: 18500, imagen: "src/assets/quesadillas.jpg", categoria: "Tacos y Wraps", stock: 12 },
  { nombre: "Arepa con Todo", descripcion: "Arepa de maiz rellena de carne, pollo, queso y huevo de codorniz.", precio: 15500, imagen: "src/assets/arepaT.jpg", categoria: "Tacos y Wraps", stock: 20 },
  { nombre: "Costillas BBQ", descripcion: "Costilas de cerdo bañadas en salsa BBQ ahumada con papas.", precio: 45000, imagen: "src/assets/costillasB.jpg", categoria: "Platos Fuertes", stock: 8 },
  { nombre: "Churrasco 300g", descripcion: "Corte de res a la parrilla con chimichurri artesanal.", precio: 42000, imagen: "src/assets/churrasco.jpg", categoria: "Platos Fuertes", stock: 7 },
  { nombre: "Parrillada Mixta", descripcion: "Res, pollo, cerdo, chorizo y acompanamientos.", precio: 55000, imagen: "src/assets/parrilladaM.jpg", categoria: "Platos Fuertes", stock: 6 },
  { nombre: "Arroz Atollado", descripcion: "Arroz humedo tipico con pollo, cerdo y longaniza.", precio: 28000, imagen: "src/assets/arrozA.jpg", categoria: "Platos Fuertes", stock: 10 },
  { nombre: "Ceviche de Camaron", descripcion: "Camarones frescos marinados en limon citrico y cebolla roja.", precio: 35000, imagen: "src/assets/cevicheC.jpg", categoria: "Platos Fuertes", stock: 15 },
  { nombre: "Sushi Roll Filadelfia", descripcion: "Salmon, queso crema y aguacate, envuelto en sesamo.", precio: 30000, imagen: "src/assets/sushiR.jpg", categoria: "Platos Fuertes", stock: 20 },
  { nombre: "Poke de Atun", descripcion: "Atun marinado, mango, rabano y arroz de sushi.", precio: 31000, imagen: "src/assets/pokeA.jpg", categoria: "Platos Fuertes", stock: 12 },
  { nombre: "Bowl de Salmon", descripcion: "Base de quinoa, salmon fresco, edamame y aderezo ginger.", precio: 38000, imagen: "src/assets/bowlS.jpg", categoria: "Platos Fuertes", stock: 10 },
  { nombre: "Filete de Pescado", descripcion: "Pescado blanco al ajillo con ensalada verde.", precio: 29000, imagen: "src/assets/fileteP.jpg", categoria: "Platos Fuertes", stock: 9 },
  { nombre: "Pizza Pepperoni", descripcion: "Masa artesanal, salsa de tomate italiana y doble pepperoni.", precio: 28000, imagen: "src/assets/pizzaP.jpg", categoria: "Platos Fuertes", stock: 15 },
  { nombre: "Pasta Carbonara", descripcion: "Espagueti con salsa de huevo, queso pecorino y guanciale.", precio: 27000, imagen: "src/assets/pastaC.jpg", categoria: "Platos Fuertes", stock: 12 },
  { nombre: "Lasana de Carne", descripcion: "Capas de pasta con bolonesa de la casa y salsa bechamel.", precio: 26000, imagen: "src/assets/lasañaC.jpg", categoria: "Platos Fuertes", stock: 10 },
  { nombre: "Ramen Tonkotsu", descripcion: "Caldo de cerdo, fideos frescos, huevo marinado y cerdo chashu.", precio: 33000, imagen: "src/assets/ramenT.jpg", categoria: "Platos Fuertes", stock: 8 },
  { nombre: "Sopa de Tortilla", descripcion: "Caldo de tomate con tiras de tortilla, aguacate y crema.", precio: 15000, imagen: "src/assets/sopaT.jpg", categoria: "Platos Fuertes", stock: 20 },
  { nombre: "Ensalada de Pollo", descripcion: "Mix de lechugas, pechuga a la plancha, croutons y aderezo cesar.", precio: 22000, imagen: "src/assets/ensaladaP.jpg", categoria: "Platos Fuertes", stock: 15 },
  { nombre: "Brownie con Helado", descripcion: "Postre de chocolate melcochudo con helado de vainilla.", precio: 12500, imagen: "src/assets/brownieH.jpg", categoria: "Postres", stock: 25 },
  { nombre: "Limonada Cerezada", descripcion: "Refrescante mezcla de limon y cerezas maceradas.", precio: 12000, imagen: "src/assets/limonadaC.jpg", categoria: "Bebidas", stock: 50 },
  { nombre: "Jugo de Mango", descripcion: "Jugo natural de mango maduro, 100% fruta.", precio: 9500, imagen: "src/assets/jugoM.jpg", categoria: "Bebidas", stock: 40 },
  { nombre: "Soda de Frutos Rojos", descripcion: "Soda artesanal con infusion de fresa, mora y arandanos.", precio: 13900, imagen: "src/assets/sodaF.jpg", categoria: "Bebidas", stock: 35 },
  { nombre: "Malteada de Vainilla", descripcion: "Cremosa malteada con crema batida y chispas.", precio: 16500, imagen: "src/assets/malteadaV.jpg", categoria: "Bebidas", stock: 20 },
  { nombre: "Vino Tinto (Copa)", descripcion: "Seleccion de la casa, ideal para carnes rojas.", precio: 18000, imagen: "src/assets/copaV.jpg", categoria: "Bebidas", stock: 15 },
  { nombre: "Cheesecake de Oreo", descripcion: "Postre cremoso de queso con trozos de galleta oreo.", precio: 15000, imagen: "src/assets/oreoC.jpg", categoria: "Postres", stock: 12 },
  { nombre: "Tarta de Manzana", descripcion: "Tarta horneada con canela y helado de vainilla.", precio: 14000, imagen: "src/assets/tartaM.jpg", categoria: "Postres", stock: 10 },
  { nombre: "Salmon Teriyaki", descripcion: "Salmon glaseado en salsa dulce con vegetales salteados.", precio: 49000, imagen: "src/assets/salmonT.jpg", categoria: "Platos Fuertes", stock: 15 },
  { nombre: "Lomo a la Pimienta", descripcion: "Corte premium de res con salsa cremosa de pimienta.", precio: 54000, imagen: "src/assets/lomoP.jpg", categoria: "Platos Fuertes", stock: 10 },
  { nombre: "Risotto de Hongos", descripcion: "Arroz cremoso con variedad de setas y queso parmesano.", precio: 39500, imagen: "src/assets/risottoH.jpg", categoria: "Platos Fuertes", stock: 7 },
];

const platosGuardados = localStorage.getItem('platosbristo');
const platos = ref(platosGuardados ? JSON.parse(platosGuardados) : PLATOSINICIO);

const categorias = ref(["Todas", "Rapidas", "Snacks", "Tacos y Wraps", "Platos Fuertes", "Bebidas", "Postres"]);
const categoriaSeleccionada = ref("Todas");
const platosFiltrados = ref([...platos.value]);
const carrito = ref([]);
const mostrarModal = ref(false);
const totalUnidades = ref(0);
const propinaPorcentaje = ref(0);
const cliente = ref({ nombre: '', mesa: '', metodoPago: '', notas: '' });
const totales = ref({ subtotal: 0, iva: 0, propina: 0, total: 0 });
const textoBusqueda = ref("");

// --- NUEVO ESTADO PARA PRODUCTOS ---
const mostrarFormPlato = ref(false);
const editandoPlatoNombre = ref(null);
const menuAbierto = ref(null);
const formPlato = ref({
  nombre: '',
  descripcion: '',
  precio: 0,
  imagen: '',
  categoria: 'Rapidas',
  stock: 0
});

function abrirFormPlato(plato = null) {
  if (plato) {
    editandoPlatoNombre.value = plato.nombre;
    formPlato.value = { ...plato };
  } else {
    editandoPlatoNombre.value = null;
    formPlato.value = {
      nombre: '',
      descripcion: '',
      precio: 0,
      imagen: '',
      categoria: 'Rapidas',
      stock: 0
    };
  }
  mostrarFormPlato.value = true;
  menuAbierto.value = null;
}

function cerrarFormPlato() {
  mostrarFormPlato.value = false;
}

function manejarArchivo(evento) {
  const archivo = evento.target.files[0];
  if (archivo) {
    const reader = new FileReader();
    reader.onload = (evento) => {
      formPlato.value.imagen = evento.target.result;
    };
    reader.readAsDataURL(archivo);
  }
}

function toggleMenu(nombre) {
  menuAbierto.value = menuAbierto.value === nombre ? null : nombre;
}

function platosStorage() {
  const platosGuardar = platos.value.map(p => {
    const itemEnCarrito = carrito.value.find(c => c.nombre === p.nombre);
    return {
      ...p,
      stock: p.stock + (itemEnCarrito ? itemEnCarrito.cantidad : 0)
    };
  });
  localStorage.setItem('platosbristo', JSON.stringify(platosGuardar));
}

const datosGuardados = localStorage.getItem('carrito_bistro');
if (datosGuardados) {
  const guardado = JSON.parse(datosGuardados);
  carrito.value = guardado;
  guardado.forEach(itemCarrito => {
    const plato = platos.value.find(producto => producto.nombre === itemCarrito.nombre);
    if (plato) {
      plato.stock -= itemCarrito.cantidad;
    }
  });
}

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
  localStorage.setItem('carrito_bistro', JSON.stringify(carrito.value));
}

actualizarTotales();

function aplicarFiltros() {
  const prefijo = textoBusqueda.value.toLowerCase().trim();
  platosFiltrados.value = platos.value.filter(plato => {
    const coincideCategoria = categoriaSeleccionada.value === "Todas" || plato.categoria === categoriaSeleccionada.value;
    const coincideNombre = plato.nombre.toLowerCase().startsWith(prefijo);
    return coincideCategoria && coincideNombre;
  });
}

function filtrarPlatos(categoria) {
  categoriaSeleccionada.value = categoria;
  aplicarFiltros();
}

function cantidadEnCarrito(nombre) {
  for (let i = 0; i < carrito.value.length; i++) {
    if (carrito.value[i].nombre === nombre) return carrito.value[i].cantidad;
  }
  return 0;
}

function guardarPlato() {
  const { nombre, descripcion, precio, imagen, categoria, stock } = formPlato.value;
  if (!nombre || !descripcion || !precio || !imagen || !categoria) {
    return Swal.fire('Error', 'Todos los campos son obligatorios', 'error');
  }
  if (editandoPlatoNombre.value) {
    const index = platos.value.findIndex(p => p.nombre === editandoPlatoNombre.value);
    if (index !== -1) platos.value[index] = { ...formPlato.value };
  } else {
    if (platos.value.find(producto => producto.nombre === nombre)) return Swal.fire('Error', 'Ya existe un plato con ese nombre', 'error');
    platos.value.push({ ...formPlato.value });
  }
  cerrarFormPlato();
  platosStorage();
  aplicarFiltros();
  Swal.fire('¡Éxito!', editandoPlatoNombre.value ? 'Producto actualizado' : 'Producto agregado', 'success');
}

function eliminarPlato(nombre) {
  Swal.fire({
    title: '¿Eliminar producto?',
    text: "Esta acción no se puede deshacer",
    icon: 'warning',
    showCancelButton: true,
    confirmButtonColor: '#FF4D4D',
    cancelButtonColor: '#011627',
    confirmButtonText: 'Sí, eliminar'
  }).then((result) => {
    if (result.isConfirmed) {
      platos.value = platos.value.filter(producto => producto.nombre !== nombre);
      platosStorage();
      aplicarFiltros();
      Swal.fire('Eliminado', 'El producto ha sido eliminado', 'success');
    }
  });
}

function agregarAlCarrito(producto) {
  if (producto.stock <= 0) return Swal.fire({ icon: 'error', title: 'Sin stock', text: 'Lo sentimos, este producto se ha agotado.' });
  let item = null;
  for (let i = 0; i < carrito.value.length; i++) {
    if (carrito.value[i].nombre === producto.nombre) {
      item = carrito.value[i];
      break;
    }
  }
  if (item) {
    if (item.cantidad >= 3) return Swal.fire({ icon: 'warning', title: 'Límite alcanzado', text: 'Solo se permiten máximo 3 unidades de cada producto por cliente.', confirmButtonColor: '#E76F51' });
    item.cantidad++;
  } else {
    carrito.value.push({ nombre: producto.nombre, precio: producto.precio, cantidad: 1, categoria: producto.categoria });
  }
  producto.stock--;
  Swal.fire({ toast: true, position: 'top-end', icon: 'success', title: '¡Agregado!', showConfirmButton: false, timer: 1000, background: '#2A9D8F', color: '#fff' });
  actualizarTotales();
}

function cambiarCantidad(index, valor) {
  const itemCarrito = carrito.value[index];
  let productoOriginal = platos.value.find(producto => producto.nombre === itemCarrito.nombre);
  if (valor > 0) {
    if (itemCarrito.cantidad >= 3) return Swal.fire({ icon: 'warning', title: 'Límite alcanzado', text: 'Solo se permiten máximo 3 unidades de cada producto por cliente.', confirmButtonColor: '#E76F51' });
    if (productoOriginal.stock <= 0) return Swal.fire({ icon: 'error', title: 'Sin stock', text: 'No hay más unidades disponibles.' });
    itemCarrito.cantidad++;
    productoOriginal.stock--;
  } else {
    itemCarrito.cantidad--;
    productoOriginal.stock++;
    if (itemCarrito.cantidad <= 0) carrito.value.splice(index, 1);
  }
  actualizarTotales();
}

function vaciarCarrito() {
  Swal.fire({ icon: 'warning', title: '¿Vaciar carrito?', text: 'Perderás el progreso de la compra', showCancelButton: true, confirmButtonText: 'Vaciar carrito', confirmButtonColor: '#E76F51' }).then(function (resultado) {
    if (resultado.isConfirmed) {
      for (let i = 0; i < carrito.value.length; i++) {
        const itemCarrito = carrito.value[i];
        const platoOriginal = platos.value.find(p => p.nombre === itemCarrito.nombre);
        if (platoOriginal) platoOriginal.stock += itemCarrito.cantidad;
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

function abrirModal() { mostrarModal.value = true; }
function cerrarModal() { mostrarModal.value = false; }
function cerrarSiEsFuera(evento) { if (evento.target.classList.contains('modal-overlay')) cerrarModal(); }

function finalizarPedido() {
  const { nombre, mesa, metodoPago, notas } = cliente.value;
  if (!carrito.value.length) return Swal.fire('Error', 'Carrito vacío', 'error');
  if (!nombre || !mesa || !metodoPago) return Swal.fire('Atención', 'Completa tus datos', 'info');

  Swal.fire({
    title: 'PROCESANDO PAGO',
    html: 'Procesando',
    allowOutsideClick: false,
    didOpen: () => { Swal.showLoading(); },
    timer: 2000,
    timerProgressBar: true,
  }).then(() => {
    const filas = carrito.value.map(i => `
      <tr>
        <td>${i.nombre} x${i.cantidad}</td>
        <td style="text-align:right;">$${(i.precio * i.cantidad).toLocaleString()}</td>
      </tr>`).join('');

    const propinaHtml = propinaPorcentaje.value > 0 ? `<div class="ticket-linea-total"><span>Propina (${propinaPorcentaje.value}%):</span><span>$${totales.value.propina.toLocaleString()}</span></div>` : '';
    const notasHtml = notas ? `<div class="ticket-notas"><strong>NOTAS:</strong> ${notas}</div>` : '';

    const ticketHtml = `
      <div id="ticket-impresion" class="ticket-container">
        <h1 class="ticket-header">RESTAURANTE BRISTO</h1>
        <div class="ticket-info-cliente">
          <p><strong>MESA:</strong> ${mesa} | <strong>CLIENTE:</strong> ${nombre}</p>
          <span class="ticket-metodo-pago">PAGO: ${metodoPago}</span>
        </div>
        <table class="ticket-table">${filas}</table>
        <div class="ticket-totales">
          <div class="ticket-linea-total"><span>Subtotal:</span><span>$${totales.value.subtotal.toLocaleString()}</span></div>
          <div class="ticket-linea-total"><span>IVA (19%):</span><span>$${totales.value.iva.toLocaleString()}</span></div>
          ${propinaHtml}
          <div class="ticket-total-grande"><span>TOTAL:</span><span>$${totales.value.total.toLocaleString()}</span></div>
        </div>
        ${notasHtml}
        <p class="ticket-footer">¡GRACIAS POR SU COMPRA!</p>
      </div>
    `;

    const iframe = document.createElement('iframe');
    iframe.style.position = 'fixed';
    iframe.style.right = '0';
    iframe.style.bottom = '0';
    iframe.style.width = '0';
    iframe.style.height = '0';
    iframe.style.border = '0';
    document.body.appendChild(iframe);
    const doc = iframe.contentWindow.document;
    doc.open();
    doc.write(`<html><head><link rel="stylesheet" href="src/style.css"></head><body>${ticketHtml}</body></html>`);
    doc.close();

    setTimeout(() => {
      iframe.contentWindow.focus();
      iframe.contentWindow.print();
      document.body.removeChild(iframe);
      Swal.fire({ title: '¡ÉXITO!', text: 'Pedido procesado correctamente', icon: 'success', confirmButtonColor: '#FF4D4D' }).then(() => {
        carrito.value = [];
        cliente.value = { nombre: '', mesa: '', metodoPago: '', notas: '' };
        propinaPorcentaje.value = 0;
        localStorage.setItem('platosbristo', JSON.stringify(platos.value));
        actualizarTotales();
        cerrarModal();
      });
    }, 500);
  });
}
</script>
