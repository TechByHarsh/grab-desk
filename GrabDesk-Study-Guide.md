# GrabDesk Study Guide

A developer's guide to the GrabDesk codebase (`TechByHarsh/grab-desk`). It follows the path of a request from the React client to MongoDB, then walks through each backend and frontend module.

**How to use this guide:** read sections 1–2 for the big picture, then keep sections 3–5 open next to the code. Section 6 gets the app running locally. Appendix A lists inconsistencies found in the code while writing this guide. Fix them or keep them in mind.

---

## 1. Overview

GrabDesk is a **MERN** (MongoDB, Express, React, Node.js) e-commerce app with personalized product recommendations (source: `README.md`). It has three parts:

| Part | What it does | Main code |
|---|---|---|
| **Customer storefront** | Landing page, catalog with search/filters, product detail, cart, wishlist, checkout, profile (orders, addresses, messages) | `client/src/pages/`, `client/src/components/` |
| **Recommendation engine** | Scores each user's interest in product tags from clicks, wishlist, cart and purchase actions, then ranks products by tag overlap | `server/utils/recommendation.utils.js`, `getRecommendedProducts` in `server/controllers/product.controller.js` |
| **Admin dashboard** | Analytics charts, product/category/order/user management, notifications, announcements, direct messages, live activity logs | `client/src/admin/`, `server/routes/admin.routes.js` |

**Tech stack (from `package.json` files)**

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite 7, React Router 7, Tailwind CSS 4, Axios, Recharts, socket.io-client, xlsx, react-icons |
| Backend | Node.js, Express 5, Mongoose 8, jsonwebtoken, bcryptjs, socket.io 4, cloudinary 2, dotenv, cors |
| Database | MongoDB (Atlas or local) |
| Hosting (README) | Vercel (frontend), Render (backend), MongoDB Atlas |

**Repository layout**

```text
grab-desk/
├── README.md                  # project overview
├── BACKEND_ARCHITECTURE.md    # backend design notes (viva/interview style)
├── shared/constants/productConstants.js
├── client/                    # React + Vite frontend
│   └── src/
│       ├── main.jsx, App.jsx  # bootstrap + routing
│       ├── api/axios.js       # shared Axios instance (JWT interceptor)
│       ├── context/           # ToastContext, WishlistContext
│       ├── pages/             # customer pages
│       ├── components/        # reusable UI (+ charts/)
│       ├── admin/             # admin dashboard sections
│       └── utils/stringUtils.js
└── server/                    # Express backend
    ├── server.js              # entry point
    ├── config/db.js           # Mongo connection
    ├── routes/                # 5 routers
    ├── controllers/           # business logic
    ├── models/                # 8 Mongoose models
    ├── middleware/            # protect (JWT), adminOnly
    ├── utils/                 # cloudinary, categoryMapper, recommendations, notifications
    └── *.js                   # seeding, verification and test scripts
```

---

## 2. Architecture

### 2.1 Client/server split

The client and server are separate npm projects. They communicate only over HTTP (REST/JSON) and WebSockets (Socket.IO).

- **Client** (`client/`): a Vite single-page app on `http://localhost:5173`. It calls the API through one Axios instance (`client/src/api/axios.js`). The base URL is `VITE_BACKEND_URL`, or `http://localhost:5000/api` if that is not set.
- **Server** (`server/`): an Express app on `PORT` (default `5000`). It mounts every router under `/api/*` and attaches a Socket.IO server to the same HTTP server.
- **Media**: product images and profile photos go to **Cloudinary**. MongoDB stores only the URLs.

### 2.2 Request flow

```text
React component (e.g. pages/Cart.jsx)
   │  api.get('/users/cart')                       ← client/src/api/axios.js
   │  request interceptor adds  Authorization: Bearer <token from localStorage>
   ▼
Express app (server/server.js)
   │  cors() → express.json({ limit: '50mb' }) → app.use('/api/users', userRoutes)
   ▼
Router (server/routes/user.routes.js)
   │  router.use(protect)  → router.get('/cart', getCart)
   ▼
Middleware (server/middleware/auth.middleware.js)
   │  jwt.verify(token) → req.user = User.findById(decoded.userId)
   ▼
Controller (server/controllers/user.controller.js → getCart)
   │  User.findById(req.user.id).populate('cart.productId')
   ▼
Mongoose model (server/models/User.model.js, Product.model.js)
   ▼
MongoDB  → JSON response → React setState → re-render
```

The response interceptor in `axios.js` handles **401** responses for the whole app. It clears `token` and `user` from `localStorage` and redirects to `/login`:

```js
// client/src/api/axios.js
api.interceptors.request.use((config) => {
    const token = localStorage.getItem('token');
    if (token) config.headers.Authorization = `Bearer ${token}`;
    return config;
});
api.interceptors.response.use((r) => r, (error) => {
    if (error.response && error.response.status === 401) {
        localStorage.removeItem('token');
        localStorage.removeItem('user');
        if (window.location.pathname !== '/login') window.location.href = '/login';
    }
    return Promise.reject(error);
});
```

### 2.3 Socket.IO real-time events

`server.js` creates one `Server` with two channels:

- **Main namespace `/`**: clients may emit `join-admin-room` with `{ role: 'admin' }` to join the `admin-room` room.
- **`/activity` namespace**: used only for the live activity-log stream.

The `io` instance is available to controllers as `req.app.get('io')` and as `global.io` (used by `utils/adminNotify.js`).

```js
// server/server.js
const io = new Server(server, { cors: { origin: '*', methods: ['GET', 'POST'] } });
const activityNamespace = io.of('/activity');
io.on('connection', (socket) => {
  socket.on('join-admin-room', (userData) => {
    if (userData && userData.role === 'admin') socket.join('admin-room');
  });
});
global.io = io;
app.set('io', io);
```

