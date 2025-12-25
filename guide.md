# פרויקט Express.js: מערכת כרטיסים (Event Ticket Server) - מבנה מודולרי

מדריך זה מרכז את כל קוד הפרויקט כשהוא מחולק למודולים (קבצים נפרדים) לטובת קריאות, תחזוקה ולימוד של ארכיטקטורת שרת נכונה.

## 1. הקמת הסביבה ומבנה הפרויקט

לפני שמתחילים לכתוב קוד, יש לפתוח מסוף (Terminal) בתיקייה חדשה ולהריץ את הפקודות הבאות:

```
npm init -y
npm install express
```

לאחר מכן, צור את מבנה התיקיות והקבצים הבא:

```
/my-ticket-server
│
├── /data                # אחסון הנתונים (JSON)
│   ├── users.json
│   ├── events.json
│   └── receipts.json
│
├── /utils               # פונקציות עזר כלליות
│   └── fileHelper.js
│
├── /middleware          # שכבת אימות (בדיקת סיסמאות)
│   └── authMiddleware.js
│
├── /controllers         # לוגיקה עסקית (מה הפונקציה עושה)
│   ├── userController.js
│   └── eventController.js
│
├── /routes              # ניתוב בקשות (URL -> Controller)
│   ├── userRoutes.js
│   └── eventRoutes.js
│
├── server.js            # קובץ הכניסה הראשי
└── package.json
```

---

## 2. קבצי הנתונים (Data Layer)

יש ליצור 3 קבצים בתוך תיקיית `data`. כל קובץ צריך להכיל מערך ריק בתור התחלה.

**קובץ: `data/users.json`**
```
[]
```

**קובץ: `data/events.json`**
```
[]
```

**קובץ: `data/receipts.json`**
```
[]
```

---

## 3. שכבת העזר (Utils)

מטרת השכבה: לטפל בקריאה וכתיבה לקבצים כדי שהקונטרולרים יישארו נקיים.

**קובץ: `utils/fileHelper.js`**

```
const fs = require('fs').promises;
const path = require('path');

// פונקציה לקריאת קובץ JSON והחזרת אובייקט JavaScript
const readJsonFile = async (filename) => {
    const filePath = path.join(__dirname, '..', 'data', filename);
    try {
        const data = await fs.readFile(filePath, 'utf8');
        return JSON.parse(data);
    } catch (error) {
        // אם הקובץ לא קיים או שיש שגיאה, נחזיר מערך ריק למניעת קריסת השרת
        return [];
    }
};

// פונקציה לכתיבת אובייקט JavaScript לתוך קובץ JSON
const writeJsonFile = async (filename, data) => {
    const filePath = path.join(__dirname, '..', 'data', filename);
    await fs.writeFile(filePath, JSON.stringify(data, null, 2), 'utf8');
};

module.exports = { readJsonFile, writeJsonFile };
```

---

## 4. שכבת ה-Middleware (אימות)

מטרת השכבה: לוודא ששם המשתמש והסיסמה נכונים לפני שמאפשרים לבצע פעולה (כמו קניית כרטיס או יצירת אירוע).

**קובץ: `middleware/authMiddleware.js`**

```
const { readJsonFile } = require('../utils/fileHelper');

const verifyUser = async (req, res, next) => {
    // שליפת שם משתמש וסיסמה מגוף הבקשה
    const { username, password } = req.body;

    // בדיקת תקינות בסיסית
    if (!username || !password) {
        return res.status(400).json({ message: "Username and password are required" });
    }

    // טעינת המשתמשים ובדיקה האם קיים צירוף מתאים
    const users = await readJsonFile('users.json');
    const user = users.find(u => u.username === username && u.password === password);

    if (!user) {
        return res.status(401).json({ message: "Invalid credentials" }); // 401 = Unauthorized
    }

    // אם הכל תקין, ממשיכים לפונקציה הבאה בשרשרת
    next();
};

module.exports = verifyUser;
```

---

## 5. הקונטרולרים (Controllers)

מטרת השכבה: להכיל את הלוגיקה של האפליקציה (מה קורה כשנרשמים, מה קורה כשקונים כרטיס).

**קובץ: `controllers/userController.js`**

