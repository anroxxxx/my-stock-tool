
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>港美股交易成本助手</title>
    <style>
        :root {
            --bg: #EAE7E2; --primary: #9A8C98; --text: #4A4E69;
            --card: #FFFFFF; --border: #D1CEC7; --accent: #B5838D;
        }
        body {
            background-color: var(--bg); color: var(--text);
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            transition: all 0.6s ease; display: flex; justify-content: center; padding: 20px;
        }
        .container { width: 100%; max-width: 700px; }
        .card {
            background: var(--card); padding: 30px; border-radius: 20px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.05); margin-bottom: 20px;
        }
        .tabs { display: flex; gap: 10px; margin-bottom: 20px; }
        .tab-btn {
            flex: 1; padding: 12px; border: none; border-radius: 10px;
            background: rgba(0,0,0,0.05); cursor: pointer; font-weight: bold; transition: 0.3s;
        }
        .tab-btn.active { background: var(--primary); color: white; }
        
        .mode-switch {
            display: flex; background: #f0f0f0; border-radius: 8px; margin-bottom: 20px; padding: 4px;
        }
        .mode-btn {
            flex: 1; border: none; padding: 8px; cursor: pointer; border-radius: 6px; font-size: 0.85rem;
        }
        .mode-btn.active { background: white; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }

        .batch-item { display: grid; grid-template-columns: 1fr 1fr 40px; gap: 10px; margin-bottom: 10px; }
        input {
            width: 100%; padding: 12px; border: 1px solid var(--border);
            border-radius: 10px; box-sizing: border-box; outline: none; background: #fafafa;
        }
        .btn { cursor: pointer; border: none; border-radius: 10px; font-weight: bold; transition: 0.3s; }
        .btn-add { background: none; border: 1.5px dashed var(--primary); color: var(--primary); padding: 10px; width: 100%; margin: 10px 0 20px; }
        .btn-calc { background: var(--primary); color: white; padding: 18px; width: 100%; font-size: 1rem; }
        
        .results { margin-top: 25px; padding-top: 20px; border-top: 1px solid var(--border); }
        .res-row { display: flex; justify-content: space-between; margin-bottom: 12px; }
        .highlight { color: var(--accent); font-weight: bold; font-size: 1.3rem; }

        .history-section { max-height: 400px; overflow-y: auto; }
        table { width: 100%; border-collapse: collapse; font-size: 0.85rem; }
        th, td { padding: 12px 8px; border-bottom: 1px solid var(--border); text-align: left; }
    </style>
</head>
<body>

<div class="container">
    <div class="tabs">
        <button class="tab-btn active" onclick="setMarket('HK')">🇭🇰 港股模式</button>
        <button class="tab-btn" onclick="setMarket('US')">🇺🇸 美股模式</button>
    </div>

    <div class="card">
        <h2 id="title" style="text-align:center; margin-top:0;">交易計算機</h2>
        
        <div style="margin-bottom: 15px;">
            <label style="font-size:0.9rem; font-weight:bold;">代號 / 股名</label>
            <input type="text" id="symbol" placeholder="例: 0700 或 NVDA" oninput="fetchName()">
            <div id="stockNameNotice" style="font-size:0.8rem; color:var(--primary); margin-top:5px;"></div>
        </div>

        <div id="batchArea">
            <label style="font-size:0.9rem; font-weight:bold;">分批買入 (單價 / 股數)</label>
            <div class="batch-item">
                <input type="number" class="p-in" placeholder="價格" step="0.001">
                <input type="number" class="s-in" placeholder="股數">
                <span></span>
            </div>
        </div>
        <button class="btn btn-add" onclick="addBatch()">+ 新增買入批次</button>

        <div class="mode-switch">
            <button class="mode-btn active" onclick="setMode('PROFIT')">設定目標利潤%</button>
            <button class="mode-btn" onclick="setMode('PRICE')">設定預期賣出價</button>
        </div>

        <div id="inputBox" style="margin-bottom: 20px;">
            <label id="modeLabel" style="font-size:0.9rem; font-weight:bold;">目標利潤 (%)</label>
            <input type="number" id="modeValue" value="10" step="0.01">
        </div>

        <button class="btn btn-calc" onclick="run()">立即計算並變換顏色</button>

        <div id="res" class="results" style="display:none;">
            <div class="res-row"><span>平均買入成本</span><span id="resAvg" style="font-weight:bold;">-</span></div>
            <div class="res-row"><span>總交易稅費 (買+賣)</span><span id="resFees">-</span></div>
            <div class="res-row"><span id="lbl1">預期賣出價</span><span id="val1" class="highlight">-</span></div>
            <div class="res-row"><span id="lbl2">最終純收益</span><span id="val2" class="highlight">-</span></div>
            <div class="res-row"><span>保本價</span><span id="resBreak">-</span></div>
        </div>
    </div>

    <div class="card history-section">
        <h3 style="margin-top:0;">📜 交易紀錄</h3>
        <table id="hTable">
            <thead><tr><th>市場</th><th>股票</th><th>成本</th><th>賣出</th><th>純利</th></tr></thead>
            <tbody></tbody>
        </table>
    </div>
</div>

