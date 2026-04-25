<template>
  <div class="todo">

    <!-- ── HEADER ────────────────────────────────────────────── -->
    <header class="header">
      <div class="header-top">

        <div class="logo">
          <img src="/logo.png" alt="FoodApp" class="logo-img" />
        </div>

        <div class="search-wrapper">
          <span class="search-icon">🔍</span>
          <input
            class="search-input"
            v-model="searchQuery"
            @input="applyFilters"
            placeholder="Buscar comida..."
          />
          <button class="search-clear" v-if="searchQuery" @click="searchQuery = ''; applyFilters()">✕</button>
        </div>

        <button class="btn-reset-stock" @click="resetStock" title="Restaurar stock original">↺</button>

        <button class="cart-btn" @click="cartOpen = true">
          🛒
          <span class="cart-badge" v-if="getCartTotal() > 0">{{ getCartTotal() }}</span>
        </button>
      </div>

      <div class="categories">
        <button
          class="pill"
          :class="activeCategory === '' ? 'pill-active' : ''"
          @click="activeCategory = ''; applyFilters()"
        >Todas</button>
        <button
          v-for="cat in categories"
          :key="cat"
          class="pill"
          :class="activeCategory === cat ? 'pill-active' : ''"
          @click="activeCategory = cat; applyFilters()"
        >{{ cat }}</button>
      </div>
    </header>

    <!-- ── OVERLAY + SIDEBAR ──────────────────────────────────── -->
    <div class="overlay" v-if="cartOpen" @click="cartOpen = false"></div>

    <aside class="sidebar" :class="cartOpen ? 'sidebar-open' : ''">
      <div class="sidebar-header">
        <h2>Tu pedido</h2>
        <button class="sidebar-close" @click="cartOpen = false">✕</button>
      </div>

      <div class="sidebar-empty" v-if="getCartTotal() === 0">
        <span>🛒</span>
        <p>Tu carrito está vacío</p>
      </div>

      <div class="sidebar-items" v-if="getCartTotal() > 0">
        <div
          v-for="product in products"
          :key="product.id"
          class="sidebar-item"
          v-show="cart[product.id] > 0"
        >
          <img :src="product.image" :alt="product.name" class="sidebar-img" />
          <div class="sidebar-info">
            <p class="sidebar-name">{{ product.name }}</p>
            <p class="sidebar-price">${{ (product.price * (cart[product.id] || 0)).toFixed(2) }}</p>
          </div>
          <div class="sidebar-qty">
            <button class="qty-btn" @click="removeFromSidebar(product)">−</button>
            <span>{{ cart[product.id] }}</span>
            <button class="qty-btn" @click="addFromSidebar(product)" :disabled="product.stock === 0">+</button>
          </div>
        </div>
      </div>

      <div class="sidebar-footer" v-if="getCartTotal() > 0">
        <div class="sidebar-total">
          <span>Total</span>
          <span>${{ getCartPrice() }}</span>
        </div>
        <button class="btn-checkout" @click="abrirFactura">Generar factura</button>
      </div>
    </aside>

    <!-- ── PRODUCTOS ──────────────────────────────────────────── -->
    <main class="comida">
      <h1>{{ activeCategory || 'Todas las comidas' }}</h1>

      <div class="productos">
        <div class="card" v-for="product in filteredProducts" :key="product.id">

          <button
            class="btn-cart"
            @click="sendToCart(product)"
            :disabled="!selected[product.id] || selected[product.id] === 0 || product.stock === 0"
            :class="selected[product.id] > 0 ? 'btn-cart-active' : ''"
          >🛒</button>

          <div class="image-wrapper">
            <img :src="product.image" :alt="product.name" class="product-img" />
          </div>

          <div class="card-body">
            <h2 class="product-name">{{ product.name }}</h2>
            <p class="product-description">{{ product.description }}</p>

            <div
              class="stock-badge"
              :class="product.stock === 0 ? 'stock-out' : product.stock <= 5 ? 'stock-low' : 'stock-ok'"
            >
              <span class="stock-dot"></span>
              <span v-if="product.stock === 0">Agotado</span>
              <span v-else-if="product.stock <= 5">Solo {{ product.stock }} disponibles</span>
              <span v-else>{{ product.stock }} en stock</span>
            </div>

            <div class="price-row">
              <span class="price">${{ product.price.toFixed(2) }}</span>

              <div class="quantity-controls" v-if="product.stock > 0">
                <button class="qty-btn" @click="decreaseSelected(product)" :disabled="!selected[product.id] || selected[product.id] === 0">−</button>
                <span class="qty-count">{{ selected[product.id] || 0 }}</span>
                <button class="qty-btn" @click="increaseSelected(product)" :disabled="(selected[product.id] || 0) >= product.stock">+</button>
              </div>

              <span v-else class="sin-stock-label">Agotado</span>
            </div>
          </div>

        </div>

        <div class="no-results" v-if="filteredProducts.length === 0">
          <span>🍽️</span>
          <p>No se encontraron productos</p>
        </div>
      </div>
    </main>

    <!-- ── FAB ───────────────────────────────────────────────── -->
    <button class="fab" @click="modalOpen = true" title="Agregar producto">+</button>

    <!-- ── MODAL AGREGAR PRODUCTO ────────────────────────────── -->
    <div class="modal-overlay" v-if="modalOpen" @click.self="closeModal">
      <div class="modal">
        <div class="modal-header">
          <h2>Nuevo producto</h2>
          <button class="modal-close" @click="closeModal">✕</button>
        </div>

        <div class="modal-body">

          <div class="field">
            <label>Nombre *</label>
            <input v-model="newProduct.name" type="text" placeholder="Ej: Pizza Hawaiana" />
          </div>

          <div class="field">
            <label>Descripción *</label>
            <textarea v-model="newProduct.description" placeholder="Describe el producto..." rows="3"></textarea>
          </div>

          <div class="field-row">
            <div class="field">
              <label>Precio * ($)</label>
              <input v-model="newProduct.price" type="number" min="0" step="0.01" placeholder="0.00" />
            </div>
            <div class="field">
              <label>Stock *</label>
              <input v-model="newProduct.stock" type="number" min="0" step="1" placeholder="0" />
            </div>
          </div>

          <div class="field">
            <label>Categoría *</label>
            <select v-model="newProduct.category">
              <option value="" disabled>Selecciona una categoría</option>
              <option v-for="cat in categories" :key="cat" :value="cat">{{ cat }}</option>
              <option value="__nueva__">+ Nueva categoría</option>
            </select>
          </div>

          <div class="field" v-if="newProduct.category === '__nueva__'">
            <label>Nombre de la nueva categoría *</label>
            <input v-model="newProduct.newCategory" type="text" placeholder="Ej: Wraps" />
          </div>

          <div class="field">
            <label>URL de imagen *</label>
            <input v-model="newProduct.image" type="text" placeholder="https://..." />
            <div class="img-preview" v-if="newProduct.image">
              <img :src="newProduct.image" alt="Preview" @error="newProduct.imageError = true" @load="newProduct.imageError = false" />
              <p v-if="newProduct.imageError" class="img-error">URL de imagen no válida</p>
            </div>
          </div>

          <p class="modal-error" v-if="modalError">{{ modalError }}</p>
        </div>

        <div class="modal-footer">
          <button class="btn-cancel" @click="closeModal">Cancelar</button>
          <button class="btn-save" @click="saveNewProduct">Agregar producto</button>
        </div>
      </div>
    </div>

    <!-- ── MODAL FACTURA ─────────────────────────────────────── -->
    <div class="modal-overlay" v-if="facturaOpen" @click.self="facturaOpen = false">
      <div class="modal modal-factura">
        <div class="modal-header">
          <h2>Vista previa de factura</h2>
          <button class="modal-close" @click="facturaOpen = false">✕</button>
        </div>

        <div class="modal-body factura-body">
          <div class="factura" id="factura-preview">
            <div class="factura-logo">
              <img src="/logo.png" alt="FoodApp" />
            </div>
            <h1 class="factura-title">FoodApp</h1>
            <p class="factura-sub">Factura de pedido</p>
            <p class="factura-fecha">{{ facturaFecha }}</p>

            <div class="factura-divider"></div>

            <table class="factura-table">
              <thead>
                <tr>
                  <th>Producto</th>
                  <th>Cant.</th>
                  <th>Precio</th>
                  <th>Total</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in facturaItems" :key="item.id">
                  <td>{{ item.name }}</td>
                  <td class="center">{{ item.qty }}</td>
                  <td class="right">${{ item.price.toFixed(2) }}</td>
                  <td class="right">${{ (item.price * item.qty).toFixed(2) }}</td>
                </tr>
              </tbody>
            </table>

            <div class="factura-divider"></div>

            <div class="factura-total-row">
              <span>Total a pagar</span>
              <span class="factura-total-valor">${{ getCartPrice() }}</span>
            </div>

            <p class="factura-gracias">¡Gracias por tu pedido! 🍽️</p>
          </div>
        </div>

        <div class="modal-footer">
          <button class="btn-cancel" @click="facturaOpen = false">Cerrar</button>
          <button class="btn-save" @click="descargarFactura">⬇ Descargar PDF</button>
        </div>
      </div>
    </div>

    <!-- Toast de éxito -->
    <div class="toast" v-if="toastVisible">✅ ¡Factura descargada correctamente!</div>

  </div>
