<template>
  <canvas id="c" ref="c"></canvas>
</template>

<script>
import * as THREE from "three";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
export default {
  name: "threeExamples",
  data() {
    return {
      down: false,
    };
  },
  methods: {
    main() {
      const canvas = document.querySelector("#c");
      const renderer = new THREE.WebGLRenderer({ canvas });

      const fov = 45;
      const aspect = 2; // the canvas default
      const near = 0.1;
      const far = 1000;
      const camera = new THREE.PerspectiveCamera(fov, aspect, near, far);
      camera.position.set(0, 10, 20);

      const controls = new OrbitControls(camera, canvas);
      controls.target.set(0, 0, 0);
      controls.update();

      const scene = new THREE.Scene();
      scene.background = new THREE.Color("lightblue");

      {
        const color = 0xffffff;
        const intensity = 1;
        const light = new THREE.DirectionalLight(color, intensity);
        light.position.set(0, 10, 0);
        light.target.position.set(-5, 0, 0);
        scene.add(light);
        scene.add(light.target);
      }

      const gridHelper = new THREE.GridHelper(100, 10);
      scene.add(gridHelper);
      gridHelper.position.set(0, -5, 0);

      const cube = new THREE.Mesh(
        new THREE.BoxBufferGeometry(1, 1, 1),
        new THREE.MeshPhongMaterial({ color: "red" })
      );
      scene.add(cube);

      function resizeRendererToDisplaySize(renderer) {
        const canvas = renderer.domElement;
        const width = canvas.clientWidth;
        const height = canvas.clientHeight;
        const needResize = canvas.width !== width || canvas.height !== height;
        if (needResize) {
          renderer.setSize(width, height, false);
        }
        return needResize;
      }

      let then = 0;
      function render(now) {
        now *= 0.001; // convert to seconds
        const deltaTime = now - then;
        then = now;

        if (resizeRendererToDisplaySize(renderer)) {
          const canvas = renderer.domElement;
          camera.aspect = canvas.clientWidth / canvas.clientHeight;
          camera.updateProjectionMatrix();
        }

        cube.rotation.x = now;
        cube.rotation.y = now * 1.1;

        // move cube in front of camera
        {
          const distanceFromCamera = 3; // 3 units
          const target = new THREE.Vector3(0, 0, -distanceFromCamera);
          target.applyMatrix4(camera.matrixWorld);

          const moveSpeed = 60; // units per second
          const distance = cube.position.distanceTo(target);
          console.log(cube.position.distanceTo(target));
          if (distance > 0) {
            const amount = Math.min(moveSpeed * deltaTime, distance) / distance;
            cube.position.lerp(target, amount);
            cube.material.color.set("green");
          } else {
            cube.material.color.set("red");
          }
        }

        renderer.render(scene, camera);

        requestAnimationFrame(render);
      }

      requestAnimationFrame(render);
    },
    render() {
      // time *= 0.001; //convert ms to s

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

      this.picked = null;
      if (intersects.length > 0) {
        this.picked = intersects[0].object;

        console.log("picked: ", this.picked);
        this.picked.position.lerp(this.camera.position, 0.01);
        // this.camera.lookAt(this.picked.position);
        // this.controls.target = this.picked.position;
        // console.log("camera pos: ", this.camera.position);
      }

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
    },
    initEventListeners() {
      this.$refs.c.addEventListener("mousemove", this.onMouseMove, false);
      //   this.$refs.c.addEventListener("mousedown", this.onMouseDown, false);
      //   this.$refs.c.addEventListener("mouseup", this.onMouseUp, false);
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
      console.log("mouseup");
    },
    loadGltf() {
      this.gltfLoader = new GLTFLoader();
      const url = "http://localhost:8000/scene.gltf";
      this.gltfLoader.load(
        url,
        (gltf) => {
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
      const boxWidth = 1;
      const boxHeight = 1;
      const boxDepth = 1;
      const geometry = new THREE.BoxBufferGeometry(
        boxWidth,
        boxHeight,
        boxDepth
      );
      //   const material = new THREE.MeshPhongMaterial({ color: 0x44aa88 }); //phong material is affected by light
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
