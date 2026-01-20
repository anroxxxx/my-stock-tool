<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>港美股交易助手</title>
    <style>
        :root {
            --bg: #EAE7E2; --primary: #9A8C98; --text: #4A4E69;
            --card: #FFFFFF; --border: #D1CEC7; --accent: #B5838D;
        }
        body {
            background-color: var(--bg); color: var(--text);
            font-family: sans-serif; transition: all 0.5s ease;
            display: flex; justify-content: center; padding: 20px;
        }
        .main-container { width: 100%; max-width: 650px; }
        .card {
            background: var(--card); padding: 25px;
            border-radius: 15px; box-shadow: 0 4px 20px rgba(0,0,0,0.08);
            margin-bottom: 20px;
        }
        /* Tab 樣式 */
        .tabs { display: flex; margin-bottom: 20px; gap: 10px; }
        .tab-btn {
            flex: 1; padding: 12px; border: none; border-radius: 8px;
            background: rgba(0,0,0,0.05); color: var(--text);
            cursor: pointer; font-weight: bold; transition: 0.3s;
        }
        .tab-btn.active { background: var(--primary); color: white; }

        h2 { text-align: center; color: var(--primary); margin-top: 0; }
        .input-group { margin-bottom: 15px; }
        label { display: block; margin-bottom: 5px; font-weight: bold; font-size: 0.9rem; }
        input {
            width: 100%; padding: 10px; border: 1px solid var(--border);
            border-radius: 8px; box-sizing: border-box; outline: none;
        }
        .batch-item {
            display: grid; grid-template-columns: 1fr 1fr 40px;
            gap: 8px; margin-bottom: 8px;
        }
        .btn { cursor: pointer; border: none; border-radius: 8px; width: 100%; font-weight: bold; }
        .btn-add { background: none; border: 1px dashed var(--primary); color: var(--primary); padding: 8px; margin-bottom: 15px; }
        .btn-calc { background: var(--primary); color: white; padding: 15px; font-size: 1rem; }
        .remove-btn { color: var(--accent); background: none; font-size: 1.2rem; cursor: pointer; }
        
        .results {
            margin-top: 20px; padding: 15px; border-radius: 10px;
            background: rgba(0,0,0,0.02); border: 1px solid var(--border);
        }
        .res-row { display: flex; justify-content: space-between; margin-bottom: 10px; }
        .highlight { color: var(--accent); font-weight: bold; font-size: 1.2rem; }
        
        .history-table { width: 100%; border-collapse: collapse; font-size: 0.8rem; margin-top: 10px; }
        .history-table th, .history-table td { padding: 8px; border-bottom: 1px solid var(--border); text-align: left; }
        #stockNameNotice { font-size: 0.8rem; color: var(--primary); min-height: 1.2rem; margin-top: 4px; }
    </style>
</head>
<body>

<div class="main-container">
    <div class="tabs">
        <button class="tab-btn active" onclick="switchMarket('HK')">🇭🇰 港股模式</button>
        <button class="tab-btn" onclick="switchMarket('US')">🇺🇸 美股模式</button>
    </div>

    <div class="card">
        <h2 id="marketTitle">港股交易計算機</h2>
        
        <div class="input-group">
            <label id="tickerLabel">港股代號</label>
            <input type="text" id="symbol" placeholder="例: 0700" oninput="autoFetchName()">
            <div id="stockNameNotice">輸入代號自動獲取...</div>
        </div>

        <div id="batchContainer">
            <label>買入紀錄 (價格 / 股數)</label>
            <div class="batch-item">
                <input type="number" class="price-in" placeholder="價格" step="0.001">
                <input type="number" class="shares-in" placeholder="股數">
                <span></span>
            </div>
        </div>
        <button class="btn btn-add" onclick="addNewBatch()">+ 新增買入批次</button>

        <div class="input-group">
            <label>預期純利潤 (%)</label>
            <input type="number" id="targetProfit" value="10">
        </div>

        <button class="btn btn-calc" onclick="calculateAll()">計算結果並換色</button>

        <div id="resultDisplay" class="results" style="display:none;">
            <div class="res-row"><span>平均買入成本</span><span id="avgPrice" style="font-weight:bold;">-</span></div>
            <div class="res-row"><span>保本賣出價</span><span id="breakevenPrice" style="font-weight:bold;">-</span></div>
            <div class="res-row"><span>建議賣出價</span><span id="targetSellPrice" class="highlight">-</span></div>
            <div class="res-row"><span>最終預估純收益</span><span id="netProfit" class="highlight">-</span></div>
        </div>
    </div>

    <div class="card">
        <h3>交易紀錄存檔</h3>
        <table class="history-table" id="historyTable">
            <thead>
                <tr><th>市場</th><th>股票</th><th>平均成本</th><th>目標賣出</th><th>純利</th></tr>
            </thead>
            <tbody></tbody>
        </table>
    </div>