</template>

<script setup>
import { ref } from 'vue'

const STORAGE_KEY = 'food_app_products'

const defaultProducts = [
  { id: 1,  category: 'Hamburguesas',    name: 'Chicken Burger',             description: 'Hamburguesa de pollo crujiente con lechuga, tomate y salsa especial.',             price: 3.50,  stock: 15, image: 'https://images.unsplash.com/photo-1568901346375-23c9450c58cd?w=400&h=400&fit=crop' },
  { id: 2,  category: 'Hamburguesas',    name: 'Classic Smash Burger',       description: 'Doble carne aplastada, queso americano derretido y cebolla caramelizada.',         price: 5.20,  stock: 10, image: 'https://images.unsplash.com/photo-1553979459-d2229ba7433b?w=400&h=400&fit=crop' },
  { id: 3,  category: 'Hamburguesas',    name: 'BBQ Bacon Burger',           description: 'Carne de res, tocino crocante, salsa BBQ ahumada y aros de cebolla.',              price: 6.00,  stock: 8,  image: 'https://images.unsplash.com/photo-1594212699903-ec8a3eca50f5?w=400&h=400&fit=crop' },
  { id: 4,  category: 'Hamburguesas',    name: 'Veggie Burger',              description: 'Medallón de garbanzo y espinaca, aguacate, pepino y yogur de hierbas.',            price: 4.80,  stock: 12, image: 'https://images.unsplash.com/photo-1520072959219-c595dc870360?w=400&h=400&fit=crop' },
  { id: 5,  category: 'Pizzas',          name: 'Pizza Margherita',           description: 'Tomate San Marzano, mozzarella fresca y albahaca sobre masa napolitana.',          price: 7.50,  stock: 20, image: 'https://images.unsplash.com/photo-1574071318508-1cdbab80d002?w=400&h=400&fit=crop' },
  { id: 6,  category: 'Pizzas',          name: 'Pizza Pepperoni',            description: 'Abundante pepperoni, mozzarella extra y orégano sobre salsa de tomate.',          price: 8.00,  stock: 18, image: 'https://images.unsplash.com/photo-1628840042765-356cda07504e?w=400&h=400&fit=crop' },
  { id: 7,  category: 'Pizzas',          name: 'Pizza BBQ Chicken',          description: 'Pollo a la BBQ, cebolla morada, miel y queso gouda ahumado.',                     price: 9.00,  stock: 5,  image: 'https://images.unsplash.com/photo-1565299624946-b28f40a0ae38?w=400&h=400&fit=crop' },
  { id: 8,  category: 'Pizzas',          name: 'Pizza 4 Quesos',             description: 'Mozzarella, parmesano, gorgonzola y brie sobre base de crema de ajo.',            price: 9.50,  stock: 7,  image: 'https://images.unsplash.com/photo-1513104890138-7c749659a591?w=400&h=400&fit=crop' },
  { id: 9,  category: 'Pastas',          name: 'Spaghetti Bolognese',        description: 'Espagueti con ragú de res lento, zanahoria, apio y vino tinto.',                  price: 6.50,  stock: 14, image: 'https://images.unsplash.com/photo-1555949258-eb67b1ef0ceb?w=400&h=400&fit=crop' },
  { id: 10, category: 'Pastas',          name: 'Fettuccine Alfredo',         description: 'Fettuccine en salsa de mantequilla, parmesano y crema ligera.',                   price: 6.00,  stock: 11, image: 'https://www.nutmegnanny.com/wp-content/uploads/2024/02/Cajun-Shrimp-Fettuccine-Alfredo-4.jpg' },
  { id: 11, category: 'Pastas',          name: 'Penne Arrabbiata',           description: 'Penne con salsa de tomate picante, ajo y pimiento calabrés.',                     price: 5.80,  stock: 9,  image: 'https://images.unsplash.com/photo-1547592180-85f173990554?w=400&h=400&fit=crop' },
  { id: 12, category: 'Pastas',          name: 'Lasaña Clásica',             description: 'Capas de pasta fresca, bechamel, boloñesa y parmesano gratinado.',               price: 8.50,  stock: 6,  image: 'https://images.unsplash.com/photo-1574894709920-11b28e7367e3?w=400&h=400&fit=crop' },
  { id: 13, category: 'Sushi',           name: 'California Roll',            description: 'Cangrejo, aguacate y pepino envueltos en arroz y ajonjolí tostado.',              price: 7.00,  stock: 20, image: 'https://www.craftycookbook.com/wp-content/uploads/2024/03/tobiko-roll-1200.jpg' },
  { id: 14, category: 'Sushi',           name: 'Salmón Nigiri (x4)',         description: 'Rectángulos de arroz de sushi coronados con salmón fresco y wasabi.',             price: 8.00,  stock: 15, image: 'https://aisforappleau.com/wp-content/uploads/2023/07/how-to-make-sushi-salmon-nigiri-6.jpg' },
  { id: 15, category: 'Sushi',           name: 'Spicy Tuna Roll',            description: 'Atún con mayonesa picante, pepino y tobiko, enrollado en nori.',                  price: 8.50,  stock: 3,  image: 'https://food.fnr.sndimg.com/content/dam/images/food/fullset/2009/2/10/0/BT0511-4_Spicy-Tuna-Roll_s4x3.jpg.rend.hgtvcom.1280.960.suffix/1382724147395.webp' },
  { id: 16, category: 'Sushi',           name: 'Dragon Roll',                description: 'Camarón tempura, aguacate exterior, anguila glaseada y salsa teriyaki.',          price: 10.00, stock: 8,  image: 'https://images.unsplash.com/photo-1611143669185-af224c5e3252?w=400&h=400&fit=crop' },
  { id: 17, category: 'Tacos',           name: 'Taco de Carnitas',           description: 'Tortilla de maíz, carnitas de cerdo, cebolla, cilantro y salsa verde.',           price: 2.50,  stock: 25, image: 'https://images.unsplash.com/photo-1613514785940-daed07799d9b?w=400&h=400&fit=crop' },
  { id: 18, category: 'Tacos',           name: 'Taco de Camarón',            description: 'Camarones al ajillo, repollo morado, crema de chipotle y limón.',                price: 3.20,  stock: 18, image: 'https://images.unsplash.com/photo-1565299585323-38d6b0865b47?w=400&h=400&fit=crop' },
  { id: 19, category: 'Ensaladas',       name: 'César con Pollo',            description: 'Lechuga romana, pollo a la plancha, crutones, parmesano y aderezo César.',       price: 5.50,  stock: 16, image: 'https://images.unsplash.com/photo-1550304943-4f24f54ddde9?w=400&h=400&fit=crop' },
  { id: 20, category: 'Ensaladas',       name: 'Bowl Mediterráneo',          description: 'Quínoa, tomate cherry, aceitunas kalamata, feta y aderezo de limón.',            price: 6.00,  stock: 10, image: 'https://images.unsplash.com/photo-1512621776951-a57141f2eefd?w=400&h=400&fit=crop' },
  { id: 21, category: 'Postres',         name: 'Cheesecake de Frutos Rojos', description: 'Base de galleta, crema de queso suave y coulis de fresas y arándanos.',          price: 4.00,  stock: 12, image: 'https://images.unsplash.com/photo-1565958011703-44f9829ba187?w=400&h=400&fit=crop' },
  { id: 22, category: 'Postres',         name: 'Brownie con Helado',         description: 'Brownie de chocolate oscuro tibio con bola de vainilla y caramelo.',             price: 3.80,  stock: 20, image: 'https://images.unsplash.com/photo-1606313564200-e75d5e30476c?w=400&h=400&fit=crop' },
  { id: 23, category: 'Postres',         name: 'Tiramisú',                   description: 'Capas de savoiardi, mascarpone, espresso y cacao en polvo.',                     price: 4.50,  stock: 7,  image: 'https://images.unsplash.com/photo-1571877227200-a0d98ea607e9?w=400&h=400&fit=crop' },
  { id: 24, category: 'Postres',         name: 'Waffle Belga',               description: 'Waffle crujiente con fresas, crema chantilly y jarabe de maple.',               price: 4.20,  stock: 9,  image: 'https://images.unsplash.com/photo-1562376552-0d160a2f238d?w=400&h=400&fit=crop' },
  { id: 25, category: 'Bebidas',         name: 'Limonada de Menta',          description: 'Limones frescos, menta, agua mineral y un toque de jengibre.',                  price: 2.00,  stock: 30, image: 'https://images.unsplash.com/photo-1556679343-c7306c1976bc?w=400&h=400&fit=crop' },
  { id: 26, category: 'Bebidas',         name: 'Smoothie Tropical',          description: 'Mango, piña, maracuyá y leche de coco batidos al momento.',                     price: 3.00,  stock: 22, image: 'https://images.unsplash.com/photo-1505252585461-04db1eb84625?w=400&h=400&fit=crop' },
  { id: 27, category: 'Bebidas',         name: 'Cold Brew Coffee',           description: 'Café de extracción en frío 18 horas, servido con hielo y leche de avena.',      price: 2.80,  stock: 15, image: 'https://images.unsplash.com/photo-1461023058943-07fcbe16d735?w=400&h=400&fit=crop' },
  { id: 28, category: 'Acompañamientos', name: 'Papas Fritas',               description: 'Corte clásico, fritas dos veces para mayor crocancia. Con dip de queso.',       price: 2.50,  stock: 40, image: 'https://images.unsplash.com/photo-1573080496219-bb080dd4f877?w=400&h=400&fit=crop' },
  { id: 29, category: 'Acompañamientos', name: 'Aros de Cebolla',            description: 'Cebolla dulce, empanizada en cerveza y frita. Con salsa ranch.',                price: 3.00,  stock: 5,  image: 'https://images.unsplash.com/photo-1639024471283-03518883512d?w=400&h=400&fit=crop' },
  { id: 30, category: 'Acompañamientos', name: 'Alitas BBQ (x6)',            description: 'Alitas de pollo asadas y bañadas en salsa BBQ casera ahumada.',                 price: 5.50,  stock: 13, image: 'https://images.unsplash.com/photo-1567620832903-9fc6debc209f?w=400&h=400&fit=crop' },
]

