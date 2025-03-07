import express from "express";
import mongoose from "mongoose";
import axios from "axios";
import cors from "cors";

const app = express();
app.use(express.json());
app.use(cors());

mongoose.connect("mongodb://localhost:27017/transactionsDB", {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});

const transactionSchema = new mongoose.Schema({
  title: String,
  description: String,
  price: Number,
  category: String,
  sold: Boolean,
  dateOfSale: Date,
});

const Transaction = mongoose.model("Transaction", transactionSchema);

// Initialize Database
app.get("/initialize", async (req, res) => {
  const { data } = await axios.get("https://s3.amazonaws.com/roxiler.com/product_transaction.json");
  await Transaction.deleteMany({});
  await Transaction.insertMany(data);
  res.send({ message: "Database initialized!" });
});

// Get Transactions with Search and Pagination
app.get("/transactions", async (req, res) => {
  const { month, search = "", page = 1, per_page = 10 } = req.query;
  const startDate = new Date(`${month} 1, 2023`);
  const endDate = new Date(`${month} 31, 2023`);
 
  const filter = {
    dateOfSale: { $gte: startDate, $lte: endDate },
    $or: [
      { title: new RegExp(search, "i") },
      { description: new RegExp(search, "i") },
      { price: isNaN(search) ? { $exists: true } : Number(search) },
    ],
  };

  const transactions = await Transaction.find(filter)
    .skip((page - 1) * per_page)
    .limit(Number(per_page));
  res.send(transactions);
});

// Statistics API
app.get("/statistics", async (req, res) => {
  const { month } = req.query;
  const startDate = new Date(`${month} 1, 2023`);
  const endDate = new Date(`${month} 31, 2023`);

  const transactions = await Transaction.find({ dateOfSale: { $gte: startDate, $lte: endDate } });
  const totalSales = transactions.reduce((sum, t) => sum + (t.sold ? t.price : 0), 0);
  const soldItems = transactions.filter((t) => t.sold).length;
  const unsoldItems = transactions.filter((t) => !t.sold).length;

  res.send({ totalSales, soldItems, unsoldItems });
});

// Bar Chart API
app.get("/bar-chart", async (req, res) => {
  const { month } = req.query;
  const startDate = new Date(`${month} 1, 2023`);
  const endDate = new Date(`${month} 31, 2023`);

  const transactions = await Transaction.find({ dateOfSale: { $gte: startDate, $lte: endDate } });
  const ranges = {
    "0-100": 0, "101-200": 0, "201-300": 0, "301-400": 0,
    "401-500": 0, "501-600": 0, "601-700": 0, "701-800": 0,
    "801-900": 0, "901+": 0
  };
 
  transactions.forEach(t => {
    if (t.price <= 100) ranges["0-100"]++;
    else if (t.price <= 200) ranges["101-200"]++;
    else if (t.price <= 300) ranges["201-300"]++;
    else if (t.price <= 400) ranges["301-400"]++;
    else if (t.price <= 500) ranges["401-500"]++;
    else if (t.price <= 600) ranges["501-600"]++;
    else if (t.price <= 700) ranges["601-700"]++;
    else if (t.price <= 800) ranges["701-800"]++;
    else if (t.price <= 900) ranges["801-900"]++;
    else ranges["901+"]++;
  });

  res.send(ranges);
});

// Pie Chart API
app.get("/pie-chart", async (req, res) => {
  const { month } = req.query;
  const startDate = new Date(`${month} 1, 2023`);
  const endDate = new Date(`${month} 31, 2023`);

  const transactions = await Transaction.find({ dateOfSale: { $gte: startDate, $lte: endDate } });
  const categories = {};
  transactions.forEach(t => {
    categories[t.category] = (categories[t.category] || 0) + 1;
  });

  res.send(categories);
});

// Combined Dashboard API
app.get("/dashboard", async (req, res) => {
  const { month } = req.query;
  const [stats, barData, pieData] = await Promise.all([
    axios.get(`http://localhost:5000/statistics?month=${month}`),
    axios.get(`http://localhost:5000/bar-chart?month=${month}`),
    axios.get(`http://localhost:5000/pie-chart?month=${month}`)
  ]);

  res.send({
    statistics: stats.data,
    barChart: barData.data,
    pieChart: pieData.data
  });
});

const PORT = 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
