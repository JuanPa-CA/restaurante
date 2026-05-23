<template>
  <div id="app">
    <!-- Header -->
    <header>
      <h1>RESTAURANTE BRISTO</h1>
      <div class="header-actions">
        <button class="btn-pedido" v-on:click="abrirModal">
          <span>{{ carrito.length > 0 ? 'MI CARRITO ' : 'CARRITO' }}</span>
          <span class="badge-carrito">{{ totalUnidades }}</span>
        </button>
      </div>
    </header>

    <!-- Barra de Categorias -->
    <div class="carrusel-categorias">
      <button class="btn-flecha flecha-izq" v-on:click="scrollCategorias(-1)">‹</button>
      
      <nav class="barra-categorias" ref="navCategorias">
        <template v-for="categoria in categorias">
          <button v-if="categoriaSeleccionada === categoria" class="btn-categoria active"
            v-on:click="filtrarPlatos(categoria)">
            {{ categoria }}
          </button>
          <button v-else class="btn-categoria" v-on:click="filtrarPlatos(categoria)">
            {{ categoria }}
          </button>
        </template>
        <button class="btn-agregar-cat" v-on:click="abrirModalCategorias">Agregar Nueva</button>
      </nav>

      <button class="btn-flecha flecha-der" v-on:click="scrollCategorias(1)">›</button>
    </div>

    <div>
      <button class="btn-agregar-plus" v-on:click="abrirFormPlato()">+</button>
    </div>

    <!--Barra de busqueda-->
    <div class="contenedor-busqueda">
      <div class="busqueda-wrapper">
        <span class="icon-busqueda">🔍</span>
        <input type="text" placeholder="¿Que estás buscando?" v-model="textoBusqueda" v-on:input="aplicarFiltros"
          class="input-busqueda">
      </div>
    </div>

    <!-- Grid de Productos -->
    <main>
      <div class="grid-productos">
        <div v-for="producto in platosFiltrados" class="card" :key="producto.nombre">

          <div class="card-options" v-on:click.stop>
            <button class="btn-dots" v-on:click.stop="toggleMenu(producto.nombre)">⋮</button>
            <div v-if="menuAbierto === producto.nombre" class="dropdown-menu">
              <button v-on:click="abrirFormPlato(producto)">EDITAR PLATO</button>
              <button class="btn-eliminar" v-on:click="eliminarPlato(producto.nombre)">ELIMINAR PLATO</button>
            </div>
          </div>

          <div class="card-en-pedido" v-if="cantidadEnCarrito(producto.nombre) > 0">
            {{ cantidadEnCarrito(producto.nombre) }} en pedido
          </div>

          <div class="card-img-wrapper">
  <img :src="producto.imagen">
  <span class="card-categoria-badge">{{ producto.categoria }}</span>