// ── Estado — declarado PRIMERO para que todo lo de abajo pueda usarlo ─────────

const products        = ref(loadProducts())
const filteredProducts = ref([...products.value])
const cart            = ref({})
const selected        = ref({})
const searchQuery     = ref('')
const activeCategory  = ref('')
const cartOpen        = ref(false)
const categories      = [...new Set(defaultProducts.map(p => p.category))]

// ── Modal agregar producto ────────────────────────────────────────────────────

const modalOpen  = ref(false)
const modalError = ref('')

function emptyProduct() {
  return { name: '', description: '', price: '', stock: '', category: '', newCategory: '', image: '', imageError: false }
}

const newProduct = ref(emptyProduct())

function closeModal() {
  modalOpen.value  = false
  modalError.value = ''
  newProduct.value = emptyProduct()
}

// ── Factura ───────────────────────────────────────────────────────────────────

const facturaOpen  = ref(false)
const toastVisible = ref(false)
const facturaFecha = ref('')
const facturaItems = ref([])

function abrirFactura() {
  // Armar items del carrito
  facturaItems.value = products.value
    .filter(p => cart.value[p.id] > 0)
    .map(p => ({ id: p.id, name: p.name, price: p.price, qty: cart.value[p.id] }))

  // Fecha actual
  const now = new Date()
  facturaFecha.value = now.toLocaleDateString('es-CO', {
    year: 'numeric', month: 'long', day: 'numeric',
    hour: '2-digit', minute: '2-digit'
  })

  cartOpen.value   = false
  facturaOpen.value = true
}

