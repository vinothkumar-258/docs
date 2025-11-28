# 📌 Stock Analytics API – Clickable Example Endpoints

### **1. Import Stock Data**
**Link:**  
https://us-central1-brilliant-tide-478610-b2.cloudfunctions.net/import-stock-data?symbol=GOOG  
Imports GOOG profile + historical prices + weekly & daily metrics.

---

### **2. Export Stock Data**
**Link:**  
https://us-central1-brilliant-tide-478610-b2.cloudfunctions.net/export-stock-data?symbol=GOOG  
Exports stored price + metrics for GOOG.

---

### **3. Correlation (Two Symbols, Excel Output)**
**Link:**  
https://us-central1-brilliant-tide-478610-b2.cloudfunctions.net/correlation?symbols=AAPL,GOOG  
Generates 26-week & 52-week rolling correlation between AAPL and GOOG.

---

### **4. Daily SMA Status — Above Threshold**
**Link:**  
http://us-central1-brilliant-tide-478610-b2.cloudfunctions.net/dailySmaStatus?symbol=AAPL&SMA=50&above=150  
Shows AAPL streaks where price stayed **above 50-day SMA for ≥ 150 days**.

---

### **5. Daily SMA Status — Below Threshold**
**Link:**  
http://us-central1-brilliant-tide-478610-b2.cloudfunctions.net/dailySmaStatus?symbol=AAPL&SMA=50&below=5  
Shows AAPL streaks where price stayed **below 50-day SMA for ≤ 5 days**.

---

### **6. Seasonality – Yearly (PDF)**
**Link:**  
https://us-central1-brilliant-tide-478610-b2.cloudfunctions.net/seasonality-yearly?symbol=AAPL&start=2020-01-01&end=2025-11-30  
Generates a PDF of **year-by-year monthly returns + annual returns** for AAPL.

---

### **7. Seasonality – Monthly (PDF)**
**Link:**  
https://us-central1-brilliant-tide-478610-b2.cloudfunctions.net/seasonality-monthly?symbol=AAPL&start=2020-01-01&end=2025-11-30  
Generates a PDF showing **day-of-month seasonality patterns** for AAPL.

---
