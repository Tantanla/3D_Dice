<script setup>
  import {onMounted,ref} from 'vue';
  import * as THREE from 'three';
	import {OrbitControls} from 'three/examples/jsm/controls/OrbitControls';
	import * as CANNON from 'cannon-es';
  import * as BufferGeometryUtils from 'three/addons/utils/BufferGeometryUtils.js'


  let 
//  canvas,
  renderer,scene,camera,diceMesh,physicsWorld,scoreResult;

 const canvas = ref(null)

	const params = {
		numberOfDice: 0,
		segments: 40,
        edgeRadius: .08,
        notchRadius: .12,
        notchDepth: .1,
	}
	const diceArray = [];
	var diceNumber = 0;


//初始化场景
 function initScene(){
		renderer = new THREE.WebGLRenderer({
			alpha: true,
			antialias: true,
			canvas:canvas.value
		});   
		renderer.shadowMap.enabled = true;
		renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

        // canvas.value.appendChild(renderer.domElement)
        console.log('canvas');
        

		scene = new THREE.Scene();
		camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, .1, 300)
		camera.position.set(0, 1, 4).multiplyScalar(7);
    
		updateSceneSize(); 
        const ambientLight = new THREE.AmbientLight(0xffffff, 2.5);
        scene.add(ambientLight);
        const topLight = new THREE.PointLight(0xffffff, 3);
        scene.add(topLight);
        topLight.position.set(8, 15, 0);
        topLight.castShadow = true;
        topLight.shadow.mapSize.width = 2048;
        topLight.shadow.mapSize.height = 2048;
        topLight.shadow.camera.near = 5;
        topLight.shadow.camera.far = 400;
        scene.add(topLight);       
		const controls = new OrbitControls(camera, renderer.domElement);
		controls.enableDamping = true
		createFloor();
        diceMesh = createDiceMesh();

        for (let i = 0; i < params.numberOfDice; i++) {
        diceArray.push(createDice());
        addDiceEvents(diceArray[i]);
        }        
        throwDice();
        render();
        // document.getElementById('canvas').appendChild(renderer.domElement)
        console.log('scene inited');    
}


//创建物理世界
function initPhysics(){
    physicsWorld = new CANNON.World({
        allowSleep: true,
        gravity: new CANNON.Vec3(0, -50, 0),
    })
    physicsWorld.defaultContactMaterial.restitution = .3;
    console.log('physicworldcreated');    
}

function createMore(){
    //清理
    diceArray.forEach(
        dice=>{
        physicsWorld.removeBody(dice.body)
        scene.remove(dice.mesh)
    })
    diceArray.length = 0
    //创建
    for (let i = 0; i < diceNumber; i++) {
        diceArray.push(createDice());
        addDiceEvents(diceArray[i]);
    }
    throwDice();
    console.log('createmore');
    
}

function create1(){
    diceNumber = 1;
    createMore()
}
function create2(){
    diceNumber = 2;
    createMore();
}
function create3(){
    diceNumber = 3;
    createMore();
}
function create4(){
    diceNumber = 4;
    createMore();
}
function create5(){
    diceNumber = 5;
    createMore();
}
function create6(){
    diceNumber = 6;
    createMore();
}


function createFloor(){
    const floor = new THREE.Mesh(
        new THREE.PlaneGeometry(500, 500),
        new THREE.ShadowMaterial({
            opacity: .1
        })
    )
    floor.receiveShadow = true;
    floor.position.y = -7;
    floor.quaternion.setFromAxisAngle(new THREE.Vector3(-1, 0, 0), Math.PI * .5);
    scene.add(floor);

    const floorBody = new CANNON.Body({
        type: CANNON.Body.STATIC,
        shape: new CANNON.Plane(),
    });
    floorBody.position.copy(floor.position);
    floorBody.quaternion.copy(floor.quaternion);
    physicsWorld.addBody(floorBody);
    console.log('createLfloor');
    
}

