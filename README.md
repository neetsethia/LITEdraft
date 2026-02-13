<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>LiteDraft Studio | Enterprise</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
    <style>
        :root {
            --bg: #f4f4f5; --canvas-bg: #ffffff;
            --ui-bg: rgba(255, 255, 255, 0.95);
            --accent: #6366f1; --text: #18181b;
            --border: #e4e4e7; --shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
        }
        body { margin: 0; font-family: 'Inter', system-ui, sans-serif; background: var(--bg); color: var(--text); overflow: hidden; }

        /* --- UI COMPONENTS --- */
        #toolbar {
            position: absolute; top: 20px; left: 50%; transform: translateX(-50%);
            background: var(--ui-bg); border: 1px solid var(--border); border-radius: 12px;
            display: flex; gap: 8px; padding: 8px; box-shadow: var(--shadow); z-index: 100;
            backdrop-filter: blur(8px);
        }
        
        #inspector {
            position: absolute; right: 20px; top: 20px; width: 260px;
            background: var(--ui-bg); border: 1px solid var(--border); border-radius: 12px;
            padding: 20px; box-shadow: var(--shadow); z-index: 90; backdrop-filter: blur(8px);
        }

        .btn {
            width: 40px; height: 40px; border: none; background: transparent; border-radius: 8px;
            cursor: pointer; color: #52525b; transition: all 0.2s; display: flex; align-items: center; justify-content: center;
        }
        .btn:hover { background: #f4f4f5; color: var(--text); }
        .btn.active { background: var(--accent); color: white; }
        
        .section-header { 
            font-size: 11px; font-weight: 600; text-transform: uppercase; letter-spacing: 0.05em; 
            color: #71717a; margin: 16px 0 8px 0; 
        }
        
        #hud {
            position: absolute; background: var(--text); color: white; padding: 6px 12px;
            border-radius: 6px; font-family: 'JetBrains Mono', monospace; font-size: 12px;
            pointer-events: none; display: none; z-index: 200; transform: translate(15px, 15px);
        }

        canvas { display: block; background: var(--canvas-bg); cursor: crosshair; }
    </style>
</head>
<body>

<div id="hud"></div>

<nav id="toolbar">
    <button class="btn active" id="tool-line" onclick="setTool('line')" title="Draw Wall (L)">📏</button>
    <button class="btn" id="tool-dim" onclick="setTool('dim')" title="Dimension (D)">📐</button>
    <div style="width:1px; background:var(--border); margin:4px 0;"></div>
    <button class="btn" id="tool-select" onclick="setTool('select')" title="Select & Edit (V)">🖐️</button>
    <button class="btn" onclick="undo()" title="Undo (Cmd+Z)">⟲</button>
    <div style="width:1px; background:var(--border); margin:4px 0;"></div>
    <button class="btn" onclick="exportPDF()" title="Export with Legend">📄</button>
</nav>

<aside id="inspector">
    <div style="font-weight:700; font-size:16px; margin-bottom:4px;">Project Specs</div>
    <div style="font-size:12px; color:#71717a;">Enterprise Suite v2.0</div>

    <div class="section-header">Active Layer</div>
    <select id="layerSelect" onchange="activeLayer=this.value; render();" style="width:100%; padding:8px; border-radius:6px; border:1px solid var(--border);">
        <option value="Structural">Structural (Wall)</option>
        <option value="Furniture">Furniture</option>
        <option value="Utilities">Utilities (Gas/Elec)</option>
    </select>

    <div class="section-header">Visual Style</div>
    <label style="display:flex; justify-content:space-between; font-size:13px; margin-bottom:8px;">
        <span>Rough Sketch</span>
        <input type="checkbox" id="roughToggle" onchange="render()">
    </label>
    
    <div class="section-header">Cost Estimate (BOM)</div>
    <div style="background:#f4f4f5; padding:12px; border-radius:8px; font-size:12px;">
        <div style="display:flex; justify-content:space-between; margin-bottom:4px;">
            <span>Wall Length:</span> <strong id="bom-ft">0.0'</strong>
        </div>
        <div style="display:flex; justify-content:space-between; color:var(--accent);">
            <span>Total Cost:</span> <strong id="bom-cost">$0.00</strong>
        </div>
    </div>
