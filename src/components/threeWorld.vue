<template>
  <canvas id="c" ref="c">
    <!-- <div id="guiBlock"></div> -->
  </canvas>
</template>

<script>
import * as THREE from "three";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import {GUI} from "three/examples/js/libs/dat.gui.min.js"
export default {
  name: "threeWorld",
  data() {
    return {
      down: false,
      then: 0,
      clicked: false,
      isTravelling: false,
      travellingObject: {},
      isMovingBack: false,
      hasTravelled: false,
      objectSavedPosition: {},
      guiParams: {
        xPos: 0, 
        yPos: 0, 
        zPos: 0
      }
    };
  },
  methods: {
    main() {
      const canvas = document.querySelector("#c");
      this.renderer = new THREE.WebGLRenderer({ canvas });
      this.createCamera();
      this.scene = new THREE.Scene();
      this.homeTarget = new THREE.Vector3(0, 0, 0);
      this.baseQuaternion = new THREE.Quaternion(0, 0, 0, 1);
      this.scene.background = new THREE.Color(0xaaaaaa);
      this.addLight();
      this.yellow = new THREE.Color("yellow");
      this.black = new THREE.Color("black");
      this.setControls();
      this.initEventListeners();
      this.initRaycaster();

      // LOAD 3D Object (fire Extinguisher)
      this.loadGltf();
      this.createBox();
      this.gui = new GUI();

      // GUI STUFF, updates reactive data
      //https://github.com/dataarts/dat.gui/blob/master/API.md
      this.gui.add(this.guiParams, 'xPos', -10, 10, 1).name('x position').onChange(this.updatePosition)
      this.gui.add(this.guiParams, 'yPos', -10, 10, 1).name('y position').onChange(this.updatePosition)
      this.gui.add(this.guiParams, 'zPos', -10, 10, 1).name('z position').onChange(this.updatePosition)

      //   this.loadTextures();
      //   this.loadTexture();
      // this.createBoxGeometry(1, 1, 1);
      //   this.coneGeometry = new THREE.ConeBufferGeometry(0.5, 1.5, 20);
      //   this.cubes = [];
      //   this.loadManager.onLoad = () => {
      //     console.log("in Load manger");
      //     this.cubes = [
      //       this.makeInstance(this.boxGeometry, 0x44aa88, 2),
      //       //   this.makeInstance(this.coneGeometry, 0x12ab22, 0),
      //       this.makeInstance(this.boxGeometry, 0xaa8844, -2),
      //       this.makeInstance(this.boxGeometry, 0xaa8844, 0),
      //       //   this.makeInstance(this.coneGeometry, 0xaa2233, 1),
      //     ];
      //   };

      this.render();
      requestAnimationFrame(this.render);
    },
    render(now) {
      now *= 0.001; //convert ms to s
      const deltaTime = now - this.then;
      this.then = now;
      //   this.controls.enabled = true;

      if (this.resizeRendererToDisplaySize(this.renderer)) {
        const canvas = this.renderer.domElement;
        this.camera.aspect = canvas.clientWidth / canvas.clientHeight;
        this.camera.updateProjectionMatrix();
      }

      this.raycaster.setFromCamera(this.mouse, this.camera);

      let intersects = this.raycaster.intersectObjects(
        this.scene.children,
        true
      );

      this.distanceFromCamera = 2;
      // Check if something was previously picked
      if (this.picked) {
        //if something was picked LAST FRAME, remove the highlighting
        this.picked.material.emissive = this.pickedSavedColor;
        this.picked = null;
      }

      // something is picked
      if (intersects.length) {
        //grab the intersected object
        this.picked = intersects[0].object;
        this.pickedSavedColor = this.picked.material.emissive;
        this.picked.material.emissive = this.yellow;
        //move to travel function
        if (this.clicked && !this.hasTravelled) {
          this.clicked = false;
          this.objectSavedPosition = this.picked.position;
          this.savedQuaternion = this.picked.quaternion;
          // console.log("pos: ", this.objectSavedPosition);
          this.isTravelling = true;
          // this.hasTravelled = true;
          this.travellingObject = this.picked;
          //   this.clicked = false;
        } else if (this.clicked && this.hasTravelled) {
          this.clicked = false;
          this.isMovingBack = true;
          this.travellingObject = this.picked;
        }

        // console.log("picked: ", this.picked.material.emissive);
        // this.picked.position.lerp(this.camera.position, 0.01);
        // this.camera.lookAt(this.picked.position);
        // this.controls.target = this.picked.position;
        // console.log("camera pos: ", this.camera.position);
      }
      if (this.isTravelling) {
        // code from gman
        // https://stackoverflow.com/questions/59638065/three-js-move-an-object-to-front-of-camera
        const target = new THREE.Vector3(0, 0, -this.distanceFromCamera);
        console.log("target pos1:", target);
        this.camera.updateMatrixWorld();
        target.applyMatrix4(this.camera.matrixWorld);
        console.log("camera matrixWorld: ", this.camera.matrixWorld);
        console.log("targeta after matrix: ", target);
        const moveSpeed = 15;
        const distance = this.travellingObject.position.distanceTo(target);
        const amount = Math.min(moveSpeed * deltaTime, distance) / distance;
        if (distance > 0) {
          console.log("distance: ", distance);
          this.travellingObject.position.lerp(target, amount);
        } else {
          this.isTravelling = false;
          this.hasTravelled = true;
          this.controls.target = this.travellingObject.position;
          //to debug: saved position is getting updated with travel.
          // console.log("saved pos after travel: ", this.objectSavedPosition);
          // console.log(
          //   "travelling object pos: ",
          //   this.travellingObject.position
          // );
          // console.log(
          //   "quaternion after travel: ",
          //   this.travellingObject.quaternion
          // );
        }
      }

      //MOVING BACK CODE, position is hard coded
      if (this.isMovingBack) {
        // const target = this.objectSavedPosition;
        // const target = new THREE.Vector3(0, 0, 0);// move to main, only create once
        // target.applyMatrix4(this.camera.matrixWorld);
        const moveSpeed = 15;
        const distance = this.travellingObject.position.distanceTo(
          this.homeTarget
        );
        const amount = Math.min(moveSpeed * deltaTime, distance) / distance;
        if (distance > 0) {
          this.travellingObject.position.lerp(this.homeTarget, amount);
          // this.travellingObject.quaternion.rotateTowards(
          //   this.baseQuaternion,
          //   3
          // );
        } else {
          this.isMovingBack = false;
          this.hasTravelled = false;
          this.controls.enabled = true;
          this.controls.target = this.travellingObject.position;
        }
      }

      // object has moved in front of camera
      if (this.hasTravelled) {
        //disable orbit controls
        this.controls.enabled = false;

        // rotate the object when mouse clicked
        if (this.down) {
          // TODO: learn more about quaternions for rotation, commented code below looks
          // kinda funny...
          // this.travellingObject.quaternion.y += this.mouse.x - this.lastMouseX;
          // this.travellingObject.quaternion.x += this.lastMouseY - this.mouse.y;
          this.travellingObject.rotateY((this.mouse.x - this.lastMouseX)*2)
          this.travellingObject.rotateX(this.lastMouseY - this.mouse.y)
        }
      }

      this.lastMouseX = this.mouse.x;
      this.lastMouseY = this.mouse.y;
      this.clicked = false;
      this.controls.update();

      this.renderer.render(this.scene, this.camera);
      requestAnimationFrame(this.render); //passing it the render function
    },
    resizeRendererToDisplaySize(renderer) {
      const canvas = renderer.domElement;
      const pixelRatio = window.devicePixelRatio;
      const width = (canvas.clientWidth * pixelRatio) | 0;
      const height = (canvas.clientHeight * pixelRatio) | 0;
      const needResize = canvas.width !== width || canvas.height !== height;
      if (needResize) {
        console.log("needs resize");
        this.renderer.setSize(width, height, false);
      }
      return needResize;
    },
    createCamera() {
      const fov = 75;
      const aspect = 2;
      const near = 0.1;
      const far = 100;
      this.camera = new THREE.PerspectiveCamera(fov, aspect, near, far);
      this.camera.position.set(0, 0, 5);
      this.camera.lookAt(0, 0, 0);
    },
    addLight() {
      const color = 0xffffff;
      const intensity = 0.75;
      const light = new THREE.DirectionalLight(color, intensity);
      const ambLight = new THREE.AmbientLight(color, intensity);
      light.position.set(-1, 2, 4);
      this.scene.add(light);
      this.scene.add(ambLight);
    },
    setControls() {
      this.controls = new OrbitControls(this.camera, this.renderer.domElement);
    },
    initRaycaster() {
      this.raycaster = new THREE.Raycaster();
      this.mouse = new THREE.Vector2();
      this.picked = null;
      this.pickedSavedColor = 0;
    },
    initEventListeners() {
      this.$refs.c.addEventListener("mousemove", this.onMouseMove, false);
      //   this.controls.addEventListener("mousedown", this.onMouseDown, false);
      this.$refs.c.addEventListener("mousedown", this.onMouseDown, false);
      this.$refs.c.addEventListener("mouseup", this.onMouseUp, false);
      this.$refs.c.addEventListener("click", this.mouseClicked, false);
    },
    onMouseMove() {
      this.mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
      this.mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    },
    onMouseDown() {
      this.down = true;
      console.log("mouseDown");
    },
    onMouseUp() {
      this.down = false;
      console.log("mouseup");
    },
    mouseClicked() {
      this.clicked = true;
      console.log("click!");
    },
    loadGltf() {
      this.FE = {}
      this.gltfLoader = new GLTFLoader();
      const url = "http://localhost:8000/scene.gltf";
      this.gltfLoader.load(
        url,
        (gltf) => {
          this.FE = gltf.scene
          const root = gltf.scene;
          this.scene.add(root);
        },
        undefined,
        (error) => {
          console.log("error loading gltf", error);
        }
      );
    },
    loadTexture() {
      this.loadManager = new THREE.LoadingManager();
      const loader = new THREE.TextureLoader(this.loadManager);
      this.material = new THREE.MeshBasicMaterial({
        map: loader.load("http://localhost:8000/mip-low-res-enlarged.png"),
      });
    },
    loadTextures() {
      this.loadManager = new THREE.LoadingManager();
      const loader = new THREE.TextureLoader(this.loadManager);
      this.materials = [
        new THREE.MeshBasicMaterial({
          map: loader.load("http://localhost:8000/flower-1.jpg"),
        }),
        new THREE.MeshBasicMaterial({
          map: loader.load("http://localhost:8000/flower-2.jpg"),
        }),
        new THREE.MeshBasicMaterial({
          map: loader.load("http://localhost:8000/flower-3.jpg"),
        }),
        new THREE.MeshBasicMaterial({
          map: loader.load("http://localhost:8000/flower-4.jpg"),
        }),
        new THREE.MeshBasicMaterial({
          map: loader.load("http://localhost:8000/flower-5.jpg"),
        }),
        new THREE.MeshBasicMaterial({
          map: loader.load("http://localhost:8000/flower-6.jpg"),
        }),
      ];
    },
    createBox() {
      const boxWidth = 0.5;
      const boxHeight = 0.5;
      const boxDepth = 0.5;
      const geometry = new THREE.BoxBufferGeometry(
        boxWidth,
        boxHeight,
        boxDepth
      );
      this.material = new THREE.MeshPhongMaterial({ color: 0x44aa88 }); //phong material is affected by light
      this.cubeMesh = new THREE.Mesh(geometry, this.material);
      this.scene.add(this.cubeMesh);
    },
    createBoxGeometry(width, height, depth) {
      this.boxGeometry = new THREE.BoxGeometry(width, height, depth);
    },
    makeInstance(geometry, color, xPos) {
      //   const material = new THREE.MeshPhongMaterial({ color }); //same as color: color
      const cubeMesh = new THREE.Mesh(geometry, this.material);
      this.scene.add(cubeMesh);
      cubeMesh.position.x = xPos;

      return cubeMesh;
    },
    updatePosition() {
      this.FE.position.x = this.guiParams.xPos
      this.FE.position.y = this.guiParams.yPos
      this.FE.position.z = this.guiParams.zPos
    }
  },
  mounted() {
    this.main();
  },
};
</script>

<style lang="css" scoped>
#c {
  width: 100vw;
  height: 100vh;
  display: block;
  overflow: hidden;
}
</style>