</div>
          <div class="card-content">
            <div class="card-title">{{ producto.nombre }}</div>
            <div class="card-price-tag">${{ (producto.precio || 0).toLocaleString() }}</div>

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
              <input type="text" v-model="formPlato.nombre" placeholder="Pizza Hawayana">
            </div>
            <div class="form-field">
              <label>DESCRIPCIÓN</label>
              <textarea v-model="formPlato.descripcion" rows="2" placeholder="Ingredientes y detalles"></textarea>
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
          <p>Restaurante Bistro - Mesa #{{ cliente.mesa.toString().slice(0, 2) || ' ' }}</p>
          <button v-on:click="cerrarModal" class="btn-cerrar-x">x</button>
        </div>

        <div class="factura-body">

          <!-- Carrito vacio -->
          <div v-if="carrito.length === 0" class="factura-vacia">
            <p class="vacia-sub">¡Tu carrito está vacío!</p>
            <p class="vacia-sub">Parece que aún no has elegido nada.</p>
            <button class="btn-pedido btn-oscuro" v-on:click="cerrarModal">EXPLORAR</button>
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
              <div class="item-precio-unit">${{ (item.precio || 0).toLocaleString() }} c/u</div>
            </div>
            <div class="item-qty-selector">
              <button class="qty-btn" v-on:click="cambiarCantidad(index, -1)">-</button>
              <span class="item-nombre">{{ item.cantidad }}</span>
              <button class="qty-btn" v-on:click="cambiarCantidad(index, 1)">+</button>
            </div>
            <div class="item-total-line">
              ${{ ((item.precio || 0) * item.cantidad).toLocaleString() }}
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
                <input type="number" v-model="cliente.mesa" placeholder="01" min="1" max="99" oninput="if(this.value.length > 2) this.value = this.value.slice(0, 2);">
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
                <button class="btn-pedido" :disabled="procesandoPedido" :class="{ 'btn-desactivado': procesandoPedido }" v-on:click="finalizarPedido">
                  {{ procesandoPedido ? 'GENERANDO...' : 'PAGAR AHORA' }}
                </button>
              </div>
            </div>

          </div>
        </div>
      </div>
    </div>

    <!-- Modal Gestión de Categorías -->
    <div v-if="mostrarModalCategorias" class="modal-overlay" v-on:click.self="cerrarModalCategorias">
      <div class="factura-box formulario-producto">
        <div class="factura-header">
          <h2>CATEGORÍAS</h2>
          <button v-on:click="cerrarModalCategorias" class="btn-cerrar-x">x</button>
        </div>
        <div class="factura-body">
          <div class="form-datos">

            <!-- Input para nueva categoría -->
            <div class="cat-input-row">
              <input
                type="text"
                v-model="nuevaCategoriaTexto"
                placeholder="Nombre de nueva categoría"
                class="cat-input"
                v-on:keydown.enter="agregarCategoria"
                maxlength="12"
              >
              <button class="btn-pedido btn-cat-add" v-on:click="agregarCategoria">+ AGREGAR</button>
            </div>

            <!-- Categorías personalizadas (editables/eliminables) -->
            <div v-if="categoriasPersonalizadas.length > 0" class="cat-seccion">
              <label class="cat-seccion-label">MIS CATEGORÍAS</label>
              <div class="cat-lista">
                <div v-for="cat in categoriasPersonalizadas" class="cat-item cat-item-custom" :key="cat">
                  <div v-if="categoriaEditando === cat" class="cat-edit-row">
                    <input
                      type="text"
                      v-model="categoriaEditandoTexto"
                      class="cat-input cat-input-sm"
                      maxlength="12"
                      v-on:keydown.enter="confirmarEdicionCategoria(cat)"
                    >
                    <button class="btn-cat-action btn-cat-ok" v-on:click="confirmarEdicionCategoria(cat)">✓</button>
                    <button class="btn-cat-action btn-cat-cancel" v-on:click="cancelarEdicionCategoria">✕</button>
                  </div>
                  <div v-else class="cat-item-contenido">
                    <span class="cat-nombre">{{ cat }}</span>
                    <div class="cat-acciones">
                      <button class="btn-cat-action btn-cat-edit" v-on:click="iniciarEdicionCategoria(cat)" title="Editar">✏️</button>
                      <button class="btn-cat-action btn-cat-del" v-on:click="eliminarCategoria(cat)" title="Eliminar">🗑️</button>
                    </div>
                  </div>
                </div>
              </div>
            </div>

            <!-- Categorías fijas -->
            <div class="cat-seccion">
              <label class="cat-seccion-label">CATEGORÍAS FIJAS</label>
              <div class="cat-lista">
                <div v-for="cat in categoriasFijas" class="cat-item cat-item-fija" :key="cat">
                  <span class="cat-nombre">{{ cat }}</span>
                  <span class="cat-badge-fija">FIJA</span>
                </div>
              </div>
            </div>

          </div>
        </div>
      </div>
    </div>

    <!-- Footer -->
    <footer>
      <h2 class="footer-logo">RESTAURANTE BISTRO</h2>
      <p class="footer-info">Calle de los Sabores #123 - Abierto de 8 am a 7 pm</p>
    </footer>
  </div>

  <Teleport to="body">
    <div v-if="ticketParaImprimir" class="ticket-print-root" :class="{ 'ticket-print-root--active': imprimiendoTicket }" aria-hidden="true">
      <div class="ticket-print-page">
        <h1 class="ticket-print-header">RESTAURANTE BRISTO</h1>
        <div class="ticket-print-meta">
          <p><strong>MESA:</strong> {{ ticketParaImprimir.mesa }} | <strong>CLIENTE:</strong> {{ ticketParaImprimir.cliente }}</p>
          <span class="ticket-print-payment">PAGO: {{ ticketParaImprimir.metodoPago }}</span>
        </div>

        <div class="ticket-print-items">
          <div v-for="item in ticketParaImprimir.items" :key="`${item.nombre}-${item.cantidad}`" class="ticket-print-item">
            <span>{{ item.nombre }} x{{ item.cantidad }}</span>
            <span>${{ item.total.toLocaleString() }}</span>
          </div>
        </div>

        <div class="ticket-print-totals">
          <div class="ticket-print-total-line">
            <span>Subtotal:</span>
            <span>${{ ticketParaImprimir.subtotal.toLocaleString() }}</span>
          </div>
          <div class="ticket-print-total-line">
            <span>IVA (19%):</span>
            <span>${{ ticketParaImprimir.iva.toLocaleString() }}</span>
          </div>
          <div v-if="ticketParaImprimir.propinaPorcentaje > 0" class="ticket-print-total-line">
            <span>Propina ({{ ticketParaImprimir.propinaPorcentaje }}%):</span>
            <span>${{ ticketParaImprimir.propina.toLocaleString() }}</span>
          </div>
          <div class="ticket-print-grand-total">
            <span>TOTAL:</span>
            <span>${{ ticketParaImprimir.total.toLocaleString() }}</span>
          </div>
        </div>

        <div v-if="ticketParaImprimir.notas" class="ticket-print-notes">
          <strong>NOTAS:</strong> {{ ticketParaImprimir.notas }}
        </div>

        <p class="ticket-print-footer">¡GRACIAS POR SU COMPRA!</p>
      </div>
    </div>
  </Teleport>
</template>

<script setup>
import { nextTick, ref } from 'vue';
import Swal from 'sweetalert2';

// ========================================
// 1. CONSTANTES Y DATOS INICIALES
// ========================================

//array de categorias
const CATEGORIAS_FIJAS = ["Todas", "Rapidas", "Snacks", "Tacos y Wraps", "Platos Fuertes", "Bebidas", "Postres"];

