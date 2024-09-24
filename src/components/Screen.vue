<template>
  <div>
    <canvas ref="experience" class="half-size-canvas"></canvas>

    <div class="cols">
      <ul>
        <li>
          <button @click="toggleMarkers('red')">Toggle Red</button>
        </li>
        <li>
          <button @click="toggleMarkers('green')">Toggle Green</button>
        </li>
        <li>
          <button @click="toggleMarkers('blue')">Toggle Blue</button>
        </li>
      </ul>
      <div class="output">
        <p>Deviation clicked:</p>
        <p>{{ output }}</p>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { onMounted, ref, shallowRef } from 'vue';
import {
  AxesHelper,
  PerspectiveCamera,
  Scene,
  WebGLRenderer,
  DirectionalLight,
  Raycaster,
  Vector2,
  Mesh,
  MeshMatcapMaterial,
  SphereGeometry,
  TextureLoader
} from 'three';
import { STLLoader } from 'three/examples/jsm/loaders/STLLoader.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export default {
  props: {
    modelUrl: {
      type: String,
      required: true,
    },
    lightIntensity: {
      type: Number,
      required: true,
    },
    data: {
      type: Array,
      required: false,
    },
    textureUrl: {
      type: String,
      required: false,
    },
  },

  setup(props) {
    const experience = ref<HTMLCanvasElement | null>(null);
    const output = shallowRef<string>('');

    const scene = new Scene();
    const camera = new PerspectiveCamera(50, window.innerWidth / window.innerHeight, 1, 1000);
    
    // Set the camera position and ensure it looks at the center of the scene
    camera.position.set(75, 0, 0); // Adjust the position as needed
    camera.lookAt(0, 0, 0); // Ensure camera is looking at the center

    // Directional light with shadows
    const directionalLight = new DirectionalLight(0xffffff, 1.0);
    directionalLight.position.set(10, 10, 10);
    directionalLight.castShadow = true;
    directionalLight.shadow.mapSize.width = 2048;
    directionalLight.shadow.mapSize.height = 2048;
    directionalLight.shadow.bias = -0.0005;
    scene.add(directionalLight);

    let renderer: WebGLRenderer | null = null;
    const raycaster = new Raycaster();
    const mouse = new Vector2();

    const spheres: Mesh[] = [];

    const createSpheresFromData = (data: any, scene: Scene) => {
      data.forEach((sphereInfo: any) => {
        const geometry = new SphereGeometry(
          sphereInfo.radius,
          sphereInfo.widthSegments,
          sphereInfo.heightSegments
        );
        const material = new MeshMatcapMaterial({
          color: sphereInfo.color,
          matcap: new TextureLoader().load(props.textureUrl || '')
        });
        const sphere = new Mesh(geometry, material);
        sphere.userData.tag = sphereInfo.tag;
        sphere.position.set(sphereInfo.position.x, sphereInfo.position.y, sphereInfo.position.z);
        scene.add(sphere);
        spheres.push(sphere);
      });
    };

    const onMouseClick = (event: MouseEvent) => {
      if (!renderer) return;

      const rect = experience.value?.getBoundingClientRect();
      if (!rect) return;

      mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
      mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

      raycaster.setFromCamera(mouse, camera);
      const intersects = raycaster.intersectObjects(spheres);
      if (intersects.length > 0) {
        const sphereTag = intersects[0].object.userData.tag;
        output.value = sphereTag;
        console.log(sphereTag);
        const event = new CustomEvent('deviation-clicked-event', { detail: sphereTag });
        window.document.dispatchEvent(event);
      }
    };

    const toggleMarkers = (color: string) => {
      spheres.forEach((element) => {
        if (element.userData.tag === color) element.visible = !element.visible;
      });
    };

    onMounted(() => {
      if (experience.value) {
        renderer = new WebGLRenderer({
          canvas: experience.value,
          antialias: true,
        });
        renderer.setSize(1600, 1200);
        renderer.shadowMap.enabled = true;
        // renderer.shadowMap.type = PCFSoftShadowMap; // Softer shadows

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.update();

        const animate = () => {
          requestAnimationFrame(animate);
          controls.update();
          renderer!.render(scene, camera);
        };
        animate();

        const loader = new STLLoader();
        loader.load(
          props.modelUrl,
          (geometry) => {
            const material = new MeshMatcapMaterial({
              color: 0xcccccc,
              matcap: new TextureLoader().load(props.textureUrl || '')
            });
            const mesh = new Mesh(geometry, material);

            mesh.geometry.computeVertexNormals();
            mesh.geometry.center();

            mesh.scale.set(0.1, 0.1, 0.1);
            scene.add(mesh);

            // Add AxesHelper to show model rotation
            const axesHelper = new AxesHelper(50); // Adjust the size if needed
            
            mesh.add(axesHelper);

            // Create spheres from the provided data
            createSpheresFromData(props.data, scene);

            window.addEventListener('click', onMouseClick, false);
          },
          (xhr) => {
            console.log(`${(xhr.loaded / xhr.total) * 100}% loaded`);
          },
          (error) => {
            console.error('An error occurred while loading the STL model', error);
          }
        );
      }
    });

    return {
      experience,
      output,
      toggleMarkers,
    };
  },
};
</script>

<style scoped>
.half-size-canvas {
  border: 1px solid black;
}
.cols {
  display: flex;
}
.output {
  margin-left: 3rem;
}
</style>