async function descargarFactura() {
  // Cargar jsPDF desde CDN dinámicamente
  if (!window.jspdf) {
    await new Promise((resolve, reject) => {
      const script = document.createElement('script')
      script.src = 'https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js'
      script.onload = resolve
      script.onerror = reject
      document.head.appendChild(script)
    })
  }

  const { jsPDF } = window.jspdf
  const doc = new jsPDF({ unit: 'pt', format: 'a4' })

  const verde    = [106, 191, 105]
  const oscuro   = [26, 26, 26]
  const gris     = [120, 120, 120]
  const lineaY   = (n) => 100 + n * 22
  const pageW    = doc.internal.pageSize.getWidth()

  // Encabezado verde
  doc.setFillColor(...verde)
  doc.rect(0, 0, pageW, 70, 'F')

  doc.setTextColor(255, 255, 255)
  doc.setFont('helvetica', 'bold')
  doc.setFontSize(26)
  doc.text('FoodApp', 40, 44)

  doc.setFontSize(11)
  doc.setFont('helvetica', 'normal')
  doc.text('Factura de pedido', 40, 60)

  // Fecha
  doc.setTextColor(...gris)
  doc.setFontSize(10)
  doc.text(facturaFecha.value, pageW - 40, 44, { align: 'right' })

  // Línea separadora
  doc.setDrawColor(...verde)
  doc.setLineWidth(1.5)
  doc.line(40, 85, pageW - 40, 85)

  // Encabezados tabla
  doc.setFillColor(245, 245, 245)
  doc.rect(40, 90, pageW - 80, 22, 'F')

  doc.setTextColor(...oscuro)
  doc.setFont('helvetica', 'bold')
  doc.setFontSize(10)
  doc.text('Producto',    50,  106)
  doc.text('Cant.',      340,  106, { align: 'center' })
  doc.text('Precio',     430,  106, { align: 'right' })
  doc.text('Total',  pageW - 50, 106, { align: 'right' })

  // Filas
  doc.setFont('helvetica', 'normal')
  let y = 128
  facturaItems.value.forEach((item, i) => {
    if (i % 2 === 0) {
      doc.setFillColor(252, 252, 252)
      doc.rect(40, y - 14, pageW - 80, 20, 'F')
    }
    doc.setTextColor(...oscuro)
    doc.setFontSize(10)
    doc.text(item.name,                              50,  y)
    doc.text(String(item.qty),                      340,  y, { align: 'center' })
    doc.text(`$${item.price.toFixed(2)}`,           430,  y, { align: 'right' })
    doc.text(`$${(item.price * item.qty).toFixed(2)}`, pageW - 50, y, { align: 'right' })
    y += 22
  })

  // Línea total
  y += 8
  doc.setDrawColor(...verde)
  doc.setLineWidth(1)
  doc.line(40, y, pageW - 40, y)
  y += 18

  doc.setFont('helvetica', 'bold')
  doc.setFontSize(13)
  doc.setTextColor(...verde)
  doc.text('Total a pagar:', 50, y)
  doc.text(`$${getCartPrice()}`, pageW - 50, y, { align: 'right' })

  // Pie
  y += 40
  doc.setFont('helvetica', 'italic')
  doc.setFontSize(10)
  doc.setTextColor(...gris)
  doc.text('¡Gracias por tu pedido! — FoodApp', pageW / 2, y, { align: 'center' })

  doc.save(`factura-foodapp-${Date.now()}.pdf`)

  // Toast y limpiar carrito
  facturaOpen.value  = false
  toastVisible.value = true
  cart.value         = {}
  selected.value     = {}
  setTimeout(() => { toastVisible.value = false }, 3500)
}