//array de platos
const PLATOSINICIO = [
  { nombre: "Hamburguesa Especial", descripcion: "Carne angus 200g, queso cheddar, tocino y vegetales frescos.", precio: 25000, imagen: "hamburguesaE.jpg", categoria: "Rapidas", stock: 8 },
  { nombre: "Club Sandwich", descripcion: "Triple piso con pollo, jamon, huevo, queso y tocino.", precio: 21000, imagen: "clubS.jpg", categoria: "Rapidas", stock: 12 },
  { nombre: "Hot-dog Picante", descripcion: "Salchicha premium, jalapeños, cebolla caramelizada y salsa brava.", precio: 18000, imagen: "hotdogP.jpg", categoria: "Rapidas", stock: 15 },
  { nombre: "Nuggets de Pollo", descripcion: "12 piezas de pechuga apanada con papas fritas pequeñas.", precio: 16000, imagen: "nuggetsP.jpg", categoria: "Snacks", stock: 20 },
  { nombre: "Salchipapa Monstruosa", descripcion: "Cama de papas fritas, salchicha suiza, pollo desmechado y queso.", precio: 32000, imagen: "salchipapaM.jpg", categoria: "Rapidas", stock: 6 },
  { nombre: "Empanadas (x3)", descripcion: "Tradicionales de carne y papa, acompañadas de aji casero.", precio: 12000, imagen: "empanadasT.jpg", categoria: "Snacks", stock: 30 },
  { nombre: "Nachos Locos", descripcion: "Totopos crujientes con chili, queso fundido y pico de gallo.", precio: 19000, imagen: "nachosL.jpg", categoria: "Snacks", stock: 10 },
  { nombre: "Papas Bravas", descripcion: "Cubos de papa frita con salsa picante y alioli.", precio: 14000, imagen: "papasB.jpg", categoria: "Snacks", stock: 18 },
  { nombre: "Alitas BBQ (x10)", descripcion: "Alitas crujientes bañadas en salsa a eleccion.", precio: 29000, imagen: "alitasB.jpg", categoria: "Snacks", stock: 12 },
  { nombre: "Tacos al Pastor", descripcion: "3 tacos de cerdo marinado con piña, cilantro y cebolla.", precio: 20000, imagen: "tacosP.jpg", categoria: "Tacos y Wraps", stock: 25 },
  { nombre: "Burrito Supremo", descripcion: "Tortilla gigante de harina rellena de carne, frijol y guacamole.", precio: 24000, imagen: "burritoP.jpg", categoria: "Tacos y Wraps", stock: 10 },
  { nombre: "Wrap de Vegetales", descripcion: "Tortilla integral, hummus, vegetales asados y queso feta.", precio: 17000, imagen: "wrapV.jpg", categoria: "Tacos y Wraps", stock: 14 },
  { nombre: "Quesadilla de Pollo", descripcion: "Tortilla de trigo con queso fundido y pollo desmechado.", precio: 18500, imagen: "quesadillas.jpg", categoria: "Tacos y Wraps", stock: 12 },
  { nombre: "Arepa con Todo", descripcion: "Arepa de maiz rellena de carne, pollo, queso y huevo de codorniz.", precio: 15500, imagen: "arepaT.jpg", categoria: "Tacos y Wraps", stock: 20 },
  { nombre: "Costillas BBQ", descripcion: "Costilas de cerdo bañadas en salsa BBQ ahumada con papas.", precio: 45000, imagen: "costillasB.jpg", categoria: "Platos Fuertes", stock: 8 },
  { nombre: "Churrasco 300g", descripcion: "Corte de res a la parrilla con chimichurri artesanal.", precio: 42000, imagen: "churrasco.jpg", categoria: "Platos Fuertes", stock: 7 },
  { nombre: "Parrillada Mixta", descripcion: "Res, pollo, cerdo, chorizo y acompanamientos.", precio: 55000, imagen: "parrilladaM.jpg", categoria: "Platos Fuertes", stock: 6 },
  { nombre: "Arroz Atollado", descripcion: "Arroz humedo tipico con pollo, cerdo y longaniza.", precio: 28000, imagen: "arrozA.jpg", categoria: "Platos Fuertes", stock: 10 },
  { nombre: "Ceviche de Camaron", descripcion: "Camarones frescos marinados en limon citrico y cebolla roja.", precio: 35000, imagen: "cevicheC.jpg", categoria: "Platos Fuertes", stock: 15 },
  { nombre: "Sushi Roll Filadelfia", descripcion: "Salmon, queso crema y aguacate, envuelto en sesamo.", precio: 30000, imagen: "sushiR.jpg", categoria: "Platos Fuertes", stock: 20 },
  { nombre: "Poke de Atun", descripcion: "Atun marinado, mango, rabano y arroz de sushi.", precio: 31000, imagen: "pokeA.jpg", categoria: "Platos Fuertes", stock: 12 },
  { nombre: "Bowl de Salmon", descripcion: "Base de quinoa, salmon fresco, edamame y aderezo ginger.", precio: 38000, imagen: "bowlS.jpg", categoria: "Platos Fuertes", stock: 10 },
  { nombre: "Filete de Pescado", descripcion: "Pescado blanco al ajillo con ensalada verde.", precio: 29000, imagen: "fileteP.jpg", categoria: "Platos Fuertes", stock: 9 },
  { nombre: "Pizza Pepperoni", descripcion: "Masa artesanal, salsa de tomate italiana y doble pepperoni.", precio: 28000, imagen: "pizzaP.jpg", categoria: "Rapidas", stock: 15 },
  { nombre: "Pasta Carbonara", descripcion: "Espagueti con salsa de huevo, queso pecorino y guanciale.", precio: 27000, imagen: "pastaC.jpg", categoria: "Platos Fuertes", stock: 12 },
  { nombre: "Lasaña de Carne", descripcion: "Capas de pasta con bolonesa de la casa y salsa bechamel.", precio: 26000, imagen: "lasañaC.jpg", categoria: "Platos Fuertes", stock: 10 },
  { nombre: "Ramen Tonkotsu", descripcion: "Caldo de cerdo, fideos frescos, huevo marinado y cerdo chashu.", precio: 33000, imagen: "ramenT.jpg", categoria: "Platos Fuertes", stock: 8 },
  { nombre: "Sopa de Tortilla", descripcion: "Caldo de tomate con tiras de tortilla, aguacate y crema.", precio: 15000, imagen: "sopaT.jpg", categoria: "Platos Fuertes", stock: 20 },
  { nombre: "Ensalada de Pollo", descripcion: "Mix de lechugas, pechuga a la plancha, croutons y aderezo cesar.", precio: 22000, imagen: "ensaladaP.jpg", categoria: "Platos Fuertes", stock: 15 },
  { nombre: "Brownie con Helado", descripcion: "Postre de chocolate melcochudo con helado de vainilla.", precio: 12500, imagen: "brownieH.jpg", categoria: "Postres", stock: 25 },
  { nombre: "Limonada Cerezada", descripcion: "Refrescante mezcla de limon y cerezas maceradas.", precio: 12000, imagen: "limonadaC.jpg", categoria: "Bebidas", stock: 50 },
  { nombre: "Jugo de Mango", descripcion: "Jugo natural de mango maduro, 100% fruta.", precio: 9500, imagen: "jugoM.jpg", categoria: "Bebidas", stock: 40 },
  { nombre: "Soda de Frutos Rojos", descripcion: "Soda artesanal con infusion de fresa, mora y arandanos.", precio: 13900, imagen: "sodaF.jpg", categoria: "Bebidas", stock: 35 },
  { nombre: "Malteada de Vainilla", descripcion: "Cremosa malteada con crema batida y chispas.", precio: 16500, imagen: "malteadaV.jpg", categoria: "Bebidas", stock: 20 },
  { nombre: "Vino Tinto", descripcion: "Seleccion de la casa, ideal para carnes rojas.", precio: 18000, imagen: "copaV.jpg", categoria: "Bebidas", stock: 15 },
  { nombre: "Cheesecake de Oreo", descripcion: "Postre cremoso de queso con trozos de galleta oreo.", precio: 15000, imagen: "oreoC.jpg", categoria: "Postres", stock: 12 },
  { nombre: "Tarta de Manzana", descripcion: "Tarta horneada con canela y helado de vainilla.", precio: 14000, imagen: "tartaM.jpg", categoria: "Postres", stock: 10 },
  { nombre: "Salmon Teriyaki", descripcion: "Salmon glaseado en salsa dulce con vegetales salteados.", precio: 49000, imagen: "salmonT.jpg", categoria: "Platos Fuertes", stock: 15 },
  { nombre: "Lomo a la Pimienta", descripcion: "Corte premium de res con salsa cremosa de pimienta.", precio: 54000, imagen: "lomoP.jpg", categoria: "Platos Fuertes", stock: 10 },
  { nombre: "Risotto de Hongos", descripcion: "Arroz cremoso con variedad de setas y queso parmesano.", precio: 39500, imagen: "risottoH.jpg", categoria: "Platos Fuertes", stock: 7 },
];