<script>
    let market = 'HK';
    let mode = 'PROFIT';
    const themes = [
        { bg: "#EAE7E2", primary: "#9A8C98", text: "#4A4E69", accent: "#B5838D" },
        { bg: "#D8E2DC", primary: "#8D99AE", text: "#2B2D42", accent: "#E5989B" },
        { bg: "#ECE4DB", primary: "#A68A64", text: "#582F0E", accent: "#84A59D" },
        { bg: "#D6E2E9", primary: "#9DB4C0", text: "#253237", accent: "#E29578" },
        { bg: "#EAF4F4", primary: "#6B9080", text: "#2F3E46", accent: "#D67D7D" }
    ];

    function setMarket(m) {
        market = m;
        document.querySelectorAll('.tab-btn').forEach(b => b.classList.toggle('active', b.innerText.includes(m)));
        document.getElementById('title').innerText = m === 'HK' ? '港股交易計算機' : '美股交易計算機';
    }

    function setMode(m) {
        mode = m;
        document.querySelectorAll('.mode-btn').forEach(b => b.classList.toggle('active', b.onclick.toString().includes(m)));
        document.getElementById('modeLabel').innerText = m === 'PROFIT' ? '目標利潤 (%)' : '預期賣出價 (單價)';
        document.getElementById('modeValue').value = m === 'PROFIT' ? "10" : "";
    }

    function addBatch() {
        const div = document.createElement('div');
        div.className = 'batch-item';
        div.innerHTML = `<input type="number" class="p-in" placeholder="價格" step="0.001"><input type="number" class="s-in" placeholder="股數"><button style="color:red; background:none; border:none; cursor:pointer; font-size:1.2rem;" onclick="this.parentElement.remove()">×</button>`;
        document.getElementById('batchArea').appendChild(div);
    }

    async function fetchName() {
        let q = document.getElementById('symbol').value.trim();
        if (q.length < 1) return;
        if (market === 'HK' && !isNaN(q)) q = q.padStart(4, '0') + ".HK";
        try {
            const res = await fetch(`https://query1.finance.yahoo.com/v1/finance/search?q=${q}`);
            const data = await res.json();
            document.getElementById('stockNameNotice').innerText = data.quotes[0]?.shortname || data.quotes[0]?.longname || "";
        } catch (e) {}
    }

    function run() {
        // 1. 換色
        const t = themes[Math.floor(Math.random() * themes.length)];
        const s = document.documentElement.style;
        s.setProperty('--bg', t.bg); s.setProperty('--primary', t.primary); s.setProperty('--text', t.text); s.setProperty('--accent', t.accent);

        // 2. 稅率計算
        const hkRate = 0.0025 + 0.001 + 0.000042 + 0.000027 + 0.0000565 + 0.0000015;
        const usRate = 0.0008;
        const rate = (market === 'HK') ? hkRate : usRate;

        // 3. 匯總買入資料
        const pIn = document.getElementsByClassName('p-in');
        const sIn = document.getElementsByClassName('s-in');
        let totalPrincipal = 0, totalQty = 0;
        for (let i = 0; i < pIn.length; i++) {
            let p = parseFloat(pIn[i].value), q = parseFloat(sIn[i].value);
            if (p > 0 && q > 0) { totalPrincipal += (p * q); totalQty += q; }
        }
        if (totalQty === 0) return alert("請輸入有效的價格與股數");

        const buyFees = totalPrincipal * rate;
        const totalCost = totalPrincipal + buyFees;
        const inputVal = parseFloat(document.getElementById('modeValue').value);

        let sellPrice, netProfit, profitPct;

        if (mode === 'PROFIT') {
            const targetPct = inputVal / 100;
            sellPrice = (totalCost * (1 + targetPct)) / (1 - rate) / totalQty;
            const sellAmount = sellPrice * totalQty;
            const sellFees = sellAmount * rate;
            netProfit = sellAmount - sellFees - totalCost;
            profitPct = inputVal;
        } else {
            sellPrice = inputVal;
            const sellAmount = sellPrice * totalQty;
            const sellFees = sellAmount * rate;
            netProfit = sellAmount - sellFees - totalCost;
            profitPct = (netProfit / totalCost) * 100;
        }

        const breakeven = totalCost / (1 - rate) / totalQty;
        const totalAllFees = buyFees + (sellPrice * totalQty * rate);

        // 4. 顯示結果
        document.getElementById('res').style.display = 'block';
        document.getElementById('resAvg').innerText = "$" + (totalPrincipal / totalQty).toFixed(3);
        document.getElementById('resFees').innerText = "$" + totalAllFees.toFixed(2);
        document.getElementById('resBreak').innerText = "$" + breakeven.toFixed(3);
        
        if (mode === 'PROFIT') {
            document.getElementById('lbl1').innerText = "預期賣出價"; document.getElementById('val1').innerText = "$" + sellPrice.toFixed(3);
            document.getElementById('lbl2').innerText = "最終純收益"; document.getElementById('val2').innerText = "$" + netProfit.toFixed(2);
        } else {
            document.getElementById('lbl1').innerText = "預期利潤 %"; document.getElementById('val1').innerText = profitPct.toFixed(2) + "%";
            document.getElementById('lbl2').innerText = "最終純收益"; document.getElementById('val2').innerText = "$" + netProfit.toFixed(2);
        }

        // 5. 紀錄
        const row = document.getElementById('hTable').querySelector('tbody').insertRow(0);
        const name = document.getElementById('stockNameNotice').innerText || document.getElementById('symbol').value;
        row.innerHTML = `<td>${market}</td><td>${name}</td><td>$${(totalPrincipal/totalQty).toFixed(2)}</td><td>$${sellPrice.toFixed(2)}</td><td style="color:var(--accent); font-weight:bold;">$${netProfit.toFixed(0)}</td>`;
    }
</script>
</body>
</html>
