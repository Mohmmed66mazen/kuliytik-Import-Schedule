# 💡 ImportScheduleHandler — دليل Config JSON

هذا المستند يشرح **شكل ملف config (JSON)** الذي يُستخدم لإنشاء كائن `ImportScheduleHandler` — وهو المسؤول عن تشغيل أداة استيراد الجدول الجامعي عبر WebView.

---

## 1. ما هو هذا الملف؟ 🤔

`ImportScheduleHandler` هو **كلاس Config** يتحكم في سلوك أداة استيراد الجدول. يتم تمرير إعداداته إما:

- **يدوياً** عبر الكود (Dart constructor)
- **أو كملف JSON** (يُقرأ ويُمرر إلى `fromJson`)

الـ JSON يحدد:

- 🧭 أين يبدأ WebView (`startUrl`)
- 📋 خطوات التعليمات للمستخدم (`steps`)
- 📝 سكريبت JavaScript لاستخراج البيانات (`importScript`)
- 🔄 سكريبتات تُشغل أثناء التنقل (`scripts`)
- 🚦 قواعد إعادة التوجيه (`redirectConfig`)

---

## 2. هيكل Config الكامل

```jsonc
{
  // ===== Required Fields =====

  "steps": ["Step 1", "Step 2", "..."],
  "startUrl": "https://...",
  "importScript": "// JavaScript code to extract the table...",

  // ===== Optional Fields =====

  "scripts": [
    {
      "targetUrl": "https://...",
      "script": "// JavaScript code...",
    },
  ],
  "redirectConfig": {
    "allowedUrls": ["https://..."],
    "redirectUrl": "https://...",
  },
}
```

---

## 3. الحقول الرئيسية — نظرة سريعة

| الحقل            |      النوع      | مطلوب؟ | وظيفته                                                       |
| ---------------- | :-------------: | :----: | ------------------------------------------------------------ |
| `steps`          | `Array<String>` |   ✅   | قائمة التعليمات التي تظهر للمستخدم خطوة بخطوة                |
| `startUrl`       |    `String`     |   ✅   | رابط البداية الذي يفتحه WebView                              |
| `importScript`   |    `String`     |   ✅   | سكريبت JavaScript لاستخراج الجدول عند الضغط على زر الاستيراد |
| `scripts`        | `Array<Object>` |   ❌   | سكريبتات تُشغل تلقائياً عند زيارة صفحات معينة                |
| `redirectConfig` |    `Object`     |   ❌   | قواعد لإعادة التوجيه إذا خرج المستخدم عن الصفحات المسموحة    |

---

## 4. شرح كل حقل بالتفصيل 🔍

### 4.1 `steps` — خطوات التعليمات

```json
"steps": [
    "قم بتسجيل الدخول إلى حسابك الجامعي",
    "انتقل إلى صفحة الجدول.",
    "ستلاحظ أن الزر تغيّر إلى \"استيراد جدول\"",
    "اضغط على الزر ليتم استيراد الجدول"
]
```

| الحقل           | `steps`                                                                     |
| --------------- | --------------------------------------------------------------------------- |
| **النوع**       | `Array<String>` (مصفوفة نصوص)                                               |
| **مطلوب؟**      | ✅ **نعم**                                                                  |
| **الوظيفة**     | يعرض للمستخدم سلسلة من التعليمات في `ImportInstructionsBottomSheet` ليتبعها |
| **عدد العناصر** | من 1 إلى أي عدد                                                             |
| **مثال**        | `["سجل الدخول", "اذهب للجدول", "اضغط استيراد"]`                             |

> 💡 **نصيحة:** اجعل الخطوات واضحة ومختصرة، وابدأ من تسجيل الدخول حتى الضغط على زر الاستيراد.

---

### 4.2 `startUrl` — رابط البداية

```json
"startUrl": "https://sms.uot.edu.ly/eng/login_ing.php"
```

| الحقل       | `startUrl`                                                       |
| ----------- | ---------------------------------------------------------------- |
| **النوع**   | `String` (نص — رابط URL)                                         |
| **مطلوب؟**  | ✅ **نعم**                                                       |
| **الوظيفة** | الصفحة الأولى التي يتم تحميلها في WebView عند فتح أداة الاستيراد |
| **مثال**    | `"https://university.edu/login.php"`                             |

> 💡 **نصيحة:** ضع رابط صفحة تسجيل الدخول أو الصفحة الرئيسية للنظام الجامعي.

---

### 4.3 `importScript` — سكريبت الاستيراد الرئيسي

```json
"importScript": "const scheduleCode = {}; ... return JSON.stringify(schedule);"
```

