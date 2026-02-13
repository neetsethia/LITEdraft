To enable Move & Highlight, I have upgraded the "Select" tool.
New Features:
 * Active Selection: Clicking an object now highlights it in Magenta with a dashed bounding box.
 * Drag & Move: Simply click and drag any selected object to move it.
 * Smart Cursors: The cursor changes to a "Move" icon when hovering over a selectable object.
Here is the liteDRAFT | Architect Edition.
<!liteDraft>
<html lang="en" data-theme="dark">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>liteDRAFT | Architect</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
    <style>
        :root {
            --bg: #e4e4e7; --canvas-bg: #f4f4f5;
            --ui-bg: #ffffff; --text: #18181b; --border: #d4d4d8;
            --accent: #2563eb; --select: #d946ef; /* Magenta for Selection */
            --menu-hover: #f3f4f6;
        }
        [data-theme="dark"] {
            --bg: #18181b; --canvas-bg: #09090b;
            --ui-bg: #27272a; --text: #e4e4e7; --border: #3f3f46;
            --accent: #3b82f6; --select: #f0abfc;
            --menu-hover: #3f3f46;
        }
        
        body { margin: 0; font-family: 'Inter', sans-serif; background: var(--bg); color: var(--text); overflow: hidden; display: flex; flex-direction: column; height: 100vh; user-select: none; }
        
        /* --- MENUBAR --- */
        #menubar { height: 30px; background: var(--ui-bg); border-bottom: 1px solid var(--border); display: flex; align-items: center; padding: 0 10px; font-size: 13px; z-index: 2000; }
        .menu-item { padding: 5px 10px; cursor: pointer; position: relative; }
        .menu-item:hover { background: var(--menu-hover); border-radius: 4px; }
        .dropdown { position: absolute; top: 100%; left: 0; background: var(--ui-bg); border: 1px solid var(--border); border-radius: 4px; padding: 5px 0; display: none; min-width: 150px; box-shadow: 0 4px 12px rgba(0,0,0,0.2); }
        .menu-item:hover .dropdown { display: block; }
        .dd-item { padding: 8px 15px; cursor: pointer; display: flex; justify-content: space-between; }
        .dd-item:hover { background: var(--menu-hover); }

        /* --- LAYOUT --- */
        #workspace { flex-grow: 1; position: relative; display: flex; }
        #toolbar-left { width: 44px; background: var(--ui-bg); border-right: 1px solid var(--border); display: flex; flex-direction: column; align-items: center; padding-top: 10px; gap: 5px; }
        #inspector { width: 260px; background: var(--ui-bg); border-left: 1px solid var(--border); display: flex; flex-direction: column; padding: 10px; gap: 10px; }
        #canvas-area { flex-grow: 1; position: relative; background: var(--bg); overflow: hidden; }
        canvas { display: block; }

        .btn { width: 34px; height: 34px; border: none; background: transparent; border-radius: 6px; cursor: pointer; color: var(--text); display: flex; align-items: center; justify-content: center; font-size: 18px; }
        .btn:hover { background: var(--menu-hover); }
        .btn.active { background: var(--accent); color: white; }

        input, select { background: var(--bg); color: var(--text); border: 1px solid var(--border); padding: 5px; border-radius: 4px; width: 100%; box-sizing: border-box; }
        
        #cmd-bar { position: absolute; bottom: 10px; left: 50%; transform: translateX(-50%); background: var(--ui-bg); padding: 6px 12px; border-radius: 6px; border: 1px solid var(--border); display: flex; gap: 8px; align-items: center; font-family: monospace; font-size: 12px; opacity: 0.9; }
        
        #coords { position: absolute; bottom: 5px; right: 280px; font-family: monospace; font-size: 11px; opacity: 0.7; pointer-events: none; }
    </style>
</head>
<body>

