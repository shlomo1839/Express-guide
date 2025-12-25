# Express.js Starter Template - השלד האולטימטיבי

זהו שלד (Boilerplate) מקצועי לפיתוח שרתי REST API באמצעות Node.js ו-Express.
הוא בנוי בצורה מודולרית (Separation of Concerns) כדי לאפשר גדילה קלה של הפרויקט ותחזוקה נוחה.

---

## 1. הקמת הפרויקט (Setup)

בכל פרויקט חדש, פתח את הטרמינל בתיקייה הריקה והרץ את הפקודות הבאות:

```
# 1. אתחול הפרויקט
npm init -y

# 2. התקנת חבילות בסיס (חובה)
npm install express dotenv cors

# 3. התקנת חבילות לפיתוח (מומלץ)
npm install --save-dev nodemon morgan
```

**הסבר על החבילות:**
*   `dotenv`: לטעינת משתני סביבה (כמו סיסמאות) מקובץ `.env`.
*   `cors`: מאפשר לדפדפן (React/Vue) לפנות לשרת שלך מדומיין אחר.
*   `morgan`: מדפיס לוגים של בקשות לטרמינל (טוב לדיבאגינג).
*   `nodemon`: מפעיל מחדש את השרת אוטומטית בכל שמירה.

---

## 2. מבנה התיקיות

צור את המבנה הבא. זהו המבנה הסטנדרטי בתעשייה:

```
/my-new-project
│
├── /config              # הגדרות מערכת וחיבור לדאטה-בייס
│   └── db.js
│
├── /controllers         # הלוגיקה העסקית (מה הפונקציה עושה)
│   └── exampleController.js
│
├── /routes              # ניתוב בקשות (API Endpoints)
│   └── exampleRoutes.js
│
├── /middleware          # פונקציות ביניים (אימות, שגיאות)
│   ├── errorMiddleware.js
│   └── authMiddleware.js
│
├── /utils               # פונקציות עזר כלליות
│   └── helpers.js
│
├── .env                 # משתנים רגישים (לא עולה לגיט!)
├── .gitignore           # קבצים שגיט צריך להתעלם מהם
├── server.js            # נקודת הכניסה לשרת
└── package.json
```

---

## 3. קבצי הקוד (העתק-הדבק)

### א. קובץ ההגדרות (`.env`)
כאן שומרים פורטים, סיסמאות ומפתחות API.
```
PORT=3000
NODE_ENV=development
MONGO_URI=your_database_connection_string_here
```

### ב. קובץ התעלמות מגיט (`.gitignore`)
חובה כדי לא להעלות את `node_modules` הכבד.
```
node_modules
.env
.DS_Store
```

### ג. נקודת הכניסה (`server.js`)
זהו הלב של האפליקציה. הוא מחבר את הכל יחד.

```
require('dotenv').config(); // טעינת משתני סביבה
const express = require('express');
const cors = require('cors');
const morgan = require('morgan'); // אופציונלי - ללוגים

// ייבוא ראוטרים
const exampleRoutes = require('./routes/exampleRoutes');

// ייבוא Middleware לשגיאות
const { errorHandler } = require('./middleware/errorMiddleware');

const app = express();

// --- Middlewares גלובליים ---
app.use(express.json()); // מאפשר קריאת JSON מה-Body
app.use(cors());         // מאפשר גישה מקליינטים שונים
if (process.env.NODE_ENV === 'development') {
    app.use(morgan('dev')); // לוגים בטרמינל
}

// --- הגדרת נתיבים (Routes) ---
app.get('/', (req, res) => {
    res.send('API is running...');
});

// חיבור קבצי הראוטר
app.use('/api/examples', exampleRoutes);

// --- טיפול בשגיאות (חייב להיות בסוף) ---
app.use(errorHandler);

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Server running in ${process.env.NODE_ENV} mode on port ${PORT}`);
});
```

### ד. הראוטר (`routes/exampleRoutes.js`)
מגדיר איזה נתיב מפעיל איזו פונקציה.

```
const express = require('express');
const router = express.Router();
const { getExamples, createExample } = require('../controllers/exampleController');

// הגדרת הכתובות
// כתובת מלאה: /api/examples/
router.get('/', getExamples);
router.post('/', createExample);

module.exports = router;
```

### ה. הקונטרולר (`controllers/exampleController.js`)
הלוגיקה עצמה. כאן כותבים את הקוד.

```
// @desc    Get all examples
// @route   GET /api/examples
// @access  Public
const getExamples = async (req, res, next) => {
    try {
        // כאן תהיה שליפה מהדאטה-בייס
        const mockData = [{ id: 1, name: "Test Item" }];
        
        res.status(200).json({
            success: true,
            data: mockData
        });
    } catch (error) {
        // העברת השגיאה ל-Middleware המרכזי
        next(error);
    }
};

// @desc    Create new example
// @route   POST /api/examples
// @access  Public
const createExample = async (req, res, next) => {
    try {
        const { name } = req.body;
        
        if (!name) {
            res.status(400);
            throw new Error('Please provide a name field');
        }

        res.status(201).json({
            success: true,
            message: `Created item: ${name}`
        });
    } catch (error) {
        next(error);
    }
};

module.exports = { getExamples, createExample };
```

### ו. מנהל השגיאות (`middleware/errorMiddleware.js`)
דואג שאם יש שגיאה, השרת לא יקרוס אלא יחזיר תשובה מסודרת בפורמט אחיד.

```
const errorHandler = (err, req, res, next) => {
    // אם הסטטוס הוא 200 (הצלחה) אבל יש שגיאה, נשנה ל-500
    const statusCode = res.statusCode === 200 ? 500 : res.statusCode;

    res.status(statusCode);

    res.json({
        success: false,
        message: err.message,
        // הצגת ה-Stack Trace רק בסביבת פיתוח
        stack: process.env.NODE_ENV === 'production' ? null : err.stack,
    });
};

module.exports = { errorHandler };
```

---

## 4. הגדרת סקריפטים (`package.json`)

כדי להריץ את השרת בנוחות עם `nodemon`, הוסף את השורות הבאות תחת "scripts" בקובץ `package.json`:

```
"scripts": {
  "start": "node server.js",
  "dev": "nodemon server.js"
}
```

---

## 5. איך מתחילים לעבוד?

1.  מריצים בטרמינל: `npm run dev`.
2.  פותחים ב-Postman או בדפדפן את הכתובת: `http://localhost:3000/api/examples`.
3.  כדי להוסיף ישות חדשה (למשל "Users"):
    *   צור `controllers/userController.js`.
    *   צור `routes/userRoutes.js`.
    *   חבר אותם ב-`server.js` עם `app.use('/api/users', userRoutes)`.

בהצלחה!
```

### איך להשתמש בזה עכשיו?
1.  צור קובץ חדש ב-VS Code בשם `express-template.md`.
2.  הדבק את התוכן שלמעלה.
3.  שמור את הקובץ.
4.  לחץ `Ctrl + Shift + V` כדי לראות אותו כדף הוראות מסודר.
5.  בכל פעם שאתה מתחיל פרויקט חדש, פתח את הקובץ הזה ועקוב אחר ההוראות שלב-אחר-שלב.