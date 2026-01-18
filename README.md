# the-game-.5                            
the  games code
you can add to the code 
ctrl c+v the game  code  to this  site https://codepen.io/pen/

!DOCTYPE html>
<html lang="">
  <head>
    <meta charset="utf-8">
    <title></title>
  </head>
  <body>
    <header></header>
    <main></main>
    <footer></footer>
  </body>
<!DOCTYPE html>
<html>
<head>
    <title>3D Soldier Strike: Menu Edition</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Segoe UI', sans-serif; }

        /* Main Menu Styling */
        #main-menu {
            position: absolute; width: 100%; height: 100%;
            background: radial-gradient(circle, #1a1a2e, #000);
            display: flex; flex-direction: column; align-items: center; justify-content: center;
            z-index: 2000; color: gold; text-align: center;
        }
        #main-menu h1 { font-size: 70px; margin-bottom: 10px; text-shadow: 0 0 20px #ff4500; }

        /* UI Layer */
        #ui { position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; color: white; display: none; }

        #unit-sidebar { position: absolute; left: 15px; top: 15px; pointer-events: auto; display: flex; flex-direction: column; gap: 10px; }
        #missile-area { position: absolute; left: 15px; bottom: 30px; pointer-events: auto; background: rgba(0,0,0,0.8); padding: 15px; border-radius: 12px; border: 2px solid #ff4500; }

        #boss-ui { position: absolute; top: 20px; left: 50%; transform: translateX(-50%); width: 400px; text-align: center; }
        #hp-container { width: 100%; height: 20px; background: #333; border: 2px solid #fff; border-radius: 10px; overflow: hidden; margin-top: 5px; }
        #hp-bar { width: 100%; height: 100%; background: linear-gradient(90deg, #ff0000, #ff4500); transition: width 0.3s; }

        .btn { width: 220px; padding: 18px; border: none; border-radius: 8px; color: white; font-weight: bold; cursor: pointer; font-size: 16px; transition: 0.2s; }
        .u-btn { background: #2196f3; border-bottom: 4px solid #0d47a1; }
        .m-btn { background: #ff4500; border-bottom: 4px solid #bf360c; }
        .start-btn { background: #4caf50; border-bottom: 6px solid #2e7d32; font-size: 24px; pointer-events: auto; }
        .btn:hover { transform: scale(1.05); }

        #crosshair { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 40px; height: 40px; border: 2px solid #0f0; border-radius: 50%; display: none; }
    </style>
</head>
<body>

<div id="main-menu">
    <h1>BATTLEFRONT 3D</h1>
    <p style="color: white;">Command your army. Launch the barrage. Lead from the front.</p>
    <br>
    <button class="btn start-btn" onclick="startGame()">START MISSION</button>
</div>

<div id="ui">
    <div id="boss-ui">
        <div style="font-weight: bold;">ENEMY FORTRESS</div>
        <div id="hp-container"><div id="hp-bar"></div></div>
    </div>

    <div id="unit-sidebar">
        <div id="gold-display" style="font-size: 30px; color: gold;">$ <span id="g-val">50</span></div>
        <button class="btn u-btn" onclick="spawnUnit('player', 'soldier', 10)">SOLDIER (10G)</button>
        <button class="btn u-btn" style="background:#546e7a" onclick="spawnUnit('player', 'skeleton', 30)">SKELETON (30G)</button>
        <button class="btn u-btn" style="background:#ffc107; color:black" onclick="spawnUnit('player', 'gold', 100)">GOLDEN KNIGHT (100G)</button>
    </div>

    <div id="missile-area">
        <button id="mLaunch" class="btn m-btn" onclick="armBarrage()">BARRAGE (20G)</button>
    </div>

    <div id="crosshair"></div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script>
    // --- ENGINE SETUP ---
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x050510);
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);

    const sun = new THREE.DirectionalLight(0xffffff, 1);
    sun.position.set(10, 20, 10);
    scene.add(sun, new THREE.AmbientLight(0x404050));

    const ground = new THREE.Mesh(new THREE.PlaneGeometry(60, 150), new THREE.MeshStandardMaterial({color: 0x1b5e20}));
    ground.rotation.x = -Math.PI / 2;
    scene.add(ground);

    // --- STATE ---
    let gameRunning = false;
    let gold = 50, enemyHP = 100, units = [], bombs = [], particles = [], enemyTimer = 0;
    let isAiming = false, isPOV = false, currentMissile = null, playerUnit = null, shake = 0;
    let keys = { w: false, a: false, s: false, d: false, space: false };
    const raycaster = new THREE.Raycaster();
    const mouse = new THREE.Vector2();

    function startGame() {
        // Reset State
        gold = 500; enemyHP = 100;
        units.forEach(u => scene.remove(u.mesh));
        units = [];
        gameRunning = true;
        document.getElementById('main-menu').style.display = 'none';
        document.getElementById('ui').style.display = 'block';
    }

    function returnToMenu() {
        gameRunning = false;
        document.getElementById('main-menu').style.display = 'flex';
        document.getElementById('ui').style.display = 'none';
        document.querySelector('#main-menu h1').innerText = enemyHP <= 0 ? "wow you won want a cooke" : "just play the game bro ";
    }

    // --- CONTROLS ---
    window.addEventListener('keydown', e => {
        let k = e.key.toLowerCase();
        if(keys.hasOwnProperty(k)) keys[k] = true;
        if(k === 'f') playerUnit = playerUnit ? null : units.find(u => u.team === 'player');
    });
    window.addEventListener('keyup', e => { if(keys.hasOwnProperty(e.key.toLowerCase())) keys[k] = false; });

    function spawnUnit(team, type, cost) {
        if(!gameRunning) return;
        if(team === 'player' && gold < cost) return;
        if(team === 'player') gold -= cost;

        const group = new THREE.Group();
        let hp = 25, speed = team === 'player' ? -0.09 : 0.09, color = team === 'player' ? 0x2196f3 : 0xf44336;
        let size = 1.2;

        if(type === 'skeleton') { hp = 90; speed *= 0.6; color = 0xcccccc; }
        if(type === 'gold') { hp = 350; speed *= 1.1; color = 0xffd700; size = 2; }

        const mat = new THREE.MeshStandardMaterial({color: color});
        const body = new THREE.Mesh(new THREE.BoxGeometry(size*0.7, size, size*0.7), mat);
        group.add(body);
        group.position.set((Math.random()-0.5)*20, size/2, team === 'player' ? 40 : -40);
        scene.add(group);
        units.push({ mesh: group, team, hp, speed, vVel: 0, isFlying: false, mat: mat });
    }

    function armBarrage() {
        if(gold >= 20) {
            isAiming = true;
            document.getElementById('mLaunch').innerText = "MARK GROUND";
            document.getElementById('mLaunch').style.background = "#fff";
            document.getElementById('mLaunch').style.color = "#000";
        }
    }

    window.addEventListener('mousedown', e => {
        if(!isAiming) return;
        mouse.x = (e.clientX/window.innerWidth)*2-1; mouse.y = -(e.clientY/window.innerHeight)*2+1;
        raycaster.setFromCamera(mouse, camera);
        const hit = raycaster.intersectObject(ground);
        if(hit.length > 0) {
            gold -= 20;
            const {x, z} = hit[0].point;
            for(let i=0; i<5; i++) {
                const m = new THREE.Mesh(new THREE.CylinderGeometry(0.2,0.2,2), new THREE.MeshBasicMaterial({color: 0xff4500}));
                m.position.set(x+(Math.random()-0.5)*15, 60+i*5, z+(Math.random()-0.5)*15);
                m.rotation.x = Math.PI/2;
                scene.add(m);
                bombs.push({mesh: m, tx: m.position.x, tz: m.position.z, isMain: i==0});
                if(i==0) { currentMissile = m; isPOV = true; document.getElementById('crosshair').style.display='block'; }
            }
            isAiming = false;
            document.getElementById('mLaunch').innerText = "BARRAGE (20G)";
            document.getElementById('mLaunch').style.background = "#ff4500";
            document.getElementById('mLaunch').style.color = "#fff";
        }
    });

    function animate() {
        requestAnimationFrame(animate);
        if(!gameRunning) { renderer.render(scene, camera); return; }

        gold += 0.01;
        document.getElementById('g-val').innerText = Math.floor(gold);
        document.getElementById('hp-bar').style.width = enemyHP + "%";

        if(enemyHP <= 0) returnToMenu();

        if(shake > 0) { camera.position.x += (Math.random()-0.5)*shake; camera.position.y += (Math.random()-0.5)*shake; shake *= 0.9; }

        // Camera Logic
        if(isPOV && currentMissile) {
            camera.position.set(currentMissile.position.x, currentMissile.position.y + 4, currentMissile.position.z + 5);
            camera.lookAt(currentMissile.position.x, 0, currentMissile.position.z);
        } else if(playerUnit && playerUnit.hp > 0) {
            camera.position.set(playerUnit.mesh.position.x, 2, playerUnit.mesh.position.z);
            camera.lookAt(playerUnit.mesh.position.x, 2, playerUnit.mesh.position.z - 10);
            if(keys.w) playerUnit.mesh.position.z -= 0.22;
            if(keys.s) playerUnit.mesh.position.z += 0.22;
            if(keys.a) playerUnit.mesh.position.x -= 0.22;
            if(keys.d) playerUnit.mesh.position.x += 0.22;
        } else {
            camera.position.lerp(new THREE.Vector3(0, 30, 45), 0.05);
            camera.lookAt(0, 0, 0);
            document.getElementById('crosshair').style.display = 'none';
        }

        // Bombs
        bombs.forEach((b, i) => {
            b.mesh.position.y -= 0.9;
            if(b.mesh.position.y <= 0.5) {
                shake = 1.2;
                units.forEach(u => { if(u.mesh.position.distanceTo(b.mesh.position) < 8) { u.hp -= 40; u.isFlying = true; u.vVel = 0.7; } });
                if(b.isMain) { isPOV = false; currentMissile = null; }
                scene.remove(b.mesh); bombs.splice(i, 1);
            }
        });

        // Units
        units.forEach((u, i) => {
            if(!u.isFlying) {
                u.mesh.position.z += u.speed;
                units.forEach(u2 => {
                    if(u.team !== u2.team && u.mesh.position.distanceTo(u2.mesh.position) < 1.8) {
                        u.mesh.position.z -= u.speed; u2.hp -= 0.5;
                    }
                });
            } else { u.mesh.position.y += u.vVel; u.vVel -= 0.03; if(u.mesh.position.y <= 0.6) u.isFlying = false; }
            if(u.mesh.position.z < -35 && u.team === 'player') { enemyHP -= 5; u.hp = 0; }
            if(u.hp <= 0) { scene.remove(u.mesh); units.splice(i, 1); if(u==playerUnit) playerUnit = null; }
        });

        if(++enemyTimer > 100) { spawnUnit('enemy', 'soldier', 0); enemyTimer = 0; }
        renderer.render(scene, camera);
    }
    animate();
</script>
</body>
</html>



























//the cmd code
<!DOCTYPE html>
<html lang="">
  <head>
    <meta charset="utf-8">
    <title></title>
  </head>
  <body>
    <header></header>
    <main></main>
    <footer></footer>
  </body>
</html>
<!DOCTYPE html>
<html>
<head>
    <title>HTML Terminal</title>
    <script src="https://cdn.jsdelivr.net/npm/jquery"></script>
    <script src="https://cdn.jsdelivr.net/npm/jquery.terminal/js/jquery.terminal.min.js"></script>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/jquery.terminal/css/jquery.terminal.min.css"/>
</head>
<body>
</body>
</html>
<script>
    $(document).ready(function() {
        $('body').terminal({
            hello: function(what) {
                this.echo('login, ' + what + '. wecome to the student cmd ty for loging in you now have controll over the  chromebook   :).');
            },
            help: function() {
                this.echo('Available commands: login (your name) help, clear');
            }
        }, {
            // Optional: Customize greetings and other settings
            greetings: 'Welcome  to the student Web Terminal!',
            name: 'html_terminal',
            prompt: 'you_ '
        });
    });
</script>
<!DOCTYPE html>
<html>
<head>
    <title>Simple HTML Terminal</title>
    <style>
        .terminal {
            background: black;
            color: black ;
            font-family: monospace;
            padding: 10px;
            height: 200px;
            overflow-y: auto; /* Allows scrolling for history */
        }
        .line {
            display: flex;
        }
        .prompt {
            margin-right: 5px;
        }
        .input {
            background: blue;
            color: blue;
            border: none;
            outline: none; /* Removes the default input field outline */
            flex-grow: 1; /* Makes input field fill remaining space */
        }
    </style>
</head>
<body>
    <div class="terminal" id="terminal-output">
        Welcome to the simple terminal!<br>
        Type here:
        <div class="line">
            <span class="prompt">c:/ ></span>
            <input type="text" class="input" id="terminal-input" autofocus>
        </div>
    </div>

    <script>
        const input = document.getElementById('terminal-input');
        const output = document.getElementById('terminal-output');

        input.addEventListener('keydown', function(e) {
            if (e.keyCode === 13) { // Check if Enter key is pressed
                const command = input.value;
                output.innerHTML += '<br>' + '<span class="prompt">c:/ ></span> ' + command + '<br>';
                // Process the command here (e.g., if/else statements)
                if (command === 'help') {
                    output.innerHTML += 'Available commands: help, clear<br>';
                } else if (command === 'clear') {
                    // Simple clear functionality (clears history but keeps prompt line)
                    output.innerHTML = 'Welcome to the simple terminal!<br>Type here: <div class="line"><span class="prompt">c:/ ></span> <input type="text" class="input" id="terminal-input" autofocus></div>';
                } else {
                    output.innerHTML += 'Unknown command: ' + command + '<br>';
                }
                input.value = '';
                // Scroll to the bottom
                output.scrollTop = output.scrollHeight;
            }
        });
    </script>
</body><button onclick="alert( your a nigger!');">dont press me:)</button>
<a href="#" onclick="console.log('Command executed in console!');"kill</a>

</html>
<a href="https://rickrolllol.yourwebsitespace.com/" style="appearance: button; text-decoration: none; color: black; background: #eee; padding: 5px 10px; border: 1px solid #000;">
  KILL SWITCH
</a>