| Event | Namespace / target | Emitted by | Consumed by |
|---|---|---|---|
| `newLog` | `/activity` | `auth.controller` (signup/login), `product.controller` (add/bulk), `order.controller` (status), `activityLog.controller`, `utils/activityLogger.js` | `admin/ActivityLogs.jsx` |
| `product:created` | `/` broadcast | `addProduct` | `admin/CategoryManagement.jsx` (refetches counts) |
| `category:created` / `:updated` / `:deleted` | `/` broadcast | `category.controller` | `admin/CategoryManagement.jsx` |
| `announcement:new` | `/` broadcast | `notification.controller.createAnnouncement` | `components/Navbar.jsx` |
| `admin-message:<userId>` | `/` broadcast (user-specific event name) | `user.controller.sendMessageToUser` | `components/Navbar.jsx` |
| `admin:activity` | room `admin-room` | `utils/adminNotify.js` (cart add, order, signup) | — (no client listener found) |

### 2.4 Cloudinary media uploads

The client sends images as **base64 data URLs** in JSON, which is why `express.json` has a 50 MB limit. `server/utils/cloudinary.js` handles them:

- `uploadToCloudinary(base64, folder)`: uploads with `quality: 'auto'`, `fetch_format: 'auto'` and a max size of 800×800. Returns `secure_url`.
- `processProductImages(images)`: uploads strings that are base64 (`isBase64Image`) and keeps existing URLs as they are.
- `uploadProfilePicture(base64)` returns `{ url, publicId }`. `deleteFromCloudinary(publicId)` deletes an old avatar.
- Configuration comes from `CLOUD_NAME`, `CLOUD_API_KEY` and `CLOUD_API_SECRET`. See `server/CLOUDINARY_SETUP.md`.
- When `NODE_ENV !== 'production'`, the module sets `NODE_TLS_REJECT_UNAUTHORIZED='0'` to work around self-signed-certificate errors. Never do this in production.

```js
// server/utils/cloudinary.js
const result = await cloudinary.uploader.upload(base64Image, {
    folder, resource_type: 'image', quality: 'auto', fetch_format: 'auto',
    transformation: [{ width: 800, height: 800, crop: 'limit' }]
});
```

If image processing fails, `addProduct` returns **500** and does not save the product, so raw base64 never ends up in MongoDB.

---

## 3. Backend (`server/`)

### 3.1 Entry point: `server/server.js`

`server.js` starts the backend in this order:

1. `dotenv.config()` loads `server/.env`.
2. `connectDB()` (`config/db.js`) runs `mongoose.connect(process.env.MONGODB_URI)` and calls `process.exit(1)` if the connection fails.
3. It creates `express()` and `http.createServer(app)`, then attaches Socket.IO (see 2.3).
4. It adds the `cors` middleware (origin `*`, methods GET/POST/PUT/PATCH/DELETE/OPTIONS) and `express.json` / `urlencoded` with a 50 MB limit.
5. It mounts the routers:

```js
app.use('/api/auth',     require('./routes/auth.routes'));
app.use('/api/users',    require('./routes/user.routes'));
app.use('/api/orders',   require('./routes/order.routes'));
app.use('/api/products', require('./routes/product.routes'));
app.use('/api/admin',    require('./routes/admin.routes'));
app.get('/api/health', (req, res) => res.json({ status: 'OK' }));
```

6. `server.listen(PORT || 5000)`.

### 3.2 Routes (`server/routes/`)

Legend: **P** = `protect` (valid JWT required), **A** = `adminOnly`.

**`auth.routes.js` → `/api/auth`** (`auth.controller.js`)

| Method | Path | Guard | Handler |
|---|---|---|---|
| POST | `/signup` | — | `signup` |
| POST | `/login` | — | `login` |
| GET | `/me` | P | `getMe` |
| PUT | `/reset-password` | P | `resetPassword` |

**`user.routes.js` → `/api/users`** (all routes behind `router.use(protect)`, handled in `user.controller.js` unless noted)

| Method | Path | Handler |
|---|---|---|
| GET | `/me` | `getMe` |
| GET | `/messages` | `getMessages` |
| PUT | `/messages/mark-read` | `markMessagesRead` |
| POST | `/profile-picture` | `uploadProfilePictureHandler` (`user.controller_PROFILE.js`) |
| GET / POST / DELETE | `/cart` | `getCart` / `addToCart` / `clearCart` |
| PUT / DELETE | `/cart/:productId` | `updateCartItem` / `removeFromCart` |
| GET / POST / DELETE | `/wishlist` | `getWishlist` / `addToWishlist` / `clearWishlist` |
| DELETE | `/wishlist/:productId` | `removeFromWishlist` |
| POST | `/address` | `addAddress` |
| PUT / DELETE | `/address/:index` | `updateAddress` / `deleteAddress` |

**`order.routes.js` → `/api/orders`** (all routes P, handled in `order.controller.js`)

| Method | Path | Handler |
|---|---|---|
| GET | `/my` | `getMyOrders` |
| POST | `/` | `createOrder` |

**`product.routes.js` → `/api/products`** (`product.controller.js`)

| Method | Path | Guard | Handler |
|---|---|---|---|
| GET | `/` | — | `getProducts` (search + pagination) |
| GET | `/recommended` | P | `getRecommendedProducts` |
| GET | `/:id` | — | `getProductById` |
| GET | `/admin/products` | P | `getAllProductsAdmin` |
| POST | `/admin/products` | P | `addProduct` |
| POST | `/admin/products/bulk` | P | `bulkCreateProducts` |
| DELETE | `/:id` | P | `deleteProduct` |

