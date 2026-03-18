<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Minecraft TD - Final Edition</title>
    <style>
        body { margin: 0; background: #000; display: flex; flex-direction: column; align-items: center; font-family: 'Courier New', monospace; color: white; overflow: hidden; }
        #lobby { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: #1a1a1a; display: flex; flex-direction: column; justify-content: center; align-items: center; z-index: 1000; border: 10px solid #5a8d2d; box-sizing: border-box; }
        .mc-btn { background: #4e4e4e; border: 3px solid #000; border-top-color: #8b8b8b; border-left-color: #8b8b8b; color: #e0e0e0; padding: 12px; cursor: pointer; font-family: 'Courier New', monospace; font-size: 1rem; margin: 4px; width: 260px; text-align: center; }
        .mc-btn:hover { background: #5a8d2d; color: white; }
        .mc-btn.active { border-color: #fff; background: #5a8d2d; }
        #game-ui { display: none; flex-direction: column; align-items: center; }
        #hud { width: 1080px; display: flex; justify-content: space-between; padding: 10px; background: #000; border: 4px solid #333; margin-top: 5px; box-sizing: border-box; align-items: center; }
        #game-wrapper { position: relative; display: flex; gap: 10px; padding: 10px; }
        canvas { border: 4px solid #000; background: #5a8d2d; cursor: crosshair; }
        #unit-menu { width: 260px; height: 600px; background: #000; border: 4px solid #333; display: flex; flex-direction: column; padding: 10px; overflow-y: auto; box-sizing: border-box; }
        .unit-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 6px; }
        .unit-card { background: #111; border: 2px solid #333; padding: 5px; cursor: pointer; font-size: 0.6rem; text-align: center; }
        .unit-card:hover { border-color: #5a8d2d; }
        .unit-card.selected { border-color: #fbff00; background: #222; }
        .config-panel { font-size: 0.7rem; background: #222; padding: 5px; border: 1px solid #444; }
        input[type=range] { vertical-align: middle; cursor: pointer; }
    </style>
</head>
<body>

<div id="lobby">
    <h1 style="color: #5a8d2d; font-size: 3.5rem; text-shadow: 5px 5px #000;">MINECRAFT TD</h1>
    <p>DIFICULDADE:</p>
    <button class="mc-btn active" id="d-ez" onclick="setDiff('Easy')">FÁCIL (10 Ondas)</button>
    <button class="mc-btn" id="d-med" onclick="setDiff('Medium')">MÉDIO (25 Ondas)</button>
    <button class="mc-btn" id="d-hard" onclick="setDiff('Hard')">DIFÍCIL (50 Ondas - 2x HP)</button>
    <button class="mc-btn" id="d-inf" onclick="setDiff('Infinite')">MODO INFINITO</button>
    <br>
    <button class="mc-btn" style="background: #3a5a1d; font-weight: bold; width: 300px; height: 60px;" onclick="startGame()">INICIAR AVENTURA</button>
</div>

<div id="game-ui">
    <div id="hud">
        <div id="hp-disp">❤❤❤❤❤</div>
        <div style="color: #fbff00;">ESMERALDAS: <span id="money-display">500</span></div>
        <div id="wave-disp">WAVE: 0</div>
        <div class="config-panel">
            🎵 Som: <input type="range" id="vol-slider" min="0" max="100" value="30" oninput="updateVol(this.value)">
        </div>
    </div>
    <div id="game-wrapper">
        <canvas id="gameCanvas"></canvas>
        <div id="unit-menu">
            <button class="mc-btn" style="width: 100%; font-size: 0.8rem; margin: 0 0 10px 0;" onclick="startWave()">LANÇAR ONDA</button>
            <div id="menu-content" class="unit-grid"></div>
        </div>
    </div>
</div>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    canvas.width = 800; canvas.height = 600;

    let money = 500, lives = 5, wave = 0, maxWaves = 10;
    let difficulty = 'Easy', hpMult = 1, isInfinite = false;
    let towers = [], enemies = [], projectiles = [], frameCount = 0;
    let selectedUnit = null;

    const path = [{x:0,y:5},{x:4,y:5},{x:4,y:9},{x:8,y:9},{x:8,y:7},{x:19,y:7}]; // Pontos principais

    // --- MÚSICA 8-BIT ---
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    const gainNode = audioCtx.createGain();
    gainNode.connect(audioCtx.destination);
    gainNode.gain.value = 0.3;

    function playNote(freq, start, duration) {
        const osc = audioCtx.createOscillator();
        osc.type = 'square';
        osc.frequency.setValueAtTime(freq, start);
        osc.connect(gainNode);
        osc.start(start);
        osc.stop(start + duration);
    }

    function startMusic() {
        const tempo = 0.2;
        setInterval(() => {
            let now = audioCtx.currentTime;
            const melody = [261, 329, 392, 523, 392, 329];
            melody.forEach((note, i) => playNote(note, now + (i * tempo), 0.1));
        }, 1200);
    }

    function updateVol(v) { gainNode.gain.value = v / 100; }

    // --- LÓGICA DO JOGO ---
    function setDiff(d) {
        difficulty = d;
        isInfinite = (d === 'Infinite');
        hpMult = (d === 'Hard') ? 2 : 1;
        maxWaves = d === 'Easy' ? 10 : (d === 'Medium' ? 25 : 50);
        document.querySelectorAll('#lobby .mc-btn').forEach(b => b.classList.remove('active'));
        document.getElementById('d-' + (d==='Easy'?'ez':d==='Medium'?'med':d==='Hard'?'hard':'inf')).classList.add('active');
    }

    function startGame() {
        audioCtx.resume();
        startMusic();
        document.getElementById('lobby').style.display = 'none';
        document.getElementById('game-ui').style.display = 'flex';
        initMenu();
        loop();
    }

    function initMenu() {
        const grid = document.getElementById('menu-content');
        for(let i=0; i<25; i++) {
            const u = { 
                name: i < 20 ? `Torre ${i+1}` : `Herói ${i-19}`,
                cost: i < 20 ? 50 + (i*15) : 1000 + (i*200),
                atk: i < 20 ? 5 + i : 50,
                rng: 120 + (i*2),
                color: `hsl(${i * 15}, 60%, 50%)`,
                hero: i >= 20
            };
            const card = document.createElement('div');
            card.className = 'unit-card';
            card.innerHTML = `<div style="width:20px;height:20px;background:${u.color};margin:auto"></div>${u.name}<br><span style="color:#fb0">$${u.cost}</span>`;
            card.onclick = () => { 
                selectedUnit = u;
                document.querySelectorAll('.unit-card').forEach(c => c.classList.remove('selected'));
                card.classList.add('selected');
            };
            grid.appendChild(card);
        }
    }

    class Enemy {
        constructor() {
            this.x = path[0].x * 40; this.y = path[0].y * 40;
            this.pIdx = 0; this.maxHp = (20 + (wave * 15)) * hpMult;
            this.hp = this.maxHp; this.speed = 1.2;
        }
        update() {
            let target = path[this.pIdx + 1];
            if(!target) { lives--; this.hp = 0; return; }
            let tx = target.x * 40, ty = target.y * 40;
            let dist = Math.hypot(tx - this.x, ty - this.y);
            if(dist < 2) this.pIdx++;
            else { this.x += (tx - this.x)/dist * this.speed; this.y += (ty - this.y)/dist * this.speed; }
        }
        draw() {
            ctx.fillStyle = "#2d4d22"; ctx.fillRect(this.x+10, this.y+10, 20, 20);
            ctx.fillStyle = "#000"; ctx.fillRect(this.x, this.y-5, 40, 4);
            ctx.fillStyle = "#f00"; ctx.fillRect(this.x, this.y-5, (this.hp/this.maxHp)*40, 4);
        }
    }

    function startWave() {
        wave++;
        document.getElementById('wave-disp').innerText = `WAVE: ${wave}`;
        let count = 0;
        let spawn = setInterval(() => {
            enemies.push(new Enemy());
            if(++count >= 5 + wave) clearInterval(spawn);
        }, 600);
    }

    canvas.addEventListener('mousedown', (e) => {
        if(!selectedUnit || money < selectedUnit.cost) return;
        const rect = canvas.getBoundingClientRect();
        const gx = Math.floor(((e.clientX - rect.left)*(800/rect.width))/40)*40;
        const gy = Math.floor(((e.clientY - rect.top)*(600/rect.height))/40)*40;
        towers.push({...selectedUnit, x: gx, y: gy, lastShot: 0});
        money -= selectedUnit.cost;
        document.getElementById('money-display').innerText = money;
    });

    function loop() {
        ctx.fillStyle = "#5a8d2d"; ctx.fillRect(0,0,800,600);
        frameCount++;

        // Desenha caminho
        ctx.fillStyle = "#723232";
        for(let i=0; i<path.length-1; i++) {
            let p1 = path[i], p2 = path[i+1];
            ctx.fillRect(Math.min(p1.x, p2.x)*40, Math.min(p1.y, p2.y)*40, Math.abs(p1.x-p2.x)*40+40, Math.abs(p1.y-p2.y)*40+40);
        }

        enemies.forEach((en, i) => {
            en.update(); en.draw();
            if(en.hp <= 0) { enemies.splice(i, 1); if(en.pIdx < path.length-1) money += 20; document.getElementById('money-display').innerText = money; }
        });

        towers.forEach(t => {
            ctx.fillStyle = t.color; ctx.fillRect(t.x+5, t.y+5, 30, 30);
            if(t.hero) { ctx.strokeStyle = "#a0f"; ctx.strokeRect(t.x+2, t.y+2, 36, 36); }
            
            // Lógica de tiro
            if(frameCount - t.lastShot > 30) {
                let target = enemies.find(en => Math.hypot(en.x - t.x, en.y - t.y) < t.rng);
                if(target) {
                    projectiles.push({x: t.x+20, y: t.y+20, tx: target.x+20, ty: target.y+20, atk: t.atk, target: target});
                    t.lastShot = frameCount;
                }
            }
        });

        projectiles.forEach((p, i) => {
            let dist = Math.hypot(p.tx - p.x, p.ty - p.y);
            p.x += (p.tx - p.x)/dist * 5; p.y += (p.ty - p.y)/dist * 5;
            ctx.fillStyle = "#fff"; ctx.beginPath(); ctx.arc(p.x, p.y, 3, 0, 7); ctx.fill();
            if(dist < 5) { p.target.hp -= p.atk; projectiles.splice(i, 1); }
        });

        document.getElementById('hp-disp').innerText = "❤".repeat(lives);
        if(lives <= 0) { alert("Fim de jogo!"); location.reload(); }

        requestAnimationFrame(loop);
    }
</script>
</body>
</html>
