<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>港股交易助手 - 穩定版</title>
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
        .main-container { width: 100%; max-width: 600px; }
        .card {
            background: var(--card); padding: 25px;
            border-radius: 15px; box-shadow: 0 4px 20px rgba(0,0,0,0.08);
            margin-bottom: 20px;
        }
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
        .btn {
            cursor: pointer; border: none; border-radius: 8px;
            width: 100%; transition: 0.3s; font-weight: bold;
        }
        .btn-add {
            background: none; border: 1px dashed var(--primary);
            color: var(--primary); padding: 8px; margin-bottom: 15px;
        }
        .btn-calc {
            background: var(--primary); color: white; padding: 15px; font-size: 1rem;
        }
        .remove-btn { color: var(--accent); background: none; font-size: 1.2rem; cursor: pointer; }
        .results {
            margin-top: 20px; padding-top: 15px; border-top: 1px solid var(--border);
        }
        .res-row { display: flex; justify-content: space-between; margin-bottom: 10px; }
        .res-val { font-weight: bold; }
        .highlight { color: var(--accent); font-size: 1.2rem; }
        .history-table { width: 100%; border-collapse: collapse; font-size: 0.8rem; }
        .history-table th, .history-table td { padding: 8px; border-bottom: 1px solid var(--border); text-align: left; }
        #stockNameNotice { font-size: 0.8rem; color: var(--primary); min-height: 1.2rem; margin-top: 4px; }
    </style>
</head>
<body>

<div class="main-container">
    <div class="card">
        <h2>港股交易計算機</h2>
        
        <div class="input-group">
            <label>港股代號</label>
            <input type="text" id="symbol" placeholder="例: 0700" oninput="autoFetchName()">
            <div id="stockNameNotice">輸入代號自動獲取股名...</div>
        </div>

        <div id="batchContainer">
            <label>買入紀錄 (價格 / 股數)</label>
            <div class="batch-item">
                <input type="number" class="price-in" placeholder="買入價" step="0.001">
                <input type="number" class="shares-in" placeholder="股數">
                <span></span>
            </div>
        </div>
        <button class="btn btn-add" onclick="addNewBatch()">+ 新增買入批次</button>

        <div class="input-group">
            <label>預期純利潤 (%)</label>
            <input type="number" id="targetProfit" value="10">
        </div>

        <button class="btn btn-calc" onclick="calculateAll()">立即計算並換色</button>

        <div id="resultDisplay" class="results" style="display:none;">
            <div class="res-row"><span>平均買入成本</span><span id="avgPrice" class="res-val">-</span></div>
            <div class="res-row"><span>總交易稅費</span><span id="totalAllFees" class="res-val">-</span></div>
            <div class="res-row"><span>保本賣出價</span><span id="breakevenPrice" class="res-val">-</span></div>
            <div class="res-row"><span>建議賣出價</span><span id="targetSellPrice" class="res-val highlight">-</span></div>
            <div class="res-row"><span>最終預估純收益</span><span id="netProfit" class="res-val highlight">-</span></div>
        </div>
    </div>

    <div class="card">
        <h3>交易紀錄存檔</h3>
        <table class="history-table" id="historyTable">
            <thead>
                <tr><th>股票</th><th>平均成本</th><th>目標賣出</th><th>預期利潤</th></tr>
            </thead>
            <tbody></tbody>
        </table>
    </div>
</div>

