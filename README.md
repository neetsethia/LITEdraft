<!DOCTYPE html>
<html lang="en" data-theme="light">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>liteDRAFT</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
    <style>
        :root {
            --bg: #f4f4f5; --canvas-bg: #ffffff;
            --ui-bg: rgba(255, 255, 255, 0.98);
            --text: #18181b; --border: #e4e4e7;
            --accent: #6366f1; --highlight: #e0e7ff;
            --wall-color: #18181b;
        }
        [data-theme="dark"] {
            --bg: #18181b; --canvas-bg: #09090b;
            --ui-bg: rgba(24, 24, 27, 0.95);
            --text: #f4f4f5; --border: #27272a;
            --accent: #818cf8; --highlight: #312e81;
            --wall-color: #ffffff;
        }
        body { margin: 0; font-family: 'Inter', system-ui, sans-serif; background: var(--bg); color: var(--text); overflow: hidden; }

        /* --- UI LAYOUT --- */
        #toolbar {
            position: absolute; top: 20px; left: 50%; transform: translateX(-50%);
            background: var(--ui-bg); border: 1px solid var(--border); border-radius: 12px;
            display: flex; gap: 8px; padding: 8px; z-index: 100; box-shadow: 0 4px 12px rgba(0,0,0,0.1);
        }
        
        #inspector {
            position: absolute; right: 20px; top: 20px; width: 280px; height: 85vh;
            background: var(--ui-bg); border: 1px solid var(--border); border-radius: 16px;
            padding: 20px; z-index: 90; box-shadow: 0 4px 12px rgba(0,0,0,0.1);
            display: flex; flex-direction: column;
        }

        .btn {
            width: 40px; height: 40px; border: none; background: transparent; border-radius: 8px;
            cursor: pointer; color: var(--text); opacity: 0.7; transition: all 0.2s; 
            display: flex; align-items: center; justify-content: center; font-size: 18px;
        }
        .btn:hover { background: var(--border); opacity: 1; }
        .btn.active { background: var(--accent); color: white; opacity: 1; }
        
        /* --- OBJECT TREE STYLES --- */
        #object-tree {
            flex-grow: 1; overflow-y: auto; margin-top: 15px; border-top: 1px solid var(--border); padding-top: 10px;
        }
        .tree-item {
            display: flex; align-items: center; justify-content: space-between;
            padding: 8px; border-radius: 6px; font-size: 12px; cursor: pointer;
            border-bottom: 1px solid transparent; transition: background 0.1s;
        }
        .tree-item:hover { background: var(--bg); }
        .tree-item.selected { background: var(--highlight); border-left: 3px solid var(--accent); }
        .tree-label { flex-grow: 1; margin: 0 8px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
        .tree-icon { opacity: 0.5; font-size: 14px; }
        
        /* Inputs */
        input, select { background: var(--bg); color: var(--text); border: 1px solid var(--border); padding: 6px; border-radius: 6px; width: 100%; box-sizing: border-box; }

        #hud {
            position: absolute; background: var(--accent); color: white; padding: 4px 10px;
            border-radius: 4px; font-family: monospace; font-size: 11px;
            pointer-events: none; display: none; z-index: 200;
        }

        canvas { display: block; background: var(--canvas-bg); cursor: crosshair; }
    </style>
</head>
<body>

<div id="hud"></div>

<nav id="toolbar">
    <button class="btn active" id="tool-line" onclick="setTool('line')" title="Draw (L)">📏</button>
    <button class="btn" id="tool-dim" onclick="setTool('dim')" title="Dimension (D)">📐</button>
    <div style="width:1px; background:var(--border); margin:4px 0;"></div>
    <button class="btn" id="tool-select" onclick="setTool('select')" title="Select (V)">🖐️</button>
    <button class="btn" onclick="deleteSelected()" title="Delete">🗑️</button>
    <div style="width:1px; background:var(--border); margin:4px 0;"></div>
    <button class="btn" onclick="toggleTheme()" title="Theme">🌓</button>
</nav>