```
const { readJsonFile, writeJsonFile } = require('../utils/fileHelper');

// --- 1. הרשמת משתמש ---
exports.registerUser = async (req, res) => {
    const { username, password } = req.body;

    if (!username || !password) {
        return res.status(400).json({ message: "Missing username or password" });
    }

    const users = await readJsonFile('users.json');
    
    // וידוא שהמשתמש לא קיים כבר
    if (users.find(u => u.username === username)) {
        return res.status(409).json({ message: "Username already exists" });
    }

    // הוספה ושמירה
    users.push({ username, password });
    await writeJsonFile('users.json', users);

    res.json({ message: "User registered successfully" });
};

// --- 2. קניית כרטיסים ---
exports.buyTicket = async (req, res) => {
    const { username, eventName, quantity } = req.body;
    
    // (המשתמש כבר אומת ב-Middleware לפני שהגיע לכאן)

    let events = await readJsonFile('events.json');
    
    // חיפוש האירוע (Case Insensitive)
    const eventIndex = events.findIndex(e => e.eventName.toLowerCase() === eventName.toLowerCase());

    if (eventIndex === -1) {
        return res.status(404).json({ message: "Event not found" });
    }

    const event = events[eventIndex];

    // בדיקת מלאי
    if (event.ticketsAvailable < quantity) {
        return res.status(400).json({ message: "Not enough tickets available" });
    }

    // עדכון המלאי
    event.ticketsAvailable -= quantity;
    events[eventIndex] = event;
    await writeJsonFile('events.json', events);

    // יצירת קבלה ושמירה
    const receipts = await readJsonFile('receipts.json');
    receipts.push({ 
        username, 
        eventName: event.eventName, // שומרים את השם המקורי של האירוע
        ticketsBought: quantity 
    });
    await writeJsonFile('receipts.json', receipts);

    res.json({ message: "Tickets purchased successfully" });
};

// --- 3. סיכום למשתמש ---
exports.getUserSummary = async (req, res) => {
    const { username } = req.params; // מקבלים מה-URL ולא מה-Body
    const receipts = await readJsonFile('receipts.json');

    // סינון כל הקבלות של המשתמש הזה
    const userReceipts = receipts.filter(r => r.username === username);

    // טיפול במקרה שאין קבלות
    if (userReceipts.length === 0) {
        return res.json({
            totalTicketsBought: 0,
            events: [],
            averageTicketsPerEvent: 0
        });
    }

    // חישובים סטטיסטיים
    const totalTicketsBought = userReceipts.reduce((sum, r) => sum + r.ticketsBought, 0);
    // Set מסנן כפילויות באופן אוטומטי
    const uniqueEvents = [...new Set(userReceipts.map(r => r.eventName))];
    const averageTicketsPerEvent = totalTicketsBought / uniqueEvents.length;

    res.json({
        totalTicketsBought,
        events: uniqueEvents,
        averageTicketsPerEvent
    });
};
```

**קובץ: `controllers/eventController.js`**

```
const { readJsonFile, writeJsonFile } = require('../utils/fileHelper');

// --- יצירת אירוע חדש ---
exports.createEvent = async (req, res) => {
    const { eventName, ticketsForSale, username } = req.body;

    const events = await readJsonFile('events.json');

    // יצירת האובייקט לפי הדרישות
    const newEvent = {
        eventName,
        ticketsAvailable: ticketsForSale, 
        createdBy: username
    };

    events.push(newEvent);
    await writeJsonFile('events.json', events);

    res.json({ message: "Event created successfully" });
};
```

---

## 6. הראוטרים (Routes)

מטרת השכבה: להגדיר את כתובות ה-URL ולחבר אותן לפונקציות בקונטרולר, תוך שיבוץ ה-Middleware כשצריך.

**קובץ: `routes/userRoutes.js`**

```
const express = require('express');
const router = express.Router();
const userController = require('../controllers/userController');
const verifyUser = require('../middleware/authMiddleware');

// הרשמה (אין צורך באימות מקדים כי המשתמש עדיין לא קיים)
router.post('/register', userController.registerUser);

// קניית כרטיסים (מחייב שם משתמש וסיסמה תקינים)
router.post('/tickets/buy', verifyUser, userController.buyTicket);

// קבלת סיכום (פרמטר ב-URL)
router.get('/:username/summary', userController.getUserSummary);

module.exports = router;
```

**קובץ: `routes/eventRoutes.js`**

```
const express = require('express');
const router = express.Router();
const eventController = require('../controllers/eventController');
const verifyUser = require('../middleware/authMiddleware');

// יצירת אירוע (מחייב שם משתמש וסיסמה תקינים)
router.post('/events', verifyUser, eventController.createEvent);

module.exports = router;
```

---

## 7. קובץ השרת הראשי (Server Entry Point)

מטרת הקובץ: לאחד את כל החלקים ולהרים את השרת לאוויר.

**קובץ: `server.js`**

```
const express = require('express');
const app = express();
const PORT = 3000;

// מאפשר לשרת לקרוא JSON שנשלח ב-Body
app.use(express.json());

// ייבוא הראוטרים
const userRoutes = require('./routes/userRoutes');
const eventRoutes = require('./routes/eventRoutes');

// --- הגדרת ה-End Points ---

// 1. טיפול בבקשות שקשורות למשתמשים
// הקובץ userRoutes מטפל גם ב: /user/register וגם ב: /users/tickets...
// לכן נגדיר אותו פעמיים עם Prefixes שונים כדי להתאים לדרישות המדויקות במבחן

// עבור: POST /user/register
app.use('/user', userRoutes);   

// עבור: POST /users/tickets/buy ועבור GET /users/:username/summary
app.use('/users', userRoutes);  

// 2. טיפול בבקשות יוצרים (יצירת אירועים)
// עבור: POST /creator/events
app.use('/creator', eventRoutes); 

// הפעלת השרת
app.listen(PORT, () => {
    console.log(`Server running on http://localhost:${PORT}`);
});
```
```