function saveNewProduct() {
  const p = newProduct.value

  // Validación
  if (!p.name.trim())        { modalError.value = 'El nombre es obligatorio.'; return }
  if (!p.description.trim()) { modalError.value = 'La descripción es obligatoria.'; return }
  if (!p.price || isNaN(p.price) || Number(p.price) < 0) { modalError.value = 'Ingresa un precio válido.'; return }
  if (!p.stock || isNaN(p.stock) || Number(p.stock) < 0) { modalError.value = 'Ingresa un stock válido.'; return }
  if (!p.category)           { modalError.value = 'Selecciona una categoría.'; return }
  if (p.category === '__nueva__' && !p.newCategory.trim()) { modalError.value = 'Escribe el nombre de la nueva categoría.'; return }
  if (!p.image.trim())       { modalError.value = 'La URL de imagen es obligatoria.'; return }
  if (p.imageError)          { modalError.value = 'La URL de imagen no es válida.'; return }

  const finalCategory = p.category === '__nueva__' ? p.newCategory.trim() : p.category

  // Nuevo ID: máximo actual + 1
  const maxId = products.value.reduce((max, prod) => prod.id > max ? prod.id : max, 0)

  const producto = {
    id:          maxId + 1,
    category:    finalCategory,
    name:        p.name.trim(),
    description: p.description.trim(),
    price:       Number(Number(p.price).toFixed(2)),
    stock:       Number(p.stock),
    image:       p.image.trim(),
  }

  products.value.push(producto)

  // Si es categoría nueva, agregarla a la lista
  if (!categories.includes(finalCategory)) {
    categories.push(finalCategory)
  }

  saveProducts()
  applyFilters()
  closeModal()
}

// ── localStorage ──────────────────────────────────────────────────────────────

function loadProducts() {
  try {
    const stored = localStorage.getItem(STORAGE_KEY)
    return stored ? JSON.parse(stored) : [...defaultProducts]
  } catch {
    return [...defaultProducts]
  }
}

function saveProducts() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(products.value))
}

function resetStock() {
  const defaultIds = new Set(defaultProducts.map(d => d.id))
  products.value = products.value.map(p => {
    if (!defaultIds.has(p.id)) return p  // productos manuales: intactos
    const original = defaultProducts.find(d => d.id === p.id)
    return { ...p, stock: original.stock }
  })
  saveProducts()
  applyFilters()
}

// ── Filtros ───────────────────────────────────────────────────────────────────

function applyFilters() {
  filteredProducts.value = products.value.filter(p => {
    const matchCat    = activeCategory.value === '' || p.category === activeCategory.value
    const matchSearch = p.name.toLowerCase().includes(searchQuery.value.toLowerCase())
    return matchCat && matchSearch
  })
}

// ── Selección en la card (no toca el carrito) ────────────────────────────────

function increaseSelected(product) {
  const current = selected.value[product.id] || 0
  if (current >= product.stock) return
  selected.value[product.id] = current + 1
}

function decreaseSelected(product) {
  const current = selected.value[product.id] || 0
  if (current <= 0) return
  selected.value[product.id] = current - 1
}

// ── Enviar al carrito (botón 🛒 de la card) ───────────────────────────────────

function sendToCart(product) {
  const qty = selected.value[product.id] || 0
  if (qty <= 0) return

  // Sumar al carrito
  cart.value[product.id] = (cart.value[product.id] || 0) + qty

  // Descontar stock del producto
  const index = products.value.findIndex(p => p.id === product.id)
  if (index !== -1) {
    products.value[index].stock -= qty
    saveProducts()
    applyFilters() // refrescar filteredProducts con el stock actualizado
  }

  // Limpiar selección de esta card
  selected.value[product.id] = 0
}

// ── Carrito (sidebar) ─────────────────────────────────────────────────────────

function getCartTotal() {
  return Object.values(cart.value).reduce((sum, qty) => sum + qty, 0)
}

function getCartPrice() {
  return products.value.reduce((total, p) => {
    return total + (p.price * (cart.value[p.id] || 0))
  }, 0).toFixed(2)
}

function removeFromSidebar(product) {
  const current = cart.value[product.id] || 0
  if (current <= 0) return
  cart.value[product.id] = current - 1

  // Devolver 1 unidad al stock
  const index = products.value.findIndex(p => p.id === product.id)
  if (index !== -1) {
    products.value[index].stock += 1
    saveProducts()
    applyFilters()
  }
}

function addFromSidebar(product) {
  const stockDisponible = product.stock
  if (stockDisponible <= 0) return
  cart.value[product.id] = (cart.value[product.id] || 0) + 1

  const index = products.value.findIndex(p => p.id === product.id)
  if (index !== -1) {
    products.value[index].stock -= 1
    saveProducts()
    applyFilters()
  }
}

</script>

<style>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  background: #f5f5f5;
  font-family: 'Nunito', 'Segoe UI', sans-serif;
}

.todo { min-height: 100vh; }

/* ── HEADER ────────────────────────────────────────────────── */

.header {
  position: sticky;
  top: 0;
  z-index: 100;
  background: #ffffff;
  box-shadow: 0 2px 16px rgba(0,0,0,0.08);
  padding: 1rem 2rem 0;
}

.header-top {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  padding-bottom: 1rem;
}

.logo { display: flex; align-items: center; flex-shrink: 0; }
.logo-img { height: 64px; width: auto; object-fit: contain; }