`/recommended` is declared before `/:id`, so Express does not treat the word "recommended" as a product id.

**`admin.routes.js` → `/api/admin`** (`router.use(protect); router.use(adminOnly);` applies to every route)

| Area | Routes | Controller |
|---|---|---|
| Users | `GET /users`, `PATCH /users/:id/status`, `POST /users/:id/reset-password`, `POST /send-message` | `user.controller.js` |
| Orders | `GET /orders`, `PATCH /orders/:id/status` | `order.controller.js` |
| Categories | `GET/POST /categories`, `PATCH/DELETE /categories/:id` | `category.controller.js` |
| Notifications | `GET /notifications`, `PATCH /notifications/:id/read`, `DELETE /notifications/:id`, `POST /notifications/announcement` | `notification.controller.js` |
| Notification settings | `GET /notification-settings`, `PUT /notification-settings/email`, `PUT /notification-settings/sms` | `notificationSettings.controller.js` |
| Activity logs | `GET /activity-logs`, `GET /activity-logs/latest`, `POST /activity-logs` | `activityLog.controller.js` |
| Products | `GET /products`, `POST /products`, `POST /products/bulk`, `PUT /products/:id` | `product.controller.js` |
| Analytics | `GET /analytics/stats`, `/analytics/sales-history`, `/analytics/daily-sales`, `/analytics/top-products` | `analytics.controller.js` |

### 3.3 Auth and JWT middleware

**Token creation** (`auth.controller.js`):

```js
const generateToken = (userId, role) =>
    jwt.sign({ userId, role }, process.env.JWT_SECRET, { expiresIn: '1d' });
```

- `signup`: requires `name`, `email`, `password` and `phone`. It blocks the reserved admin email `admin123@grabdesk.in` and rejects duplicate emails. It hashes the password with `bcrypt.genSalt(10)` + `bcrypt.hash`, creates the user with `role: 'customer'`, emits a `newLog` and calls `notifyAdminActivity({ actionType: 'user_joined' })`. It returns `{ token, user }`.
- `login`: runs `bcrypt.compare(password, user.passwordHash)`, updates `lastLogin`, emits a `newLog` and returns `{ token, user }`. Wrong credentials get a **401**.
- `resetPassword`: checks `currentPassword` and requires `newPassword` to be at least 6 characters, then re-hashes.
- No API endpoint creates admins. An admin account has to be created directly in the database (`role: 'admin'`).

**`protect`** (`middleware/auth.middleware.js`) reads `Authorization: Bearer <token>`, verifies it, and **loads the user from the database on every request**, so a deleted user loses access immediately:

```js
const decoded = jwt.verify(token, process.env.JWT_SECRET);
req.user = await User.findById(decoded.userId).select('-passwordHash');
if (!req.user) return res.status(401).json({ message: 'Not authorized, user not found' });
next();
```

It returns separate 401 messages for `JsonWebTokenError`, `TokenExpiredError` and a missing token.

**`adminOnly`** (`middleware/admin.middleware.js`): `req.user.role === 'admin'`, otherwise **403**.

### 3.4 Mongoose models (`server/models/`)

Every model except `AdminNotification` and `NotificationSettings` uses `{ timestamps: true }`, which adds `createdAt` and `updatedAt`.

#### `User.model.js` → `User`

| Field | Type / constraints |
|---|---|
| `name` | String, required |
| `email` | String, required, **unique**, lowercase, trim |
| `passwordHash` | String, required (bcrypt hash) |
| `phone` | String, required |
| `role` | enum `customer` \| `admin` \| `moderator`, default `customer` |
| `status` | enum `Active` \| `Blocked`, default `Active` |
| `profilePhotoUrl`, `cloudinaryPublicId` | String (Cloudinary avatar) |
| `addresses` | `[Object]` (free-form `{ street, city, state, country, zipCode, ... }`) |
| `cart` | `[{ productId → Product, quantity, size, color }]` |
| `wishlist` | `[ObjectId → Product]` |
| `personalizedTags` | `Map<String, Number>`, tag → interest score |
| `lastLogin` | Date |
| `messages` | `[{ title, description, read, createdAt }]` (admin DMs and announcements) |

**Relationships:** `cart.productId` and `wishlist` reference `Product`. Cart and wishlist are embedded in the user document; there is no separate Cart collection. **Indexes:** a unique index on `email`.

#### `Product.model.js` → `Product`

| Field | Type / constraints |
|---|---|
| `name` | String, required, trim |
| `description` | String |
| `basePrice` | Number, required |
| `discountPercent` | Number 0–100, default 0 |
| `finalPrice` | Number, required (computed) |
| `stock` | Number, required, default 0 |
| `sku` | String, **unique + sparse** |
| `isActive` | Boolean, default true |
| `category` | String, default `Electronics`, **indexed** (no enum; validated in the controller) |
| `categories` | `[String]`, defaults to `[category]` |
| `tags` | `[tagSchema]`: `{ type: 'category' / 'attribute' / 'usage', value, weight=1 }` (no `_id`) |
| `brand` | String, default `Grabdesk` |
| `model`, `color`, `material` | String |
| `sizeAvailable`, `images` | `[String]` |
| `ratingAverage`, `ratingCount` | Number |

`finalPrice` is recalculated in a pre-save hook:

```js
productSchema.pre('save', function (next) {
    if (this.isModified('basePrice') || this.isModified('discountPercent')) {
        this.finalPrice = this.basePrice - (this.basePrice * (this.discountPercent / 100));
    }
    next();
});
```

**Indexes:** `category` (single field) and `sku` (unique, sparse). Sparse means products without an SKU do not collide with each other.