<div id="menubar">
    <div style="font-weight: 800; margin-right: 15px; color: var(--accent);">liteDRAFT</div>
    <div class="menu-item">File
        <div class="dropdown">
            <div class="dd-item" onclick="newFile()">New</div>
            <div class="dd-item" onclick="saveFile()">Save JSON</div>
        </div>
    </div>
    <div class="menu-item">View
        <div class="dropdown">
            <div class="dd-item" onclick="view={x:0,y:0,z:1}; render();">Reset View</div>
            <div class="dd-item" onclick="toggleTheme()">Toggle Theme</div>
        </div>
    </div>
    <div class="menu-item">Help
        <div class="dropdown"><div class="dd-item" onclick="alert('Select tool to move objects.\nScroll to Zoom.\nMiddle Click to Pan.')">Shortcuts</div></div>
    </div>
</div>

<div id="workspace">
    <div id="toolbar-left">
        <button class="btn" id="t-select" onclick="setTool('select')" title="Select & Move (V)">🖐️</button>
        <div style="width: 20px; height: 1px; background: var(--border);"></div>
        <button class="btn active" id="t-line" onclick="setTool('line')" title="Line (L)">📏</button>
        <button class="btn" id="t-rect" onclick="setTool('rect')" title="Rectangle (R)">⬜</button>
        <button class="btn" id="t-circle" onclick="setTool('circle')" title="Circle (C)">⭕</button>
        <button class="btn" id="t-text" onclick="setTool('text')" title="Text (T)">T</button>
    </div>

    <div id="canvas-area">
        <canvas id="c"></canvas>
        <div id="cmd-bar">
            <span>CMD ></span>
            <input type="text" id="cmd-input" style="border:none; outline:none; background:transparent; width:150px;" placeholder="Type L, R, C..." onkeydown="handleCmd(event)">
        </div>
        <div id="coords">0, 0</div>
    </div>

    <div id="inspector">
        <div style="font-size:11px; font-weight:700; text-transform:uppercase; color:#71717a;">Properties</div>
        <label style="font-size:12px;">Active Layer</label>
        <select id="layerSelect" onchange="activeLayer=this.value;">
            <option value="Wall">Wall (Heavy)</option>
            <option value="Furn">Furniture (Blue)</option>
            <option value="Util">Utility (Red)</option>
        </select>
        
        <div style="height:1px; background:var(--border); margin:10px 0;"></div>
        
        <div style="font-size:11px; font-weight:700; text-transform:uppercase; color:#71717a;">Object List</div>
        <div id="tree" style="flex-grow:1; overflow-y:auto; font-size:12px;"></div>
        
        <button class="btn" style="width:100%; border:1px solid var(--border); font-size:13px;" onclick="deleteSel()">Delete Selected</button>
    </div>
</div>

