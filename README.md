<!DOCTYPE html>
<html lang="en" data-theme="dark">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>liteDRAFT | CAD Edition</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
    <style>
        :root {
            --bg: #e4e4e7; --canvas-bg: #f4f4f5;
            --ui-bg: #ffffff; --text: #18181b; --border: #d4d4d8;
            --accent: #2563eb; --select: #3b82f6;
            --grid-maj: #d4d4d8; --grid-min: #e4e4e7;
        }
        [data-theme="dark"] {
            --bg: #18181b; --canvas-bg: #09090b;
            --ui-bg: #27272a; --text: #e4e4e7; --border: #3f3f46;
            --accent: #3b82f6; --select: #60a5fa;
            --grid-maj: #27272a; --grid-min: #18181b;
        }
        
        body { margin: 0; font-family: 'Inter', sans-serif; background: var(--bg); color: var(--text); overflow: hidden; user-select: none; }
        
        /* LAYOUT */
        #toolbar-left {
            position: absolute; left: 10px; top: 50%; transform: translateY(-50%);
            background: var(--ui-bg); border: 1px solid var(--border); border-radius: 8px;
            display: flex; flex-direction: column; gap: 4px; padding: 6px; z-index: 100;
        }
        
        #inspector {
            position: absolute; right: 10px; top: 10px; bottom: 50px; width: 240px;
            background: var(--ui-bg); border: 1px solid var(--border); border-radius: 8px;
            padding: 15px; z-index: 90; display: flex; flex-direction: column; gap: 15px;
        }

        #command-bar {
            position: absolute; bottom: 10px; left: 10px; right: 260px;
            background: var(--ui-bg); border: 1px solid var(--border); border-radius: 6px;
            padding: 8px 15px; z-index: 100; font-family: 'JetBrains Mono', monospace; font-size: 12px;
            display: flex; align-items: center; color: var(--text);
        }
        
        .btn {
            width: 36px; height: 36px; border: none; background: transparent; border-radius: 6px;
            cursor: pointer; color: var(--text); opacity: 0.7; font-size: 16px;
            display: flex; align-items: center; justify-content: center;
        }
        .btn:hover { background: var(--bg); opacity: 1; }
        .btn.active { background: var(--accent); color: white; opacity: 1; }
        
        /* INPUTS */
        input, select { background: var(--bg); color: var(--text); border: 1px solid var(--border); padding: 6px; border-radius: 4px; width: 100%; box-sizing: border-box; }
        .row { display: flex; justify-content: space-between; align-items: center; font-size: 11px; font-weight: 600; text-transform: uppercase; color: #71717a; }

        #hud { position: absolute; pointer-events: none; color: var(--accent); font-family: monospace; font-size: 12px; z-index: 200; display:none; }
        
        canvas { display: block; cursor: crosshair; }
    </style>
</head>
<body>

<div id="hud"></div>

<nav id="toolbar-left">
    <button class="btn active" id="t-line" onclick="setTool('line')" title="Line (L)">📏</button>
    <button class="btn" id="t-poly" onclick="setTool('poly')" title="Polyline (PL)">⚡</button>
    <button class="btn" id="t-rect" onclick="setTool('rect')" title="Rectangle (REC)">⬜</button>
    <button class="btn" id="t-circle" onclick="setTool('circle')" title="Circle (C)">⭕</button>
    <div style="height:1px; background:var(--border); margin:2px 0;"></div>
    <button class="btn" id="t-offset" onclick="setTool('offset')" title="Offset (O)">⫷</button>
    <button class="btn" id="t-dim" onclick="setTool('dim')" title="Dimension (D)">📐</button>
    <div style="height:1px; background:var(--border); margin:2px 0;"></div>
    <button class="btn" id="t-select" onclick="setTool('select')" title="Select (Space)">🖐️</button>
</nav>

<div id="command-bar">
    <span style="opacity:0.5; margin-right:10px;">COMMAND ></span>
    <input type="text" id="cmd-input" style="border:none; background:transparent; outline:none; font-family:inherit; color:inherit; width:100%;" placeholder="Type a command (L, C, O, REC)..." onkeydown="handleCmd(event)">
</div>

<aside id="inspector">
    <div style="font-weight:800; font-size:18px; letter-spacing:-0.5px;">liteDRAFT <span style="font-size:10px; opacity:0.5; font-weight:400; vertical-align:middle; margin-left:5px;">CAD</span></div>
    
    <div>
        <div class="row" style="margin-bottom:5px;">Layer Properties</div>
        <select id="layerSelect" onchange="activeLayer=this.value; render();">
            <option value="Wall">Wall (Heavy)</option>
            <option value="Furn">Furniture (Blue)</option>
            <option value="Dim">Dimension (Gray)</option>
            <option value="Util">Utility (Dashed)</option>
        </select>
    </div>

    <div id="prop-panel" style="display:none; background:var(--bg); padding:10px; border-radius:6px;">
        <div class="row" style="margin-bottom:8px;">Selection</div>
        <input type="text" id="prop-label" placeholder="Label..." oninput="updateSel('label', this.value)">
    </div>

    <div style="flex-grow:1; display:flex; flex-direction:column;">
        <div class="row" style="margin-bottom:5px;">Object Tree</div>
        <div id="tree" style="flex-grow:1; overflow-y:auto; font-size:12px;"></div>
    </div>
    
    <div style="display:flex; gap:5px;">
        <button class="btn" onclick="toggleTheme()" style="flex:1;">🌓</button>
        <button class="btn" onclick="exportPDF()" style="flex:1;">📄 PDF</button>
    </div>
</aside>

<canvas id="c"></canvas>

<script>
    const canvas = document.getElementById('c'), ctx = canvas.getContext('2d');
    
    // --- CORE STATE ---
    let shapes = [], view = { x: 0, y: 0, z: 1 };
    let tool = 'line', activeLayer = 'Wall';
    let p1 = null, p2 = null, isDown = false;
    let dragStart = null, selectedId = null;
    let inputBuffer = "";
    
    // Polyline State
    let polyPoints = [];
    
    // Config
    const LAYERS = {
        'Wall': { c: 'var(--text)', w: 3, d: [] },
        'Furn': { c: '#3b82f6', w: 2, d: [] },
        'Dim':  { c: '#a1a1aa', w: 1, d: [] },
        'Util': { c: '#ef4444', w: 2, d: [8, 4] }
    };

    // --- COORDINATE SYSTEMS ---
    function toWorld(ex, ey) { return { x: (ex - view.x) / view.z, y: (ey - view.y) / view.z }; }
    function toScreen(wx, wy) { return { x: wx * view.z + view.x, y: wy * view.z + view.y }; }
    function snap(w) { return { x: Math.round(w.x/12)*12, y: Math.round(w.y/12)*12 }; } // Snap to 12 unit (1ft) grid

    // --- RENDER LOOP ---
    function render() {
        // Handle Resize
        if(canvas.width !== window.innerWidth) { canvas.width = window.innerWidth; canvas.height = window.innerHeight; }
        
        // Clear
        const style = getComputedStyle(document.body);
        ctx.fillStyle = style.getPropertyValue('--canvas-bg');
        ctx.fillRect(0,0,canvas.width, canvas.height);
        
        drawGrid();

        // Transform View
        ctx.save();
        ctx.translate(view.x, view.y);
        ctx.scale(view.z, view.z);

        // Draw Shapes
        shapes.forEach(s => {
            const l = LAYERS[s.layer] || LAYERS.Wall;
            const isSel = s.id === selectedId;
            
            ctx.beginPath();
            ctx.strokeStyle = isSel ? style.getPropertyValue('--select') : (l.c.startsWith('var') ? style.getPropertyValue(l.c.substring(4, l.c.length-1)) : l.c);
            ctx.lineWidth = l.w / view.z * (isSel ? 2 : 1);
            ctx.setLineDash(l.d.map(x => x/view.z));
            
            drawGeometry(ctx, s);
            ctx.stroke();

            // Draw Labels/Dims
            if(s.type === 'dim' || s.label) drawLabel(ctx, s);
        });

        // Draw Interactive
        if(tool === 'poly' && polyPoints.length > 0) {
            ctx.strokeStyle = style.getPropertyValue('--accent');
            ctx.beginPath();
            ctx.moveTo(polyPoints[0].x, polyPoints[0].y);
            for(let i=1; i<polyPoints.length; i++) ctx.lineTo(polyPoints[i].x, polyPoints[i].y);
            if(p2) ctx.lineTo(p2.x, p2.y);
            ctx.stroke();
        } else if (isDown && p1 && p2 && tool !== 'select' && tool !== 'pan') {
            ctx.strokeStyle = style.getPropertyValue('--accent');
            ctx.setLineDash([4/view.z, 4/view.z]);
            ctx.beginPath();
            if(tool === 'rect') ctx.rect(p1.x, p1.y, p2.x-p1.x, p2.y-p1.y);
            else if(tool === 'circle') ctx.arc(p1.x, p1.y, Math.hypot(p2.x-p1.x, p2.y-p1.y), 0, Math.PI*2);
            else { ctx.moveTo(p1.x, p1.y); ctx.lineTo(p2.x, p2.y); }
            ctx.stroke();
        }

        ctx.restore();
    }

    function drawGeometry(c, s) {
        if(s.type === 'line' || s.type === 'dim') { c.moveTo(s.x1, s.y1); c.lineTo(s.x2, s.y2); }
        else if(s.type === 'rect') c.rect(s.x, s.y, s.w, s.h);
        else if(s.type === 'circle') c.arc(s.x, s.y, s.r, 0, Math.PI*2);
        else if(s.type === 'poly') {
            c.moveTo(s.pts[0].x, s.pts[0].y);
            for(let i=1; i<s.pts.length; i++) c.lineTo(s.pts[i].x, s.pts[i].y);
        }
    }

    function drawLabel(c, s) {
        c.save();
        let mx, my, txt;
        if(s.type === 'line' || s.type === 'dim') { mx=(s.x1+s.x2)/2; my=(s.y1+s.y2)/2; txt=(Math.hypot(s.x2-s.x1, s.y2-s.y1)/12).toFixed(1)+"'"; }
        else if(s.type === 'rect') { mx=s.x+s.w/2; my=s.y+s.h/2; txt="RECT"; }
        else if(s.type === 'circle') { mx=s.x; my=s.y; txt="R"+(s.r/12).toFixed(1)+"'"; }
        else { mx=s.pts[0].x; my=s.pts[0].y; txt="POLY"; }
        
        if(s.label) txt = s.label;

        c.fillStyle = getComputedStyle(document.body).getPropertyValue('--canvas-bg');
        c.font = `${12/view.z}px monospace`;
        const m = c.measureText(txt);
        c.fillRect(mx - m.width/2 - 2, my - 6/view.z, m.width + 4, 12/view.z);
        c.fillStyle = '#888';
        c.textAlign = 'center'; c.textBaseline = 'middle';
        c.fillText(txt, mx, my);
        c.restore();
    }

    function drawGrid() {
        ctx.beginPath();
        const step = 24 * view.z;
        const offX = view.x % step, offY = view.y % step;
        ctx.strokeStyle = getComputedStyle(document.body).getPropertyValue('--grid-maj');
        ctx.lineWidth = 1;
        for(let x=offX; x<canvas.width; x+=step) { ctx.moveTo(x,0); ctx.lineTo(x,canvas.height); }
        for(let y=offY; y<canvas.height; y+=step) { ctx.moveTo(0,y); ctx.lineTo(canvas.width,y); }
        ctx.stroke();
    }

    // --- INTERACTION ---
    canvas.addEventListener('wheel', e => {
        e.preventDefault();
        const zoomSpeed = 0.001;
        const newZ = Math.max(0.1, Math.min(5, view.z - e.deltaY * zoomSpeed));
        // Zoom towards mouse
        const mouseW = toWorld(e.offsetX, e.offsetY);
        view.x -= (mouseW.x * newZ + view.x - e.offsetX) - (mouseW.x * view.z + view.x - e.offsetX);
        view.y -= (mouseW.y * newZ + view.y - e.offsetY) - (mouseW.y * view.z + view.y - e.offsetY);
        view.z = newZ;
        render();
    });

    canvas.addEventListener('pointerdown', e => {
        const w = snap(toWorld(e.offsetX, e.offsetY));
        isDown = true; dragStart = {x: e.offsetX, y: e.offsetY};
        
        if(e.button === 1 || e.code === 'Space') { tool = 'pan'; return; } // Middle click Pan

        if(tool === 'select' || tool === 'offset') {
            // Simple hit test
            const clickW = toWorld(e.offsetX, e.offsetY);
            const hit = shapes.find(s => {
                // Dist checks simplified
                if(s.type === 'line') return Math.hypot(clickW.x-(s.x1+s.x2)/2, clickW.y-(s.y1+s.y2)/2) < 20;
                if(s.type === 'rect') return clickW.x > s.x && clickW.x < s.x+s.w && clickW.y > s.y && clickW.y < s.y+s.h;
                if(s.type === 'circle') return Math.abs(Math.hypot(clickW.x-s.x, clickW.y-s.y) - s.r) < 10;
                return false;
            });
            
            if(tool === 'offset' && hit) {
                // Offset Logic (Simple visual clone offset)
                const offDist = 12; // 1ft default offset
                if(hit.type === 'line') shapes.push({ ...hit, id: Date.now(), x1: hit.x1+offDist, y1: hit.y1+offDist, x2: hit.x2+offDist, y2: hit.y2+offDist });
                else if(hit.type === 'circle') shapes.push({ ...hit, id: Date.now(), r: hit.r + offDist });
                else if(hit.type === 'rect') shapes.push({ ...hit, id: Date.now(), x: hit.x-offDist, y: hit.y-offDist, w: hit.w+offDist*2, h: hit.h+offDist*2 });
                render(); updateTree(); tool = 'select'; // Reset
                return;
            }

            select(hit ? hit.id : null);
            return;
        }

        if(tool === 'poly') {
            polyPoints.push(w);
            return;
        }

        p1 = w; p2 = w;
    });

    canvas.addEventListener('pointermove', e => {
        const w = snap(toWorld(e.offsetX, e.offsetY));
        
        if(tool === 'pan' && isDown) {
            view.x += e.offsetX - dragStart.x;
            view.y += e.offsetY - dragStart.y;
            dragStart = {x: e.offsetX, y: e.offsetY};
            render();
            return;
        }

        p2 = w;
        render();
        
        // HUD Update
        const hud = document.getElementById('hud');
        hud.style.display = 'block';
        hud.style.left = (e.clientX+15)+'px'; hud.style.top = (e.clientY+15)+'px';
        hud.innerText = `X:${Math.round(w.x/12)}' Y:${Math.round(w.y/12)}'`;
    });

    canvas.addEventListener('pointerup', e => {
        if(tool === 'pan') { isDown = false; tool = 'select'; return; }
        if(!isDown || tool === 'select' || tool === 'poly') { isDown = false; return; }
        
        // Commit Shape
        const id = Date.now();
        if(tool === 'line' || tool === 'dim') shapes.push({ id, type: tool, x1: p1.x, y1: p1.y, x2: p2.x, y2: p2.y, layer: activeLayer });
        if(tool === 'rect') shapes.push({ id, type: 'rect', x: Math.min(p1.x,p2.x), y: Math.min(p1.y,p2.y), w: Math.abs(p2.x-p1.x), h: Math.abs(p2.y-p1.y), layer: activeLayer });
        if(tool === 'circle') shapes.push({ id, type: 'circle', x: p1.x, y: p1.y, r: Math.hypot(p2.x-p1.x, p2.y-p1.y), layer: activeLayer });
        
        isDown = false; p1 = null; p2 = null;
        updateTree(); render();
    });

    // End Polyline with Enter
    window.addEventListener('keydown', e => {
        if(e.key === 'Enter') {
            if(tool === 'poly' && polyPoints.length > 1) {
                shapes.push({ id: Date.now(), type: 'poly', pts: [...polyPoints], layer: activeLayer });
                polyPoints = []; render(); updateTree();
            }
            // Handle Command Input
            const cmd = document.getElementById('cmd-input');
            if(document.activeElement === cmd) { handleCmd(cmd.value); cmd.value = ''; }
        }
        if(e.key === 'Delete') {
            if(selectedId) { shapes = shapes.filter(s => s.id !== selectedId); select(null); updateTree(); render(); }
        }
    });

    // --- APP LOGIC ---
    function select(id) {
        selectedId = id;
        const panel = document.getElementById('prop-panel');
        const s = shapes.find(x => x.id === id);
        if(s) {
            panel.style.display = 'block';
            document.getElementById('prop-label').value = s.label || '';
        } else {
            panel.style.display = 'none';
        }
        render();
        // Highlight in tree
        const items = document.querySelectorAll('.tree-item');
        items.forEach(i => i.style.background = 'transparent');
        if(id) document.getElementById('tree-'+id).style.background = 'var(--select)';
    }

    function updateSel(prop, val) {
        const s = shapes.find(x => x.id === selectedId);
        if(s) { s[prop] = val; render(); }
    }

    function updateTree() {
        const t = document.getElementById('tree');
        t.innerHTML = shapes.map(s => `
            <div id="tree-${s.id}" class="tree-item" onclick="select(${s.id})" style="padding:4px; cursor:pointer; border-bottom:1px solid var(--border);">
                ${s.type.toUpperCase()} <span style="opacity:0.5">#${s.id.toString().slice(-3)}</span>
            </div>
        `).join('');
    }

    function setTool(t) {
        if(t === 'poly' && polyPoints.length > 0) return; // Finish current poly first?
        tool = t;
        document.querySelectorAll('.btn').forEach(b => b.classList.remove('active'));
        document.getElementById('t-'+t).classList.add('active');
        polyPoints = []; // Reset poly if switching
    }

    function handleCmd(val) {
        if(typeof val !== 'string') return; 
        const c = val.toUpperCase().trim();
        if(c === 'L') setTool('line');
        else if(c === 'C') setTool('circle');
        else if(c === 'REC') setTool('rect');
        else if(c === 'PL') setTool('poly');
        else if(c === 'O') setTool('offset');
        else if(c === 'D') setTool('dim');
    }

    function toggleTheme() {
        const r = document.documentElement;
        r.setAttribute('data-theme', r.getAttribute('data-theme') === 'dark' ? 'light' : 'dark');
        render();
    }
    
    function exportPDF() {
        const { jsPDF } = window.jspdf;
        const doc = new jsPDF();
        shapes.forEach(s => {
            if(s.type === 'line' || s.type === 'dim') doc.line(s.x1, s.y1, s.x2, s.y2);
            if(s.type === 'rect') doc.rect(s.x, s.y, s.w, s.h);
            if(s.type === 'circle') doc.circle(s.x, s.y, s.r);
        });
        doc.save("draft.pdf");
    }

    // Init
    window.onresize = render;
    render();
</script>
</body>
</html>
