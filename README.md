# Chemical_engineering_plant
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>3D Chemical Engineering Virtual Plant</title>

<style>
    body {
        margin: 0;
        overflow: hidden;
        font-family: Arial, sans-serif;
        background: #111827;
        color: white;
    }

    #ui {
        position: absolute;
        top: 15px;
        left: 15px;
        width: 290px;
        padding: 16px;
        background: rgba(15, 23, 42, 0.94);
        border: 1px solid #475569;
        border-radius: 10px;
        z-index: 10;
    }

    h2 {
        margin: 0 0 10px;
        color: #38bdf8;
        font-size: 20px;
    }

    h3 {
        color: #facc15;
        margin-bottom: 5px;
    }

    label {
        display: block;
        margin-top: 12px;
        color: #cbd5e1;
    }

    input, select {
        width: 100%;
        margin-top: 5px;
    }

    #data {
        margin-top: 14px;
        padding: 10px;
        background: #1e293b;
        border-radius: 6px;
        line-height: 1.5;
        font-size: 13px;
    }

    #help {
        position: absolute;
        right: 15px;
        bottom: 15px;
        padding: 10px;
        background: rgba(15, 23, 42, 0.85);
        border-radius: 7px;
        color: #cbd5e1;
        z-index: 10;
        font-size: 13px;
    }
</style>
</head>

<body>

<div id="ui">
    <h2>Virtual Chemical Plant</h2>

    <label>Selected Equipment</label>
    <select id="equipment">
        <option value="overview">Plant Overview</option>
        <option value="tank">Feed Tank</option>
        <option value="pump">Pump</option>
        <option value="hx">Heat Exchanger</option>
        <option value="reactor">CSTR Reactor</option>
        <option value="column">Distillation Column</option>
        <option value="condenser">Condenser</option>
    </select>

    <label>
        Feed Flow Rate:
        <span id="flowValue">10</span> kg/s
    </label>
    <input id="flow" type="range" min="1" max="20" value="10">

    <label>
        Reactor Temperature:
        <span id="tempValue">350</span> K
    </label>
    <input id="temperature" type="range" min="280" max="500" value="350">

    <label>
        Reflux Ratio:
        <span id="refluxValue">2.0</span>
    </label>
    <input id="reflux" type="range" min="0.5" max="5" step="0.1" value="2">

    <div id="data"></div>
</div>

<div id="help">
    Drag to rotate | Scroll to zoom | Right-click drag to pan
</div>

<script type="module">

import * as THREE from
'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';

import { OrbitControls } from
'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';

let scene, camera, renderer, controls;
let particles = [];
let equipmentMeshes = {};
let plantParameters = {
    flow: 10,
    temperature: 350,
    reflux: 2
};

init();
animate();

function init() {

    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0f172a);

    camera = new THREE.PerspectiveCamera(
        55,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
    );

    camera.position.set(12, 12, 18);

    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.shadowMap.enabled = true;
    document.body.appendChild(renderer.domElement);

    controls = new OrbitControls(camera, renderer.domElement);
    controls.target.set(0, 1, 0);
    controls.enableDamping = true;

    createLighting();
    createGround();
    createPlant();
    createFlowParticles();
    createUI();

    window.addEventListener("resize", onResize);
}

function createLighting() {

    const ambient = new THREE.AmbientLight(0xffffff, 0.55);
    scene.add(ambient);

    const light = new THREE.DirectionalLight(0xffffff, 1.2);
    light.position.set(8, 15, 10);
    light.castShadow = true;
    scene.add(light);

    const blueLight = new THREE.PointLight(0x38bdf8, 2, 30);
    blueLight.position.set(0, 8, 4);
    scene.add(blueLight);
}

function createGround() {

    const groundGeometry = new THREE.BoxGeometry(30, 0.3, 18);
    const groundMaterial = new THREE.MeshStandardMaterial({
        color: 0x1e293b
    });

    const ground = new THREE.Mesh(groundGeometry, groundMaterial);
    ground.position.y = -0.3;
    ground.receiveShadow = true;
    scene.add(ground);

    const grid = new THREE.GridHelper(30, 30, 0x475569, 0x334155);
    grid.position.y = -0.14;
    scene.add(grid);
}

function material(color) {
    return new THREE.MeshStandardMaterial({
        color: color,
        metalness: 0.2,
        roughness: 0.65
    });
}

