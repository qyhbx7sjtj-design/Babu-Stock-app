# Babu-Stock-app
<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>波段風控回測 APP</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; }
        .card { background-color: #1e293b; border-radius: 12px; padding: 16px; margin-bottom: 16px; box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1); }
    </style>
</head>
<body class="bg-slate-900 text-white p-4 pb-20">

    <!-- 標題區 -->
    <div class="flex justify-between items-center mb-6">
        <div>
            <h1 class="text-xl font-bold text-emerald-400">波段風控回測 APP</h1>
            <p class="text-xs text-slate-400">基於 VIP 電子書策略 & 壓縮爆發型態</p>
        </div>
        <div class="bg-emerald-900 text-emerald-300 text-xs px-2 py-1 rounded">V1.0</div>
    </div>

    <!-- 參數設定區 -->
    <div class="card border border-slate-700">
        <h2 class="text-sm font-bold text-slate-300 mb-3 border-b border-slate-700 pb-2">參數設定 (參照書本 P.143)</h2>
        <div class="grid grid-cols-2 gap-4">
            <div>
                <label class="block text-xs text-slate-500 mb-1">總資金 (Capital)</label>
                <input type="number" id="capital" value="1000000" class="w-full bg-slate-800 border border-slate-600 rounded p-2 text-sm focus:outline-none focus:border-emerald-500">
            </div>
            <div>
                <label class="block text-xs text-slate-500 mb-1">單筆風險 %</label>
                <select id="riskPct" class="w-full bg-slate-800 border border-slate-600 rounded p-2 text-sm">
                    <option value="0.01">1% (保守)</option>
                    <option value="0.02" selected>2% (標準)</option>
                    <option value="0.05">5% (積極)</option>
                </select>
            </div>
        </div>
        <button onclick="runSimulation()" class="w-full mt-4 bg-emerald-600 hover:bg-emerald-500 text-white font-bold py-3 rounded-lg transition shadow-lg shadow-emerald-900/50">
            執行回測 / 生成新盤勢
        </button>
    </div>

    <!-- 結果數據區 -->
    <div class="grid grid-cols-3 gap-2 mb-4">
        <div class="bg-slate-800 p-3 rounded-lg text-center border border-slate-700">
            <div class="text-xs text-slate-400">最終資產</div>
            <div id="finalEquity" class="text-sm font-bold text-white">--</div>
        </div>
        <div class="bg-slate-800 p-3 rounded-lg text-center border border-slate-700">
            <div class="text-xs text-slate-400">總獲利</div>
            <div id="totalReturn" class="text-sm font-bold text-white">--</div>
        </div>
        <div class="bg-slate-800 p-3 rounded-lg text-center border border-slate-700">
            <div class="text-xs text-slate-400">交易次數</div>
            <div id="tradeCount" class="text-sm font-bold text-white">--</div>
        </div>
    </div>

    <!-- 圖表區 -->
    <div class="card border border-slate-700 relative" style="height: 350px;">
        <canvas id="strategyChart"></canvas>
    </div>

    <!-- 交易日誌區 -->
    <div class="card border border-slate-700">
        <h2 class="text-sm font-bold text-slate-300 mb-3">詳細交易紀錄 (風控計算)</h2>
        <div class="overflow-x-auto">
            <table class="w-full text-xs text-left text-slate-300">
                <thead class="text-slate-500 bg-slate-800 uppercase">
                    <tr>
                        <th class="px-2 py-2">動作</th>
                        <th class="px-2 py-2">價格</th>
                        <th class="px-2 py-2">張數</th>
                        <th class="px-2 py-2">損益</th>
                    </tr>
                </thead>
                <tbody id="tradeLogBody">
                    <!-- JS 生成內容 -->
                </tbody>
            </table>
        </div>
    </div>

    <script>
        let chartInstance = null;

        // 1. 模擬數據生成器 (模擬壓縮後爆發)
        function generateData(days = 200, startPrice = 50) {
            let prices = [startPrice];
            let ma20 = [];
            let boxHighs = [];
            let volumes = [];
            
            // 隨機決定爆發點
            let breakoutPoint = Math.floor(days * 0.6); 
            
            for (let i = 1; i < days; i++) {
                let prev = prices[i-1];
                let change = 0;
                let vol = 0;

                if (i < breakoutPoint) {
                    // 壓縮期：波動極小
                    change = (Math.random() - 0.5) * 0.015; 
                    vol = Math.floor(Math.random() * 1000) + 500; // 量縮
                } else if (i === breakoutPoint) {
                    // 爆發日：大漲 + 爆量
                    change = 0.06 + (Math.random() * 0.02); 
                    vol = Math.floor(Math.random() * 8000) + 5000; // 爆量
                } else {
                    // 趨勢期：波動變大，趨勢向上
                    change = (Math.random() - 0.4) * 0.03; 
                    vol = Math.floor(Math.random() * 3000) + 1000;
                }

                let newPrice = prev * (1 + change);
                prices.push(newPrice);
                volumes.push(vol);
            }

            // 計算 MA20 和 BoxHigh (過去20天高點)
            for (let i = 0; i < days; i++) {
                if (i < 20) {
                    ma20.push(null);
                    boxHighs.push(null);
                } else {
                    let sum = 0;
                    let maxP = 0;
                    for (let j = 0; j < 20; j++) {
                        sum += prices[i-j];
                        if (prices[i-j-1] > maxP) maxP = prices[i-j-1]; // 昨天的20日高
                    }
                    ma20.push(sum / 20);
                    boxHighs.push(maxP);
                }
            }

            return { prices, ma20, boxHighs, volumes };
        }

        // 2. 核心策略邏輯
        function runSimulation() {
            const capitalInput = parseFloat(document.getElementById('capital').value);
            const riskPct = parseFloat(document.getElementById('riskPct').value);
            
            const data = generateData(150, 50); // 生成150天數據
            
            let cash = capitalInput;
            let shares = 0;
            let entryPrice = 0;
            let stopLoss = 0;
            let trades = [];
            let buySignals = []; // 用於圖表標記
            let sellSignals = [];

            // 遍歷每一天
            for (let i = 21; i < data.prices.length; i++) {
                let price = data.prices[i];
                let ma = data.ma20[i];
                let boxHigh = data.boxHighs[i];
                let vol = data.volumes[i];
                
                // 計算 5日均量
                let volSum = 0;
                for(let k=1; k<=5; k++) volSum += data.volumes[i-k];
                let volMa5 = volSum / 5;

                // --- 進場判斷 ---
                // 1. 空手
                // 2. 突破箱型高點 (Box High)
                // 3. 爆量 (大於2倍均量)
                // 4. 收盤大於 MA20
                if (shares === 0) {
                    if (price > boxHigh && vol > 2 * volMa5 && price > ma) {
                        
                        // [風控核心 P.143]
                        let proposedStop = boxHigh * 0.94; // 假設停損設在突破點下方6%
                        stopLoss = proposedStop;
                        
                        let riskAmount = cash * riskPct; // 可承受虧損金額
                        let riskPerShare = price - stopLoss; // 每股虧損
                        
                        // 計算張數
                        let canBuyShares = Math.floor(riskAmount / riskPerShare);
                        // 不能買超過總現金
                        let maxAffordable = Math.floor(cash / price);
                        let finalShares = Math.min(canBuyShares, maxAffordable);

                        if (finalShares > 0) {
                            shares = finalShares;
                            cash -= shares * price;
                            entryPrice = price;
                            
                            trades.push({
                                type: 'BUY', date: i, price: price, shares: finalShares, 
                                note: `風控買進 ${finalShares} 股`
                            });
                            buySignals.push({x: i, y: price});
                        }
                    }
                }
                // --- 出場判斷 ---
                // 1. 跌破 MA20 (趨勢改變)
                else if (shares > 0) {
                    if (price < ma) {
                        cash += shares * price;
                        let pnl = (price - entryPrice) * shares;
                        trades.push({
                            type: 'SELL', date: i, price: price, shares: shares, 
                            pnl: pnl, note: `跌破月線`
                        });
                        sellSignals.push({x: i, y: price});
                        shares = 0;
                    }
                }
            }
            
            // 如果最後還持有，強制平倉計算損益
            let finalValue = cash;
            if (shares > 0) {
                finalValue += shares * data.prices[data.prices.length-1];
            }

            updateUI(finalValue, capitalInput, trades);
            drawChart(data, buySignals, sellSignals);
        }

        // 3. 更新介面
        function updateUI(final, initial, trades) {
            const profit = final - initial;
            const returnPct = ((profit / initial) * 100).toFixed(2);
            
            document.getElementById('finalEquity').innerText = Math.round(final).toLocaleString();
            
            const returnEl = document.getElementById('totalReturn');
            returnEl.innerText = `${Math.round(profit).toLocaleString()} (${returnPct}%)`;
            returnEl.className = profit >= 0 ? "text-sm font-bold text-emerald-400" : "text-sm font-bold text-rose-400";
            
            document.getElementById('tradeCount').innerText = trades.length;

            // 填充日誌
            const logBody = document.getElementById('tradeLogBody');
            logBody.innerHTML = '';
            trades.forEach(t => {
                let row = document.createElement('tr');
                row.className = "border-b border-slate-700 hover:bg-slate-800";
                
                let pnlClass = "";
                let pnlText = "-";
                if(t.type === 'SELL') {
                    pnlText = Math.round(t.pnl).toLocaleString();
                    pnlClass = t.pnl >= 0 ? "text-emerald-400" : "text-rose-400";
                }

                row.innerHTML = `
                    <td class="px-2 py-2 font-bold ${t.type === 'BUY' ? 'text-emerald-400' : 'text-rose-300'}">${t.type}</td>
                    <td class="px-2 py-2">$${t.price.toFixed(1)}</td>
                    <td class="px-2 py-2">${t.shares}</td>
                    <td class="px-2 py-2 ${pnlClass}">${pnlText}</td>
                `;
                logBody.appendChild(row);
            });
        }

        // 4. 繪製圖表 (Chart.js)
        function drawChart(data, buys, sells) {
            const ctx = document.getElementById('strategyChart').getContext('2d');
            
            if (chartInstance) chartInstance.destroy();

            const labels = data.prices.map((_, i) => i);
            
            // 轉換買賣訊號為圖表數據
            const buyPoints = labels.map(i => {
                const signal = buys.find(b => b.x === i);
                return signal ? signal.y : null;
            });
            const sellPoints = labels.map(i => {
                const signal = sells.find(s => s.x === i);
                return signal ? signal.y : null;
            });

            chartInstance = new Chart(ctx, {
                type: 'line',
                data: {
                    labels: labels,
                    datasets: [
                        {
                            label: '股價 (Close)',
                            data: data.prices,
                            borderColor: '#94a3b8',
                            borderWidth: 2,
                            pointRadius: 0,
                            tension: 0.1
                        },
                        {
                            label: '月線 (MA20)',
                            data: data.ma20,
                            borderColor: '#fbbf24', // 黃色
                            borderWidth: 1.5,
                            pointRadius: 0,
                            borderDash: [5, 5]
                        },
                        {
                            label: '買進訊號',
                            data: buyPoints,
                            backgroundColor: '#10b981', // 綠色
                            borderColor: '#10b981',
                            pointStyle: 'triangle',
                            pointRadius: 8,
                            pointRotation: 0,
                            showLine: false
                        },
                        {
                            label: '賣出訊號',
                            data: sellPoints,
                            backgroundColor: '#f43f5e', // 紅色
                            borderColor: '#f43f5e',
                            pointStyle: 'triangle',
                            pointRadius: 8,
                            pointRotation: 180,
                            showLine: false
                        }
                    ]
                },
                options: {
                    responsive: true,
                    maintainAspectRatio: false,
                    interaction: { intersect: false, mode: 'index' },
                    plugins: {
                        legend: { labels: { color: '#cbd5e1' } },
                        tooltip: { 
                            callbacks: {
                                label: function(context) {
                                    return context.dataset.label + ': ' + context.parsed.y.toFixed(2);
                                }
                            }
                        }
                    },
                    scales: {
                        x: { display: false },
                        y: { 
                            grid: { color: '#334155' },
                            ticks: { color: '#94a3b8' }
                        }
                    }
                }
            });
        }

        // 初始化執行一次
        runSimulation();

    </script>
</body>
</html>