// ========================================
// 2. ESTADO REACTIVO — CATÁLOGO
// ========================================

const platosGuardados = localStorage.getItem('platosbristo');
const platos = ref(platosGuardados ? JSON.parse(platosGuardados) : PLATOSINICIO);
const platosFiltrados = ref([...platos.value]);
const textoBusqueda = ref("");
const categoriaSeleccionada = ref("Todas");

const _categoriasCustom = JSON.parse(localStorage.getItem('categorias_custom_bistro') || '[]');
const categorias = ref([...CATEGORIAS_FIJAS, ..._categoriasCustom]);
const categoriasPersonalizadas = ref([..._categoriasCustom]);
const categoriasFijas = ref(CATEGORIAS_FIJAS.filter(c => c !== 'Todas'));


// ========================================
// 3. ESTADO REACTIVO — CARRITO Y PEDIDO
// ========================================

const carrito = ref([]);
const totalUnidades = ref(0);
const propinaPorcentaje = ref(0);
const cliente = ref({ nombre: '', mesa: '', metodoPago: '', notas: '' });
const totales = ref({ subtotal: 0, iva: 0, propina: 0, total: 0 });


// ========================================
// 4. ESTADO REACTIVO — UI / MODALES
// ========================================

const mostrarModal = ref(false);
const mostrarFormPlato = ref(false);
const mostrarModalCategorias = ref(false);
const ticketParaImprimir = ref(null);
const imprimiendoTicket = ref(false);
const procesandoPedido = ref(false);
const menuAbierto = ref(null);
const editandoPlatoNombre = ref(null);
const navCategorias = ref(null);

const formPlato = ref({
  nombre: '',
  descripcion: '',
  precio: 0,
  imagen: '',
  categoria: 'Rapidas',
  stock: 0,
});

