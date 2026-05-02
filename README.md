<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Secure Mobile Tracker</title>
    <script src="https://unpkg.com/html5-qrcode"></script>
    <style>
        :root { --primary: #2563eb; --secondary: #0f172a; --success: #10b981; --bg: #f8fafc; }
        body { font-family: -apple-system, system-ui, sans-serif; background: var(--bg); margin: 0; padding-bottom: 100px; }
        .top-dash { position: sticky; top: 0; z-index: 100; background: var(--secondary); color: white; padding: 15px 20px; display: flex; justify-content: space-between; align-items: center; }
        .inr-text { font-size: 1.5rem; font-weight: 800; color: var(--success); }
        .container { max-width: 800px; margin: 15px auto; padding: 0 12px; }
        .item-card { background: white; border-radius: 12px; padding: 15px; margin-bottom: 15px; box-shadow: 0 1px 4px rgba(0,0,0,0.1); border: 1px solid #e2e8f0; display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
        .full-width { grid-column: span 2; }
        label { font-size: 10px; font-weight: 700; color: #64748b; text-transform: uppercase; display: block; margin-bottom: 5px; }
        input, select { width: 100%; box-sizing: border-box; padding: 10px; border: 1px solid #e2e8f0; border-radius: 8px; font-size: 14px; }
        .scan-group { display: flex; gap: 8px; }
        .btn-scan { background: var(--primary); color: white; border: none; padding: 0 15px; border-radius: 8px; cursor: pointer; }
        #scanner-overlay { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: #000; z-index: 9999; flex-direction: column; }
        #reader { width: 100%; flex-grow: 1; }
        .scanner-footer { height: 80px; display: flex; justify-content: center; align-items: center; background: #000; }
        .btn-close { background: #ef4444; color: white; border: none; padding: 10px 30px; border-radius: 50px; font-weight: bold; }
        .bottom-actions { position: fixed; bottom: 0; left: 0; right: 0; background: white; padding: 15px; display: flex; justify-content: space-around; border-top: 1px solid #e2e8f0; }
        .btn-footer { border: none; background: none; font-weight: 700; color: #64748b; cursor: pointer; display: flex; flex-direction: column; align-items: center; font-size: 11px; }
        .btn-main { background: var(--primary); color: white; padding: 10px 25px; border-radius: 50px; flex-direction: row; gap: 5px; font-size: 14px; }
    </style>
</head>
<body>

<div class="top-dash">
    <div><div style="font-size: 10px; color: #94a3b8;">TOTAL NET</div><div id="rateLabel" style="font-size: 11px;">Loading...</div></div>
    <div class="inr-text" id="totalINR">₹0.00</div>
</div>

<div class="container">
    <select id="yearFilter" onchange="refresh()" style="width: auto; margin-bottom: 15px;"></select>
    <div id="expenseList"></div>
</div>

<div id="scanner-overlay">
    <div id="reader"></div>
    <div class="scanner-footer"><button class="btn-close" onclick="stopScanner()">CANCEL</button></div>
</div>

<div class="bottom-actions">
    <button class="btn-footer" onclick="document.getElementById('importFile').click()">📥<br>Import</button>
    <button class="btn-footer btn-main" onclick="addNew()" style="color:white;">＋ Add Entry</button>
    <button class="btn-footer" onclick="exportData()">📤<br>Export</button>
</div>

<input type="file" id="importFile" hidden onchange="importData(event)">

<script>
    const DB_KEY = "Tracker_Final_Live_V8";
    let html5QrCode = null;
    let currentInput = null;

    function init() {
        const saved = JSON.parse(localStorage.getItem(DB_KEY)) || [];
        document.getElementById('expenseList').innerHTML = "";
        if(saved.length === 0) addNew();
        else saved.forEach(s => addNew(s));
        
        const yf = document.getElementById('yearFilter');
        const cy = new Date().getFullYear();
        for(let i=cy-2; i<=cy+2; i++) {
            let o = document.createElement('option'); o.value = i; o.innerText = i;
            if(i === cy) o.selected = true; yf.appendChild(o);
        }
        refresh();
    }

    function addNew(data = {}) {
        const div = document.createElement('div');
        div.className = "item-card";
        div.innerHTML = `
            <div><label>Date</label><input type="date" class="d-date" value="${data.date || ''}" onchange="save(); refresh();"></div>
            <div class="full-width"><label>Description</label><input type="text" class="d-item" value="${data.item || ''}" oninput="save();"></div>
            <div><label>Ref ID</label><div class="scan-group">
                <input type="text" class="d-ref" value="${data.ref || ''}" oninput="save();">
                <button class="btn-scan" onclick="startScanner(this)">📷</button>
            </div></div>
            <div><label>Price (₹)</label><input type="number" class="d-price" value="${data.price || ''}" oninput="save(); refresh();"></div>
            <div><label>Disc %</label><input type="number" class="d-disc" value="${data.disc || ''}" oninput="save(); refresh();"></div>
            <div><label>Gift Code</label><input type="text" class="d-gccode" value="${data.gccode || ''}" oninput="save();"></div>
            <div><label>Redeemed?</label><select class="d-redeemed" onchange="save();">
                <option value="No" ${data.redeemed === 'No' ? 'selected' : ''}>No</option>
                <option value="Yes" ${data.redeemed === 'Yes' ? 'selected' : ''}>Yes</option>
            </select></div>
        `;
        document.getElementById('expenseList').appendChild(div);
    }

    async function startScanner(btn) {
        currentInput = btn.parentElement.querySelector('.d-ref');
        document.getElementById('scanner-overlay').style.display = 'flex';
        if (!html5QrCode) html5QrCode = new Html5Qrcode("reader");
        try {
            await html5QrCode.start({ facingMode: "environment" }, { fps: 15, qrbox: 250 }, (text) => {
                currentInput.value = text; stopScanner(); save();
            });
        } catch (err) { alert("Camera failed. Use HTTPS."); stopScanner(); }
    }

    async function stopScanner() {
        if (html5QrCode && html5QrCode.isScanning) await html5QrCode.stop();
        document.getElementById('scanner-overlay').style.display = 'none';
    }

    function refresh() {
        const year = document.getElementById('yearFilter').value;
        let total = 0;
        document.querySelectorAll('.item-card').forEach(card => {
            const dt = card.querySelector('.d-date').value;
            const p = parseFloat(card.querySelector('.d-price').value) || 0;
            const d = parseFloat(card.querySelector('.d-disc').value) || 0;
            if(!dt || dt.startsWith(year)) { total += (p - (p * (d / 100))); card.style.display = "grid"; }
            else card.style.display = "none";
        });
        document.getElementById('totalINR').innerText = `₹${total.toLocaleString('en-IN')}`;
    }

    function save() {
        const list = Array.from(document.querySelectorAll('.item-card')).map(c => ({
            date: c.querySelector('.d-date').value, item: c.querySelector('.d-item').value,
            ref: c.querySelector('.d-ref').value, price: c.querySelector('.d-price').value,
            disc: c.querySelector('.d-disc').value, gccode: c.querySelector('.d-gccode').value,
            redeemed: c.querySelector('.d-redeemed').value
        }));
        localStorage.setItem(DB_KEY, JSON.stringify(list));
    }

    function exportData() {
        const a = document.createElement('a');
        a.href = URL.createObjectURL(new Blob([localStorage.getItem(DB_KEY)], {type: 'application/json'}));
        a.download = `Backup.json`; a.click();
    }

    function importData(e) {
        const r = new FileReader();
        r.onload = () => { localStorage.setItem(DB_KEY, r.result); init(); };
        r.readAsText(e.target.files[0]);
    }

    window.onload = init;
</script>
</body>
</html>
