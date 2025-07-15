# Ecommerce_Assignment


 // Create a .env file with the following keys:
PORT=5000  
MONGO_URI=mongodb://localhost:27017/ecommerce  
JWT_SECRET=yourSecretKey

// In server.js, connect to MongoDB using Mongoose and start the server

const app = require('./app');  
const mongoose = require('mongoose');  
require('dotenv').config();  
mongoose.connect(process.env.MONGO_URI)  
.then(() => {  
  console.log("MongoDB connected");  
  app.listen(process.env.PORT, () => console.log(`Server running on port ${process.env.PORT}`));  
})  
.catch(err => console.log(err));

// In app.js, set up Express middleware and routes:
const express = require('express');  
const cors = require('cors');  
const app = express();  
app.use(cors());  
app.use(express.json());  
app.use('/api/auth', require('./routes/authRoutes'));  
app.use('/api/products', require('./routes/productRoutes'));  
app.use('/api/cart', require('./routes/cartRoutes'));  
app.use('/api/orders', require('./routes/orderRoutes'));  
module.exports = app;

// Define the User model in models/User.js:

const mongoose = require('mongoose');  
const bcrypt = require('bcryptjs');  
const userSchema = new mongoose.Schema({  
  username: { type: String, required: true, unique: true },  
  password: { type: String, required: true },  
  role: { type: String, enum: ['customer', 'admin'], default: 'customer' },  
  cart: [{ productId: String, quantity: Number }]  
});  
userSchema.pre('save', async function () {  
  if (this.isModified('password')) {  
    this.password = await bcrypt.hash(this.password, 10);  
  }  
});  
module.exports = mongoose.model('User', userSchema);

// Define the Product model in models/Product.js:

const mongoose = require('mongoose');  
const productSchema = new mongoose.Schema({  
  name: String,  
  category: String,  
  price: Number,  
  description: String  
});  
module.exports = mongoose.model('Product', productSchema);

// Define the Order model in models/Order.js:

const mongoose = require('mongoose');  
const orderSchema = new mongoose.Schema({  
  userId: String,  
  items: [{ productId: String, quantity: Number }],  
  createdAt: { type: Date, default: Date.now }  
});  
module.exports = mongoose.model('Order', orderSchema);

Create middleware/authMiddleware.js to verify JWT:

const jwt = require('jsonwebtoken');  
module.exports = function (req, res, next) {  
  const token = req.headers.authorization?.split(" ")[1];  
  if (!token) return res.status(401).json({ message: "Unauthorized" });  
  try {  
    const decoded = jwt.verify(token, process.env.JWT_SECRET);  
    req.user = decoded;  
    next();  
  } catch {  
    res.status(401).json({ message: "Invalid token" });  
  }  
};

Create middleware/roleMiddleware.js to check admin role:
module.exports = function (role) {  
  return (req, res, next) => {  
    if (req.user.role !== role) return res.status(403).json({ message: "Forbidden" });  
    next();  
  };  
};

In controllers/authController.js, implement register and login:

const User = require('../models/User');  
const jwt = require('jsonwebtoken');  
const bcrypt = require('bcryptjs');  
exports.register = async (req, res) => {  
  try {  
    const user = await new User(req.body).save();  
    res.json({ message: "User registered" });  
  } catch (err) {  
    res.status(400).json({ error: err.message });  
  }  
};  
exports.login = async (req, res) => {  
  const { username, password } = req.body;  
  const user = await User.findOne({ username });  
  if (!user || !(await bcrypt.compare(password, user.password))) {  
    return res.status(400).json({ message: "Invalid credentials" });  
  }  
  const token = jwt.sign({ id: user._id, role: user.role }, process.env.JWT_SECRET);  
  res.json({ token });  
};

In controllers/productController.js, implement CRUD and search:

const Product = require('../models/Product');  
exports.getAll = async (req, res) => {  
  const { page = 1, limit = 5, search = "" } = req.query;  
  const filter = { name: { $regex: search, $options: "i" } };  
  const products = await Product.find(filter)  
    .limit(limit * 1)  
    .skip((page - 1) * limit);  
  res.json(products);  
};  
exports.create = async (req, res) => {  
  const product = await new Product(req.body).save();  
  res.json(product);  
};  
exports.update = async (req, res) => {  
  const updated = await Product.findByIdAndUpdate(req.params.id, req.body, { new: true });  
  res.json(updated);  
};  
exports.delete = async (req, res) => {  
  await Product.findByIdAndDelete(req.params.id);  
  res.json({ message: "Product deleted" });  
};

In controllers/cartController.js, implement cart logic:

const User = require('../models/User');  
exports.addToCart = async (req, res) => {  
  const user = await User.findById(req.user.id);  
  const existing = user.cart.find(i => i.productId === req.body.productId);  
  if (existing) existing.quantity += req.body.quantity;  
  else user.cart.push(req.body);  
  await user.save();  
  res.json(user.cart);  
};  
exports.removeFromCart = async (req, res) => {  
  const user = await User.findById(req.user.id);  
  user.cart = user.cart.filter(i => i.productId !== req.params.productId);  
  await user.save();  
  res.json(user.cart);  
};  
exports.getCart = async (req, res) => {  
  const user = await User.findById(req.user.id);  
  res.json(user.cart);  
};

In controllers/orderController.js, implement order placement:

const Order = require('../models/Order');  
const User = require('../models/User');  
exports.createOrder = async (req, res) => {  
  const user = await User.findById(req.user.id);  
  const order = await new Order({  
    userId: user._id,  
    items: user.cart  
  }).save();  
  user.cart = [];  
  await user.save();  
  res.json(order);  
};  
exports.getOrders = async (req, res) => {  
  const orders = await Order.find({ userId: req.user.id });  
  res.json(orders);  
};

create the route files:In routes/authRoutes.js:

const express = require('express');  
const router = express.Router();  
const { register, login } = require('../controllers/authController');  
router.post('/register', register);  
router.post('/login', login);  
module.exports = router;


In routes/productRoutes.js

const express = require('express');  
const router = express.Router();  
const auth = require('../middleware/authMiddleware');  
const checkRole = require('../middleware/roleMiddleware');  
const productController = require('../controllers/productController');  
router.get('/', productController.getAll);  
router.post('/', auth, checkRole('admin'), productController.create);  
router.put('/:id', auth, checkRole('admin'), productController.update);  
router.delete('/:id', auth, checkRole('admin'), productController.delete);  
module.exports = router;


// In routes/cartRoutes.js:
const express = require('express');  
const router = express.Router();  
const auth = require('../middleware/authMiddleware');  
const orderController = require('../controllers/orderController');  
router.post('/', auth, orderController.createOrder);  
router.get('/', auth, orderController.getOrders);  
module.exports = router;