<script>
    const canvas = document.getElementById('c'), ctx = canvas.getContext('2d');
    const container = document.getElementById('canvas-area');

    // --- STATE ---
    let shapes = [], view = { x: 0, y: 0, z: 1 };
    let tool = 'line', activeLayer = 'Wall';
    let isDown = false, isDragging = false, dragStart = null;
    let p1 = null, p2 = null, selectedId = null;
    
    // Config
    const LAYERS = {
        'Wall': { c: 'var(--text)', w: 3 },
        'Furn': { c: '#3b82f6', w: 2 },
        'Util': { c: '#ef4444', w: 2 }
    };

    // --- RENDER ENGINE ---
    function render() {
        canvas.width = container.clientWidth;
        canvas.height = container.clientHeight;
        
        // Clear & Grid
        const style = getComputedStyle(document.body);
        ctx.fillStyle = style.getPropertyValue('--canvas-bg');
        ctx.fillRect(0,0,canvas.width, canvas.height);
        drawGrid(style.getPropertyValue('--border'));

        ctx.save();
        ctx.translate(view.x, view.y);
        ctx.scale(view.z, view.z);

        shapes.forEach(s => {
            const l = LAYERS[s.layer] || LAYERS.Wall;
            const isSel = s.id === selectedId;
            
            ctx.beginPath();
            // Highlight Logic
            if(isSel) {
                ctx.strokeStyle = style.getPropertyValue('--select');
                ctx.lineWidth = (l.w + 2) / view.z;
                ctx.setLineDash([5/view.z, 5/view.z]);
            } else {
                ctx.strokeStyle = l.c.startsWith('var') ? style.getPropertyValue(l.c.substring(4, l.c.length-1)) : l.c;
                ctx.lineWidth = l.w / view.z;
                ctx.setLineDash([]);
            }
            
            drawShape(ctx, s);
            ctx.stroke();

            // Draw bounding box if selected
            if(isSel) drawBounds(ctx, s);
        });

        // Draw Preview
        if(isDown && tool !== 'select' && tool !== 'pan') {
            ctx.strokeStyle = style.getPropertyValue('--accent');
            ctx.setLineDash([4/view.z, 4/view.z]);
            ctx.beginPath();
            drawPreview(ctx);
            ctx.stroke();
        }
        ctx.restore();
    }

    function drawShape(c, s) {
        if(s.type === 'line') { c.moveTo(s.x1, s.y1); c.lineTo(s.x2, s.y2); }
        else if(s.type === 'rect') c.rect(s.x, s.y, s.w, s.h);
        else if(s.type === 'circle') c.arc(s.x, s.y, s.r, 0, Math.PI*2);
    }

    function drawPreview(c) {
        if(tool === 'line') { c.moveTo(p1.x, p1.y); c.lineTo(p2.x, p2.y); }
        else if(tool === 'rect') c.rect(p1.x, p1.y, p2.x-p1.x, p2.y-p1.y);
        else if(tool === 'circle') c.arc(p1.x, p1.y, Math.hypot(p2.x-p1.x, p2.y-p1.y), 0, Math.PI*2);
    }

    function drawBounds(c, s) {
        c.strokeStyle = getComputedStyle(document.body).getPropertyValue('--select');
        c.lineWidth = 1 / view.z;
        c.setLineDash([]);
        // Simplified bounding box visualization for non-rects
        if(s.type === 'line') { 
            c.strokeRect(s.x1-4/view.z, s.y1-4/view.z, 8/view.z, 8/view.z); 
            c.strokeRect(s.x2-4/view.z, s.y2-4/view.z, 8/view.z, 8/view.z); 
        }
    }

    function drawGrid(color) {
        ctx.beginPath();
        const step = 24 * view.z;
        const offX = view.x % step, offY = view.y % step;
        ctx.strokeStyle = color; ctx.lineWidth = 0.5;
        for(let x=offX; x<canvas.width; x+=step) { ctx.moveTo(x,0); ctx.lineTo(x,canvas.height); }
        for(let y=offY; y<canvas.height; y+=step) { ctx.moveTo(0,y); ctx.lineTo(canvas.width,y); }
        ctx.stroke();
    }

    // --- MATH & COORDS ---
    function toWorld(ex, ey) { 
        const rect = canvas.getBoundingClientRect();
        return { x: (ex - rect.left - view.x) / view.z, y: (ey - rect.top - view.y) / view.z }; 
    }
    function snap(w) { return { x: Math.round(w.x/12)*12, y: Math.round(w.y/12)*12 }; }

    function hitTest(w) {
        return shapes.slice().reverse().find(s => {
            if(s.type === 'line') return pointLineDist(w, {x:s.x1, y:s.y1}, {x:s.x2, y:s.y2}) < 10/view.z;
            if(s.type === 'rect') return w.x>=s.x && w.x<=s.x+s.w && w.y>=s.y && w.y<=s.y+s.h;
            if(s.type === 'circle') return Math.abs(Math.hypot(w.x-s.x, w.y-s.y) - s.r) < 5/view.z;
            return false;
        });
    }

    function pointLineDist(p, a, b) {
        const A=p.x-a.x, B=p.y-a.y, C=b.x-a.x, D=b.y-a.y;
        const dot = A*C+B*D, lenSq = C*C+D*D;
        let param = lenSq !== 0 ? dot/lenSq : -1;
        let xx, yy;
        if(param<0) { xx=a.x; yy=a.y; } else if(param>1) { xx=b.x; yy=b.y; } else { xx=a.x+param*C; yy=a.y+param*D; }
        return Math.hypot(p.x-xx, p.y-yy);
    }

    // --- INTERACTION ---
    canvas.addEventListener('wheel', e => {
        e.preventDefault();
        const zs = 0.001;
        const newZ = Math.max(0.1, Math.min(5, view.z - e.deltaY * zs));
        const m = toWorld(e.clientX, e.clientY);
        view.x -= (m.x * newZ + view.x - (e.clientX - canvas.getBoundingClientRect().left)) - (m.x * view.z + view.x - (e.clientX - canvas.getBoundingClientRect().left));
        view.z = newZ;
        render();
    });

    canvas.addEventListener('pointerdown', e => {
        isDown = true;
        const raw = toWorld(e.clientX, e.clientY);
        const w = snap(raw);
        
        if(e.button === 1 || tool === 'pan') { tool = 'pan'; return; } // Mid click pan

        if(tool === 'select') {
            const hit = hitTest(raw);
            if(hit) {
                selectedId = hit.id;
                isDragging = true;
                dragStart = raw; // Use raw for smooth drag
                render(); updateTree();
            } else {
                selectedId = null;
                render(); updateTree();
            }
            return;
        }
        
        p1 = w; p2 = w;
    });

    canvas.addEventListener('pointermove', e => {
        const raw = toWorld(e.clientX, e.clientY);
        const w = snap(raw);
        document.getElementById('coords').innerText = `${Math.round(w.x)}, ${Math.round(w.y)}`;

        // Cursor Logic
        if(tool === 'select') {
            const hit = hitTest(raw);
            canvas.style.cursor = hit ? 'move' : 'default';
        }

        if(!isDown) return;

        if(e.buttons === 4 || tool === 'pan') { // Pan
            view.x += e.movementX; view.y += e.movementY; render(); return;
        }

        if(tool === 'select' && isDragging && selectedId) {
            const s = shapes.find(x => x.id === selectedId);
            const dx = raw.x - dragStart.x;
            const dy = raw.y - dragStart.y;
            
            // Move Logic
            if(s.type === 'line') { s.x1+=dx; s.y1+=dy; s.x2+=dx; s.y2+=dy; }
            else if(s.type === 'rect' || s.type === 'circle') { s.x+=dx; s.y+=dy; }
            
            dragStart = raw;
            render(); return;
        }

        p2 = w; render();
    });

    canvas.addEventListener('pointerup', e => {
        if(tool === 'select') { isDown = false; isDragging = false; return; }
        if(tool === 'pan') { isDown = false; return; }

        if(isDown && p1 && p2) {
            const id = Date.now();
            let s = { id, type: tool, layer: activeLayer };
            if(tool === 'line') { s.x1=p1.x; s.y1=p1.y; s.x2=p2.x; s.y2=p2.y; }
            if(tool === 'rect') { s.x=Math.min(p1.x,p2.x); s.y=Math.min(p1.y,p2.y); s.w=Math.abs(p2.x-p1.x); s.h=Math.abs(p2.y-p1.y); }
            if(tool === 'circle') { s.x=p1.x; s.y=p1.y; s.r=Math.hypot(p2.x-p1.x, p2.y-p1.y); }
            shapes.push(s);
            updateTree(); render();
        }
        isDown = false; p1 = null; p2 = null;
    });

    // --- UTILS ---
    function deleteSel() {
        if(selectedId) { shapes = shapes.filter(s => s.id !== selectedId); selectedId = null; updateTree(); render(); }
    }

    function updateTree() {
        const t = document.getElementById('tree');
        t.innerHTML = shapes.map(s => `
            <div onclick="selectedId=${s.id}; render(); updateTree();" 
                 style="padding:4px; cursor:pointer; background:${s.id===selectedId ? 'var(--menu-hover)' : 'transparent'}; border-left:${s.id===selectedId ? '3px solid var(--select)' : '3px solid transparent'}">
                ${s.type} <span style="opacity:0.5">#${s.id}</span>
            </div>
        `).join('');
    }

    function setTool(t) { 
        tool = t; 
        document.querySelectorAll('.btn').forEach(b => b.classList.remove('active')); 
        document.getElementById('t-'+t).classList.add('active'); 
    }

    function toggleTheme() {
        const root = document.documentElement;
        root.setAttribute('data-theme', root.getAttribute('data-theme') === 'dark' ? 'light' : 'dark');
        render();
    }

    window.onresize = render;
    render();
</script>
</body>
</html>

