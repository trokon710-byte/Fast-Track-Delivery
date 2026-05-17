<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Fast Trick Delivery Service</title>
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
<style>
  body { margin:0; font-family: Arial, sans-serif; background:#0f172a; color:#fff; }
  header { background:#16a34a; padding:15px; text-align:center; font-size:20px; font-weight:bold; }
  #map { height: 35vh; width: 100%; }
  .panel { padding:15px; background:#1e293b; min-height: 50vh; }
  input, button, select { padding:12px; margin:6px 0; width:100%; border:none; border-radius:8px; font-size:16px; }
  button { background:#16a34a; color:white; font-weight:bold; cursor:pointer; }
  button:disabled { background:#4b5563; cursor:not-allowed; }
  .tabs { display:flex; background:#334155; }
  .tab { flex:1; text-align:center; padding:12px; cursor:pointer; }
  .tab.active { background:#16a34a; }
  .card { background:#334155; padding:12px; margin:8px 0; border-radius:8px; }
  .payment-method { display:flex; align-items:center; gap:10px; padding:12px; margin:8px 0; border:2px solid #4b5563; border-radius:8px; cursor:pointer; }
  .payment-method.selected { border-color:#16a34a; background:#166534; }
  .price { color:#4ade80; font-weight:bold; font-size:18px; }
  .status { padding:10px; border-radius:8px; margin-top:10px; text-align:center; }
  .status.success { background:#166534; }
  .status.pending { background:#92400e; }
</style>
</head>
<body>

<header>🚚 Fast Trick Delivery Service</header>

<div class="tabs">
  <div class="tab active" onclick="showTab('shops')">Shops</div>
  <div class="tab" onclick="showTab('cart')">Cart</div>
  <div class="tab" onclick="showTab('rider')">Rider</div>
</div>

<div id="map"></div>

<div class="panel" id="shopsTab">
  <h3>Restaurants</h3>
  <div id="restaurants"></div>
  <h3>Clothes & Fashion</h3>
  <div id="stores"></div>
</div>

<div class="panel" id="cartTab" style="display:none;">
  <h3>Your Order</h3>
  <div id="cartItems"></div>
  <div id="cartTotal" class="price"></div>

  <h4>Delivery Location - Tap map to set</h4>
  <input id="deliveryAddress" placeholder="Tap map to set location" readonly>

  <h4>Payment Method</h4>
  <div class="payment-method" onclick="selectPayment('momo')">
    <span>📱</span> <div><b>MTN Mobile Money</b><br><small>Pay with MoMo</small></div>
  </div>
  <div class="payment-method" onclick="selectPayment('orange')">
    <span>🧡</span> <div><b>Orange Money</b><br><small>Pay with Orange Money</small></div>
  </div>
  <div class="payment-method" onclick="selectPayment('card')">
    <span>💳</span> <div><b>Card</b><br><small>Visa/Mastercard via Flutterwave</small></div>
  </div>
  <div class="payment-method" onclick="selectPayment('cash')">
    <span>💵</span> <div><b>Cash on Delivery</b></div>
  </div>

  <input id="customerPhone" placeholder="Phone number for payment">
  <button id="payBtn" onclick="processPayment()" disabled>Pay & Place Order</button>
  <div id="paymentStatus"></div>
</div>

<div class="panel" id="riderTab" style="display:none;">
  <h3>Rider Dashboard</h3>
  <button onclick="startRiderLocation()">Go Online & Share Location</button>
  <div id="orders"></div>
</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-app.js";
import { getFirestore, collection, addDoc, onSnapshot, doc, updateDoc, query, where } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-firestore.js";
import { getAuth, signInAnonymously } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-auth.js";

// REPLACE WITH YOUR FIREBASE CONFIG
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT.appspot.com",
  messagingSenderId: "XXX",
  appId: "XXX"
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const auth = getAuth(app);
signInAnonymously(auth);

const map = L.map('map').setView([6.3008, -10.8000], 13); // Monrovia, Liberia
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
  attribution: '© OpenStreetMap'
}).addTo(map);

let cart = [], selectedPayment = null, deliveryLatLng = null, userId;
let riderMarker = null;

// Sample shops in Monrovia - replace with real ones
const shops = {
  restaurants: [
    {id:1, name:"Bush Chicken", lat:6.3100, lng:-10.8050,
     menu:[{name:"Chicken & Rice", price:500}, {name:"Jollof Rice", price:400}]},
    {id:2, name:"Pizza Hut Monrovia", lat:6.3050, lng:-10.8000,
     menu:[{name:"Pepperoni Pizza", price:1200}, {name:"Chicken Pizza", price:1100}]}
  ],
  stores: [
    {id:3, name:"Fashion House", lat:6.3150, lng:-10.7950,
     menu:[{name:"T-Shirt", price:600}, {name:"Jeans", price:1500}]},
    {id:4, name:"City Mall Boutique", lat:6.3000, lng:-10.8020,
     menu:[{name:"Dress", price:2000}, {name:"Shoes", price:1800}]}
  ]
};

function loadShops(){
  document.getElementById('restaurants').innerHTML = shops.restaurants.map(s => `
    <div class="card" onclick="viewShop(${s.id}, 'restaurant')">
      <h4>${s.name}</h4><small>Restaurant</small>
    </div>
  `).join('');
  
  document.getElementById('stores').innerHTML = shops.stores.map(s => `
    <div class="card" onclick="viewShop(${s.id}, 'store')">
      <h4>${s.name}</h4><small>Clothes & Fashion</small>
    </div>
  `).join('');
}

window.viewShop = function(id, type){
  const allShops = [...shops.restaurants, ...shops.stores];
  const shop = allShops.find(s=>s.id===id);
  const items = shop.menu.map((item,i)=>`
    <div class="card">
      <div style="display:flex; justify-content:space-between;">
        <span>${item.name}</span><span class="price">${item.price} LRD</span>
      </div>
      <button onclick="addToCart(${id},${i})">Add to Cart</button>
    </div>
  `).join('');
  document.getElementById('shopsTab').innerHTML = `<button onclick="loadShops()">← Back</button><h3>${shop.name}</h3>`+items;
  map.setView([shop.lat, shop.lng], 15);
  L.marker([shop.lat, shop.lng]).addTo(map).bindPopup(shop.name).openPopup();
}

window.addToCart = function(shopId, itemIndex){
  const allShops = [...shops.restaurants, ...shops.stores];
  const shop = allShops.find(s=>s.id===shopId);
  cart.push({...shop.menu[itemIndex], shop:shop.name, shopLat:shop.lat, shopLng:shop.lng});
  updateCart();
  showTab('cart');
}

function updateCart(){
  const total = cart.reduce((s,i)=>s+i.price,0);
  document.getElementById('cartItems').innerHTML = cart.map(i=>`<div class="card">${i.name} - ${i.price} LRD</div>`).join('');
  document.getElementById('cartTotal').innerText = `Total: ${total} LRD`;
  document.getElementById('payBtn').disabled = cart.length===0 ||!selectedPayment ||!deliveryLatLng;
}

window.selectPayment = function(method){
  selectedPayment = method;
  document.querySelectorAll('.payment-method').forEach(el=>el.classList.remove('selected'));
  event.target.closest('.payment-method').classList.add('selected');
  updateCart();
}

map.on('click', function(e){
  deliveryLatLng = e.latlng;
  document.getElementById('deliveryAddress').value = `${e.latlng.lat.toFixed(5)}, ${e.latlng.lng.toFixed(5)}`;
  L.marker(e.latlng).addTo(map).bindPopup("Delivery Location").openPopup();
  updateCart();
});

window.processPayment = async function(){
  const total = cart.reduce((s,i)=>s+i.price,0);
  const phone = document.getElementById('customerPhone').value;
  const statusDiv = document.getElementById('paymentStatus');

  if(!phone) return alert("Enter phone number");

  statusDiv.innerHTML = `<div class="status pending">Processing ${selectedPayment} payment...</div>`;

  // DEMO MODE - Replace with real API calls
  setTimeout(async ()=>{
    if(selectedPayment === 'momo' || selectedPayment === 'orange'){
      // For real: Call your backend that uses MTN MoMo API or Orange Money API
      // POST to /api/pay with phone, amount, method
    }
    if(selectedPayment === 'card'){
      // For real: Use Flutterwave or Paystack. They work in Liberia
      // window.location = paymentLink from Flutterwave API
    }
    statusDiv.innerHTML = `<div class="status success">Payment OK! Placing order...</div>`;
    await placeOrder();
  }, 2000);
}

async function placeOrder(){
  const order = {
    items: cart,
    total: cart.reduce((s,i)=>s+i.price,0),
    paymentMethod: selectedPayment,
    phone: document.getElementById('customerPhone').value,
    deliveryLocation: deliveryLatLng,
    shopLocation: {lat: cart[0].shopLat, lng: cart[0].shopLng},
    status: "pending",
    customerId: userId,
    createdAt: Date.now()
  };
  await addDoc(collection(db, "orders"), order);
  cart = []; updateCart();
  alert("Order placed successfully!");
  showTab('shops');
  loadShops();
}

window.startRiderLocation = function(){
  navigator.geolocation.watchPosition(pos => {
    const lat = pos.coords.latitude, lng = pos.coords.longitude;
    if(riderMarker) map.removeLayer(riderMarker);
    riderMarker = L.marker([lat, lng]).addTo(map).bindPopup("Rider");
    
    const q = query(collection(db, "orders"), where("riderId", "==", userId), where("status", "==", "accepted"));
    onSnapshot(q, snap => {
      snap.forEach(docSnap => {
        updateDoc(doc(db, "orders", docSnap.id), {riderLat: lat, riderLng: lng});
      });
    });
  });
  
  const q = query(collection(db, "orders"), where("status", "==", "pending"));
  onSnapshot(q, snap => {
    document.getElementById('orders').innerHTML = "";
    snap.forEach(docSnap => {
      const o = docSnap.data();
      document.getElementById('orders').innerHTML += `
        <div class="card">
          <b>${o.items[0].shop}</b><br>
          Items: ${o.items.map(i=>i.name).join(', ')}<br>
          Total: ${o.total} LRD<br>
          Payment: ${o.paymentMethod}<br>
          <button onclick="acceptOrder('${docSnap.id}')">Accept Order</button>
        </div>
      `;
    });
  });
}

window.acceptOrder = async function(orderId){
  await updateDoc(doc(db, "orders", orderId), {status: "accepted", riderId: userId});
  alert("Order accepted!");
}

window.showTab = function(tab){
  document.getElementById('shopsTab').style.display = tab==='shops'? 'block' : 'none';
  document.getElementById('cartTab').style.display = tab==='cart'? 'block' : 'none';
  document.getElementById('riderTab').style.display = tab==='rider'? 'block' : 'none';
  document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
  event.target.classList.add('active');
}

loadShops();
auth.onAuthStateChanged(user => { if(user) userId = user.uid; });
</script>
</body>
</html>
