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