| الحقل       | `importScript`                                                                                                            |
| ----------- | ------------------------------------------------------------------------------------------------------------------------- |
| **النوع**   | `String` (نص — كود JavaScript كامل)                                                                                       |
| **مطلوب؟**  | ✅ **نعم**                                                                                                                |
| **الوظيفة** | السكريبت الذي يُشغل عندما يضغط المستخدم على زر **"استيراد جدول"**. يقوم بتصفح DOM واستخراج بيانات المواد وإرجاعها كـ JSON |

#### آلية العمل:

```
المستخدم يضغط "استيراد جدول"
         │
         ▼
controller.runJavaScript(importScript)  ← يُحقن السكريبت في WebView
         │
         ▼
السكريبت يتصفح DOM ويستخرج البيانات
         │
         ▼
return JSON.stringify(data)  ← يُرسل النتيجة إلى Dart
```

#### أهم النقاط:

| النقطة                  | الشرح                                                |
| ----------------------- | ---------------------------------------------------- |
| ✅ **آخر جملة**         | يجب أن تكون `return JSON.stringify(data);`           |
| ✅ **DOM Query**        | استخدم `document.querySelector` و `querySelectorAll` |
| ✅ **الدوال المساعدة**  | يمكن تعريف دوال داخلية مثل `timeToMinutes()`         |
| ✅ **البيانات المرجعة** | يجب أن تطابق هيكل `Course.fromJson`                  |

#### الهيكل المتوقع للبيانات المرجعة:

```json
{
  "اسم المادة": {
    "courseCode": "رمز المادة (اختياري)",
    "timings": [
      {
        "doctor": "اسم الدكتور (اختياري)",
        "place": "القاعة (اختياري)",
        "startTime": 480,
        "endTime": 570,
        "days": [0]
      }
    ]
  }
}
```

> 📖 لمزيد من التفاصيل عن شكل البيانات المرجعة، راجع ملف `ImportReturnData.md`

---

### 4.4 `scripts` — سكريبتات التنقل (اختياري)

```json
"scripts": [
    {
        "targetUrl": "https://sms.uot.edu.ly/utstudent/index.php",
        "script": "const params = new URLSearchParams(window.location.search); if('mytable' === params.get('page')) { window.isTargeted.postMessage('true'); }"
    }
]
```

| الحقل            | `scripts`                                                                                         |
| ---------------- | ------------------------------------------------------------------------------------------------- |
| **النوع**        | `Array<Object>` (مصفوفة كائنات)                                                                   |
| **مطلوب؟**       | ❌ اختياري                                                                                        |
| **الوظيفة**      | تنفيذ سكريبتات معينة **تلقائياً** عند زيارة صفحات محددة أثناء التنقل (قبل الضغط على زر الاستيراد) |
| **متى يُستخدم؟** | للكشف عن الصفحة المستهدفة، أو تعديل DOM، أو إرسال إشارات إلى التطبيق                              |

#### الحقول الفرعية لكل كائن:

| الحقل       |  النوع   | مطلوب؟ | الوصف                                                                           |
| ----------- | :------: | :----: | ------------------------------------------------------------------------------- |
| `targetUrl` | `String` |   ✅   | رابط الصفحة التي سيتم تشغيل السكريبت عند زيارتها (تتم المقارنة بـ `startsWith`) |
| `script`    | `String` |   ✅   | كود JavaScript الذي سيتم تنفيذه                                                 |

#### مثال — الكشف عن الصفحة المستهدفة:

```json
{
  "targetUrl": "https://university.edu/student/",
  "script": "const params = new URLSearchParams(window.location.search); if(params.get('page') === 'mytable') { window.isTargeted.postMessage('true'); }"
}
```

في Dart، يتم استقبال الإشارة عبر `JavaScriptChannel` المسمى `isTargeted`، مما يؤدي إلى إظهار زر "استيراد جدول".

---

### 4.5 `redirectConfig` — إعادة التوجيه (اختياري)

```json
"redirectConfig": {
    "allowedUrls": ["https://sms.uot.edu.ly/"],
    "redirectUrl": "https://sms.uot.edu.ly/eng/login_ing.php"
}
```

| الحقل       | `redirectConfig`                                                                         |
| ----------- | ---------------------------------------------------------------------------------------- |
| **النوع**   | `Object` أو `null`                                                                       |
| **مطلوب؟**  | ❌ اختياري (إذا كان `null`، تُعطّل الميزة)                                               |
| **الوظيفة** | إذا انتقل المستخدم إلى رابط خارج `allowedUrls`، يتم إعادة توجيهه قسراً إلى `redirectUrl` |

#### الحقول الفرعية:

| الحقل         |      النوع      | مطلوب؟ | الوصف                               |
| ------------- | :-------------: | :----: | ----------------------------------- |
| `allowedUrls` | `Array<String>` |   ✅   | قائمة بالروابط المسموح البقاء فيها  |
| `redirectUrl` |    `String`     |   ✅   | الرابط الذي سيتم إعادة التوجيه إليه |