function createTank(name, position, color = 0x22c55e) {

    const group = new THREE.Group();

    const body = new THREE.Mesh(
        new THREE.CylinderGeometry(1.25, 1.25, 2.7, 32),
        material(color)
    );

    const top = new THREE.Mesh(
        new THREE.ConeGeometry(1.25, 0.65, 32),
        material(color)
    );

    const bottom = new THREE.Mesh(
        new THREE.ConeGeometry(1.15, 0.5, 32),
        material(color)
    );

    body.position.y = 1.5;
    top.position.y = 3.15;
    bottom.position.y = 0.15;

    group.add(body, top, bottom);
    group.position.set(...position);

    body.castShadow = true;
    equipmentMeshes[name] = group;
    scene.add(group);

    return group;
}

function createPump(position) {

    const group = new THREE.Group();

    const base = new THREE.Mesh(
        new THREE.BoxGeometry(1.3, 0.7, 1.2),
        material(0xef4444)
    );

    const motor = new THREE.Mesh(
        new THREE.CylinderGeometry(0.45, 0.45, 1.2, 24),
        material(0xf97316)
    );

    base.position.y = 0.45;
    motor.rotation.z = Math.PI / 2;
    motor.position.set(0.5, 0.9, 0);

    group.add(base, motor);
    group.position.set(...position);

    equipmentMeshes.pump = group;
    scene.add(group);
}

function createHeatExchanger(position) {

    const group = new THREE.Group();

    const shell = new THREE.Mesh(
        new THREE.CylinderGeometry(0.8, 0.8, 3.2, 32),
        material(0x64748b)
    );

    shell.rotation.z = Math.PI / 2;
    shell.position.y = 1.2;

    const hotPipe = new THREE.Mesh(
        new THREE.CylinderGeometry(0.18, 0.18, 3.5, 16),
        material(0xef4444)
    );

    hotPipe.rotation.z = Math.PI / 2;
    hotPipe.position.set(0, 1.2, 0.4);

    const coldPipe = new THREE.Mesh(
        new THREE.CylinderGeometry(0.18, 0.18, 3.5, 16),
        material(0x38bdf8)
    );

    coldPipe.rotation.z = Math.PI / 2;
    coldPipe.position.set(0, 1.2, -0.4);

    group.add(shell, hotPipe, coldPipe);
    group.position.set(...position);

    equipmentMeshes.hx = group;
    scene.add(group);
}

function createReactor(position) {

    const group = new THREE.Group();

    const vessel = new THREE.Mesh(
        new THREE.CylinderGeometry(1.1, 1.1, 2.8, 32),
        material(0xa855f7)
    );

    const top = new THREE.Mesh(
        new THREE.ConeGeometry(1.1, 0.6, 32),
        material(0xa855f7)
    );

    const jacket = new THREE.Mesh(
        new THREE.CylinderGeometry(1.22, 1.22, 2.2, 32, 1, true),
        new THREE.MeshStandardMaterial({
            color: 0xf97316,
            transparent: true,
            opacity: 0.35
        })
    );

    vessel.position.y = 1.5;
    top.position.y = 3.2;
    jacket.position.y = 1.5;

    group.add(vessel, top, jacket);
    group.position.set(...position);

    equipmentMeshes.reactor = group;
    scene.add(group);
}

function createColumn(position) {

    const group = new THREE.Group();

    const column = new THREE.Mesh(
        new THREE.CylinderGeometry(0.8, 0.8, 6, 32),
        material(0xeab308)
    );

    const trayMaterial = material(0xfef08a);

    column.position.y = 3.2;
    group.add(column);

    for (let i = 0; i < 7; i++) {
        const tray = new THREE.Mesh(
            new THREE.CylinderGeometry(0.85, 0.85, 0.08, 32),
            trayMaterial
        );

        tray.position.y = 0.7 + i * 0.83;
        group.add(tray);
    }

    group.position.set(...position);

    equipmentMeshes.column = group;
    scene.add(group);
}

function createCondenser(position) {

    const group = new THREE.Group();

    const body = new THREE.Mesh(
        new THREE.BoxGeometry(2.3, 1.3, 1.3),
        material(0x0ea5e9)
    );

    const tube = new THREE.Mesh(
        new THREE.CylinderGeometry(0.25, 0.25, 2.5, 20),
        material(0xe0f2fe)
    );

    tube.rotation.z = Math.PI / 2;
    group.add(body, tube);
    group.position.set(...position);

    equipmentMeshes.condenser = group;
    scene.add(group);
}