const nuevaCategoriaTexto = ref('');
const categoriaEditando = ref(null);
const categoriaEditandoTexto = ref('');


// ========================================
// 5. LOCAL STORAGE
// ========================================

const guardarPlatosEnStorage = () => {
  const platosGuardar = platos.value.map(p => {
    const itemEnCarrito = carrito.value.find(c => c.nombre === p.nombre);
    return { ...p, stock: p.stock + (itemEnCarrito ? itemEnCarrito.cantidad : 0) };
  });
  localStorage.setItem('platosbristo', JSON.stringify(platosGuardar));
};

const actualizarCarritoEnStorage = () =>
  localStorage.setItem('carrito_bistro', JSON.stringify(carrito.value));

const guardarCategoriasEnStorage = () =>
  localStorage.setItem('categorias_custom_bistro', JSON.stringify(categoriasPersonalizadas.value));

const inicializarCarrito = () => {
  const datosGuardados = localStorage.getItem('carrito_bistro');
  if (!datosGuardados) return;

  const guardado = JSON.parse(datosGuardados);
  carrito.value = guardado.map(itemCarrito => {
    const plato = platos.value.find(p => p.nombre === itemCarrito.nombre);
    return { ...itemCarrito, precio: itemCarrito.precio || (plato ? plato.precio : 0) };
  });

  carrito.value.forEach(itemCarrito => {
    const plato = platos.value.find(p => p.nombre === itemCarrito.nombre);
    if (plato) plato.stock -= itemCarrito.cantidad;
  });
};


// ========================================
// 6. FILTROS Y BÚSQUEDA
// ========================================

const aplicarFiltros = () => {
  const prefijo = textoBusqueda.value.toLowerCase().trim();
  platosFiltrados.value = platos.value
    .filter(plato => {
      const coincideCategoria = categoriaSeleccionada.value === 'Todas' || plato.categoria === categoriaSeleccionada.value;
      const coincideNombre = plato.nombre.toLowerCase().startsWith(prefijo);
      return coincideCategoria && coincideNombre;
    })
    .sort((a, b) => (b.stock > 0) - (a.stock > 0));
};

const filtrarPlatos = categoria => {
  categoriaSeleccionada.value = categoria;
  aplicarFiltros();
};

const scrollCategorias = (direccion) => {
  if (navCategorias.value) {
    const firstButton = navCategorias.value.querySelector('.btn-categoria');
    if (firstButton) {
      const scrollAmount = firstButton.offsetWidth + 8;
      navCategorias.value.scrollBy({ left: direccion * scrollAmount, behavior: 'smooth' });
    }
  }
};


// ========================================
// 7. CARRITO
// ========================================

const cantidadEnCarrito = nombre => {
  const item = carrito.value.find(i => i.nombre === nombre);
  return item ? item.cantidad : 0;
};

const verificarStockTotal = () => {
  const sinStock = platos.value.every(p => p.stock === 0);
  if (sinStock) Swal.fire({
    icon: 'warning',
    title: '¡Sin stock!',
    text: 'Todos los productos se han agotado.',
    confirmButtonText: 'ACEPTAR',
    confirmButtonColor: '#E76F51',
  });
};

const actualizarTotales = () => {
  let subtotal = 0;
  let unidades = 0;

  carrito.value.forEach(item => {
    const platoOriginal = platos.value.find(p => p.nombre === item.nombre);
    if (platoOriginal) item.precio = platoOriginal.precio;
    subtotal += (item.precio || 0) * item.cantidad;
    unidades += item.cantidad;
  });

  totalUnidades.value = unidades;
  totales.value.subtotal = subtotal;
  totales.value.iva = subtotal * 0.19;
  totales.value.propina = subtotal * (propinaPorcentaje.value / 100);
  totales.value.total = totales.value.subtotal + totales.value.iva + totales.value.propina;

  actualizarCarritoEnStorage();
  verificarStockTotal();
};

const agregarAlCarrito = producto => {
  if (producto.stock <= 0) return Swal.fire({ icon: 'error', title: 'Sin stock', text: 'Lo sentimos, este producto se ha agotado.', confirmButtonText: 'ACEPTAR' });

  const item = carrito.value.find(c => c.nombre === producto.nombre);
  if (item) {
    item.cantidad++;
  } else {
    carrito.value.push({ nombre: producto.nombre, precio: producto.precio, cantidad: 1, categoria: producto.categoria });
  }

  producto.stock--;
  Swal.fire({ toast: true, position: 'top-end', icon: 'success', title: '¡Agregado!', showConfirmButton: false, timer: 1100, background: '#2A9D8F', color: '#fff' });
  actualizarTotales();
};

const cambiarCantidad = (index, valor) => {
  const itemCarrito = carrito.value[index];
  if (!itemCarrito) return;

  const productoOriginal = platos.value.find(p => p.nombre === itemCarrito.nombre);

  if (valor > 0) {
    if (!productoOriginal) return Swal.fire({ icon: 'error', title: 'Error', text: 'Este producto ya no está disponible en el menú.', confirmButtonText: 'ACEPTAR' });
    if (productoOriginal.stock <= 0) return Swal.fire({ icon: 'error', title: 'Sin stock', text: 'No hay más unidades disponibles.', confirmButtonText: 'ACEPTAR' });
    itemCarrito.cantidad++;
    productoOriginal.stock--;
  } else {
    itemCarrito.cantidad--;
    if (productoOriginal) productoOriginal.stock++;
    if (itemCarrito.cantidad <= 0) carrito.value.splice(index, 1);
  }

  actualizarTotales();
};