<aside id="inspector">
    <div style="font-weight:800; font-size:20px; letter-spacing:-0.5px;">liteDRAFT</div>
    <div style="font-size:11px; opacity:0.6; margin-bottom: 15px;">Object Manager</div>

    <div id="properties-panel" style="display:none; margin-bottom: 15px; background: var(--bg); padding:10px; border-radius:8px;">
        <label style="font-size:10px; font-weight:700; opacity:0.5;">NAME / LABEL</label>
        <input type="text" id="prop-name" oninput="updateProp('name', this.value)" style="margin-bottom:8px;">
        
        <label style="font-size:10px; font-weight:700; opacity:0.5;">LAYER</label>
        <select id="prop-layer" onchange="updateProp('layer', this.value)">
            <option value="Structural">Structural</option>
            <option value="Furniture">Furniture</option>
            <option value="Utilities">Utilities</option>
            <option value="Dimension">Dimension</option>
        </select>
    </div>

    <div style="font-size:10px; font-weight:700; opacity:0.5; letter-spacing:1px;">SCENE OBJECTS</div>
    <div id="object-tree">
        <div style="padding:20px; text-align:center; opacity:0.4; font-size:12px;">No objects yet.<br>Start drawing!</div>
    </div>
</aside>

<canvas id="mainCanvas"></canvas>

