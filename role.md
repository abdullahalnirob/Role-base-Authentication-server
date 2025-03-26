# Role-Based Authentication System

এই ডকুমেন্টটি একটি রোল-ভিত্তিক প্রমাণীকরণ সিস্টেম (Role-Based Authentication System) এর গঠন এবং কার্যকারিতা বর্ণনা করে।

## ডিরেক্টরি কাঠামো

নিম্নলিখিত ডিরেক্টরি কাঠামোটি এই সিস্টেমের ফাইল এবং ফোল্ডারগুলির সংগঠন দেখায়:


```
├── admin/
│   ├── adminController.js   
├── Config/
│   ├── db.js                
├── middleware/
│   ├── isAdmin.js           
├── node_modules/             
├── user/
│   ├── userController.js     
│   ├── userModel.js        
├── views/
│   ├── index.js              
├── index.js               
├── package-lock.json        
└──  package.json              
```



## মূল ফাইল (index.js)

প্রধান অ্যাপ্লিকেশন ফাইল, `index.js`, নিম্নলিখিত কোড ধারণ করে:

```javascript
import express from "express";
const app = express();
const PORT = 2000;
import cookieParser from "cookie-parser";
import connectDb from "./Config/db.js";
import router from "./user/userController.js";
import AdminRouter from "./admin/adminController.js";

// মিডলওয়্যার
app.use(express.json());
app.use(cookieParser());

// রুট
app.get("/", (_req, res) => {
    res.send("Welcome Developers!");
});
app.use("/", router);
app.use("/", AdminRouter);

// লিসেনিং
app.listen(PORT, () => {
    console.log(`Server is listening at ${PORT}`);
    connectDb();
});

```

## ডাটাবেস সংযোগ (db.js)
ডাটাবেস সংযোগ ফাইল, db.js, মঙ্গোডিবি ডাটাবেসের সাথে সংযোগ স্থাপন করে:

```javascript

import mongoose from "mongoose";

const connectDb = async () => {
    try {
        await mongoose.connect("mongodb://localhost:27017/users")
        console.log("Database connected successfully!!")
    } catch (error) {
        console.log("Database is not connect (-_-)")
    }
}
export default connectDb
```

## ব্যবহারকারী মডেল (userModel.js)
ব্যবহারকারী মডেল, userModel.js, নিম্নলিখিত স্কিমা ব্যবহার করে:

```javascript
import mongoose from "mongoose";

const UserSchema = mongoose.Schema({
    name: { type: String, required: true },
    email: { type: String, required: true },
    password: { type: String, required: true },
    role: { type: String, enum: ["admin", "user"], default: "user" }
});

const User = mongoose.model("users", UserSchema);
export default User;
```

## ব্যবহারকারী কন্ট্রোলার (userController.js)
ব্যবহারকারী কন্ট্রোলার, userController.js, ব্যবহারকারী নিবন্ধন, লগইন এবং লগআউট পরিচালনা করে:

```javascript
import express from "express";
import User from "../user/userModel.js";
import bcrypt from "bcrypt";
import jwt from "jsonwebtoken";
const router = express.Router();

router.post("/api/signup", async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    const hashedPass = await bcrypt.hash(password, 10);
    const newUser = new User({ name, email, password: hashedPass, role });
    newUser.save();
    return res.status(200).json({
      message: "User signup successfully!",
    });
  } catch (error) {
    return res.status(400).json({
      message: "User signup error!",
    });
  }
});

router.post("/api/login", async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    if (!user) {
      res.status(200).json({ message: "Auth faild!!" });
    } else {
      const isMatchPass = bcrypt.compare(password, user.password);
      if (isMatchPass) {
        const token = jwt.sign({ userId: user._id }, "Secrect");
        res.cookie("token", token, {
          httpOnly: true,
          secure: false,
          maxAge: 3600000,
        });
        res.status(200).json({ message: "Login Successfully!", token });
      } else {
        res.status(200).json({ message: "Auth faild!!" });
      }
    }
  } catch (error) {
    res.status(500).json({ message: "Internel Server error!" });
  }
});

router.post("/api/logout", async (_req, res) => {
  try {
    res.clearCookie("token")
    res.status(200).json({ message: "Logout sucessfull!" });
  } catch (error) {
    res.status(500).json({ message: "Internel Server error!" });

  }
})

export default router;

```

## অ্যাডমিন কন্ট্রোলার (adminController.js)
অ্যাডমিন কন্ট্রোলার, adminController.js, অ্যাডমিন সম্পর্কিত অপারেশন পরিচালনা করে:

```javascript
import express from "express";
import User from "../user/userModel.js";
import { isAdmin } from "../middleware/isAdmin.js";

const AdminRouter = express.Router();

AdminRouter.get("/api/admin/get-user", isAdmin, async (_req, res) => {
    try {
        const users = await User.find({});
        res.status(200).json({ users });
    } catch (error) {
        res.status(500).json({ msessage: "Internel server error!" });
    }
});

export default AdminRouter;

```


## অ্যাডমিন মিডলওয়্যার (isAdmin.js)
অ্যাডমিন মিডলওয়্যার, isAdmin.js, অ্যাডমিন রুটগুলি সুরক্ষিত করে:

```javascript
import jwt from "jsonwebtoken";
import User from "../user/userModel.js";

const isAdmin = async (req, res, next) => {
    try {
        const token = req.cookies.token;
        if (!token) {
            res.status(500).json({ message: "Unathorize user!!!" });
        }
        // console.log(token);
        const decode = jwt.verify(token, "Secrect",)
        // console.log(decode)
        const user = await User.findById(decode.userId)
        // console.log(user)
        if (user.role !== "admin") {
            res.status(200).json({ message: "You can't enter" })
        }
        req.user = user
        next()
    } catch (error) {
        res.status(500).json({ message: "Internel Server error!!!" });
        console.log(error)
    }
};

export { isAdmin };

```
