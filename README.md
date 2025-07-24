# Eee
<!DOCTYPE html>
<html dir="rtl" lang="ar">
<head>
  <meta charset="UTF-8" />
  <title>حاسبة الأسعار</title>
  <style>
    body {
      background-color: #1e1e1e;
      color: #fff;
      font-family: 'Segoe UI', sans-serif;
      padding: 20px;
    }
    select, input, button {
      padding: 10px;
      margin: 5px 0;
      width: 100%;
      font-size: 16px;
    }
    #results {
      background: #2a2a2a;
      margin-top: 20px;
      padding: 15px;
      border-radius: 8px;
    }
    .result-item {
      border-bottom: 1px solid #444;
      padding: 10px 0;
    }
    button {
      background-color: #444;
      color: white;
      border: none;
      cursor: pointer;
    }
    button:hover {
      background-color: #666;
    }
    #summary {
      margin-top: 20px;
      background: #333;
      padding: 10px;
      border-radius: 6px;
    }
  </style>
</head>
<body>

  <h1>حاسبة الأسعار</h1>

  <select id="mainType" onchange="loadSubTypes()">
    <option value="">اختر القسم الرئيسي</option>
    <option value="نافذة">نوافذ</option>
    <option value="باب">أبواب</option>
    <option value="سحب">أبواب السحب</option>
  </select>

  <select id="subType" onchange="checkFixedSize()">
    <option value="">اختر النوع الفرعي</option>
  </select>

  <input type="number" id="width" placeholder="العرض بالمتر" step="0.01" />
  <input type="number" id="height" placeholder="الطول بالمتر" step="0.01" />
  <input type="number" id="quantity" placeholder="الكمية" value="1" />

  <button onclick="calculate()">احسب</button>
  <button onclick="clearResults()">محو النتائج</button>
  <button onclick="exportToWord()">حفظ النتائج كملف Word</button>

  <div id="results"></div>
  <div id="summary"></div>

  <script>
    const prices = {
      // نوافذ
      "دبل جلاس دبل فريم ثابتة": 34,
      "دبل جلاس دبل فريم حركة": 73,
      "دبل جلاس دبل فريم حركتين": 92,
      "دبل جلاس سنجل فريم ثابتة": 26,
      "دبل جلاس سنجل فريم حركة": 46,
      "دبل جلاس سنجل فريم حركتين": 58,
      "سنجل جلاس سنجل فريم ثابتة": 20,
      "سنجل جلاس سنجل فريم حركة": 43,
      "سنجل جلاس سنجل فريم حركتين": 47,
      "نوافذ السلايدنج": 10, // تضاف بعد المساحة * سعر غير ثابت
      "النوافذ الكهربائية": 102,
      "سكاي لايت بدون مكينة": 56,
      "سكاي لايت مع مكينة": 145,
      "كارتن وول خفيف": 45,
      "كارتن وول ثقيل": 56,

      // أبواب
      "باب مدخل زينك": 66 + 10,
      "باب مدخل ستينلس ستيل": 120 + 10,
      "باب مدخل كاست المنيوم": 168 + 10,

      // WPC
      "WPC فارغ": 45,
      "WPC مع خشب": 50,
      "WPC ضد الصوت": 60,
      "WPC مع فريم المنيوم": 67,

      // المنيوم
      "المنيوم فارغ": 65,
      "المنيوم مع خشب": 75,
      "المنيوم فل": 85,
      "المنيوم مخفي": 110,
      "المنيوم خارجي": 61,

      // دورات مياه
      "دورات مياه جديد": 55,
      "دورات مياه قديم": 45,
      "دورات مياه زجاجي مخفي": 65,

      // أبواب السحب
      "سحب داخلي زجاج": 38,
      "سحب داخلي متين": 41,
      "سحب خارجي مفتوح": 55,
      "سحب خارجي جزئين": 58,
      "WPC سحب": 61,
    };

    const factors = {
      "default": 0.13,
      "WPC": 0.11,
      "المنيوم": 0.11,
      "دورات مياه": 0.11,
      "باب مدخل": 0.2,
    };

    function loadSubTypes() {
      const type = document.getElementById("mainType").value;
      const sub = document.getElementById("subType");
      sub.innerHTML = `<option value="">اختر النوع الفرعي</option>`;
      let options = [];

      if (type === "نافذة") {
        options = Object.keys(prices).filter(k => k.includes("جلاس") || k.includes("نافذة") || k.includes("كارتن") || k.includes("سكاي"));
      } else if (type === "باب") {
        options = Object.keys(prices).filter(k =>
          k.includes("باب مدخل") || k.includes("WPC") || k.includes("المنيوم") || k.includes("دورات مياه")
        );
      } else if (type === "سحب") {
        options = Object.keys(prices).filter(k => k.includes("سحب") || k.includes("WPC سحب"));
      }

      options.forEach(opt => {
        const o = document.createElement("option");
        o.value = opt;
        o.textContent = opt;
        sub.appendChild(o);
      });
    }

    function checkFixedSize() {
      const selected = document.getElementById("subType").value;
      const fixed = ["WPC", "المنيوم", "دورات مياه"].some(type => selected.includes(type));
      document.getElementById("width").value = fixed ? 1 : "";
      document.getElementById("height").value = fixed ? 2.2 : "";
      document.getElementById("width").readOnly = fixed;
      document.getElementById("height").readOnly = fixed;
    }

    const results = [];

    function calculate() {
      const type = document.getElementById("subType").value;
      const width = parseFloat(document.getElementById("width").value);
      const height = parseFloat(document.getElementById("height").value);
      const qty = parseInt(document.getElementById("quantity").value) || 1;

      if (!type || !width || !height) return alert("أدخل كل القيم");

      const area = width * height;
      const pricePer = prices[type] || 0;
      const mainType = type.includes("WPC") ? "WPC" : type.includes("المنيوم") ? "المنيوم" : type.includes("دورات مياه") ? "دورات مياه" : type.includes("مدخل") ? "باب مدخل" : "default";
      const factor = factors[mainType] || 0.13;

      let basePrice = pricePer;
      if (type === "نوافذ السلايدنج") basePrice = area * 0 + 10;

      const productPrice = area * basePrice;
      const shipping = area * factor * 48;
      const total = (productPrice + shipping) * qty;

      results.push({ type, area, qty, shipping: shipping.toFixed(2), total: total.toFixed(2) });
      displayResults();
    }

    function displayResults() {
      const container = document.getElementById("results");
      container.innerHTML = "";
      let totalSum = 0;

      results.forEach((r, i) => {
        totalSum += parseFloat(r.total);
        container.innerHTML += `
          <div class="result-item">
            <strong>${r.type}</strong><br>
            المقاس: ${r.area.toFixed(2)} م²<br>
            الكمية: ${r.qty}<br>
            الشحن: ${r.shipping} ريال<br>
            الإجمالي: ${r.total} ريال
          </div>
        `;
      });

      const commission = totalSum * 0.04;
      const grandTotal = totalSum + commission;
      document.getElementById("summary").innerHTML = `
        <p>الناتج الكلي: ${totalSum.toFixed(2)} ريال</p>
        <p>عمولة المكتب (4%): ${commission.toFixed(2)} ريال</p>
        <p><strong>المجموع النهائي: ${grandTotal.toFixed(2)} ريال</strong></p>
      `;
    }

    function clearResults() {
      results.length = 0;
      document.getElementById("results").innerHTML = "";
      document.getElementById("summary").innerHTML = "";
    }

    function exportToWord() {
      const content = document.getElementById("results").innerHTML + document.getElementById("summary").innerHTML;
      const blob = new Blob(['<html><body dir="rtl">' + content + '</body></html>'], { type: 'application/msword' });
      const link = document.createElement("a");
      link.href = URL.createObjectURL(blob);
      link.download = "النتائج.doc";
      link.click();
    }
  </script>

</body>
</html>