<script>
    const canvas = document.getElementById('mainCanvas'), ctx = canvas.getContext('2d');
    
    // --- STATE ---
    let shapes = []; 
    let selectedId = null;
    let tool = 'line';
    let drawing = false;
    let p1 = {x:0, y:0}, p2 = {x:0, y:0};
    
    // Config
    const LAYERS = {
        'Structural': { color: '#18181b', width: 3, dash: [] },
        'Furniture':  { color: '#6366f1', width: 2, dash: [] },
        'Utilities':  { color: '#ef4444', width: 2, dash: [10, 5] },
        'Dimension':  { color: '#a1a1aa', width: 1, dash: [] }
    };

    // --- RENDER ---
    function render() {
        if(canvas.width !== window.innerWidth) { canvas.width = window.innerWidth; canvas.height = window.innerHeight; }
        
        // Theme Colors
        const isDark = document.documentElement.getAttribute('data-theme') === 'dark';
        const wallCol = isDark ? '#ffffff' : '#18181b';
        LAYERS.Structural.color = wallCol;

        ctx.fillStyle = getComputedStyle(document.body).getPropertyValue('--canvas-bg');
        ctx.fillRect(0,0,canvas.width, canvas.height);
        drawGrid();

        shapes.forEach(s => {
            if(!s.visible) return;
            
            const style = LAYERS[s.layer] || LAYERS.Structural;
            const isSel = s.id === selectedId;

            ctx.beginPath();
            ctx.strokeStyle = isSel ? '#6366f1' : style.color; // Highlight Selection
            ctx.lineWidth = isSel ? style.width + 2 : style.width;
            ctx.setLineDash(style.dash);
            ctx.moveTo(s.x1, s.y1); ctx.lineTo(s.x2, s.y2);
            ctx.stroke();

            // Draw Label
            if(s.name || s.layer === 'Dimension') drawLabel(s);
            
            // Draw Grips
            if(isSel) drawGrips(s);
        });

        if(drawing) {
            ctx.beginPath(); ctx.strokeStyle = '#6366f1'; ctx.setLineDash([4, 4]);
            ctx.moveTo(p1.x, p1.y); ctx.lineTo(p2.x, p2.y); ctx.stroke();
        }
    }

    function drawGrips(s) {
        ctx.fillStyle = '#6366f1';
        ctx.fillRect(s.x1-4, s.y1-4, 8, 8);
        ctx.fillRect(s.x2-4, s.y2-4, 8, 8);
    }

    function drawLabel(s) {
        const mx = (s.x1+s.x2)/2, my = (s.y1+s.y2)/2;
        const txt = s.name || (Math.hypot(s.x2-s.x1, s.y2-s.y1)/24).toFixed(1)+"'";
        ctx.fillStyle = getComputedStyle(document.body).getPropertyValue('--canvas-bg');
        ctx.fillRect(mx-15, my-8, 30, 16);
        ctx.fillStyle = "#a1a1aa"; ctx.font = "10px Inter"; 
        ctx.textAlign = "center"; ctx.textBaseline = "middle";
        ctx.fillText(txt, mx, my);
    }

    function drawGrid() {
        ctx.strokeStyle = getComputedStyle(document.body).getPropertyValue('--border');
        ctx.lineWidth = 1;
        for(let i=0; i<canvas.width; i+=48) { ctx.beginPath(); ctx.moveTo(i,0); ctx.lineTo(i,canvas.height); ctx.stroke(); }
        for(let i=0; i<canvas.height; i+=48) { ctx.beginPath(); ctx.moveTo(0,i); ctx.lineTo(canvas.width,i); ctx.stroke(); }
    }

    // --- OBJECT TREE MANAGER ---
    function updateTree() {
        const tree = document.getElementById('object-tree');
        if(shapes.length === 0) {
            tree.innerHTML = '<div style="padding:20px; text-align:center; opacity:0.4; font-size:12px;">No objects yet.</div>';
            return;
        }

        tree.innerHTML = shapes.map(s => `
            <div class="tree-item ${s.id === selectedId ? 'selected' : ''}" onclick="selectShape(${s.id})">
                <span class="tree-icon">${s.layer === 'Dimension' ? '📐' : '📏'}</span>
                <span class="tree-label">${s.name || (s.layer + ' ' + s.id.toString().slice(-3))}</span>
                <span class="tree-icon" onclick="toggleVis(event, ${s.id})">${s.visible ? '👁️' : '🕶️'}</span>
            </div>
        `).join('');
    }

    function selectShape(id) {
        selectedId = id;
        const s = shapes.find(x => x.id === id);
        
        // Update Properties Panel
        const panel = document.getElementById('properties-panel');
        if(s) {
            panel.style.display = 'block';
            document.getElementById('prop-name').value = s.name || "";
            document.getElementById('prop-layer').value = s.layer;
            tool = 'select'; // Auto switch tool
            document.querySelectorAll('.btn').forEach(b => b.classList.remove('active'));
            document.getElementById('tool-select').classList.add('active');
        } else {
            panel.style.display = 'none';
        }

        updateTree();
        render();
    }

    function updateProp(prop, val) {
        const s = shapes.find(x => x.id === selectedId);
        if(s) {
            s[prop] = val;
            updateTree(); // Name changed, update tree
            render();
        }
    }

    function toggleVis(e, id) {
        e.stopPropagation();
        const s = shapes.find(x => x.id === id);
        if(s) { s.visible = !s.visible; updateTree(); render(); }
    }

    // --- INTERACTION ---
    canvas.addEventListener('pointerdown', e => {
        const pt = {x: e.offsetX, y: e.offsetY};
        
        if(tool === 'select') {
            // Hit Test
            const hit = shapes.slice().reverse().find(s => {
                const A=pt.x-s.x1, B=pt.y-s.y1, C=s.x2-s.x1, D=s.y2-s.y1;
                const dot = A*C+B*D, lenSq = C*C+D*D;
                let param = lenSq ? dot/lenSq : -1, xx, yy;
                if(param<0) {xx=s.x1; yy=s.y1} else if(param>1) {xx=s.x2; yy=s.y2} else {xx=s.x1+param*C; yy=s.y1+param*D}
                return Math.hypot(pt.x-xx, pt.y-yy) < 10;
            });
            
            if(hit) selectShape(hit.id);
            else selectShape(null);
        } else {
            drawing = true;
            p1 = snap(pt); p2 = p1;
        }
    });

    canvas.addEventListener('pointermove', e => {
        if(drawing) {
            p2 = snap({x: e.offsetX, y: e.offsetY});
            if(e.shiftKey) { const dx=Math.abs(p2.x-p1.x), dy=Math.abs(p2.y-p1.y); if(dx>dy) p2.y=p1.y; else p2.x=p1.x; }
            render();
            showHUD((Math.hypot(p2.x-p1.x, p2.y-p1.y)/24).toFixed(1)+"'");
        }
    });

    canvas.addEventListener('pointerup', () => {
        if(drawing) {
            if(Math.hypot(p2.x-p1.x, p2.y-p1.y) > 5) {
                const newShape = {
                    id: Date.now(),
                    x1: p1.x, y1: p1.y, x2: p2.x, y2: p2.y,
                    layer: tool === 'dim' ? 'Dimension' : 'Structural',
                    visible: true, name: ""
                };
                shapes.push(newShape);
                updateTree();
                render();
            }
            drawing = false; hideHUD();
        }
    });

    // --- UTILS ---
    function snap(p) { return {x: Math.round(p.x/24)*24, y: Math.round(p.y/24)*24}; }
    function deleteSelected() { 
        if(selectedId) { shapes = shapes.filter(s => s.id !== selectedId); selectShape(null); } 
    }
    function setTool(t) { tool = t; document.querySelectorAll('.btn').forEach(b => b.classList.remove('active')); document.getElementById('tool-'+t).classList.add('active'); }
    function showHUD(t) { const h=document.getElementById('hud'); h.style.display='block'; h.innerText=t; h.style.left=(event.clientX+15)+'px'; h.style.top=(event.clientY+15)+'px'; }
    function hideHUD() { document.getElementById('hud').style.display='none'; }
    function toggleTheme() { 
        const root = document.documentElement;
        root.setAttribute('data-theme', root.getAttribute('data-theme') === 'dark' ? 'light' : 'dark');
        render();
    }

    window.onresize = () => render();
    render();
    updateTree();
</script>
</body>
</html>
