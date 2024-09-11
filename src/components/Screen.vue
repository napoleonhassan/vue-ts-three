<template>
  <div>
    <canvas ref="experience" class="half-size-canvas"></canvas>
    <br />
    <button @click="toggleMarkers('red')">Toggle Red</button>
    <button @click="toggleMarkers('green')">Toggle Green</button>
    <button @click="toggleMarkers('blue')">Toggle Blue</button>
    <br />
    <p>{{ output }}</p>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue';
import {
  PerspectiveCamera,
  Scene,
  WebGLRenderer,
  AmbientLight,
  SphereGeometry,
  Mesh,
  MeshBasicMaterial,
  Raycaster,
  Vector2
} from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { deviations } from '../data/mock';

// Reactive properties
const experience = ref<HTMLCanvasElement | null>(null);
const output = ref<string>(''); // Make 'output' reactive

// Three.js related variables
const scene = new Scene();
const camera = new PerspectiveCamera(75, window.innerWidth / window.innerHeight, 1, 1000);
camera.position.z = 5;

const ambientLight = new AmbientLight(0xffffff, 10);
scene.add(ambientLight);

let renderer: WebGLRenderer | null = null;
const raycaster = new Raycaster();
const mouse = new Vector2();

const spheres: Mesh[] = []; // Array to hold all sphere meshes

const createSpheresFromData = (data: any, scene: Scene) => {
  data.forEach((sphereInfo: any) => {
    const geometry = new SphereGeometry(
      sphereInfo.radius,
      sphereInfo.widthSegments,
      sphereInfo.heightSegments
    );
    const material = new MeshBasicMaterial({ color: sphereInfo.color });
    const sphere = new Mesh(geometry, material);
    sphere.userData.tag = sphereInfo.tag;
    sphere.position.set(sphereInfo.position.x, sphereInfo.position.y, sphereInfo.position.z);
    scene.add(sphere);
    spheres.push(sphere); // Add sphere to the array
  });
};

const onMouseClick = (event: MouseEvent) => {
  if (!renderer) return;

  // Get the canvas position and size
  const rect = experience.value?.getBoundingClientRect();
  if (!rect) return;

  // Calculate mouse position relative to the canvas
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

  // Update raycaster with camera and mouse coordinates
  raycaster.ray.origin.setFromMatrixPosition(camera.matrixWorld);
  raycaster.ray.direction.set(mouse.x, mouse.y, 1).unproject(camera).sub(raycaster.ray.origin).normalize();

  // Check for intersections with all spheres
  const intersects = raycaster.intersectObjects(spheres);
  if (intersects.length > 0) {
    const sphereTag = intersects[0].object.userData.tag;
    const sphere = deviations.filter(d => d.tag == sphereTag);
    console.log('Deviation clicked:', sphereTag, sphere);
    output.value = `Deviation clicked: ${sphereTag}`; // Update the reactive 'output' variable
    
    const event = new CustomEvent('deviation-click-event', { detail: sphereTag });
    window.document.dispatchEvent(event);
  }
};

const toggleMarkers = (color) => {
  spheres.forEach((element) => {
    if (element.userData.tag === color) element.visible = !element.visible;
  });
};

onMounted(() => {
  if (experience.value) {
    renderer = new WebGLRenderer({
      canvas: experience.value,
      antialias: true
    });
    
    // Set renderer size to half of window width and height
    renderer.setSize(window.innerWidth / 2, window.innerHeight / 2);
    
    const controls = new OrbitControls(camera, renderer.domElement);
    camera.position.set(0, 16, 0);
    controls.update();
    
    const animate = () => {
      requestAnimationFrame(animate);
      controls.update();
      renderer!.render(scene, camera);
    };
    animate();
    
    const loader = new GLTFLoader();
    loader.load(
      '../src/assets/models/gear.gltf.glb',
      (gltf) => {
        gltf.scene.position.y = -2;
        scene.add(gltf.scene);
        createSpheresFromData(deviations, scene);
        
        // Add event listener for mouse clicks
        window.addEventListener('click', onMouseClick, false);
      },
      (xhr) => {
        console.log(`${(xhr.loaded / xhr.total) * 100}% loaded`);
      },
      (error) => {
        console.error('An error happened while loading the GLTF model', error);
      }
    );
  }
});

</script>

<style scoped>
.half-size-canvas {
  width: 50vw;
  height: 50vh;
  border: 1px solid black;
}
</style>
