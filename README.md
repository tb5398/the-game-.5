# the-game-.5                            
the  games code
you can add to the code 
ctrl c+v the game  code  to this  site https://codepen.io/pen/

<!DOCTYPE html>
<html>
<head>
    <title>3D Battlefront: CASTLE & GORE FINAL</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Segoe UI', sans-serif; }
        #dev-console {
            position: absolute; top: -100px; left: 0; width: 100%; height: 50px;
            background: rgba(0, 40, 0, 0.95); border-bottom: 2px solid #0f0;
            display: flex; align-items: center; justify-content: center;
            transition: top 0.3s; z-index: 10001; pointer-events: auto;
        }
        #dev-input {
            background: transparent; border: 1px solid #0f0; color: #0f0;
            width: 70%; padding: 8px; font-family: monospace; outline: none; font-size: 16px;
        }
        #ui { position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; color: white; }
        #unit-control { 
            position: absolute; left: 15px; top: 15px; pointer-events: auto; 
            background: rgba(0, 0, 0, 0.85); padding: 15px; border-radius: 8px; border: 2px solid #00e5ff;
        }
        .deploy-btn { width: 180px; padding: 10px; background: #00e5ff; color: #000; border: none; font-weight: bold; cursor: pointer; margin-top:5px; display: block; text-transform: uppercase; }
        .bar-container { width: 300px; height: 15px; background: #111; border: 2px solid #444; margin: 5px auto; overflow: hidden; }
        #hp-bar { width: 100%; height: 100%; background: #ff3d00; transition: width 0.2s; }
    </style>
</head>
<body>

<div id="dev-console">
    <span style="color:#0f0; font-family:monospace; margin-right:10px;">DEV_SYSTEM></span>
    <input type="text" id="dev-input" placeholder="gold 5000, nuke, freeze">
</div>

<div id="ui">
    <div style="position: absolute; top: 20px; width: 100%; text-align: center;">
        <div style="color: #ff3d00; font-weight: bold;">CASTLE INTEGRITY</div>
        <div class="bar-container"><div id="hp-bar"></div></div>
    </div>
    <div id="unit-control">
        <div style="font-size: 24px; color: #ffd700;">$ <span id="g-val">100</span></div>
        <button class="deploy-btn" onclick="spawnUnit('player', 'soldier', 10)">Soldier (10G)</button>
        <button class="deploy-btn" onclick="spawnUnit('player', 'jeep', 40)">Jeep (40G)</button>
        <button class="deploy-btn" onclick="spawnUnit('player', 'bomber', 30)">Nacho Man (30G)</button>
        <button class="deploy-btn" onclick="spawnUnit('player', 'plane', 60)">Air Strike (60G)</button>
    </div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script>
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x020205);
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);

    const sun = new THREE.DirectionalLight(0xffffff, 1.2);
    sun.position.set(10, 30, 10);
    scene.add(sun, new THREE.AmbientLight(0x444455));

    const ground = new THREE.Mesh(new THREE.PlaneGeometry(100, 300), new THREE.MeshStandardMaterial({color: 0x111111}));
    ground.rotation.x = -Math.PI / 2;
    scene.add(ground);

    // --- CASTLE & ARCHERS ---
    const castle = new THREE.Group();
    const wall = new THREE.Mesh(new THREE.BoxGeometry(40, 15, 10), new THREE.MeshStandardMaterial({color: 0x222222}));
    castle.add(wall);
    const archers = [];
    [-15, 15].forEach(x => {
        const tower = new THREE.Mesh(new THREE.BoxGeometry(6, 22, 6), new THREE.MeshStandardMaterial({color: 0x151515}));
        tower.position.set(x, 3, -5);
        castle.add(tower);
        const archer = new THREE.Mesh(new THREE.BoxGeometry(1, 1.5, 1), new THREE.MeshStandardMaterial({color: 0x00ff00}));
        archer.position.set(x, 14, -5);
        castle.add(archer);
        archers.push({ mesh: archer, shootTimer: 0 });
    });
    castle.position.set(0, 5, 65);
    scene.add(castle);

    let gold = 100, castleHP = 100, gamePaused = false;
    let units = [], projectiles = [], enemyTimer = 0;

    // --- DEV CONSOLE ---
    let devCombo = "";
    window.addEventListener('keydown', e => {
        if (e.shiftKey) {
            devCombo += e.key.toUpperCase();
            if (devCombo.includes("DEV")) {
                document.getElementById('dev-console').style.top = "0px";
                document.getElementById('dev-input').focus();
                devCombo = "";
            }
        }
    });
    document.getElementById('dev-input').addEventListener('keydown', e => {
        if (e.key === 'Enter') {
            const cmd = e.target.value.toLowerCase().split(" ");
            if (cmd[0] === "gold") gold = parseInt(cmd[1]);
            if (cmd[0] === "nuke") units.forEach(u => { if(u.team === 'enemy') u.hp = 0; });
            e.target.value = ""; document.getElementById('dev-console').style.top = "-100px"; e.target.blur();
        }
    });

    function addWound(target, type) {
        if(!target) return;
        const wound = new THREE.Mesh(
            type === 'hole' ? new THREE.CylinderGeometry(0.1, 0.1, 0.2, 6) : new THREE.BoxGeometry(0.05, 0.5, 0.2),
            new THREE.MeshBasicMaterial({color: 0x550000})
        );
        wound.position.set((Math.random()-0.5)*0.8, (Math.random()-0.5)*1.2, 0.51);
        if(type === 'hole') wound.rotation.x = Math.PI/2;
        target.add(wound);
    }

    function spawnUnit(team, type, cost) {
        if(team === 'player' && gold < cost) return;
        if(team === 'player') gold -= cost;
        const group = new THREE.Group();
        let hp = 100, speed = team === 'player' ? -0.15 : 0.15;
        const color = team === 'player' ? 0x0077ff : 0xff2222;

        if (type === 'plane') {
            const fus = new THREE.Mesh(new THREE.BoxGeometry(2, 1, 6), new THREE.MeshStandardMaterial({color: 0x555555}));
            const wings = new THREE.Mesh(new THREE.BoxGeometry(8, 0.2, 2), new THREE.MeshStandardMaterial({color: 0x444444}));
            group.add(fus, wings);
            group.position.set((Math.random()-0.5)*30, 25, 120);
            speed = -0.9; hp = 9999;
        } else if (type === 'jeep') {
            hp = 250; speed = -0.3;
            const car = new THREE.Mesh(new THREE.BoxGeometry(2, 1.2, 4), new THREE.MeshStandardMaterial({color: 0x1b5e20}));
            group.add(car); group.userData.body = car;
        } else {
            if(type === 'bomber') { hp = 60; speed = -0.22; setTimeout(() => { speechSynthesis.speak(new SpeechSynthesisUtterance("I AM NACHO MAN!")); }, 100); }
            const body = new THREE.Mesh(new THREE.BoxGeometry(1, 1.5, 1), new THREE.MeshStandardMaterial({color: type==='bomber'?0xffaa00:color}));
            group.add(body); group.userData.body = body;
        }

        if(type !== 'plane') group.position.set((Math.random()-0.5)*35, 1, team === 'player' ? 55 : -100);
        scene.add(group);
        units.push({ mesh: group, team, type, hp, speed, bombTimer: 0, isDead: false });
    }

    function animate() {
        requestAnimationFrame(animate);
        if(gamePaused) return;
        gold += 0.04;
        document.getElementById('g-val').innerText = Math.floor(gold);
        document.getElementById('hp-bar').style.width = castleHP + "%";

        // Archer Shooting
        archers.forEach(a => {
            a.shootTimer++;
            const target = units.find(u => u.team === 'enemy' && u.mesh.position.z > 0);
            if(target && a.shootTimer > 50) {
                const arr = new THREE.Mesh(new THREE.SphereGeometry(0.25), new THREE.MeshBasicMaterial({color: 0xffff00}));
                arr.position.copy(a.mesh.getWorldPosition(new THREE.Vector3()));
                const dir = new THREE.Vector3().subVectors(target.mesh.position, arr.position).normalize();
                projectiles.push({ mesh: arr, dir, type: 'arrow', life: 100 });
                scene.add(arr); a.shootTimer = 0;
            }
        });

        for (let i = units.length - 1; i >= 0; i--) {
            const u = units[i];
            if(u.isDead) continue;
            let blocked = false;

            // COMBAT COLLISION LOCK
            if(u.type !== 'plane') {
                units.forEach(u2 => {
                    if(u.team !== u2.team && u2.type !== 'plane' && u.mesh.position.distanceTo(u2.mesh.position) < 3.5) {
                        blocked = true;
                        u2.hp -= 1.0;
                        if(Math.random() < 0.05) addWound(u2.mesh.userData.body, 'slash');
                    }
                });
            }

            if(!blocked) u.mesh.position.z += u.speed;

            if(u.type === 'plane') {
                u.bombTimer++;
                if(u.bombTimer % 15 === 0) {
                    const b = new THREE.Mesh(new THREE.SphereGeometry(0.5), new THREE.MeshBasicMaterial({color: 0x000000}));
                    b.position.copy(u.mesh.position); projectiles.push({ mesh: b, type: 'bomb', life: 100 }); scene.add(b);
                }
                if(u.mesh.position.z < -160) { scene.remove(u.mesh); units.splice(i, 1); continue; }
            }

            if(u.team === 'enemy' && u.mesh.position.z > 58) { castleHP -= 5; u.hp = 0; }

            if(u.hp <= 0) {
                u.isDead = true;
                if(u.type === 'bomber') {
                    speechSynthesis.speak(new SpeechSynthesisUtterance("MUNCH ON THIS!"));
                    units.forEach(e => { if(e.mesh.position.distanceTo(u.mesh.position) < 10) e.hp -= 100; });
                }
                u.mesh.rotation.x = Math.PI/2; u.mesh.position.y = 0.1;
                if(u.mesh.userData.body) u.mesh.userData.body.material.color.setHex(0x330000);
                const dBody = u.mesh; setTimeout(() => scene.remove(dBody), 12000);
                units.splice(i, 1);
            }
        }

        projectiles.forEach((p, pi) => {
            if(p.type === 'arrow') p.mesh.position.add(p.dir.clone().multiplyScalar(1.2));
            if(p.type === 'bomb') { p.mesh.position.y -= 0.6; if(p.mesh.position.y <= 0.5) p.life = 0; }
            
            if(p.type === 'bomb' && p.life <= 0) {
                units.forEach(u => { if(u.team === 'enemy' && u.mesh.position.distanceTo(p.mesh.position) < 8) { u.hp -= 100; addWound(u.mesh.userData.body, 'hole'); }});
            }
            if(--p.life <= 0) { scene.remove(p.mesh); projectiles.splice(pi, 1); }
        });

        if(++enemyTimer > 100) { spawnUnit('enemy', 'soldier', 0); enemyTimer = 0; }
        camera.position.set(0, 60, 115); camera.lookAt(0, 0, 10);
        renderer.render(scene, camera);
    }
    animate();