function createPipe(a, b, color = 0x94a3b8) {

    const start = new THREE.Vector3(...a);
    const end = new THREE.Vector3(...b);

    const direction = new THREE.Vector3().subVectors(end, start);
    const length = direction.length();

    const pipe = new THREE.Mesh(
        new THREE.CylinderGeometry(0.12, 0.12, length, 16),
        material(color)
    );

    pipe.position.copy(start).add(end).multiplyScalar(0.5);
    pipe.quaternion.setFromUnitVectors(
        new THREE.Vector3(0, 1, 0),
        direction.normalize()
    );

    scene.add(pipe);
}

function createPlant() {

    createTank("tank", [-10, 0, 0], 0x22c55e);
    createPump([-7, 0, 0]);
    createHeatExchanger([-4, 0, 0]);
    createReactor([0, 0, 0]);
    createColumn([4.5, 0, 0]);
    createCondenser([8, 4.7, 0]);
    createTank("product", [10, 0, 3], 0x06b6d4);
    createTank("bottoms", [8, 0, -4], 0xf97316);

    createPipe([-8.8, 1.5, 0], [-7.7, 0.8, 0]);
    createPipe([-6.3, 0.8, 0], [-5.6, 1.2, 0]);
    createPipe([-2.4, 1.2, 0], [-1.1, 1.5, 0]);
    createPipe([1.1, 1.5, 0], [3.7, 2.4, 0]);
    createPipe([5.3, 6.2, 0], [8, 5.3, 0]);
    createPipe([8, 4.1, 0], [9.5, 2.5, 3]);
    createPipe([4.5, 0.2, 0], [7.5, 0.5, -4]);

    addLabel("FEED TANK", [-10, 4, 0]);
    addLabel("PUMP", [-7, 2, 0]);
    addLabel("HEAT EXCHANGER", [-4, 2.5, 0]);
    addLabel("CSTR REACTOR", [0, 4, 0]);
    addLabel("DISTILLATION COLUMN", [4.5, 7, 0]);
    addLabel("CONDENSER", [8, 6.5, 0]);
    addLabel("PRODUCT", [10, 4, 3]);
    addLabel("BOTTOMS", [8, 4, -4]);
}

function addLabel(text, position) {

    const canvas = document.createElement("canvas");
    canvas.width = 512;
    canvas.height = 80;

    const context = canvas.getContext("2d");
    context.font = "bold 30px Arial";
    context.fillStyle = "white";
    context.textAlign = "center";
    context.fillText(text, 256, 45);

    const texture = new THREE.CanvasTexture(canvas);

    const sprite = new THREE.Sprite(
        new THREE.SpriteMaterial({
            map: texture,
            transparent: true
        })
    );

    sprite.scale.set(3.2, 0.5, 1);
    sprite.position.set(...position);
    scene.add(sprite);
}

function createFlowParticles() {

    for (let i = 0; i < 45; i++) {

        const particle = new THREE.Mesh(
            new THREE.SphereGeometry(0.09, 8, 8),
            material(0x67e8f9)
        );

        particle.userData.offset = Math.random();
        particle.userData.speed = 0.003 + Math.random() * 0.003;

        scene.add(particle);
        particles.push(particle);
    }
}

function updateParticles() {

    const path = [
        new THREE.Vector3(-9, 1.5, 0),
        new THREE.Vector3(-7, 1, 0),
        new THREE.Vector3(-4, 1.2, 0),
        new THREE.Vector3(0, 1.5, 0),
        new THREE.Vector3(4.5, 2.5, 0),
        new THREE.Vector3(8, 5.3, 0),
        new THREE.Vector3(9.5, 2.5, 3)
    ];

    particles.forEach((p) => {

        p.userData.offset =
            (p.userData.offset + p.userData.speed *
            plantParameters.flow / 10) % 1;

        const t = p.userData.offset;
        const scaled = t * (path.length - 1);
        const index = Math.floor(scaled);
        const local = scaled - index;

        if (index < path.length - 1) {
            p.position.lerpVectors(
                path[index],
                path[index + 1],
                local
            );
        }
    });
}