.search-wrapper {
  flex: 1;
  display: flex;
  align-items: center;
  background: #f5f5f5;
  border-radius: 30px;
  padding: 0 1rem;
  gap: 0.5rem;
  border: 1.5px solid #e8e8e8;
  transition: border-color 0.2s;
  height: 44px;
}
.search-wrapper:focus-within { border-color: #6abf69; }

.search-icon { font-size: 0.9rem; color: #aaa; }

.search-input {
  flex: 1;
  border: none;
  background: transparent;
  font-size: 0.9rem;
  color: #333;
  outline: none;
}

.search-clear {
  background: none;
  border: none;
  color: #aaa;
  cursor: pointer;
  font-size: 0.85rem;
}
.search-clear:hover { color: #555; }

.cart-btn {
  position: relative;
  background: #6abf69;
  border: none;
  border-radius: 50%;
  width: 44px;
  height: 44px;
  font-size: 1.2rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: background 0.2s, transform 0.1s;
  box-shadow: 0 2px 10px rgba(106,191,105,0.35);
}
.cart-btn:hover { background: #57a857; }
.cart-btn:active { transform: scale(0.93); }

.cart-badge {
  position: absolute;
  top: -4px;
  right: -4px;
  background: #e53935;
  color: #fff;
  font-size: 0.65rem;
  font-weight: 800;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 2px solid #fff;
}

.categories {
  display: flex;
  gap: 0.5rem;
  overflow-x: auto;
  padding-bottom: 0.9rem;
  scrollbar-width: none;
  border-top: 1px solid #eee;
  padding-top: 0.6rem;
}
.categories::-webkit-scrollbar { display: none; }

.pill {
  flex-shrink: 0;
  background: #f0f0f0;
  border: none;
  border-radius: 20px;
  padding: 6px 16px;
  font-size: 0.82rem;
  font-weight: 600;
  color: #555;
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
}
.pill:hover { background: #e0f2e0; color: #2e7d32; }
.pill-active { background: #6abf69; color: #fff; }
.pill-active:hover { background: #57a857; color: #fff; }

/* ── SIDEBAR ───────────────────────────────────────────────── */

.overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.35);
  z-index: 200;
}

.sidebar {
  position: fixed;
  top: 0;
  right: 0;
  height: 100vh;
  width: 340px;
  background: #fff;
  z-index: 300;
  display: flex;
  flex-direction: column;
  box-shadow: -4px 0 24px rgba(0,0,0,0.12);
  transform: translateX(100%);
  transition: transform 0.3s ease;
}
.sidebar-open { transform: translateX(0); }

.sidebar-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.25rem 1.5rem;
  border-bottom: 1px solid #f0f0f0;
}
.sidebar-header h2 { font-size: 1.1rem; font-weight: 800; color: #1a1a1a; }

.sidebar-close {
  background: none;
  border: none;
  font-size: 1.1rem;
  cursor: pointer;
  color: #888;
  width: 32px;
  height: 32px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.15s;
}
.sidebar-close:hover { background: #f5f5f5; color: #333; }

.sidebar-empty {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.75rem;
  color: #aaa;
}
.sidebar-empty span { font-size: 3rem; }
.sidebar-empty p { font-size: 0.95rem; font-weight: 600; }

.sidebar-items {
  flex: 1;
  overflow-y: auto;
  padding: 1rem 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.sidebar-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.sidebar-img {
  width: 52px;
  height: 52px;
  border-radius: 12px;
  object-fit: cover;
  flex-shrink: 0;
}

.sidebar-info { flex: 1; }
.sidebar-name { font-size: 0.85rem; font-weight: 700; color: #1a1a1a; }
.sidebar-price { font-size: 0.82rem; color: #6abf69; font-weight: 700; }

.sidebar-qty {
  display: flex;
  align-items: center;
  gap: 6px;
  background: #f5f5f5;
  border-radius: 20px;
  padding: 4px 8px;
  font-size: 0.9rem;
  font-weight: 700;
  color: #333;
}

.sidebar-footer {
  padding: 1.25rem 1.5rem;
  border-top: 1px solid #f0f0f0;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.sidebar-total {
  display: flex;
  justify-content: space-between;
  font-size: 1rem;
  font-weight: 800;
  color: #1a1a1a;
}

.btn-checkout {
  background: #6abf69;
  color: #fff;
  border: none;
  border-radius: 20px;
  padding: 12px;
  font-size: 0.95rem;
  font-weight: 700;
  cursor: pointer;
  transition: background 0.2s;
  width: 100%;
}
.btn-checkout:hover { background: #57a857; }

/* ── PRODUCTOS ─────────────────────────────────────────────── */

.comida { padding: 2rem; }

.comida h1 {
  font-size: 1.6rem;
  font-weight: 800;
  color: #1a1a1a;
  margin-bottom: 1.5rem;
}

.productos {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 1.5rem;
}

.no-results {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.75rem;
  color: #aaa;
  padding: 4rem 0;
  grid-column: 1 / -1;
}
.no-results span { font-size: 3rem; }
.no-results p { font-size: 1rem; font-weight: 600; }

/* ── Card ──────────────────────────────────────────────────── */

.card {
  background: #ffffff;
  border-radius: 20px;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.08);
  width: 100%;
  height: 390px;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding-top: 1.5rem;
  padding-bottom: 1.25rem;
  position: relative;
}

.btn-cart {
  position: absolute;
  top: 12px;
  right: 12px;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  border: 2.5px solid #6abf69;
  background: #ffffff;
  color: #2a2a2a;
  font-size: 1.15rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s, border-color 0.2s, transform 0.1s;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.12);
}
.btn-cart:hover { background: #f0faf0; border-color: #57a857; transform: scale(1.08); }
.btn-cart:active { transform: scale(0.93); }
.btn-cart:disabled { opacity: 0.4; cursor: not-allowed; transform: none; }
.btn-cart-active {
  background: #6abf69;
  color: #fff;
  border-color: #6abf69;
}
.btn-cart-active:hover { background: #57a857; border-color: #57a857; }

.sin-stock-label {
  font-size: 0.82rem;
  font-weight: 700;
  color: #ccc;
  padding: 6px 10px;
}

.image-wrapper {
  width: 120px;
  height: 120px;
  border-radius: 50%;
  border: 5px solid #6abf69;
  overflow: hidden;
  flex-shrink: 0;
  background: #f4f4f4;
}

.product-img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}

.card-body {
  width: 100%;
  padding: 0.9rem 1.25rem 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.4rem;
  flex: 1;
  justify-content: space-between;
}

.product-name { font-size: 1.15rem; font-weight: 700; color: #1a1a1a; text-align: center; }
.product-description { font-size: 0.92rem; color: #888; text-align: center; line-height: 1.4; }

.stock-badge {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 0.85rem;
  font-weight: 600;
  padding: 4px 12px;
  border-radius: 20px;
}

.stock-dot { width: 7px; height: 7px; border-radius: 50%; flex-shrink: 0; }

.stock-ok  { background: #e8f5e9; color: #2e7d32; }
.stock-ok  .stock-dot { background: #4caf50; }
.stock-low { background: #fff8e1; color: #f57f17; }
.stock-low .stock-dot { background: #ffb300; }
.stock-out { background: #fdecea; color: #c62828; }
.stock-out .stock-dot { background: #ef5350; }

.price-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  gap: 0.5rem;
}

.price {
  font-size: 1.05rem;
  font-weight: 700;
  color: #333;
  border: 1.5px solid #ddd;
  border-radius: 20px;
  padding: 5px 16px;
  white-space: nowrap;
}

.btn-add {
  background: #6abf69;
  color: #fff;
  border: none;
  border-radius: 20px;
  padding: 7px 20px;
  font-size: 0.95rem;
  font-weight: 700;
  cursor: pointer;
  transition: background 0.2s, transform 0.1s;
  flex: 1;
}
.btn-add:hover:not(:disabled) { background: #57a857; }
.btn-add:active:not(:disabled) { transform: scale(0.97); }
.btn-add:disabled { background: #ccc; cursor: not-allowed; }

.quantity-controls {
  display: flex;
  align-items: center;
  gap: 6px;
  background: #f0f0f0;
  border-radius: 20px;
  padding: 3px 8px;
  flex: 1;
  justify-content: center;
}

.qty-btn {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  border: none;
  background: #6abf69;
  color: #fff;
  font-size: 1rem;
  font-weight: 700;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.15s, transform 0.1s;
}
.qty-btn:hover:not(:disabled) { background: #57a857; }
.qty-btn:active:not(:disabled) { transform: scale(0.93); }
.qty-btn:disabled { background: #bbb; cursor: not-allowed; }

.qty-count {
  font-size: 0.95rem;
  font-weight: 700;
  color: #333;
  min-width: 18px;
  text-align: center;
}

/* ── RESPONSIVIDAD ─────────────────────────────────────────── */

/* Tablet grande — 4 columnas */
@media (max-width: 1200px) {
  .productos {
    grid-template-columns: repeat(4, 1fr);
  }
}

/* Tablet — 3 columnas */
@media (max-width: 900px) {
  .productos {
    grid-template-columns: repeat(3, 1fr);
  }

  .header {
    padding: 0.75rem 1.25rem 0;
  }

  .sidebar {
    width: 300px;
  }
}

/* Móvil grande — 2 columnas */
@media (max-width: 600px) {
  .productos {
    grid-template-columns: repeat(2, 1fr);
    gap: 1rem;
  }

  .card {
    height: auto;
    min-height: 320px;
  }

  /* Header en 2 filas */
  .header {
    padding: 0.75rem 1rem 0;
  }

  .header-top {
    flex-wrap: wrap;
    gap: 0.6rem;
  }

  /* Fila 1: logo a la izquierda, carrito a la derecha */
  .logo {
    order: 1;
    flex: 1;
  }

  .cart-btn {
    order: 2;
    width: 42px;
    height: 42px;
    font-size: 1.1rem;
  }

  /* Fila 2: buscador ocupa todo el ancho */
  .search-wrapper {
    order: 3;
    width: 100%;
    flex: none;
    height: 42px;
    margin-bottom: 0.2rem;
  }

  .logo-img { height: 44px; }

  .comida { padding: 1rem; }
  .comida h1 { font-size: 1.3rem; }

  .sidebar { width: 100vw; }

  .pill {
    padding: 8px 14px;
    font-size: 0.82rem;
  }
}

/* Móvil pequeño — 1 columna */
@media (max-width: 420px) {
  .productos {
    grid-template-columns: 1fr;
    gap: 1rem;
  }

  .card {
    height: auto;
    min-height: 300px;
  }

  .image-wrapper {
    width: 110px;
    height: 110px;
  }

  .product-name { font-size: 1.05rem; }
  .product-description { font-size: 0.88rem; }

  .logo-img { height: 40px; }

  .search-wrapper {
    height: 40px;
    padding: 0 0.75rem;
  }

  .search-input { font-size: 0.85rem; }

  .cart-btn {
    width: 40px;
    height: 40px;
  }

  .comida h1 {
    font-size: 1.15rem;
    margin-bottom: 1rem;
  }

  /* Pills más fáciles de tocar */
  .pill {
    padding: 9px 14px;
    font-size: 0.82rem;
  }

  .price { font-size: 0.95rem; }
  .btn-add { font-size: 0.9rem; }
}

/* Mínimo absoluto — 300px */
@media (max-width: 320px) {
  .productos {
    grid-template-columns: 1fr;
    gap: 0.75rem;
  }

  .header { padding: 0.5rem 0.75rem 0; }
  .logo-img { height: 34px; }

  .cart-btn {
    width: 36px;
    height: 36px;
    font-size: 0.95rem;
  }

  .search-wrapper { height: 38px; }

  .comida { padding: 0.75rem; }

  .price {
    font-size: 0.88rem;
    padding: 4px 10px;
  }

  .btn-add {
    font-size: 0.82rem;
    padding: 6px 10px;
  }

  .sidebar { width: 100vw; }
}


/* ── FAB ───────────────────────────────────────────────────── */

.fab {
  position: fixed;
  bottom: 2rem;
  right: 2rem;
  width: 56px;
  height: 56px;
  border-radius: 50%;
  background: #6abf69;
  color: #fff;
  font-size: 2rem;
  font-weight: 300;
  border: none;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 4px 20px rgba(106, 191, 105, 0.5);
  z-index: 150;
  transition: background 0.2s, transform 0.15s;
  line-height: 1;
}
.fab:hover { background: #57a857; transform: scale(1.08); }
.fab:active { transform: scale(0.95); }

.btn-reset-stock {
  background: none;
  border: 1.5px solid #ddd;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  font-size: 1.1rem;
  color: #999;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: border-color 0.2s, color 0.2s;
}
.btn-reset-stock:hover { border-color: #6abf69; color: #6abf69; }

/* ── MODAL ─────────────────────────────────────────────────── */

.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.45);
  z-index: 400;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1rem;
}

.modal {
  background: #fff;
  border-radius: 20px;
  width: 100%;
  max-width: 500px;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
  box-shadow: 0 8px 40px rgba(0,0,0,0.18);
  overflow: hidden;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.25rem 1.5rem;
  border-bottom: 1px solid #f0f0f0;
  flex-shrink: 0;
}
.modal-header h2 { font-size: 1.1rem; font-weight: 800; color: #1a1a1a; }

.modal-close {
  background: none;
  border: none;
  font-size: 1.1rem;
  cursor: pointer;
  color: #888;
  width: 32px;
  height: 32px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.15s;
}
.modal-close:hover { background: #f5f5f5; color: #333; }

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 1.25rem 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
}

.field label {
  font-size: 0.82rem;
  font-weight: 700;
  color: #555;
}

.field input,
.field textarea,
.field select {
  border: 1.5px solid #e0e0e0;
  border-radius: 12px;
  padding: 0.6rem 0.9rem;
  font-size: 0.9rem;
  color: #333;
  outline: none;
  font-family: inherit;
  transition: border-color 0.2s;
  background: #fafafa;
}

.field input:focus,
.field textarea:focus,
.field select:focus {
  border-color: #6abf69;
  background: #fff;
}

.field textarea { resize: vertical; min-height: 70px; }

.field-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.img-preview {
  margin-top: 0.5rem;
  width: 80px;
  height: 80px;
  border-radius: 50%;
  border: 3px solid #6abf69;
  overflow: hidden;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f4f4f4;
}
.img-preview img { width: 100%; height: 100%; object-fit: cover; display: block; }
.img-error { font-size: 0.75rem; color: #e53935; margin-top: 0.25rem; }

.modal-error {
  font-size: 0.82rem;
  color: #e53935;
  font-weight: 600;
  background: #fdecea;
  padding: 8px 12px;
  border-radius: 10px;
}

.modal-footer {
  display: flex;
  gap: 0.75rem;
  padding: 1rem 1.5rem;
  border-top: 1px solid #f0f0f0;
  flex-shrink: 0;
}

.btn-cancel {
  flex: 1;
  background: #f0f0f0;
  color: #555;
  border: none;
  border-radius: 20px;
  padding: 10px;
  font-size: 0.9rem;
  font-weight: 700;
  cursor: pointer;
  transition: background 0.15s;
}
.btn-cancel:hover { background: #e0e0e0; }

.btn-save {
  flex: 2;
  background: #6abf69;
  color: #fff;
  border: none;
  border-radius: 20px;
  padding: 10px;
  font-size: 0.9rem;
  font-weight: 700;
  cursor: pointer;
  transition: background 0.2s;
}
.btn-save:hover { background: #57a857; }

@media (max-width: 420px) {
  .fab { bottom: 1.25rem; right: 1.25rem; width: 50px; height: 50px; font-size: 1.8rem; }
  .field-row { grid-template-columns: 1fr; }
}

/* ── FACTURA ───────────────────────────────────────────────── */

.modal-factura { max-width: 520px; }

.factura-body { background: #fafafa; }

.factura {
  background: #fff;
  border-radius: 16px;
  padding: 1.5rem;
  box-shadow: 0 2px 12px rgba(0,0,0,0.06);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
}

.factura-logo img {
  height: 52px;
  width: auto;
  object-fit: contain;
}

.factura-title {
  font-size: 1.4rem;
  font-weight: 800;
  color: #1a1a1a;
  margin: 0;
}

.factura-sub {
  font-size: 0.85rem;
  color: #888;
  margin: 0;
}

.factura-fecha {
  font-size: 0.8rem;
  color: #aaa;
  margin: 0;
}

.factura-divider {
  width: 100%;
  height: 2px;
  background: linear-gradient(to right, #6abf69, #bdffb0);
  border-radius: 2px;
  margin: 0.25rem 0;
}

.factura-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.85rem;
}

.factura-table thead tr {
  background: #f0faf0;
}

.factura-table th {
  padding: 8px 10px;
  text-align: left;
  font-weight: 700;
  color: #2e7d32;
  font-size: 0.8rem;
}

.factura-table td {
  padding: 7px 10px;
  color: #333;
  border-bottom: 1px solid #f5f5f5;
}

.factura-table td.center { text-align: center; }
.factura-table td.right  { text-align: right; }

.factura-total-row {
  width: 100%;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.25rem 0;
}

.factura-total-row span:first-child {
  font-size: 1rem;
  font-weight: 700;
  color: #1a1a1a;
}

.factura-total-valor {
  font-size: 1.3rem;
  font-weight: 800;
  color: #6abf69;
}

.factura-gracias {
  font-size: 0.85rem;
  color: #aaa;
  margin-top: 0.25rem;
}

/* ── TOAST ─────────────────────────────────────────────────── */

.toast {
  position: fixed;
  bottom: 2rem;
  left: 50%;
  transform: translateX(-50%);
  background: #1a1a1a;
  color: #fff;
  padding: 0.75rem 1.5rem;
  border-radius: 30px;
  font-size: 0.9rem;
  font-weight: 600;
  z-index: 500;
  box-shadow: 0 4px 20px rgba(0,0,0,0.2);
  white-space: nowrap;
}
</style>