const vaciarCarrito = () => {
  Swal.fire({
    icon: 'warning',
    title: '¿Vaciar carrito?',
    text: 'Perderás el progreso de la compra',
    showCancelButton: true,
    confirmButtonText: 'VACIAR',
    confirmButtonColor: '#E76F51',
    cancelButtonText: 'CANCELAR',
  }).then(resultado => {
    if (!resultado.isConfirmed) return;

    Swal.fire({
      title: 'Vaciando...',
      allowOutsideClick: false,
      didOpen: () => {
        Swal.showLoading();
        carrito.value.forEach(item => {
          const platoOriginal = platos.value.find(p => p.nombre === item.nombre);
          if (platoOriginal) platoOriginal.stock += item.cantidad;
        });
        carrito.value = [];
        actualizarTotales();
        setTimeout(() => Swal.fire({ icon: 'success', title: '¡CARRITO VACÍO!', confirmButtonText: 'CERRAR' }), 500);
      },
    });
  });
};

const seleccionarPropina = valor => {
  propinaPorcentaje.value = valor;
  actualizarTotales();
};


// ========================================
// 8. GESTIÓN DE PRODUCTOS (CRUD)
// ========================================

const toggleMenu = nombre => {
  menuAbierto.value = menuAbierto.value === nombre ? null : nombre;
};

const abrirFormPlato = plato => {
  if (plato) {
    editandoPlatoNombre.value = plato.nombre;
    formPlato.value = { ...plato };
  } else {
    editandoPlatoNombre.value = null;
    formPlato.value = { nombre: '', descripcion: '', precio: 0, imagen: '', categoria: 'Rapidas', stock: 0 };
  }
  mostrarFormPlato.value = true;
  menuAbierto.value = null;
};

const cerrarFormPlato = () => {
  mostrarFormPlato.value = false;
};

const manejarArchivo = evento => {
  const archivo = evento.target.files[0];
  if (!archivo) return;
  const reader = new FileReader();
  reader.onload = e => { formPlato.value.imagen = e.target.result; };
  reader.readAsDataURL(archivo);
};

const guardarPlato = () => {
  const { nombre, descripcion, precio, imagen, categoria, stock } = formPlato.value;

  if (editandoPlatoNombre.value) {
    const platoOriginal = platos.value.find(p => p.nombre === editandoPlatoNombre.value);
    if (platoOriginal) {
      const sinCambios =
        platoOriginal.nombre === nombre &&
        platoOriginal.descripcion === descripcion &&
        platoOriginal.precio == precio &&
        platoOriginal.stock == stock &&
        platoOriginal.categoria === categoria &&
        platoOriginal.imagen === imagen;
      if (sinCambios) return Swal.fire({ icon: 'info', title: 'Sin cambios', text: 'Por favor edite algo antes de guardar.', confirmButtonText: 'ACEPTAR', confirmButtonColor: '#E76F51' });
    }
  }

  if (!nombre || !descripcion || !precio || !imagen || !categoria) return Swal.fire({ icon: 'error', title: 'Error', text: 'Todos los campos son obligatorios.', confirmButtonText: 'ACEPTAR' });
  if (nombre.trim().length < 3) return Swal.fire({ icon: 'warning', title: 'Nombre inválido', text: 'El nombre debe tener mínimo 3 caracteres.', confirmButtonColor: '#E76F51', confirmButtonText: 'ACEPTAR' });
  if (descripcion.trim().length < 10) return Swal.fire({ icon: 'warning', title: 'Descripción inválida', text: 'La descripción debe tener mínimo 10 caracteres.', confirmButtonColor: '#E76F51', confirmButtonText: 'ACEPTAR' });
  if (precio <= 0) return Swal.fire({ icon: 'warning', title: 'Precio inválido', text: 'El precio debe ser mayor a 0.', confirmButtonColor: '#E76F51', confirmButtonText: 'ACEPTAR' });
  if (stock < 0 || stock > 100) return Swal.fire({ icon: 'warning', title: 'Stock inválido', text: 'El stock debe estar entre 0 y 100.', confirmButtonColor: '#E76F51', confirmButtonText: 'ACEPTAR' });

  if (editandoPlatoNombre.value) {
    const index = platos.value.findIndex(p => p.nombre === editandoPlatoNombre.value);
    if (index !== -1) {
      if (editandoPlatoNombre.value !== nombre)
        carrito.value.forEach(item => { if (item.nombre === editandoPlatoNombre.value) item.nombre = nombre; });
      platos.value[index] = { ...formPlato.value };
    }
  } else {
    if (platos.value.find(p => p.nombre === nombre)) return Swal.fire({ icon: 'error', title: 'Error', text: 'Ya existe un plato con ese nombre.', confirmButtonText: 'ACEPTAR' });
    platos.value.push({ ...formPlato.value });
  }

  cerrarFormPlato();
  guardarPlatosEnStorage();
  aplicarFiltros();
  actualizarTotales();
  Swal.fire({ icon: 'success', title: '¡Éxito!', text: editandoPlatoNombre.value ? 'Producto actualizado.' : 'Producto agregado.', confirmButtonText: 'ACEPTAR' });
};