</aside>

<canvas id="mainCanvas"></canvas>

<script>
    const canvas = document.getElementById('mainCanvas'), ctx = canvas.getContext('2d');
    
    // --- STATE MANAGEMENT ---
    let shapes = [];
    let history = [];
    let tool = 'line';
    let drawing = false;
    let p1 = {x:0, y:0}, p2 = {x:0, y:0};
    let activeLayer = 'Structural';
    let inputBuffer = "";
    
    // Layer Configuration (Color & Cost per Foot)
    const LAYERS = {
        'Structural': { color: '#18181b', width: 3, cost: 55, pattern: [] },
        'Furniture':  { color: '#6366f1', width: 2, cost: 0,  pattern: [] },
        'Utilities':  { color: '#ef4444', width: 2, cost: 25, pattern: [10, 5] },
        'Dimension':  { color: '#a1a1aa', width: 1, cost: 0,  pattern: [] }
    };

    // --- CORE RENDER LOOP ---
    function render() {
        // Resize Canvas to Window
        if(canvas.width !== window.innerWidth) {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }

        ctx.clearRect(0, 0, canvas.width, canvas.height);
        
        // Draw Grid
        drawGrid();

        // Draw Shapes
        shapes.forEach(s => {
            const style = LAYERS[s.layer] || LAYERS['Structural'];
            ctx.beginPath();
            ctx.strokeStyle = style.color;
            ctx.lineWidth = style.width;
            ctx.setLineDash(style.pattern);

            // Rough Mode Logic (Jitter)
            if(document.getElementById('roughToggle').checked && s.layer !== 'Dimension') {
                const jit = () => (Math.random()-0.5)*2;
                ctx.moveTo(s.x1+jit(), s.y1+jit());
                ctx.lineTo(s.x2+jit(), s.y2+jit());
            } else {
                ctx.moveTo(s.x1, s.y1);
                ctx.lineTo(s.x2, s.y2);
            }
            ctx.stroke();

            // Dimension Rendering
            if (s.layer === 'Dimension') drawDimensionText(s);
        });

        // Draw Preview Line
        if(drawing) {
            ctx.beginPath();
            ctx.strokeStyle = '#6366f1';
            ctx.setLineDash([4, 4]);
            ctx.moveTo(p1.x, p1.y);
            ctx.lineTo(p2.x, p2.y);
            ctx.stroke();
            ctx.setLineDash([]);
        }
        
        updateBOM();
    }

    function drawGrid() {
        ctx.strokeStyle = '#e4e4e7';
        ctx.lineWidth = 1;
        const step = 24; // 1ft
        for(let x=0; x<canvas.width; x+=step*2) { ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,canvas.height); ctx.stroke(); }
        for(let y=0; y<canvas.height; y+=step*2) { ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(canvas.width,y); ctx.stroke(); }
    }

    function drawDimensionText(s) {
        const midX = (s.x1 + s.x2) / 2;
        const midY = (s.y1 + s.y2) / 2;
        const len = (Math.hypot(s.x2-s.x1, s.y2-s.y1) / 24).toFixed(1) + "'";
        
        ctx.fillStyle = "#ffffff";
        ctx.fillRect(midX-15, midY-8, 30, 16); // Text Background
        ctx.fillStyle = "#71717a";
        ctx.font = "11px Inter";
        ctx.textAlign = "center";
        ctx.textBaseline = "middle";
        ctx.fillText(len, midX, midY);
    }

    // --- LOGIC & CALCULATIONS ---
    function updateBOM() {
        // Only count 'Structural' lines for cost
        let totalFt = shapes
            .filter(s => s.layer === 'Structural')
            .reduce((sum, s) => sum + Math.hypot(s.x2-s.x1, s.y2-s.y1)/24, 0);
            
        document.getElementById('bom-ft').innerText = totalFt.toFixed(1) + "'";
        document.getElementById('bom-cost').innerText = "$" + (totalFt * LAYERS.Structural.cost).toLocaleString();
    }

    // --- INTERACTION ---
    window.addEventListener('keydown', e => {
        // Direct Distance Entry
        if(drawing && /[0-9.]/.test(e.key)) {
            inputBuffer += e.key;
            showHUD(inputBuffer + "'");
        }
        // Commit Distance
        if(drawing && e.key === 'Enter' && inputBuffer) {
            const distPx = parseFloat(inputBuffer) * 24;
            const angle = Math.atan2(p2.y-p1.y, p2.x-p1.x);
            shapes.push({
                x1: p1.x, y1: p1.y,
                x2: p1.x + distPx * Math.cos(angle),
                y2: p1.y + distPx * Math.sin(angle),
                layer: tool === 'dim' ? 'Dimension' : activeLayer
            });
            drawing = false; inputBuffer = ""; hideHUD(); render();
        }
        // Undo
        if((e.metaKey || e.ctrlKey) && e.key === 'z') undo();
    });

    canvas.addEventListener('pointerdown', e => {
        drawing = true;
        p1 = { x: e.offsetX, y: e.offsetY };
        p2 = p1;
    });

    canvas.addEventListener('pointermove', e => {
        if(!drawing) return;
        p2 = { x: e.offsetX, y: e.offsetY };
        
        // Smart Snap (Orthogonal Shift)
        if(e.shiftKey) {
            const dx = Math.abs(p2.x - p1.x);
            const dy = Math.abs(p2.y - p1.y);
            if(dx > dy) p2.y = p1.y; else p2.x = p1.x;
        }
        
        render();
        const dist = (Math.hypot(p2.x-p1.x, p2.y-p1.y)/24).toFixed(1);
        showHUD(dist + "'");
    });

    canvas.addEventListener('pointerup', () => {
        if(drawing && !inputBuffer) {
            shapes.push({ x1: p1.x, y1: p1.y, x2: p2.x, y2: p2.y, layer: tool === 'dim' ? 'Dimension' : activeLayer });
            drawing = false; hideHUD(); render();
        }
    });

    // --- EXPORT PDF WITH LEGEND ---
    function exportPDF() {
        const { jsPDF } = window.jspdf;
        const doc = new jsPDF('l', 'px', [canvas.width, canvas.height]);
        
        // 1. Draw Project
        shapes.forEach(s => {
            const style = LAYERS[s.layer];
            doc.setDrawColor(style.color);
            doc.setLineWidth(style.width * 0.5); // Thin lines for PDF
            if(style.pattern.length) doc.setLineDash(style.pattern); else doc.setLineDash([]);
            doc.line(s.x1, s.y1, s.x2, s.y2);
        });

        // 2. Draw Legend (Top Right)
        const lx = canvas.width - 150, ly = 20;
        doc.setFillColor(255, 255, 255);
        doc.rect(lx, ly, 130, 100, 'F');
        doc.setDrawColor(0); doc.setLineWidth(1); doc.rect(lx, ly, 130, 100);
        
        doc.setFontSize(12); doc.text("SYMBOL LEGEND", lx+10, ly+20);
        
        let yOff = 40;
        Object.keys(LAYERS).forEach(key => {
            if(key === 'Dimension') return;
            const style = LAYERS[key];
            doc.setDrawColor(style.color);
            doc.setLineWidth(2);
            doc.line(lx+10, ly+yOff, lx+40, ly+yOff);
            
            doc.setTextColor(0); doc.setFontSize(10);
            doc.text(key, lx+50, ly+yOff+3);
            yOff += 20;
        });

        doc.save("Project_Plan_Legend.pdf");
    }

    // --- UTILS ---
    function undo() { shapes.pop(); render(); }
    function setTool(t) { tool = t; document.querySelectorAll('.btn').forEach(b => b.classList.remove('active')); document.getElementById('tool-'+t).classList.add('active'); }
    function showHUD(t) { const h = document.getElementById('hud'); h.style.display = 'block'; h.innerText = t; h.style.left = (event.clientX+15)+'px'; h.style.top = (event.clientY+15)+'px'; }
    function hideHUD() { document.getElementById('hud').style.display = 'none'; }
    
    // Init
    window.onresize = () => render();
    render();
</script>
</body>
</html>