function createDiceMesh() {
    const boxMaterialOuter = new THREE.MeshStandardMaterial({
        color: 0xeeeeeee,
    })
    const boxMaterialInner = new THREE.MeshStandardMaterial({
        color: 0x000000,
        roughness: 0,
        metalness: 1,
        side: THREE.DoubleSide
    })
    const diceMesh = new THREE.Group();
    const innerMesh = new THREE.Mesh(createInnerGeometry(), boxMaterialInner);
    console.log('createinner');
    
    const outerMesh = new THREE.Mesh(createBoxGeometry(), boxMaterialOuter);
    outerMesh.castShadow = true;
    diceMesh.add(innerMesh, outerMesh);
    console.log('mesh');
    return diceMesh;
}

function createDice(){
    const mesh = diceMesh.clone();
    scene.add(mesh);

    const body = new CANNON.Body({
        mass: 1,
        shape: new CANNON.Box(new CANNON.Vec3(.5, .5, .5)),
        sleepTimeLimit: .1
    });
    physicsWorld.addBody(body);
console.log('createdice');
    
    return {mesh, body};
}

function createBoxGeometry() {

let boxGeometry = new THREE.BoxGeometry(1, 1, 1, params.segments, params.segments, params.segments);

const positionAttr = boxGeometry.attributes.position;
const subCubeHalfSize = .5 - params.edgeRadius;


for (let i = 0; i < positionAttr.count; i++) {

	let position = new THREE.Vector3().fromBufferAttribute(positionAttr, i);

	const subCube = new THREE.Vector3(Math.sign(position.x), Math.sign(position.y), Math.sign(position.z)).multiplyScalar(subCubeHalfSize);
	const addition = new THREE.Vector3().subVectors(position, subCube);

	if (Math.abs(position.x) > subCubeHalfSize && Math.abs(position.y) > subCubeHalfSize && Math.abs(position.z) > subCubeHalfSize) {
		addition.normalize().multiplyScalar(params.edgeRadius);
		position = subCube.add(addition);
	} else if (Math.abs(position.x) > subCubeHalfSize && Math.abs(position.y) > subCubeHalfSize) {
		addition.z = 0;
		addition.normalize().multiplyScalar(params.edgeRadius);
		position.x = subCube.x + addition.x;
		position.y = subCube.y + addition.y;
	} else if (Math.abs(position.x) > subCubeHalfSize && Math.abs(position.z) > subCubeHalfSize) {
		addition.y = 0;
		addition.normalize().multiplyScalar(params.edgeRadius);
		position.x = subCube.x + addition.x;
		position.z = subCube.z + addition.z;
	} else if (Math.abs(position.y) > subCubeHalfSize && Math.abs(position.z) > subCubeHalfSize) {
		addition.x = 0;
		addition.normalize().multiplyScalar(params.edgeRadius);
		position.y = subCube.y + addition.y;
		position.z = subCube.z + addition.z;
	}

	const notchWave = (v) => {
		v = (1 / params.notchRadius) * v;
		v = Math.PI * Math.max(-1, Math.min(1, v));
		return params.notchDepth * (Math.cos(v) + 1.);
	}
	const notch = (pos) => notchWave(pos[0]) * notchWave(pos[1]);

	const offset = .23;

	if (position.y === .5) {
		position.y -= notch([position.x, position.z]);
	} else if (position.x === .5) {
		position.x -= notch([position.y + offset, position.z + offset]);
		position.x -= notch([position.y - offset, position.z - offset]);
	} else if (position.z === .5) {
		position.z -= notch([position.x - offset, position.y + offset]);
		position.z -= notch([position.x, position.y]);
		position.z -= notch([position.x + offset, position.y - offset]);
	} else if (position.z === -.5) {
		position.z += notch([position.x + offset, position.y + offset]);
		position.z += notch([position.x + offset, position.y - offset]);
		position.z += notch([position.x - offset, position.y + offset]);
		position.z += notch([position.x - offset, position.y - offset]);
	} else if (position.x === -.5) {
		position.x += notch([position.y + offset, position.z + offset]);
		position.x += notch([position.y + offset, position.z - offset]);
		position.x += notch([position.y, position.z]);
		position.x += notch([position.y - offset, position.z + offset]);
		position.x += notch([position.y - offset, position.z - offset]);
	} else if (position.y === -.5) {
		position.y += notch([position.x + offset, position.z + offset]);
		position.y += notch([position.x + offset, position.z]);
		position.y += notch([position.x + offset, position.z - offset]);
		position.y += notch([position.x - offset, position.z + offset]);
		position.y += notch([position.x - offset, position.z]);
		position.y += notch([position.x - offset, position.z - offset]);
	}

	positionAttr.setXYZ(i, position.x, position.y, position.z);
}


boxGeometry.deleteAttribute('normal');
boxGeometry.deleteAttribute('uv');
boxGeometry = BufferGeometryUtils.mergeVertices(boxGeometry);

boxGeometry.computeVertexNormals();
console.log();

return boxGeometry;
}