</script>
</body>
</html>











































<!DOCTYPE html>
<html>
<head>
    <title>WAR ENGINE: TOTAL ANNIHILATION</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Courier New', monospace; color: #f00; }
        #ui { position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; z-index: 10; }
        #panel {
            position: absolute; left: 10px; top: 10px; pointer-events: auto;
            background: rgba(20, 0, 0, 0.95); padding: 15px; border: 2px solid #f00; width: 300px;
        }
        select, button {
            width: 100%; padding: 8px; margin-top: 5px; background: #000; color: #f00;
            border: 1px solid #f00; cursor: crosshair; font-weight: bold; font-size: 11px;
        }
        button:hover { background: #f00; color: #000; }
        .gold-text { font-size: 24px; color: #ffd700; text-align: center; margin-bottom: 10px; }
        #counter { position: absolute; top: 10px; right: 10px; font-size: 18px; background: rgba(0,0,0,0.8); padding: 10px; border: 1px solid #f00; }
        #instructions { position: absolute; bottom: 50px; left: 10px; font-size: 11px; color: #666; }
        #console { position: absolute; bottom: 0; left: 0; width: 100%; height: 40px; background: rgba(0,0,0,0.9); border-top: 2px solid #f00; display: none; pointer-events: auto; z-index: 100; }
        #console-input { width: 98%; height: 100%; background: none; border: none; color: #0f0; font-family: 'Courier New', monospace; font-size: 18px; outline: none; padding-left: 20px; }
    </style>
</head>
<body>

<div id="ui">
    <div id="counter">BLOOD SPILLED: <span id="e-count">0</span></div>
    <div id="panel">
        <div class="gold-text">$<span id="g-val">1000</span></div>
        <label>WAR FACTORY:</label>
        <select id="unit-select">
            <option value="grunt">Slasher Grunt (10G)</option>
            <option value="psycho">Chain-Axe Psycho (25G)</option>
            <option value="spider">Spider Droid (60G)</option>
            <option value="tank">Gore Harvester (100G)</option>
            <option value="executioner">Executioner (250G)</option>
            <option value="titan">War Titan (400G)</option>
            <option value="reaper">Blood Reaper (750G)</option>
            <option value="void">The Void Maw (1200G)</option>
            <option value="colossus">Colossus Prime (2500G)</option>
        </select>
        <button onclick="deploy()">DEPLOY UNIT</button>
        <button onclick="document.documentElement.requestFullscreen()">FULLSCREEN</button>
    </div>
    <div id="instructions">WASD: MOVE | SPACE/SHIFT: UP/DOWN | MOUSE: LOOK | PRESS ` FOR CONSOLE</div>
</div>

<div id="console"><input type="text" id="console-input" placeholder="ENTER COMMAND..."></div>
<canvas id="staticCanvas" style="position:fixed; top:0; left:0; width:100%; height:100%; pointer-events:none; z-index:5; opacity:0.12;"></canvas>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/OrbitControls.js"></script>

<script>
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    const masterGain = audioCtx.createGain(); masterGain.gain.value = 0.08; masterGain.connect(audioCtx.destination);
    function playExplosion(vol = 0.3) {
        const osc = audioCtx.createOscillator(); const gain = audioCtx.createGain();
        osc.type = 'sine'; osc.frequency.setValueAtTime(60, audioCtx.currentTime);
        osc.frequency.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 2);
        gain.gain.setValueAtTime(vol, audioCtx.currentTime); gain.gain.linearRampToValueAtTime(0, audioCtx.currentTime + 2);
        osc.connect(gain); gain.connect(masterGain); osc.start(); osc.stop(audioCtx.currentTime + 2);
    }

    const scene = new THREE.Scene(); scene.background = new THREE.Color(0x0a0000);
    const camera = new THREE.PerspectiveCamera(70, window.innerWidth/window.innerHeight, 1, 20000);
    camera.position.set(0, 500, 1500);
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight); document.body.appendChild(renderer.domElement);

    const controls = new THREE.OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;

    scene.add(new THREE.HemisphereLight(0xffffff, 0x000000, 1.2));
    const ground = new THREE.Mesh(new THREE.PlaneGeometry(4000, 20000), new THREE.MeshStandardMaterial({color: 0x020202}));
    ground.rotation.x = -Math.PI/2; scene.add(ground);

    const sCanvas = document.getElementById('staticCanvas'); const sCtx = sCanvas.getContext('2d');
    function drawStatic() {
        sCanvas.width = 128; sCanvas.height = 128; const idata = sCtx.createImageData(128, 128);
        const data = new Uint32Array(idata.data.buffer);
        for(let i=0; i<data.length; i++) { if (Math.random() > 0.97) data[i] = 0x550000ff; }
        sCtx.putImageData(idata, 0, 0);
    }

    const keys = {};
    window.addEventListener('keydown', e => keys[e.code] = true);
    window.addEventListener('keyup', e => keys[e.code] = false);

    function handleCameraMovement() {
        const speed = 20; const dir = new THREE.Vector3(); camera.getWorldDirection(dir);
        const side = new THREE.Vector3().crossVectors(camera.up, dir).normalize();
        if (keys['KeyW']) camera.position.addScaledVector(dir, speed);
        if (keys['KeyS']) camera.position.addScaledVector(dir, -speed);
        if (keys['KeyA']) camera.position.addScaledVector(side, speed);
        if (keys['KeyD']) camera.position.addScaledVector(side, -speed);
        if (keys['Space']) camera.position.y += speed;
        if (keys['ShiftLeft']) camera.position.y -= speed;
    }

    let gold = 1000, units = [], killCount = 0;
    const unitLibrary = {
        grunt: { hp: 100, dmg: 3, speed: 0.8, size: 3, color: 0x224466, type: 'human' },
        psycho: { hp: 200, dmg: 7, speed: 1.1, size: 3.5, color: 0x440044, type: 'human' },
        spider: { hp: 150, dmg: 5, speed: 1.4, size: 2.5, color: 0x333333, type: 'spider' },
        tank: { hp: 1500, dmg: 20, speed: 0.5, size: 8, color: 0x333333, type: 'human' },
        executioner: { hp: 2500, dmg: 45, speed: 0.6, size: 10, color: 0x660000, type: 'human' },
        titan: { hp: 8000, dmg: 80, speed: 0.3, size: 22, color: 0x111111, type: 'human' },
        reaper: { hp: 5000, dmg: 100, speed: 0.9, size: 12, color: 0x000000, type: 'reaper' },
        void: { hp: 30000, dmg: 200, speed: 0.15, size: 50, color: 0x000000, type: 'human' },
        colossus: { hp: 100000, dmg: 500, speed: 0.1, size: 90, color: 0x000000, type: 'human' }
    };

    function createUnitModel(stats, team) {
        const group = new THREE.Group(); const s = stats.size;
        const mat = new THREE.MeshStandardMaterial({ color: team === 'player' ? stats.color : 0x550000 });
        const parts = {};

        if(stats.type === 'human') {
            parts.torso = new THREE.Mesh(new THREE.BoxGeometry(s, s*1.3, s*0.6), mat);
            parts.head = new THREE.Mesh(new THREE.BoxGeometry(s*0.6, s*0.6, s*0.6), mat); parts.head.position.y = s;
            parts.armL = new THREE.Mesh(new THREE.BoxGeometry(s*0.35, s, s*0.35), mat); parts.armL.position.set(-s*0.75, s*0.2, 0);
            parts.armR = parts.armL.clone(); parts.armR.position.x = s*0.75;
            parts.legL = new THREE.Mesh(new THREE.BoxGeometry(s*0.45, s*1.4, s*0.45), mat); parts.legL.position.set(-s*0.35, -s, 0);
            parts.legR = parts.legL.clone(); parts.legR.position.x = s*0.35;
            group.add(parts.torso, parts.head, parts.armL, parts.armR, parts.legL, parts.legR);
        } else if(stats.type === 'spider') {
            parts.torso = new THREE.Mesh(new THREE.BoxGeometry(s*1.5, s*0.5, s*1.5), mat);
            for(let i=0; i<4; i++) {
                let leg = new THREE.Mesh(new THREE.BoxGeometry(s*0.2, s*1.5, s*0.2), mat);
                leg.position.set(i<2?-s:s, -s*0.5, i%2?s:-s);
                group.add(leg);
            }
            group.add(parts.torso);
        } else if(stats.type === 'reaper') {
            parts.torso = new THREE.Mesh(new THREE.ConeGeometry(s, s*2, 8), mat);
            parts.head = new THREE.Mesh(new THREE.SphereGeometry(s*0.5), mat); parts.head.position.y = s;
            group.add(parts.torso, parts.head);
        }
        return { group, parts };
    }

    function spawn(team, type) {
        const stats = unitLibrary[type]; const model = createUnitModel(stats, team);
        model.group.position.set((Math.random()-0.5)*2500, stats.size*2, team==='player'?1500:-4000);
        units.push({ mesh: model.group, parts: model.parts, stats, team, hp: stats.hp, isDead: false, walkCycle: Math.random()*10 });
        scene.add(model.group);
        if(stats.size > 20) playExplosion(0.6);
    }

    function deploy() {
        const type = document.getElementById('unit-select').value;
        const costs = { grunt:10, psycho:25, spider:60, tank:100, executioner:250, titan:400, reaper:750, void:1200, colossus:2500 };
        if(gold >= costs[type]) { gold -= costs[type]; spawn('player', type); }
    }

    window.addEventListener('keydown', e => {
        if (e.key === '`') { const con = document.getElementById('console'); con.style.display = con.style.display==='block'?'none':'block'; if(con.style.display==='block') document.getElementById('console-input').focus(); }
        if (e.key === 'Enter' && document.activeElement.id === 'console-input') {
            const args = document.getElementById('console-input').value.split(' ');
            if (args[0] === 'gold') gold += parseInt(args[1]) || 50000;
            if (args[0] === 'spawn') spawn('player', args[1] || 'void');
            document.getElementById('console').style.display = 'none';
        }
    });

    function animate() {
        requestAnimationFrame(animate);
        gold += 0.8; document.getElementById('g-val').innerText = Math.floor(gold);
        drawStatic(); handleCameraMovement(); controls.update();

        for (let i = units.length - 1; i >= 0; i--) {
            const u = units[i]; if(u.isDead) continue;
            let target = null;
            for (let j = 0; j < units.length; j++) {
                const u2 = units[j];
                if(u.team !== u2.team && !u2.isDead) {
                    const dist = u.mesh.position.distanceTo(u2.mesh.position);
                    if(dist < (u.stats.size + u2.stats.size)*1.5) { target = u2; break; }
                }
            }
            if(target) { target.hp -= u.stats.dmg; } else {
                u.mesh.position.z += u.team==='player'?-u.stats.speed : u.stats.speed;
                u.walkCycle += 0.1;
                if(u.parts.legL) {
                    u.parts.legL.rotation.x = Math.sin(u.walkCycle)*0.5;
                    u.parts.legR.rotation.x = -Math.sin(u.walkCycle)*0.5;
                }
            }
            if(u.hp <= 0) {
                u.isDead = true; u.mesh.rotation.z = 1.5;
                killCount++; document.getElementById('e-count').innerText = killCount;
                setTimeout(() => { scene.remove(u.mesh); units.splice(units.indexOf(u), 1); }, 5000);
            }
        }
        if(Math.random() < 0.15) {
            const keys = Object.keys(unitLibrary);
            spawn('enemy', keys[Math.floor(Math.random()*keys.length)]);
        }
        renderer.render(scene, camera);
    }
    window.addEventListener('mousedown', () => { if (audioCtx.state === 'suspended') audioCtx.resume(); });
    animate();
</script>
</body>
</html>