#### `Order.model.js` → `Order`

| Field | Type / constraints |
|---|---|
| `user` | ObjectId → `User`, required |
| `items[]` | `{ product → Product, name, image, price, quantity (min 1) }`, a snapshot taken at purchase time |
| `shippingAddress` | `{ fullName, phone, street, city, state, country, zipCode }`, all required |
| `pricing` | `{ subtotal, shipping, tax, total }`, all required |
| `deliveryMethod` | enum `standard` \| `express` |
| `status` | enum `pending` \| `paid` \| `shipped` \| `delivered` \| `cancelled`, default `pending` |

#### `Category.model.js` → `Category`

`name` (String, required, trim, **unique**) and `productCount` (Number). Admins create these "managed" categories. The `productCount` returned by the API is calculated at request time (see 3.5).

#### `Notification.model.js` → `Notification`

`type` (enum `order` \| `stock` \| `user` \| `system`), `message`, `read`. These are the admin notification-center items, including stored announcements.

#### `AdminNotification.model.js` → `AdminNotification`

`actionType` (enum `cart_add` \| `order_placed` \| `user_joined`), `userId` → `User`, `username`, `userEmail`, `entityId`, `entityName`, `read`, `createdAt`. **Indexes:** `{ createdAt: -1 }`, `{ read: 1 }`.

#### `NotificationSettings.model.js` → `NotificationSettings`

`emailEnabled`, `smsEnabled`, `emailTemplates` / `smsTemplates` (`Map<String,String>`), `emailProvider` (enum `sendgrid` \| `mailgun` \| `smtp` \| `none`), `smsProvider` (enum `twilio` \| `nexmo` \| `none`), `lastUpdated`. There is one settings document. The app stores this configuration only; it never sends email or SMS.

#### `ActivityLog.model.js` → `ActivityLog`

`message` and `type` (enum `system` \| `auth` \| `product` \| `order` \| `user`). This is the persisted copy of what streams over `/activity`.

```text
User 1───* Order            (Order.user)
User *───* Product          (User.cart[].productId, User.wishlist[])
Order *──* Product          (Order.items[].product — snapshot of name/price/image)
User 1───* AdminNotification (AdminNotification.userId)
Product.category / categories ··· Category.name   (matched by string, not ObjectId)
```

### 3.5 Product catalog and category hierarchy

**`product.controller.js` key functions**

- `generateSKU(name, category)` builds `NAM-CA-123456`: the first 3 letters of the name, the first 2 of the category, and the last 6 digits of `Date.now()`.
- `normalizeBrandName(brand)` maps known brands to their proper case (`'hp' → 'HP'`). Other brands get title case. An empty brand becomes `Grabdesk`.
- `getProducts`: public catalog with a page size of **27**, using `?pageNumber=N&keyword=...`. The keyword is a case-insensitive `$regex` across `name`, `tags.value`, `brand`, `category` and `categories`. Results are sorted by `createdAt: -1`. Each product gets `resolvedCategories`, `primaryCategory` and `categoryHierarchy` (from the category mapper) added at response time. Returns `{ products, page, pages }`.
- `getProductById`: returns one product. If `req.user` is set, it scores a `CLICK` without waiting for the result.
- `addProduct`: validates the name, the category (an allow-list in the controller), the price (> 0) and the stock (≥ 0). It then calculates `finalPrice`, uploads images (`processProductImages`), and converts `tags: string[]` into `tagSchema` objects. After saving, it emits `newLog` and `product:created`. A duplicate key returns **409** and a `ValidationError` returns **400**.
- `bulkCreateProducts`: takes `{ products: [...] }` from the Excel/CSV upload. It validates each row, collects errors in `failures[]` with the row number, inserts the valid rows, and reports both.
- `getAllProductsAdmin`, `updateProduct` (`PUT /api/admin/products/:id`), `deleteProduct`.

**Recommendations** (`getRecommendedProducts` + `utils/recommendation.utils.js`)

```js
const ACTION_WEIGHTS = { CLICK: 1, WISHLIST: 2, CART: 3, PURCHASE: 5 };
// per tag: increment = actionWeight * tag.weight * multiplier
// multiplier: attribute = 1, usage = 0.5, category = 0.2
// keep only the top MAX_TAGS = 25 tags; decayUserScores() multiplies by 0.9
```

- Calls to `updatePersonalizedTags(user, product, action)` happen in `addToCart` (CART), `addToWishlist` (WISHLIST), `createOrder` (PURCHASE) and `getProductById` (CLICK).
- **Cold start:** a user with no tags gets the top 8 active, in-stock products by `ratingAverage`.
- **Personalized:** takes the user's top 5 tags and finds products whose `tags.value` is in that list. Each product scores `Σ userWeight × productTagWeight`. Results are sorted by score, then rating, then newest. The top 8 are returned.

**Category hierarchy** (`utils/categoryMapper.js`, pure functions with no database writes)

- `CATEGORY_HIERARCHY` has 11 top-level categories: Home & Living, Electronics, Fashion, Beauty & Personal Care, Sports & Fitness, Books & Stationery, Grocery, Toys & Baby Products, Automotive, Health, Office Supplies. Each has `subcategories`, `keywords` and `tags`. Only **Home & Living** has subcategories: `Furniture`, `Kitchen`, `Storage`.
- `SUBCATEGORY_TO_PARENT` is the reverse lookup, and `ALL_CATEGORIES` is the list of top-level keys.
- `inferCategoriesFromProduct(product)` applies keyword rules over the name, description and tags. For example, a name containing *chair* adds Furniture and Home & Living, and *office chair* also adds Office Supplies.
- `resolveProductCategories(product)` combines the stored `category`/`categories`, the inferred categories and the parents of any subcategories. It returns `{ primary, all, hierarchy }`.
- `calculateCategoryProductCounts(products, categories)` counts each product once per resolved category.