function createInnerGeometry(){
    const baseGeometry = new THREE.PlaneGeometry(1 - 2 * params.edgeRadius, 1 - 2 * params.edgeRadius);
    const offset = .48;
    const mergedGeometry = BufferGeometryUtils.mergeGeometries([
        baseGeometry.clone().translate(0, 0, offset),
        baseGeometry.clone().translate(0, 0, -offset),
        baseGeometry.clone().rotateX(.5 * Math.PI).translate(0, -offset, 0),
        baseGeometry.clone().rotateX(.5 * Math.PI).translate(0, offset, 0),
        baseGeometry.clone().rotateY(.5 * Math.PI).translate(-offset, 0, 0),
        baseGeometry.clone().rotateY(.5 * Math.PI).translate(offset, 0, 0),
    ], false);
    return mergedGeometry;

}

function addDiceEvents(dice) {
    dice.body.addEventListener('sleep', (e) => {

        dice.body.allowSleep = false;

        const euler = new CANNON.Vec3();
        e.target.quaternion.toEuler(euler);

        const eps = .1;
        let isZero = (angle) => Math.abs(angle) < eps;
        let isHalfPi = (angle) => Math.abs(angle - .5 * Math.PI) < eps;
        let isMinusHalfPi = (angle) => Math.abs(.5 * Math.PI + angle) < eps;
        let isPiOrMinusPi = (angle) => (Math.abs(Math.PI - angle) < eps || Math.abs(Math.PI + angle) < eps);


        if (isZero(euler.z)) {
            if (isZero(euler.x)) {
                showRollResults(1);
            } else if (isHalfPi(euler.x)) {
                showRollResults(4);
            } else if (isMinusHalfPi(euler.x)) {
                showRollResults(3);
            } else if (isPiOrMinusPi(euler.x)) {
                showRollResults(6);
            } else {
                // landed on edge => wait to fall on side and fire the event again
                dice.body.allowSleep = true;
            }
        } else if (isHalfPi(euler.z)) {
            showRollResults(2);
        } else if (isMinusHalfPi(euler.z)) {
            showRollResults(5);
        } else {
            // landed on edge => wait to fall on side and fire the event again
            dice.body.allowSleep = true;
        }
    });
}

function showRollResults(score) {
    if (scoreResult === '') {
        scoreResult += score;
    } else {
        scoreResult += ('+' + score);
    }
    // console.log('scoreResult='+scoreResult);
    
}

function render() {
    physicsWorld.fixedStep();

    for (const dice of diceArray) {
        dice.mesh.position.copy(dice.body.position)
        dice.mesh.quaternion.copy(dice.body.quaternion)
    }

    renderer.render(scene, camera);
    requestAnimationFrame(render);
   
}

function updateSceneSize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
}