const eliminarPlato = nombre => {
  Swal.fire({
    title: '¿Eliminar producto?',
    text: 'Esta acción no se puede deshacer',
    icon: 'warning',
    showCancelButton: true,
    confirmButtonColor: '#FF4D4D',
    cancelButtonColor: '#011627',
    cancelButtonText: 'CANCELAR',
    confirmButtonText: 'ELIMINAR',
  }).then(result => {
    if (!result.isConfirmed) return;
    platos.value = platos.value.filter(p => p.nombre !== nombre);
    guardarPlatosEnStorage();
    aplicarFiltros();
    Swal.fire({ icon: 'success', title: 'Eliminado', text: 'El producto ha sido eliminado.', confirmButtonText: 'ACEPTAR' });
  });
};


// ========================================
// 9. GESTIÓN DE CATEGORÍAS
// ========================================

const reconstruirCategorias = () => {
  categorias.value = [...CATEGORIAS_FIJAS, ...categoriasPersonalizadas.value];
};

const abrirModalCategorias = () => {
  nuevaCategoriaTexto.value = '';
  categoriaEditando.value = null;
  categoriaEditandoTexto.value = '';
  mostrarModalCategorias.value = true;
};

const cerrarModalCategorias = () => {
  mostrarModalCategorias.value = false;
  categoriaEditando.value = null;
  nuevaCategoriaTexto.value = '';
};

const agregarCategoria = () => {
  const nombre = nuevaCategoriaTexto.value.trim();
  if (!nombre) return Swal.fire({ icon: 'warning', title: 'Campo vacío', text: 'Ingresa un nombre para la categoría.', confirmButtonColor: '#E76F51', confirmButtonText: 'ACEPTAR' });
  if (nombre.length < 2) return Swal.fire({ icon: 'warning', title: 'Nombre muy corto', text: 'El nombre debe tener mínimo 2 caracteres.', confirmButtonColor: '#E76F51', confirmButtonText: 'ACEPTAR' });
  if (categorias.value.map(c => c.toLowerCase()).includes(nombre.toLowerCase())) return Swal.fire({ icon: 'error', title: 'Ya existe', text: 'Ya existe una categoría con ese nombre.', confirmButtonText: 'ACEPTAR' });

  categoriasPersonalizadas.value.push(nombre);
  guardarCategoriasEnStorage();
  reconstruirCategorias();
  nuevaCategoriaTexto.value = '';
  Swal.fire({ toast: true, position: 'top-end', icon: 'success', title: `Categoría "${nombre}" agregada`, showConfirmButton: false, timer: 1500 });
};

const iniciarEdicionCategoria = cat => {
  categoriaEditando.value = cat;
  categoriaEditandoTexto.value = cat;
};

const cancelarEdicionCategoria = () => {
  categoriaEditando.value = null;
  categoriaEditandoTexto.value = '';
};

const confirmarEdicionCategoria = catOriginal => {
  const nuevoNombre = categoriaEditandoTexto.value.trim();
  if (!nuevoNombre) return Swal.fire({ icon: 'warning', title: 'Campo vacío', text: 'El nombre no puede estar vacío.', confirmButtonColor: '#E76F51', confirmButtonText: 'ACEPTAR' });
  if (nuevoNombre.length < 2) return Swal.fire({ icon: 'warning', title: 'Nombre muy corto', text: 'El nombre debe tener mínimo 2 caracteres.', confirmButtonColor: '#E76F51', confirmButtonText: 'ACEPTAR' });
  if (nuevoNombre.toLowerCase() !== catOriginal.toLowerCase() && categorias.value.map(c => c.toLowerCase()).includes(nuevoNombre.toLowerCase())) return Swal.fire({ icon: 'error', title: 'Ya existe', text: 'Ya existe una categoría con ese nombre.', confirmButtonText: 'ACEPTAR' });

  Swal.fire({
    title: '¿Confirmar edición?',
    html: `Cambiar <b>${catOriginal}</b> → <b>${nuevoNombre}</b>`,
    icon: 'question',
    showCancelButton: true,
    confirmButtonColor: '#2EC4B6',
    cancelButtonColor: '#011627',
    confirmButtonText: 'GUARDAR',
    cancelButtonText: 'CANCELAR',
  }).then(result => {
    if (!result.isConfirmed) return;

    const idx = categoriasPersonalizadas.value.indexOf(catOriginal);
    if (idx !== -1) categoriasPersonalizadas.value[idx] = nuevoNombre;
    platos.value.forEach(p => { if (p.categoria === catOriginal) p.categoria = nuevoNombre; });
    if (categoriaSeleccionada.value === catOriginal) categoriaSeleccionada.value = nuevoNombre;

    guardarPlatosEnStorage();
    guardarCategoriasEnStorage();
    reconstruirCategorias();
    cancelarEdicionCategoria();
    aplicarFiltros();
    Swal.fire({ toast: true, position: 'top-end', icon: 'success', title: 'Categoría actualizada', showConfirmButton: false, timer: 1500 });
  });
};