`category.controller.getAllCategories` uses a **hybrid** approach. It loads the managed `Category` documents, resolves the categories of every product, fills in the counts, and adds **auto-detected** categories (`isManaged: false`, `id: 'auto-<name>'`) for categories that have products but no `Category` document. `addCategory`, `updateCategory` and `deleteCategory` emit `category:*` socket events.

### 3.6 Cart and order logic

**Cart** (`user.controller.js`): the cart is embedded in `User.cart`.

- `addToCart({ productId, quantity, size, color })`: if a line already has the same **product + size + color**, its quantity goes up; otherwise a new line is pushed. It then scores `CART` and calls `notifyAdminActivity('cart_add')`. It returns the full cart in the shape the frontend expects (`id, title, price=finalPrice, originalPrice=basePrice, image, color, size, quantity, inStock, maxQuantity`).
- `getCart` populates `cart.productId` and drops lines whose product was deleted.
- `updateCartItem`, `removeFromCart`, `clearCart`. The wishlist functions follow the same pattern.

**Orders** (`order.controller.js`)

`createOrder` **recalculates prices on the server** and ignores prices sent by the client:

```js
for (const item of items) {
    const product = await Product.findById(item.product || item.productId);
    const price = product.finalPrice;
    subtotal += price * item.quantity;
    orderItems.push({ product: product._id, name: product.name,
                      image: product.images?.[0] || '', price, quantity: item.quantity });
}
const shipping = deliveryMethod === 'express' ? 15.0 : subtotal > 50 ? 0 : 5.99;
const tax = subtotal * 0.085;
const total = subtotal + shipping + tax;
```

After saving, it notifies admins (`order_placed`), scores `PURCHASE` for each item, and clears the cart with `$set: { cart: [] }`. `Checkout.jsx` uses the same formulas (`calculateShipping`, `calculateTax`) so the totals on screen match the server.

- `getMyOrders`: the current user's orders, newest first.
- `getAllOrders` (admin): populates `user`, then reshapes each order for the table: title-cased `status`, `total` formatted as `₹x.xx`, `payment: 'Paid'` (hard-coded), and an `en-IN` date.
- `updateOrderStatus` (admin): lowercases the status, validates it against an allow-list, saves with `runValidators`, and emits `newLog`.
- Stock is **not** decremented when an order is placed.

**Analytics** (`analytics.controller.js`, aggregation pipelines that skip `cancelled` orders)

- `getDashboardStats` returns `totalSales` (sum of `pricing.total`), `totalProducts`, `activeUsers` (`status: 'Active'`), and the top product (`$unwind` items, `$group` by product, `$sum` quantity).
- `getSalesHistory` groups the last 6 months by `$month`/`$year`. `getDailySalesHistory` groups the last 30 days by day. `getTopProducts` returns the top 5 products by quantity.

### 3.7 Notifications, messaging and activity logs

There are three independent systems:

1. **Admin notification center** (`Notification` model, `notification.controller.js`): `getAllNotifications` (adds a relative time via `formatTime`), `toggleNotificationRead`, `deleteNotification`, and `createAnnouncement`. `createAnnouncement` saves a `system` notification, pushes `{ title: 'Announcement', description }` into **every** user's `messages` with `User.bulkWrite`, and emits `announcement:new`.
2. **Direct messages**: `sendMessageToUser({ emailOrUsername, title, message })` finds the user by email or name, pushes the message into `user.messages`, and emits `admin-message:<userId>`. Users read messages with `GET /api/users/messages` and mark them read with `PUT /messages/mark-read`. `Navbar.jsx` shows the unread count.
3. **Passive admin activity feed** (`utils/adminNotify.js → notifyAdminActivity`): writes an `AdminNotification` and emits `admin:activity` to `admin-room`. It never throws, so it cannot break the request that triggered it.
4. **Activity logs** (`activityLog.controller.js`, `utils/activityLogger.js → emitActivityLog(io, message, type)`): `ActivityLog.create` followed by `io.of('/activity').emit('newLog', ...)`.

`notificationSettings.controller.js` reads and updates the single `NotificationSettings` document (email/SMS toggles, providers and templates).

`user.controller_ADMIN.js`, `user.controller_MESSAGING.js` and `announcement.controller.js` are **not imported by any route**. They are older copies of logic that now lives in `user.controller.js` and `notification.controller.js`.

### 3.8 Seeding, import and maintenance scripts

Run each script from `server/` with `node <script>`. Most of them read `server/.env`.

| Script | Purpose |
|---|---|
| `generate-products.js` | Takes a hard-coded list of office/home products, fetches an image URL for each from the Unsplash API (`UNSPLASH_ACCESS_KEY`, 100 ms between requests), and writes **`products-bulk-upload.csv`**. Does not touch the database. |
| `import-products-csv.js` | Reads `server/products-bulk-upload.csv` (`parseCSV` / `parseCSVLine` handle quoted fields), skips duplicates and existing SKUs, generates an SKU per product (`generateSKU`), converts tag strings to `tagSchema`, and calls `Product.create` for each row. This is the main **seeding** step. |
| `deduplicate_safe.js` | Finds products with the same `name|category` and deletes the older duplicates (keeps the newest). |
| `verify_counts.js` | Prints the total product count and the number of low-stock products (`stock < 10`). |
| `verify-category-mapping.js` | Checks `categoryMapper` against real data. Uses `MONGO_URI` (note the different variable name). |
| `test-enhanced-mapping.js` | Offline check of `resolveProductCategories` on sample products. |
| `test-recommendations.js` | Creates a test user and runs `updatePersonalizedTags` against the database. |
| `verify-auth.js`, `verify-system.js`, `test-add-product.js`, `test-activity-logs.js` | HTTP smoke tests against a running server on `localhost:5000`. |
| `debug_orders.js` | Order normalization debug script. |

