<!DOCTYPE html>
<html lang="en" data-theme="dark">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>liteDRAFT | Master Edition</title>
    <style>
        :root {
            --bg: #1e1e1e; --panel: #252526; --border: #333;
            --text: #cccccc; --accent: #007acc; --hover: #2d2d2d;
            --select: #007acc; --group-color: #ffd700;
        }
        body { margin: 0; font-family: 'Segoe UI', sans-serif; background: var(--bg); color: var(--text); height: 100vh; display: flex; flex-direction: column; overflow: hidden; user-select: none; }

        /* --- TOOLBAR --- */
        #toolbar {
            height: 40px; background: var(--panel); border-bottom: 1px solid var(--border);
            display: flex; align-items: center; padding: 0 10px; gap: 8px; z-index: 10;
        }
        .btn {
            width: 32px; height: 32px; border: 1px solid transparent; background: transparent;
            border-radius: 3px; cursor: pointer; color: var(--text); display: flex; align-items: center; justify-content: center; font-size: 16px;
        }
        .btn:hover { background: var(--hover); }
        .btn.active { background: #3e3e42; border-color: var(--accent); color: white; }
        
        .separator { width: 1px; height: 20px; background: var(--border); margin: 0 6px; }

        /* --- MAIN LAYOUT --- */
        #main { display: flex; flex-grow: 1; overflow: hidden; }
        
        /* CANVAS */
        #viewport { flex-grow: 1; overflow: hidden; position: relative; background: #121212; cursor: crosshair; }
        canvas { display: block; width: 100%; height: 100%; }
        
        /* SIDEBAR */
        #sidebar {
            width: 280px; background: var(--panel); border-left: 1px solid var(--border);
            display: flex; flex-direction: column;
        }
        
        /* PROPERTY PANEL */
        #props { padding: 10px; border-bottom: 1px solid var(--border); background: #1e1e1e; }
        .prop-row { display: flex; align-items: center; margin-bottom: 8px; font-size: 12px; }
        .prop-label { width: 70px; color: #888; }
        
        input[type=text], input[type=number] { 
            background: #333; border: 1px solid #444; color: white; padding: 4px; border-radius: 2px; flex-grow: 1;
        }
        input[type=color] {
            border: none; width: 30px; height: 20px; padding: 0; background: transparent; cursor: pointer;
        }
        
        /* TREE VIEW */
        #tree-header { padding: 8px 10px; font-size: 11px; font-weight: 700; text-transform: uppercase; color: #666; background: var(--panel); }
        #tree-content { flex-grow: 1; overflow-y: auto; padding: 5px; font-family: 'Segoe UI', sans-serif; }
        
        .t-node {
            display: flex; align-items: center; padding: 4px 8px; cursor: pointer; font-size: 12px; color: #bbb; border-radius: 2px;
        }
        .t-node:hover { background: var(--hover); }
        .t-node.selected { background: #094771; color: white; }
        .t-icon { width: 16px; text-align: center; margin-right: 6px; opacity: 0.7; }
        .t-name { white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }

        /* FOOTER / STATUS BAR */
        #status-bar {
            height: 24px; background: var(--accent); color: white; font-size: 11px;
            display: flex; align-items: center; padding: 0 10px; justify-content: space-between;
        }
        #storage-meter { display: flex; align-items: center; gap: 5px; }
        progress { width: 80px; height: 8px; border-radius: 4px; }

        /* OVERLAYS */
        #zoom-floater { position: absolute; bottom: 20px; right: 20px; display: flex; gap: 5px; }
        .z-btn { width: 30px; height: 30px; background: var(--panel); border: 1px solid var(--border); color: white; cursor: pointer; box-shadow: 0 2px 5px rgba(0,0,0,0.3); }
    </style>
</head>
<body>

<div id="toolbar">
    <button class="btn active" id="t-select" onclick="setTool('select')" title="Select (V)">🖐️</button>
    <div class="separator"></div>
    <button class="btn" id="t-line" onclick="setTool('line')" title="Line (L)">📏</button>
    <button class="btn" id="t-rect" onclick="setTool('rect')" title="Rectangle (R)">⬜</button>
    <button class="btn" id="t-circle" onclick="setTool('circle')" title="Circle (C)">⭕</button>
    <div class="separator"></div>
    
    <button class="btn" onclick="groupSelection()" title="Group (Ctrl+G)">🔗</button>
    <button class="btn" onclick="ungroupSelection()" title="Ungroup (Ctrl+Shift+G)">💔</button>
    <div class="separator"></div>

    <button class="btn active" id="t-draft" onclick="toggleDraft()" title="Draft Mode (Dims)">📐</button>
    
    <div style="flex-grow:1;"></div>
    
    <button class="btn" onclick="toggleFullscreen()" title="Fullscreen Mode">⛶</button>
    
    <button class="btn" onclick="clearAll()" title="Clear All Canvas" style="color:#ff4d4d;">🗑️</button>
</div>

<div id="main">
    <div id="viewport">
        <canvas id="c"></canvas>
        <div id="zoom-floater">
            <button class="z-btn" onclick="zoom(-0.1)">-</button>
            <button class="z-btn" onclick="zoom(0.1)">+</button>
        </div>
    </div>
    
    <div id="sidebar">
        <div id="props">
            <div class="prop-row">
                <span class="prop-label">Name</span>
                <input type="text" id="p-name" placeholder="Object Name..." oninput="updateProp('name', this.value)">
            </div>
            <div class="prop-row">
                <span class="prop-label">Stroke</span>
                <input type="number" id="p-stroke" value="2" min="1" max="50" onchange="updateProp('stroke', this.value)">
            </div>
            <div class="prop-row">
                <span class="prop-label">Color</span>
                <input type="color" id="p-color" value="#d4d4d4" onchange="updateProp('color', this.value)">
            </div>
        </div>
        
        <div id="tree-header">Scene Hierarchy</div>
        <div id="tree-content"></div>
    </div>
</div>

<div id="status-bar">
    <span id="coords">0, 0</span>
    <div id="storage-meter">
        <span>Storage:</span>
        <progress id="disk-use" value="0" max="100"></progress>
        <span id="disk-text">0%</span>
    </div>
</div>

<script>
    const canvas = document.getElementById('c');
    const ctx = canvas.getContext('2d');
    const viewport = document.getElementById('viewport');
    
    // --- APP STATE ---
    let shapes = [];
    let selection = [];
    let tool = 'select';
    let isDown = false, dragStart = null, lastMouse = null;
    
    // Viewport
    let scale = 1.0;
    let offset = {x: 0, y: 0};
    
    // Settings
    let draftMode = true;
    let nextId = 1;
    let dpi = window.devicePixelRatio || 1;

    // --- INITIALIZATION ---
    function init() {
        resize();
        window.addEventListener('resize', resize);
        loadFromStorage();
        render();
    }

    function resize() {
        // High-DPI Support
        dpi = window.devicePixelRatio || 1;
        canvas.width = viewport.clientWidth * dpi;
        canvas.height = viewport.clientHeight * dpi;
        // Ensure CSS size matches viewport
        canvas.style.width = viewport.clientWidth + 'px';
        canvas.style.height = viewport.clientHeight + 'px';
        
        ctx.scale(dpi, dpi);
        render();
    }

    // --- RENDERING CORE ---
    function render() {
        // Reset transform to clear
        ctx.setTransform(1,0,0,1,0,0);
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        
        // Apply DPI scaling
        ctx.scale(dpi, dpi);

        // Background
        ctx.fillStyle = '#121212';
        ctx.fillRect(0, 0, viewport.clientWidth, viewport.clientHeight);
        
        // Apply Pan/Zoom
        ctx.save();
        ctx.translate(offset.x, offset.y);
        ctx.scale(scale, scale);

        drawGrid();

        // Render Hierarchy (Roots only)
        const roots = shapes.filter(s => !s.parentId);
        roots.forEach(root => drawNode(root));

        // Creation Preview
        if(isDown && tool !== 'select' && dragStart) {
            ctx.strokeStyle = '#007acc'; ctx.lineWidth = 2/scale;
            drawPreview();
        }

        ctx.restore();
    }

    function drawNode(s) {
        // If Group, draw children
        if(s.type === 'group') {
            const children = shapes.filter(c => c.parentId === s.id);
            if(selection.includes(s.id)) {
                const b = getBounds(s);
                ctx.strokeStyle = '#ffd700'; ctx.lineWidth = 1/scale; ctx.setLineDash([5/scale, 5/scale]);
                ctx.strokeRect(b.x, b.y, b.w, b.h);
                ctx.setLineDash([]);
            }
            children.forEach(c => drawNode(c));
            return;
        }

        const isSel = selection.includes(s.id) || (s.parentId && selection.includes(s.parentId));
        
        ctx.beginPath();
        // Use custom color or default
        ctx.strokeStyle = isSel ? '#007acc' : (s.color || '#d4d4d4');
        ctx.lineWidth = (s.stroke || 2) / scale;

        if(s.type === 'line') { ctx.moveTo(s.x, s.y); ctx.lineTo(s.x+s.w, s.y+s.h); }
        else if(s.type === 'rect') ctx.rect(s.x, s.y, s.w, s.h);
        else if(s.type === 'circle') { ctx.beginPath(); ctx.arc(s.x+s.w/2, s.y+s.h/2, Math.abs(s.w/2), 0, Math.PI*2); }
        
        ctx.stroke();

        if(draftMode) drawDimensions(s);
    }

    function drawDimensions(s) {
        if(s.type === 'group') return;
        ctx.save();
        ctx.strokeStyle = '#666'; ctx.fillStyle = '#888';
        ctx.lineWidth = 1/scale; ctx.font = `${10/scale}px sans-serif`;
        ctx.textAlign = "center"; ctx.textBaseline = "middle";
        const off = 20/scale;

        if(s.type === 'line') dimLine(s.x, s.y, s.x+s.w, s.y+s.h, off);
        else if(s.type === 'rect') {
            dimLine(s.x, s.y, s.x+s.w, s.y, -off); // Top
            dimLine(s.x, s.y, s.x, s.y+s.h, -off); // Left
        }
        else if(s.type === 'circle') {
            const r = Math.abs(s.w/2);
            ctx.fillText(`R ${Math.round(r)}`, s.x+s.w/2 + r + off, s.y+s.h/2 - off);
            ctx.beginPath(); ctx.moveTo(s.x+s.w/2, s.y+s.h/2); ctx.lineTo(s.x+s.w/2 + r, s.y+s.h/2); ctx.stroke();
        }
        ctx.restore();
    }

    function dimLine(x1, y1, x2, y2, o) {
        const ang = Math.atan2(y2-y1, x2-x1);
        const dist = Math.hypot(x2-x1, y2-y1);
        ctx.save();
        ctx.translate(x1, y1); ctx.rotate(ang);
        ctx.beginPath(); 
        ctx.moveTo(0,0); ctx.lineTo(0,o); 
        ctx.moveTo(dist,0); ctx.lineTo(dist,o);
        ctx.moveTo(0,o*0.8); ctx.lineTo(dist,o*0.8);
        ctx.stroke();
        if(Math.abs(ang) > Math.PI/2) { ctx.translate(dist/2, o); ctx.rotate(Math.PI); ctx.fillText(Math.round(dist), 0, 15/scale); }
        else ctx.fillText(Math.round(dist), dist/2, o*1.5);
        ctx.restore();
    }

    function drawGrid() {
        ctx.beginPath();
        // Dynamic Grid lines
        // Viewport bounds in world coords
        const startX = -offset.x / scale;
        const startY = -offset.y / scale;
        const endX = startX + (viewport.clientWidth / scale);
        const endY = startY + (viewport.clientHeight / scale);
        
        const step = 50; 
        // Snap to grid lines
        const firstX = Math.floor(startX / step) * step;
        const firstY = Math.floor(startY / step) * step;

        ctx.strokeStyle = '#222'; ctx.lineWidth = 1/scale;
        
        for(let x=firstX; x<endX; x+=step) { ctx.moveTo(x, startY); ctx.lineTo(x, endY); }
        for(let y=firstY; y<endY; y+=step) { ctx.moveTo(startX, y); ctx.lineTo(endX, y); }
        ctx.stroke();
    }

    // --- INTERACTION ---
    function getMouse(e) {
        const r = viewport.getBoundingClientRect();
        return { x: (e.clientX - r.left - offset.x)/scale, y: (e.clientY - r.top - offset.y)/scale };
    }

    viewport.addEventListener('pointerdown', e => {
        isDown = true;
        dragStart = getMouse(e);
        lastMouse = dragStart;

        if(e.button === 1) { return; } // Middle click pan

        if(tool === 'select') {
            const hit = hitTest(dragStart.x, dragStart.y);
            if(hit) {
                const targetId = hit.parentId ? hit.parentId : hit.id;
                if(e.shiftKey) {
                    if(selection.includes(targetId)) selection = selection.filter(id => id !== targetId);
                    else selection.push(targetId);
                } else {
                    if(!selection.includes(targetId)) selection = [targetId];
                }
            } else if(!e.shiftKey) {
                selection = [];
            }
            updateUI(); render();
        }
    });

    viewport.addEventListener('pointermove', e => {
        const curr = getMouse(e);
        document.getElementById('coords').innerText = `${Math.round(curr.x)}, ${Math.round(curr.y)}`;

        if(isDown) {
            if(e.buttons === 4) { // Pan
                offset.x += e.movementX; offset.y += e.movementY;
                render(); return;
            }

            if(tool === 'select' && selection.length > 0) {
                const dx = curr.x - lastMouse.x;
                const dy = curr.y - lastMouse.y;
                selection.forEach(id => moveRecursive(id, dx, dy));
                render();
            }
        }
        lastMouse = curr;
        if(isDown && tool !== 'select') render();
    });

    viewport.addEventListener('pointerup', e => {
        if(isDown && tool !== 'select' && dragStart) {
            const w = lastMouse.x - dragStart.x;
            const h = lastMouse.y - dragStart.y;
            if(Math.abs(w)>2 || Math.abs(h)>2) {
                shapes.push({
                    id: nextId++, type: tool,
                    x: dragStart.x, y: dragStart.y, w: w, h: h,
                    name: `${tool} ${nextId}`,
                    stroke: 2, color: '#d4d4d4'
                });
                saveToStorage();
                updateUI(); render();
            }
        }
        isDown = false; dragStart = null;
    });

    viewport.addEventListener('wheel', e => {
        e.preventDefault();
        const zoomSpeed = 0.001;
        // Zoom towards mouse pointer logic
        const r = viewport.getBoundingClientRect();
        const mouseX = e.clientX - r.left;
        const mouseY = e.clientY - r.top;
        
        const worldBefore = { x: (mouseX - offset.x)/scale, y: (mouseY - offset.y)/scale };
        
        scale = Math.max(0.1, scale - e.deltaY * zoomSpeed);
        
        // Adjust offset to keep mouse world position stable
        offset.x = mouseX - worldBefore.x * scale;
        offset.y = mouseY - worldBefore.y * scale;
        
        render();
    });

    // --- LOGIC ---
    function moveRecursive(id, dx, dy) {
        const s = shapes.find(x => x.id === id);
        if(!s) return;
        if(s.type !== 'group') { s.x += dx; s.y += dy; }
        shapes.filter(c => c.parentId === id).forEach(c => moveRecursive(c.id, dx, dy));
    }

    function hitTest(mx, my) {
        return shapes.slice().reverse().find(s => {
            if(s.type === 'group') return false;
            return mx >= s.x && mx <= s.x+s.w && my >= s.y && my <= s.y+s.h;
        });
    }
    
    function getBounds(s) {
        if(s.type !== 'group') return s;
        let children = shapes.filter(c => c.parentId === s.id);
        if(children.length === 0) return {x:0,y:0,w:0,h:0};
        let minX=Infinity, minY=Infinity, maxX=-Infinity, maxY=-Infinity;
        children.forEach(c => {
            minX = Math.min(minX, c.x); minY = Math.min(minY, c.y);
            maxX = Math.max(maxX, c.x+c.w); maxY = Math.max(maxY, c.y+c.h);
        });
        return {x:minX, y:minY, w:maxX-minX, h:maxY-minY};
    }

    // --- ACTIONS ---
    function clearAll() {
        if(confirm("Are you sure you want to clear the entire canvas?")) {
            shapes = []; selection = []; nextId = 1; saveToStorage(); updateUI(); render();
        }
    }
    
    function toggleFullscreen() {
        if (!document.fullscreenElement) { document.documentElement.requestFullscreen(); }
        else { if (document.exitFullscreen) { document.exitFullscreen(); } }
    }

    function groupSelection() {
        if(selection.length < 1) return;
        const groupId = nextId++;
        const group = { id: groupId, type: 'group', name: 'New Group', parentId: null };
        shapes.push(group);
        selection.forEach(id => {
            const s = shapes.find(x => x.id === id);
            if(s) s.parentId = groupId;
        });
        selection = [groupId];
        saveToStorage(); updateUI(); render();
    }

    function ungroupSelection() {
        selection.forEach(id => {
            const s = shapes.find(x => x.id === id);
            if(s && s.type === 'group') {
                shapes.filter(c => c.parentId === id).forEach(c => c.parentId = null);
                shapes = shapes.filter(x => x.id !== id);
            }
        });
        selection = [];
        saveToStorage(); updateUI(); render();
    }

    function saveToStorage() {
        const data = JSON.stringify({ shapes, nextId });
        const size = new Blob([data]).size;
        const limit = 5 * 1024 * 1024;
        const pct = Math.min(100, (size / limit) * 100);
        document.getElementById('disk-use').value = pct;
        document.getElementById('disk-text').innerText = (size/1024).toFixed(1) + ' KB';
        localStorage.setItem('liteDraft_Project', data);
    }

    function loadFromStorage() {
        const data = localStorage.getItem('liteDraft_Project');
        if(data) {
            const parsed = JSON.parse(data);
            shapes = parsed.shapes;
            nextId = parsed.nextId;
            updateUI();
        }
    }

    function updateUI() {
        if(selection.length === 1) {
            const s = shapes.find(x => x.id === selection[0]);
            document.getElementById('p-name').value = s.name || '';
            document.getElementById('p-stroke').value = s.stroke || 2;
            document.getElementById('p-color').value = s.color || '#d4d4d4';
        } else {
            document.getElementById('p-name').value = '';
        }

        const tree = document.getElementById('tree-content');
        tree.innerHTML = '';
        const buildTree = (parentId, depth) => {
            const nodes = shapes.filter(s => s.parentId === parentId);
            nodes.forEach(s => {
                const el = document.createElement('div');
                el.className = `t-node ${selection.includes(s.id) ? 'selected' : ''}`;
                el.style.paddingLeft = (depth * 15 + 5) + 'px';
                el.innerHTML = `<span class="t-icon">${s.type==='group'?'📁':(s.type==='line'?'📏':'⬜')}</span> <span class="t-name">${s.name}</span>`;
                el.onclick = (e) => {
                    if(e.shiftKey) { /* multi */ } else selection = [s.id];
                    updateUI(); render();
                };
                tree.appendChild(el);
                if(s.type === 'group') buildTree(s.id, depth + 1);
            });
        };
        buildTree(null, 0);
    }
    
    function updateProp(prop, val) {
        if(selection.length > 0) {
            selection.forEach(id => {
                const s = shapes.find(x => x.id === id);
                if(s) {
                    if(prop === 'stroke') s.stroke = parseInt(val);
                    else s[prop] = val;
                }
            });
            saveToStorage(); updateUI(); render();
        }
    }

    function drawPreview() {
        const w = lastMouse.x - dragStart.x;
        const h = lastMouse.y - dragStart.y;
        if(tool === 'line') { ctx.beginPath(); ctx.moveTo(dragStart.x, dragStart.y); ctx.lineTo(lastMouse.x, lastMouse.y); ctx.stroke(); }
        else if(tool === 'rect') ctx.strokeRect(dragStart.x, dragStart.y, w, h);
        else if(tool === 'circle') { ctx.beginPath(); ctx.arc(dragStart.x+w/2, dragStart.y+h/2, Math.abs(w/2), 0, Math.PI*2); ctx.stroke(); }
    }

    function setTool(t) { tool = t; document.querySelectorAll('.btn').forEach(b => b.classList.remove('active')); if(t!=='select') document.getElementById('t-'+t).classList.add('active'); else document.getElementById('t-select').classList.add('active'); }
    function toggleDraft() { draftMode = !draftMode; document.getElementById('t-draft').classList.toggle('active'); render(); }
    function zoom(d) { scale = Math.max(0.1, scale+d); render(); }

    init();
</script>
</body>
</html>