#### آلية العمل:

```
المستخدم ينتقل لصفحة جديدة
         │
         ▼
هل الرابط يبدأ بأحد allowedUrls؟
    ├── ✅ نعم → ابقَ في الصفحة
    └── ❌ لا → loadRequest(redirectUrl) ← أعد التوجيه
```

---

## 5. مثال كامل (جاهز للاستخدام) 🚀

```json
{
  "steps": [
    "قم بتسجيل الدخول إلى حسابك الجامعي",
    "انتقل إلى صفحة الجدول.",
    "ستلاحظ أن الزر تغيّر إلى \"استيراد جدول\"",
    "اضغط على الزر ليتم استيراد الجدول"
  ],
  "startUrl": "https://sms.uot.edu.ly/eng/login_ing.php",
  "importScript": "const table = document.querySelector('table'); if (!table) return; const rows = table.querySelectorAll('tr'); const schedule = {}; // extract the data return JSON.stringify(schedule);",
  "scripts": [
    {
      "targetUrl": "https://sms.uot.edu.ly/utstudent/index.php",
      "script": "const params = new URLSearchParams(window.location.search); if('mytable' === params.get('page')) { window.isTargeted.postMessage('true'); }"
    }
  ],
  "redirectConfig": {
    "allowedUrls": ["https://sms.uot.edu.ly/"],
    "redirectUrl": "https://sms.uot.edu.ly/eng/login_ing.php"
  }
}
```

---

## 6. مخطط تدفق العمل 🔄

```
📱 المستخدم يفتح الأداة
       │
       ▼
🌐 WebView يحمّل startUrl
       │
       ▼
🔄 أثناء التنقل → يتم فحص scripts[]
       │
       ├── ✅ targetUrl مطابق → runJavaScript(script)
       │
       └── ❌ لا يوجد تطابق → استمرار
       │
       ▼
🔍 هل الصفحة الحالية هي المستهدفة؟
       │
       ├── ✅ نعم → يظهر زر "استيراد جدول"
       │       │
       │       ▼
       │   المستخدم يضغط الزر
       │       │
       │       ▼
       │   runJavaScript(importScript)
       │       │
       │       ▼
       │   return JSON.stringify(data)
       │       │
       │       ▼
       │   📊 ImportResultBottomSheet يعرض النتيجة
       │
       └── ❌ لا → زر "استيراد جدول" مخفي
```

---

## 7. نصائح سريعة للمطور 💡

| 🏷️  | النصيحة                                                              |
| :-: | -------------------------------------------------------------------- |
| ✅  | ابدأ بـ `startUrl` و `importScript` — هما المطلوبان فقط لتجربة بسيطة |
| ✅  | اختبر `importScript` في **Console المتصفح** قبل إضافته               |
| ✅  | استخدم `redirectConfig` لمنع المستخدم من مغادرة النطاق المسموح       |
| ✅  | استخدم `scripts` للكشف عن وصول المستخدم للصفحة الصحيحة               |
| ⚠️  | إذا كان `scripts` فارغاً أو `null`، لن يتم تشغيل أي سكريبت تنقل      |
| ⚠️  | إذا كان `redirectConfig: null`، لن يتم إعادة التوجيه أبداً           |
| 🧪  | تأكد من صحة الـ JSON عبر [jsonlint.com](https://jsonlint.com)        |

---

## 8. مقارنة مع `UniversityAuthHandler`

نفس هيكل الـ config يُستخدم أيضاً في `UniversityAuthHandler` للمصادقة الجامعية:

| الميزة           | `ImportScheduleHandler` | `UniversityAuthHandler` |
| ---------------- | :---------------------: | :---------------------: |
| `startUrl`       |           ✅            |           ✅            |
| `scripts`        |           ✅            |           ✅            |
| `redirectConfig` |           ✅            |           ✅            |
| `steps`          |           ✅            |           ❌            |
| `importScript`   |           ✅            |           ❌            |

---

## 9. خطوات البدء السريع 🏁

1. 📝 **حدد `startUrl`** — رابط البداية (تسجيل الدخول)
2. 📋 **اكتب `steps`** — التعليمات التي سيتبعها المستخدم
3. ✍️ **اكتب `importScript`** — سكريبت استخراج الجدول (اختبره في Console المتصفح أولاً)
4. 🔄 **أضف `scripts`** (اختياري) — للكشف عن الصفحة المستهدفة تلقائياً
5. 🚦 **أضف `redirectConfig`** (اختياري) — لمنع الخروج عن النطاق المسموح
6. 🧪 **جرب الاستيراد** في التطبيق