Typical seeding flow:

```bash
cd server
node generate-products.js      # → products-bulk-upload.csv (needs UNSPLASH_ACCESS_KEY)
node import-products-csv.js    # → inserts into MongoDB (needs MONGODB_URI)
node deduplicate_safe.js       # optional clean-up
```

You can also bulk-upload an `.xlsx`/`.csv` file from the admin dashboard (`BulkUploadModal`), which calls `POST /api/admin/products/bulk`.

---

## 4. Frontend (`client/src/`)

### 4.1 Bootstrap, routing and protected routes

`main.jsx` renders `<App />` inside `StrictMode`. `App.jsx` wraps the app in providers and declares every route:

```jsx
// client/src/App.jsx
<ToastProvider>
  <WishlistProvider>
    <BrowserRouter>
      <ScrollToTop />
      <Routes>
        <Route path='/' element={<Landing />} />
        <Route path='/login' element={<Login />} />
        <Route path='/signup' element={<Signup />} />
        <Route path='/home' element={<Home />} />
        <Route path='/product' element={<Product />} />
        <Route path='/product/:id' element={<ProductDetail />} />
        <Route path='/checkout' element={<ProtectedRoute><Checkout /></ProtectedRoute>} />
        <Route path='/cart' element={<ProtectedRoute><Cart /></ProtectedRoute>} />
        <Route path='/profile' element={<ProtectedRoute><Profile /></ProtectedRoute>} />
        <Route path='/wishlist' element={<ProtectedRoute><WishlistPage /></ProtectedRoute>} />
        <Route path='/admindashbord' element={
          <ProtectedRoute adminOnly={true}><AdminDashbord /></ProtectedRoute>} />
        <Route path='/contact' element={<Contact />} />
        <Route path='/blog' element={<Blog />} />
      </Routes>
    </BrowserRouter>
  </WishlistProvider>
</ToastProvider>
```

| Path | Component | Access |
|---|---|---|
| `/` | `Landing` | public |
| `/login`, `/signup` | `Login`, `Signup` | public |
| `/home` | `Home` | public (recommendations require a login) |
| `/product`, `/product/:id` | `Product`, `ProductDetail` | public |
| `/cart`, `/checkout`, `/profile`, `/wishlist` | `Cart`, `Checkout`, `Profile`, `WishlistPage` | logged in |
| `/admindashbord` (spelled this way in the code) | `AdminDashboard` | admin |
| `/contact`, `/blog` | `Contact`, `Blog` | public |

`components/ProtectedRoute.jsx` checks the login state **on the client** from `localStorage`. The server still enforces access with `protect` and `adminOnly`.

```jsx
const ProtectedRoute = ({ children, adminOnly = false }) => {
    const token = localStorage.getItem('token');
    const user = JSON.parse(localStorage.getItem('user') || 'null');
    if (!token || !user) return <Navigate to="/login" state={{ from: location }} replace />;
    if (adminOnly && user.role !== 'admin') return <Navigate to="/home" replace />;
    return children;
};
```

**Login flow** (`pages/Login.jsx`): `api.post('/auth/login')` stores `token` and `user` (JSON) in `localStorage`, then navigates to `/admindashbord` if `role === 'admin'` and to `/home` otherwise. `Signup.jsx` works the same way and always goes to `/home`.

### 4.2 Context providers (`client/src/context/`)

- **`ToastContext.jsx`**: `ToastProvider` and the `useToast()` hook expose `addToast(message, type='success', duration=3000)` and `removeToast(id)`. It renders a fixed `ToastContainer` with animated `ToastItem`s.
- **`WishlistContext.jsx`**: `WishlistProvider` and `useWishlist()` expose `wishlist`, `addToWishlist(product)`, `removeFromWishlist(id)`, `isInWishlist(id)` and `clearWishlist()`. It loads `/users/wishlist` on mount (only when a token exists), calls the API for each change, and shows toasts. It must sit inside `ToastProvider` because it calls `useToast()`.

```jsx
const addToWishlist = async (product) => {
    if (!localStorage.getItem('token')) { addToast("Please login to add to wishlist", "error"); return; }
    if (isInWishlist(product.id)) { addToast("Item already in wishlist", "info"); return; }
    const res = await api.post('/users/wishlist', { productId: product.id });
    setWishlist(res.data);
    addToast("Added to wishlist", "success");
};
```

There is **no auth or cart context**. Pages read `localStorage` and call the API directly. The cart is fetched on each page that needs it.

### 4.3 Customer pages (`client/src/pages/`)