const eliminarCategoria = cat => {
  const productosEnCategoria = platos.value.filter(p => p.categoria === cat);
  if (productosEnCategoria.length > 0) {
    return Swal.fire({
      icon: 'error',
      title: 'No se puede eliminar',
      html: `La categoría <b>${cat}</b> tiene <b>${productosEnCategoria.length}</b> producto(s). Reasigna o elimina esos productos primero.`,
      confirmButtonColor: '#E76F51',
      confirmButtonText: 'ENTENDIDO',
    });
  }

  Swal.fire({
    title: '¿Eliminar categoría?',
    html: `Se eliminará <b>${cat}</b> permanentemente.`,
    icon: 'warning',
    showCancelButton: true,
    confirmButtonColor: '#FF4D4D',
    cancelButtonColor: '#011627',
    confirmButtonText: 'ELIMINAR',
    cancelButtonText: 'CANCELAR',
  }).then(result => {
    if (!result.isConfirmed) return;
    categoriasPersonalizadas.value = categoriasPersonalizadas.value.filter(c => c !== cat);
    if (categoriaSeleccionada.value === cat) categoriaSeleccionada.value = 'Todas';
    guardarCategoriasEnStorage();
    reconstruirCategorias();
    aplicarFiltros();
    Swal.fire({ toast: true, position: 'top-end', icon: 'success', title: 'Categoría eliminada', showConfirmButton: false, timer: 1500 });
  });
};


// ========================================
// 10. MODALES Y FINALIZACIÓN DE PEDIDO
// ========================================

const abrirModal = () => { mostrarModal.value = true; };
const cerrarModal = () => { mostrarModal.value = false; };

const cerrarSiEsFuera = evento => {
  if (evento.target.classList.contains('modal-overlay')) cerrarModal();
};

const construirTicketParaImpresion = ({ nombre, mesa, metodoPago, notas }) => ({
  cliente: nombre.trim(),
  mesa: mesa ? mesa.toString().slice(0, 2) : '',
  metodoPago,
  subtotal: totales.value.subtotal,
  iva: totales.value.iva,
  propina: totales.value.propina,
  total: totales.value.total,
  propinaPorcentaje: propinaPorcentaje.value,
  notas: notas ? notas.trim() : '',
  items: carrito.value.map(item => ({
    nombre: item.nombre,
    cantidad: item.cantidad,
    total: (item.precio || 0) * item.cantidad,
  })),
});

const limpiarPedidoFinalizado = () => {
  carrito.value = [];
  cliente.value = { nombre: '', mesa: '', metodoPago: '', notas: '' };
  propinaPorcentaje.value = 0;
  ticketParaImprimir.value = null;
  imprimiendoTicket.value = false;
  localStorage.setItem('platosbristo', JSON.stringify(platos.value));
  actualizarTotales();
  cerrarModal();
};

const cerrarImpresionTicket = () => {
  imprimiendoTicket.value = false;
  ticketParaImprimir.value = null;
  procesandoPedido.value = false;

  Swal.fire({
    title: '¡EXITO!',
    text: 'Pedido procesado correctamente',
    icon: 'success',
    confirmButtonColor: '#FF4D4D',
    confirmButtonText: '¡VAMOS!',
  }).then(() => {
    limpiarPedidoFinalizado();
  });
};

const imprimirTicket = async ticket => {
  ticketParaImprimir.value = ticket;
  imprimiendoTicket.value = true;
  await nextTick();

  let impresionCerrada = false;
  const finalizarImpresion = () => {
    if (impresionCerrada) return;
    impresionCerrada = true;
    window.removeEventListener('afterprint', finalizarImpresion);
    cerrarImpresionTicket();
  };

  window.addEventListener('afterprint', finalizarImpresion, { once: true });

  setTimeout(() => {
    window.print();
    setTimeout(finalizarImpresion, 1200);
  }, 150);
};

const finalizarPedido = () => {
  if (procesandoPedido.value) return;

  const { nombre, mesa, metodoPago, notas } = cliente.value;

  if (!carrito.value.length) return Swal.fire({ icon: 'error', title: 'Error', text: 'El carrito está vacío.', confirmButtonText: 'ACEPTAR' });
  if (!nombre || !mesa || !metodoPago) return Swal.fire({ icon: 'info', title: 'Atención', text: 'Completa todos tus datos.', confirmButtonText: 'ACEPTAR' });
  if (nombre.trim().length < 3) return Swal.fire({ icon: 'warning', title: 'Nombre inválido', text: 'El nombre debe tener mínimo 3 caracteres.', confirmButtonColor: '#E76F51', confirmButtonText: 'ACEPTAR' });
  if (mesa < 1 || mesa > 25) return Swal.fire({ icon: 'warning', title: 'Mesa inválida', text: 'El número de mesa debe estar entre 1 y 25.', confirmButtonColor: '#E76F51', confirmButtonText: 'ACEPTAR' });

  procesandoPedido.value = true;

  Swal.fire({
    title: 'PROCESANDO PAGO',
    html: 'Procesando',
    allowOutsideClick: false,
    didOpen: () => Swal.showLoading(),
    timer: 2000,
    timerProgressBar: true,
  }).then(() => {
    const ticket = construirTicketParaImpresion({ nombre, mesa, metodoPago, notas });
    mostrarModal.value = false;
    imprimirTicket(ticket);
  });
};


// ========================================
// 11. EVENTOS GLOBALES
// ========================================

document.addEventListener('click', () => { menuAbierto.value = null; });


// ========================================
// 12. INICIALIZACIÓN
// ========================================

inicializarCarrito();
aplicarFiltros();
actualizarTotales();
</script>