</div>

<script>
    let currentMarket = 'HK';
    const themes = [
        { bg: "#EAE7E2", primary: "#9A8C98", text: "#4A4E69", accent: "#B5838D" },
        { bg: "#D8E2DC", primary: "#8D99AE", text: "#2B2D42", accent: "#E5989B" },
        { bg: "#ECE4DB", primary: "#A68A64", text: "#582F0E", accent: "#84A59D" },
        { bg: "#D6E2E9", primary: "#9DB4C0", text: "#253237", accent: "#E29578" }
    ];

    function switchMarket(m) {
        currentMarket = m;
        document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
        event.target.classList.add('active');
        document.getElementById('marketTitle').innerText = m === 'HK' ? '港股交易計算機' : '美股交易計算機';
        document.getElementById('tickerLabel').innerText = m === 'HK' ? '港股代號' : '美股代號 (Ticker)';
        document.getElementById('symbol').placeholder = m === 'HK' ? '例: 0700' : 'e.g. NVDA';
        document.getElementById('resultDisplay').style.display = 'none';
    }

    function addNewBatch() {
        const div = document.createElement('div');
        div.className = 'batch-item';
        div.innerHTML = `<input type="number" class="price-in" placeholder="價格" step="0.001"><input type="number" class="shares-in" placeholder="股數"><button class="remove-btn" onclick="this.parentElement.remove()">×</button>`;
        document.getElementById('batchContainer').appendChild(div);
    }

    async function autoFetchName() {
        let code = document.getElementById('symbol').value.trim();
        if (code.length < 1) return;
        if (currentMarket === 'HK' && !code.includes('.')) code = code.padStart(4, '0') + ".HK";
        try {
            const res = await fetch(`https://query1.finance.yahoo.com/v1/finance/search?q=${code}`);
            const data = await res.json();
            document.getElementById('stockNameNotice').innerText = data.quotes[0]?.shortname || data.quotes[0]?.longname || "找到股名";
        } catch (e) { document.getElementById('stockNameNotice').innerText = "自動獲取中..."; }
    }

    function calculateAll() {
        // 1. 換色
        const theme = themes[Math.floor(Math.random() * themes.length)];
        document.documentElement.style.setProperty('--bg', theme.bg);
        document.documentElement.style.setProperty('--primary', theme.primary);
        document.documentElement.style.setProperty('--text', theme.text);
        document.documentElement.style.setProperty('--accent', theme.accent);

        // 2. 費率選擇
        let totalTaxRate;
        if (currentMarket === 'HK') {
            totalTaxRate = 0.0025 + 0.001 + 0.000042 + 0.000027 + 0.0000565 + 0.0000015; // 港股
        } else {
            totalTaxRate = 0.0008; // 美股 0.08%
        }

        // 3. 數據處理
        const pInputs = document.getElementsByClassName('price-in');
        const sInputs = document.getElementsByClassName('shares-in');
        let totalPrincipal = 0, totalQty = 0;

        for (let i = 0; i < pInputs.length; i++) {
            let p = parseFloat(pInputs[i].value), s = parseFloat(sInputs[i].value);
            if (p > 0 && s > 0) { totalPrincipal += (p * s); totalQty += s; }
        }
        if (totalQty === 0) return alert("請輸入價格和股數");

        const buyFees = totalPrincipal * totalTaxRate;
        const totalInvestment = totalPrincipal + buyFees;
        const targetPct = parseFloat(document.getElementById('targetProfit').value) / 100;

        const targetSell = (totalInvestment * (1 + targetPct)) / (1 - totalTaxRate) / totalQty;
        const breakeven = totalInvestment / (1 - totalTaxRate) / totalQty;
        const netProfit = (targetSell * totalQty * (1 - totalTaxRate)) - totalInvestment;

        // 4. 顯示
        document.getElementById('resultDisplay').style.display = 'block';
        document.getElementById('avgPrice').innerText = "$" + (totalPrincipal / totalQty).toFixed(3);
        document.getElementById('breakevenPrice').innerText = "$" + breakeven.toFixed(3);
        document.getElementById('targetSellPrice').innerText = "$" + targetSell.toFixed(3);
        document.getElementById('netProfit').innerText = "$" + netProfit.toFixed(2);

        // 5. 紀錄
        const row = document.getElementById('historyTable').querySelector('tbody').insertRow(0);
        row.innerHTML = `<td>${currentMarket}</td><td>${document.getElementById('stockNameNotice').innerText}</td><td>$${(totalPrincipal / totalQty).toFixed(2)}</td><td>$${targetSell.toFixed(2)}</td><td style="color:var(--accent); font-weight:bold;">$${netProfit.toFixed(0)}</td>`;
    }
</script>
</body>
</html>