| Page | Key behaviour | API calls |
|---|---|---|
| `Landing.jsx` | Marketing page (hero images, mission, CTA buttons via `Button` / `ButtonLight`) | — |
| `Home.jsx` | Category shortcuts (navigate to `/product` with `state.category`), image slider, **"Recommended for you"** grid | `GET /products/recommended` |
| `Product.jsx` | Catalog grid with `Searchbar`, `FilterPannel` (price, brand, category, ...), pagination (`pageNumber`, `pages`) | `GET /products?pageNumber=N` |
| `ProductDetail.jsx` | Uses `useParams().id`, has size/color/quantity selectors, add-to-cart, buy-now (goes to `/cart`), `ProductReviews`, related products (same category, max 4) | `GET /products/:id`, `GET /products`, `POST /users/cart` |
| `Cart.jsx` | Lists `CartCard`s, changes quantities, removes items, clears the cart, shows the order summary | `GET/PUT/DELETE /users/cart[...]` |
| `Checkout.jsx` | Shipping form, choose a saved address or add one, delivery method (standard/express), `handlePlaceOrder` builds `orderPayload`, then redirects to `/profile?tab=orders` | `GET /users/cart`, `GET /users/me`, `POST /users/address`, `POST /orders` |
| `Profile.jsx` | Tabs (`activeTab`): account, orders, addresses (add/edit/delete), messages (mark read), password reset (`ResetPasswordInput`), avatar upload (base64 → Cloudinary) | `/users/me`, `/orders/my`, `/users/address`, `/users/profile-picture`, `/auth/reset-password`, `/users/messages/mark-read` |
| `WishlistPage.jsx` | `WishlistHeader` with sort and clear-all; reads from `useWishlist()` | via context |
| `Login.jsx`, `Signup.jsx` | Forms built with `AuthLayout` / `AuthInput`, client-side `validate()` | `/auth/login`, `/auth/signup` |
| `Contact.jsx`, `Blog.jsx` | Thin wrappers (`ContactPage`, static content) | — |

Important shared components (`client/src/components/`):

- `Navbar.jsx`: navigation and user menu. It opens a Socket.IO connection and listens for `admin-message:<userId>` and `announcement:new`, keeps an unread counter, and can show a browser `Notification`.
- `ProductCard.jsx`: grid card that opens the detail page, adds to the cart (redirects to `/login` without a token), and toggles the wishlist. It falls back to other images when one fails to load.
- `FilterPannel.jsx`, `FilterBar.jsx`, `Searchbar.jsx`: catalog filtering UI.
- `CartCard.jsx`, `RelatedProduct.jsx`, `ProductReviews.jsx` (reviews are local component state with no backend), `ImageSlider.jsx`, `Banner.jsx`, `Footer.jsx`, `Modal.jsx`, `ScrollToTop.jsx`.

### 4.4 Admin dashboard (`client/src/admin/`)

`AdminDashboard.jsx` (component `AdminManager`) is the admin shell. It has a collapsible sidebar, the `Navbar`, `SidebarProfile`, and an `activeSection` state (default `"products"`) that selects which section renders. It fetches `/admin/products` and `/users/me` on mount and holds shared state (`products`, `orders`, `users`, `categories`) that it passes down to the sections. `handleLogout` clears `localStorage`.

| Section component | File | What it does | API / events |
|---|---|---|---|
| `AnalyticsDashboard` | `AnalyticsDashboard.jsx` | Stat cards (sales, products, active users, top product). `SalesChart` (Recharts line/area) and `ProductsChart` (bar) in `components/charts/`. A **`viewMode`** toggle switches between `monthly` and `daily`. | `/admin/analytics/stats`, `sales-history` or `daily-sales`, `top-products`. Refetches on socket events. |
| `ProductManagement` | `ProductManager.jsx` | Grid of `AdminProductCard`s, `AddProductForm` (3 required image slots, base64), `EditProductModal`, `BulkUploadModal` (parses Excel/CSV with `XLSX.read` → `sheet_to_json`), `ProductPreviewDrawer`. Applies changes to the UI before the server responds (optimistic updates). | `POST /admin/products`, `POST /admin/products/bulk`, `PUT /admin/products/:id`, `DELETE /products/:id` |
| `CategoryManagement` | `CategoryManagement.jsx` | Lists managed and auto-detected categories with counts; add, rename and delete. | `/admin/categories*`. Listens for `category:*` and `product:*`. |
| `OrderManagement` | `OrderManager.jsx` | Order table with a status filter and a status-change modal. | `GET /admin/orders`, `PATCH /admin/orders/:id/status` |
| `UserManagement` | `UserManager.jsx` | User table with role filter and search; block/unblock; password reset. | `GET /admin/users`, `PATCH /admin/users/:id/status`, `POST /admin/users/:id/reset-password` |
| `UserMessaging` | `UserMessaging.jsx` | Sends a DM to one user by email or name. | `POST /admin/send-message` |
| `NotificationCenter` | `NotificationSystem.jsx` | Notification list (mark read, delete), announcement composer, email/SMS settings. | `/admin/notifications*`, `/admin/notification-settings*` |
| `ActivityLogs` | `ActivityLogs.jsx` | Live log feed. | `GET /admin/activity-logs`, `/latest`. Socket `io('http://localhost:5000/activity')` with `on('newLog')`. |

The bulk upload parses the file like this:

```jsx
// client/src/components/BulkUploadModal.jsx
const workbook = XLSX.read(evt.target.result, { type: "binary" });
const sheet = workbook.Sheets[workbook.SheetNames[0]];
const json = XLSX.utils.sheet_to_json(sheet);   // → onUpload(rows) → POST /admin/products/bulk
```

---

## 5. Shared code (`shared/`)

`shared/constants/productConstants.js` is described as the "single source of truth" for product enums. It is written as an **ES module**:

```js
export const ALLOWED_CATEGORIES = [
  'Electronics', 'Fashion', 'Home & Living', 'Beauty & Personal Care',
  'Sports & Fitness', 'Books & Stationery', 'Grocery', 'Toys & Baby Products'
];
export const ALLOWED_BRANDS = ['Apple', 'Samsung', 'Nike', 'Adidas', 'Puma', 'Sony',
  'LG', 'Microsoft', 'Canon', 'Dell', 'HP', 'Lenovo'];
export const DEFAULT_CATEGORY = 'Electronics';
export const DEFAULT_BRAND = 'Grabdesk';
```

