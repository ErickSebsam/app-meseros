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
            <button class="qty-btn" @click="removeFromCart(product)">−</button>
            <span>{{ cart[product.id] }}</span>
            <button class="qty-btn" @click="addToCart(product)" :disabled="cart[product.id] >= product.stock">+</button>
          </div>
        </div>
      </div>

      <div class="sidebar-footer" v-if="getCartTotal() > 0">
        <div class="sidebar-total">
          <span>Total</span>
          <span>${{ getCartPrice() }}</span>
        </div>
        <button class="btn-checkout">Generar factura</button>
      </div>
    </aside>

    <!-- ── PRODUCTOS ──────────────────────────────────────────── -->
    <main class="comida">
      <h1>{{ activeCategory || 'Todas las comidas' }}</h1>

      <div class="productos">
        <div class="card" v-for="product in filteredProducts" :key="product.id">

          <button class="btn-cart" @click="registerProduct(product)">🛒</button>

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
              <span v-if="product.stock === 0">Sin stock</span>
              <span v-else-if="product.stock <= 5">Solo {{ product.stock }} disponibles</span>
              <span v-else>{{ product.stock }} en stock</span>
            </div>

            <div class="price-row">
              <span class="price">${{ product.price.toFixed(2) }}</span>

              <div class="quantity-controls" v-if="cart[product.id] > 0">
                <button class="qty-btn" @click="removeFromCart(product)">−</button>
                <span class="qty-count">{{ cart[product.id] }}</span>
                <button class="qty-btn" @click="addToCart(product)" :disabled="cart[product.id] >= product.stock">+</button>
              </div>

              <button v-else class="btn-add" @click="addToCart(product)" :disabled="product.stock === 0">
                {{ product.stock === 0 ? 'Sin stock' : 'Agregar' }}
              </button>
            </div>
          </div>

        </div>

        <div class="no-results" v-if="filteredProducts.length === 0">
          <span>🍽️</span>
          <p>No se encontraron productos</p>
        </div>
      </div>
    </main>

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

const products       = ref(loadProducts())
const filteredProducts = ref([...products.value])
const cart           = ref({})
const searchQuery    = ref('')
const activeCategory = ref('')
const cartOpen       = ref(false)
const categories     = [...new Set(defaultProducts.map(p => p.category))]

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

// ── Filtros ───────────────────────────────────────────────────────────────────

function applyFilters() {
  filteredProducts.value = products.value.filter(p => {
    const matchCat    = activeCategory.value === '' || p.category === activeCategory.value
    const matchSearch = p.name.toLowerCase().includes(searchQuery.value.toLowerCase())
    return matchCat && matchSearch
  })
}

// ── Carrito ───────────────────────────────────────────────────────────────────

function getCartTotal() {
  return Object.values(cart.value).reduce((sum, qty) => sum + qty, 0)
}

function getCartPrice() {
  return products.value.reduce((total, p) => {
    return total + (p.price * (cart.value[p.id] || 0))
  }, 0).toFixed(2)
}

function addToCart(product) {
  const current = cart.value[product.id] || 0
  if (current >= product.stock) return
  cart.value[product.id] = current + 1
}

function removeFromCart(product) {
  const current = cart.value[product.id] || 0
  if (current <= 0) return
  cart.value[product.id] = current - 1
}

function registerProduct(product) {
  console.log('Registrar producto:', product)
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

.productos { display: flex; flex-wrap: wrap; gap: 2rem; }

.no-results {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.75rem;
  color: #aaa;
  padding: 4rem 0;
  width: 100%;
}
.no-results span { font-size: 3rem; }
.no-results p { font-size: 1rem; font-weight: 600; }

/* ── Card ──────────────────────────────────────────────────── */

.card {
  background: #ffffff;
  border-radius: 20px;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.08);
  width: 220px;
  height: 340px;
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

.product-name { font-size: 1rem; font-weight: 700; color: #1a1a1a; text-align: center; }
.product-description { font-size: 0.8rem; color: #888; text-align: center; line-height: 1.4; }

.stock-badge {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 0.72rem;
  font-weight: 600;
  padding: 3px 10px;
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
  font-size: 0.95rem;
  font-weight: 700;
  color: #333;
  border: 1.5px solid #ddd;
  border-radius: 20px;
  padding: 4px 14px;
  white-space: nowrap;
}

.btn-add {
  background: #6abf69;
  color: #fff;
  border: none;
  border-radius: 20px;
  padding: 6px 18px;
  font-size: 0.85rem;
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
</style>