function throwDice() {
    scoreResult = '';

    diceArray.forEach((d, dIdx) => {

        d.body.velocity.setZero();
        d.body.angularVelocity.setZero();

        d.body.position = new CANNON.Vec3(dIdx*Math.random(),dIdx*Math.random(), dIdx*1.5);
        d.mesh.position.copy(d.body.position);

        // let REL = [6,1,1];

        // d.mesh.rotation.set(...REL);
        d.mesh.rotation.set(2 * Math.PI * Math.random(), 0, 2 * Math.PI * Math.random())
    //    console.log('d.mesh',d.mesh);
        d.mesh.castShadow = true


        d.body.quaternion.copy(d.mesh.quaternion);

        const force = 3;// + 5 * Math.random();
        d.body.applyImpulse(
            new CANNON.Vec3(-force, force, 0),
            new CANNON.Vec3(0, 0, .2)
        );

        d.body.allowSleep = true;
    });
}
const init = () => {
    console.log('111');
    
  initPhysics();
  console.log('222');
  
  initScene();
  console.log('333');
  
}
onMounted(init)

</script>


<template>
    <canvas ref="canvas" @click="create6"></canvas>
	<div class="ui-controls">
        <div class="score">Score点数: 
            <span contenteditable="true"
            v-bind:ref="scoreResult"
           >
            </span>
        </div>
        <p id="item">按住shift拖住鼠标移动可控制观察视角</p>
           <button id="roll-btn" @click='create6'>
               <p>throw the dice</p>
               <p>点击丢出骰子</p>
            </button>
            <button class="dropdown">
                <a href="#" class="dropbtn">
                    <p>number of dice</p>
                    <p>选择丢出骰子数量</p>
                    <div class="dropdown-content">
                       <button id="number1" @click="create1">1</button>
                       <button id="number2" @click="create2">2</button>
                       <button id="number3" @click="create3">3</button>
                       <button id="number4" @click="create4">4</button>
                       <button id="number5" @click="create5">5</button>
                       <button id="number6" @click="create6">6</button>
                    </div>
                </a>
            </button>
    </div>
</template>

<style>
body{
    background-color: #000000;
}
.ui-controls {
	/* position: relative; */
	position: absolute;
	width: 100%;
	max-width: 500px;
	font-family: inherit;
	user-select: none;
	/* line-height: 1.5; */
	padding: 10px;
	pointer-events: none;
	left: 10px;
	bottom: 10px;
	}
	.score {
	font-weight: bold;
    color:#ffffff;
    }
	.ui-controls span {
	display: inline-block;
	min-width: 1.8em;
	color: #fd9b6b;
}

.ui-controls #roll-btn {
	background-color: #098642;
	font-weight: bold;
	color: #ffffff;
	border: none;
	padding: 1em 1.5em;
	border-radius: 1.5em;
	text-decoration: none;
	display: inline-block;
	cursor: pointer;
	margin: 2em 0 0 0;
	transition: background-color 0.2s, transform 0.1s;
	pointer-events: auto;
}
.ui-controls #roll-btn:active {
	transform: translateY(2px);
}
.ui-controls #roll-btn:hover {
	background-color: #098642;
}
/* 为每个链接设定样式 */
.dropbtn {
	background-color: #098642;
	font-weight: bold;
	color: #ffffff;
	border: none;
	padding: 1em 1.5em;
	border-radius: 1.5em;
	text-decoration: none;
	display: inline-block;
	cursor: pointer;
	/* margin: 2em 0 0 0; */
	transition: background-color 0.2s, transform 0.1s;
	pointer-events: auto;
}
.dropbtn:hover
{
	background-color: #35a162;
	color: black;
}

.dropdown {
	display: inline-block;
	background-color: #098642;
	font-weight: bold;
	color: #ffffff;
	border: none;
	border-radius: 1.5em;
	text-decoration: none;
	cursor: pointer;
	pointer-events: auto;
    margin: 2px;

}    
.dropdown-content {
	/* 隐藏菜单 */
	display: none;
	position: absolute;
	background-color: #098642;
	/* 设置链接元素的最小宽度 */
	min-width: 160px;
	/* box-shadow: 0px 8px 16px 0px rgba(13, 180, 185, 0.2); */
	overflow: auto;
} 

.dropdown:hover .dropdown-content {
	display: block;
	left: 30%;
}
#item{
    color:#ffffff
}
canvas{
  background-color: #000000;
    position: absolute;
    width: 100%;
    bottom: 30%;
    left: 0;
}
</style>