**Validation rules and where they actually live**

| Rule | Where it is enforced |
|---|---|
| Category must be in the allow-list (new products only) | `addProduct` in `product.controller.js` (its own inline `ALLOWED_CATEGORIES`) |
| Brand: any value allowed, normalized | `normalizeBrandName` in `product.controller.js` |
| `basePrice > 0`, `stock >= 0`, `0 <= discountPercent <= 100` | `addProduct` / `bulkCreateProducts` and `Product` schema `min`/`max` |
| Unique email (lowercased) | `User` schema |
| Password at least 6 characters (on reset) | `resetPassword` |
| Order status allow-list | `updateOrderStatus` + `Order` schema enum |
| Required shipping-address fields | `Order` schema |

**Note:** no file in `client/` or `server/` imports `shared/`. The server is CommonJS and keeps its own copies. `Product.model.js` defines a 14-category / 13-brand list, while `shared/` has 8 categories and 12 brands. When you change categories, update `shared/`, `Product.model.js`, the allow-list in `addProduct` and `CATEGORY_HIERARCHY` together, or make both sides import the shared file.

---

## 6. Getting started

### 6.1 Prerequisites

- Node.js 18 or newer and npm (Vite 7 and Express 5)
- A MongoDB connection string (Atlas or local `mongodb://localhost:27017/grabdesk`)
- Optional: a Cloudinary account (image uploads) and an Unsplash API key (seed CSV generation)

### 6.2 Environment variables

**`server/.env`** (ignored by git):

```env
MONGODB_URI=mongodb://localhost:27017/grabdesk   # read by config/db.js
JWT_SECRET=change_me                             # signs/verifies JWTs
PORT=5000                                        # optional, default 5000
CLOUD_NAME=your_cloud_name                       # Cloudinary
CLOUD_API_KEY=your_api_key
CLOUD_API_SECRET=your_api_secret
NODE_ENV=development                             # 'production' re-enables TLS checks
UNSPLASH_ACCESS_KEY=...                          # only for generate-products.js
MONGO_URI=...                                    # only for verify-category-mapping.js
```

> The README and `CLOUDINARY_SETUP.md` say `MONGO_URI`, but the server reads **`MONGODB_URI`**. Set `MONGODB_URI`, or the server exits on startup.

**`client/.env`** (optional):

```env
VITE_BACKEND_URL=http://localhost:5000/api
```

The Socket.IO URLs in `Navbar.jsx` and the admin components are hard-coded to `http://localhost:5000`.

### 6.3 Install and run

```bash
git clone https://github.com/TechByHarsh/grab-desk.git
cd grab-desk

# Backend
cd server
npm install
npm run dev        # nodemon server.js   (or: npm run server → node server.js)

# Frontend (new terminal)
cd client
npm install
npm run dev        # Vite on http://localhost:5173
```

| Project | Script | Command |
|---|---|---|
| server | `server` | `node server.js` |
| server | `dev` | `nodemon server.js` |
| client | `dev` | `vite` |
| client | `build` | `vite build` (writes to `client/dist/`) |
| client | `preview` | `vite preview` |
| client | `lint` | `eslint .` |

### 6.4 First-run checklist

1. Open `GET http://localhost:5000/api/health` and check for `{ "status": "OK" }`.
2. Seed products with `node import-products-csv.js` (section 3.8), or add them from the admin UI.
3. Sign up through `/signup` (creates a customer).
4. Create an admin by setting `role: 'admin'` on a user document in MongoDB, then log in. You are redirected to `/admindashbord`.
5. Browse, add to the cart and check out. Then confirm that the order appears in **Admin → Orders** and that `/home` now shows personalized recommendations.

---

## Appendix A: Inconsistencies found in the code

The following points were found while reading the code for this guide. They are worth fixing or keeping in mind:

1. **Product admin routes are not admin-only.** In `product.routes.js`, `POST /api/products/admin/products`, `.../bulk`, `GET .../admin/products` and `DELETE /api/products/:id` use only `protect`, so any logged-in customer can call them. The matching `/api/admin/products*` routes are correctly guarded with `adminOnly`.
2. **CLICK personalization never runs.** `GET /api/products/:id` has no `protect`, so `req.user` is always undefined in `getProductById`.
3. **Order status enums do not match.** `updateOrderStatus` accepts `processing`, which the `Order` schema enum does not have, so `runValidators` rejects it. The schema has `paid`, which the controller rejects.
4. **Analytics socket listeners never fire.** `AnalyticsDashboard.jsx` listens for `newLog` on the main namespace, but the server emits `newLog` on `/activity`. Nothing emits `orderPlaced` or `userRegistered`.
5. **`admin:activity` has no listener.** Clients never emit `join-admin-room`, so `admin-room` is empty.
6. **Environment variable names differ:** `MONGODB_URI` (server) vs `MONGO_URI` (README, `CLOUDINARY_SETUP.md`, `verify-category-mapping.js`).
7. **Page size:** `getProducts` uses 27, `BACKEND_ARCHITECTURE.md` says 28, and `FilterPannel.jsx` passes a `pageSize=1000` query parameter that the server ignores.
8. **Dead code:** `user.controller_ADMIN.js`, `user.controller_MESSAGING.js` and `announcement.controller.js` are unused. `passport` and `passport-google-oauth20` are installed but never imported. `shared/` is not imported anywhere.
9. **No stock decrement** on `createOrder`. **Payment** is shown as `'Paid'` for every order.
10. **`debug_orders.js`** populates `userId`, but the field is `user`.
11. **Security in development:** CORS `origin: '*'` and `NODE_TLS_REJECT_UNAUTHORIZED='0'` are set outside production. Restrict both before deploying.