<script>
    const themes = [
        { bg: "#EAE7E2", primary: "#9A8C98", text: "#4A4E69", card: "#FFFFFF", border: "#D1CEC7", accent: "#B5838D" },
        { bg: "#D8E2DC", primary: "#8D99AE", text: "#2B2D42", card: "#FFFFFF", border: "#B8C0B0", accent: "#E5989B" },
        { bg: "#ECE4DB", primary: "#A68A64", text: "#582F0E", card: "#FFFFFF", border: "#D6CCC2", accent: "#84A59D" },
        { bg: "#EAF4F4", primary: "#6B9080", text: "#2F3E46", card: "#F6FFF8", border: "#A4C3B2", accent: "#D67D7D" }
    ];

    function addNewBatch() {
        const div = document.createElement('div');
        div.className = 'batch-item';
        div.innerHTML = `
            <input type="number" class="price-in" placeholder="買入價" step="0.001">
            <input type="number" class="shares-in" placeholder="股數">
            <button class="remove-btn" onclick="this.parentElement.remove()">×</button>
        `;
        document.getElementById('batchContainer').appendChild(div);
    }

    async function autoFetchName() {
        let code = document.getElementById('symbol').value.trim();
        if (code.length < 1) return;
        if (!code.includes('.')) code = code.padStart(4, '0') + ".HK";
        
        const notice = document.getElementById('stockNameNotice');
        try {
            // 使用更寬容的 API 接口
            const res = await fetch(`https://query1.finance.yahoo.com/v1/finance/search?q=${code}`);
            const data = await res.json();
            if(data.quotes && data.quotes[0]) {
                notice.innerText = data.quotes[0].shortname || data.quotes[0].longname;
            }
        } catch (e) {
            notice.innerText = "手動輸入股名或繼續計算";
        }
    }

    function calculateAll() {
        // 1. 換色
        const theme = themes[Math.floor(Math.random() * themes.length)];
        document.documentElement.style.setProperty('--bg', theme.bg);
        document.documentElement.style.setProperty('--primary', theme.primary);
        document.documentElement.style.setProperty('--text', theme.text);
        document.documentElement.style.setProperty('--card', theme.card);
        document.documentElement.style.setProperty('--accent', theme.accent);

        // 2. 費率 (你的設定)
        const broker = 0.0025;
        const stamp = 0.001;
        const exch = 0.000042;
        const sfc = 0.000027;
        const fee = 0.0000565;
        const afrc = 0.0000015;
        const totalTaxRate = broker + stamp + exch + sfc + fee + afrc;

        // 3. 抓數據
        const pInputs = document.getElementsByClassName('price-in');
        const sInputs = document.getElementsByClassName('shares-in');
        let totalPrincipal = 0;
        let totalQty = 0;

        for (let i = 0; i < pInputs.length; i++) {
            let p = parseFloat(pInputs[i].value);
            let s = parseFloat(sInputs[i].value);
            if (p > 0 && s > 0) {
                totalPrincipal += (p * s);
                totalQty += s;
            }
        }

        if (totalQty === 0) {
            alert("請填寫有效的買入價格和股數");
            return;
        }

        // 4. 計算
        const buyFees = totalPrincipal * totalTaxRate;
        const totalInvestment = totalPrincipal + buyFees;
        const profitTarget = parseFloat(document.getElementById('targetProfit').value) / 100;

        // 目標賣出價 = 總投入 * (1+利潤%) / (1-費率) / 總股數
        const targetSell = (totalInvestment * (1 + profitTarget)) / (1 - totalTaxRate) / totalQty;
        const breakeven = totalInvestment / (1 - totalTaxRate) / totalQty;
        
        const sellAmount = targetSell * totalQty;
        const sellFees = sellAmount * totalTaxRate;
        const netProfit = sellAmount - sellFees - totalInvestment;

        // 5. 顯示結果
        document.getElementById('resultDisplay').style.display = 'block';
        document.getElementById('avgPrice').innerText = "$" + (totalPrincipal / totalQty).toFixed(3);
        document.getElementById('totalAllFees').innerText = "$" + (buyFees + sellFees).toFixed(2);
        document.getElementById('breakevenPrice').innerText = "$" + breakeven.toFixed(3);
        document.getElementById('targetSellPrice').innerText = "$" + targetSell.toFixed(3);
        document.getElementById('netProfit').innerText = "$" + netProfit.toFixed(2);

        // 6. 紀錄
        const table = document.getElementById('historyTable').querySelector('tbody');
        const row = table.insertRow(0);
        const sName = document.getElementById('stockNameNotice').innerText;
        row.innerHTML = `
            <td>${sName.includes('輸入') ? document.getElementById('symbol').value : sName}</td>
            <td>$${(totalPrincipal / totalQty).toFixed(2)}</td>
            <td>$${targetSell.toFixed(2)}</td>
            <td style="color:var(--accent); font-weight:bold;">$${netProfit.toFixed(0)}</td>
        `;
    }
</script>

</body>
</html>