function createUI() {

    const flow = document.getElementById("flow");
    const temperature = document.getElementById("temperature");
    const reflux = document.getElementById("reflux");

    flow.addEventListener("input", () => {
        plantParameters.flow = Number(flow.value);
        document.getElementById("flowValue").textContent = flow.value;
        updateData();
    });

    temperature.addEventListener("input", () => {
        plantParameters.temperature = Number(temperature.value);
        document.getElementById("tempValue").textContent = temperature.value;
        updateReactorColor();
        updateData();
    });

    reflux.addEventListener("input", () => {
        plantParameters.reflux = Number(reflux.value);
        document.getElementById("refluxValue").textContent =
            Number(reflux.value).toFixed(1);
        updateData();
    });

    document.getElementById("equipment")
        .addEventListener("change", updateData);

    updateData();
}

function updateReactorColor() {

    const reactor = equipmentMeshes.reactor;
    const vessel = reactor.children[0];

    const normalized =
        (plantParameters.temperature - 280) / (500 - 280);

    vessel.material.color.setHSL(
        0.65 - normalized * 0.65,
        0.85,
        0.5
    );
}

function updateData() {

    const selected = document.getElementById("equipment").value;
    const flow = plantParameters.flow;
    const temp = plantParameters.temperature;
    const reflux = plantParameters.reflux;

    const heatDuty = flow * 4.18 * Math.max(temp - 300, 0);
    const conversion =
        1 - Math.exp(-0.004 * temp);

    const approximatePurity =
        Math.min(0.995, 0.60 + 0.08 * reflux);

    let text = "";

    if (selected === "overview") {
        text = `
        <b>Plant Overview</b><br>
        Process type: Continuous<br>
        Feed flow: ${flow.toFixed(1)} kg/s<br>
        Reactor temperature: ${temp} K<br>
        Reflux ratio: ${reflux.toFixed(1)}<br>
        Approx. reactor conversion: ${(conversion * 100).toFixed(1)}%<br>
        Approx. product purity: ${(approximatePurity * 100).toFixed(1)}%<br>
        Estimated heat duty: ${(heatDuty / 1000).toFixed(1)} MW
        `;
    }

    if (selected === "tank") {
        text = `
        <b>Feed Tank</b><br>
        Function: Storage and surge control<br>
        Operation: Atmospheric<br>
        Feed flow: ${flow.toFixed(1)} kg/s<br>
        Main balance: Accumulation = In − Out
        `;
    }

    if (selected === "pump") {
        text = `
        <b>Pump</b><br>
        Function: Increases liquid pressure<br>
        Flow rate: ${flow.toFixed(1)} kg/s<br>
        Approx. regime: ${flow < 8 ? "Low flow" : "Normal flow"}<br>
        Principle: Mechanical energy → pressure energy
        `;
    }

    if (selected === "hx") {
        text = `
        <b>Heat Exchanger</b><br>
        Function: Indirect heat transfer<br>
        Estimated duty: ${(heatDuty / 1000).toFixed(1)} MW<br>
        Relationship: Q = UAΔT<sub>lm</sub><br>
        Hot-side fluid: Red<br>
        Cold-side fluid: Blue
        `;
    }

    if (selected === "reactor") {
        text = `
        <b>CSTR Reactor</b><br>
        Reactor temperature: ${temp} K<br>
        Feed flow: ${flow.toFixed(1)} kg/s<br>
        Approx. conversion: ${(conversion * 100).toFixed(1)}%<br>
        Model: −r<sub>A</sub> = kC<sub>A</sub><br>
        Temperature effect: Arrhenius kinetics
        `;
    }

    if (selected === "column") {
        text = `
        <b>Distillation Column</b><br>
        Number of visible trays: 7<br>
        Reflux ratio: ${reflux.toFixed(1)}<br>
        Approx. top purity: ${(approximatePurity * 100).toFixed(1)}%<br>
        Separation mechanism: Vapor-liquid equilibrium
        `;
    }

    if (selected === "condenser") {
        text = `
        <b>Condenser</b><br>
        Function: Vapor condensation<br>
        Cooling medium: Water<br>
        Phase change: Vapor → Liquid<br>
        Energy removal increases with flow rate
        `;
    }

    document.getElementById("data").innerHTML = text;
}

function animate() {

    requestAnimationFrame(animate);

    updateParticles();
    controls.update();
    renderer.render(scene, camera);
}

function onResize() {

    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
}

</script>
</body>
</html>
