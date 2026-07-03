<!-- eslint-disable @typescript-eslint/no-explicit-any -->
<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from 'vue';
import { XR8Promise } from '@8thwall/engine-binary';
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

declare global {
  interface Window {
    THREE: typeof THREE;
    XR8: any;
    XRExtras: any;
  }
}

window.THREE = THREE;

const cameraFeed = ref<HTMLCanvasElement | null>(null);
const isLoading = ref(true);
const loadingProgress = ref(0);
const loadError = ref<string | null>(null);
const isPlaced = ref(false);

const MODEL_URL = '/assets/lake_titicaca_water_frog.glb';
const MODEL_SCALE = 0.2;

let modelRoot: THREE.Object3D | null = null;

function createSpeciesPipelineModule(XR8: any) {
  return {
    name: 'speciesar',
    onStart: () => {
      const { scene, camera } = XR8.Threejs.xrScene();

      scene.add(
        new THREE.DirectionalLight(0xffffff, 1).translateX(1).translateY(1).translateZ(1),
        new THREE.AmbientLight(0x404040, 2),
      );

      new GLTFLoader().load(
        MODEL_URL,
        (gltf) => {
          gltf.scene.scale.set(MODEL_SCALE, MODEL_SCALE, MODEL_SCALE);
          gltf.scene.visible = false;
          scene.add(gltf.scene);
          modelRoot = gltf.scene;
          isLoading.value = false;
        },
        (progress) => {
          loadingProgress.value = progress.total ? (progress.loaded / progress.total) * 100 : 0;
          if (progress.total) {
            console.log(`GLTF loading: ${loadingProgress.value.toFixed(0)}%`);
          } else {
            console.log(`GLTF loading: ${progress.loaded} bytes (total size unknown)`);
          }
        },
        (error) => {
          loadError.value = 'Failed to load 3D model.';
          console.error('GLTF load failed:', error);
        },
      );

      // Fallback so the UI doesn't hang forever with no explanation
      setTimeout(() => {
        if (isLoading.value) {
          console.warn('Model still loading after 15s — check Network tab for the .glb request');
        }
      }, 15000);

      XR8.XrController.configure({
        scale: 'absolute',
        enableLighting: true,
      });

      XR8.XrController.updateCameraProjectionMatrix({
        origin: camera.position,
        facing: camera.quaternion,
      });
    },
  };
}

function handleTap(e: TouchEvent | MouseEvent) {
  if (!modelRoot || isLoading.value) return;

  const point = 'touches' in e ? e.touches[0] : e;
  if (!point) return;

  const x = point.clientX / window.innerWidth;
  const y = point.clientY / window.innerHeight;

  const hitResults = window.XR8?.XrController?.hitTest(x, y, ['FEATURE_POINT']);
  const hit = hitResults?.[0];

  if (hit) {
    modelRoot.position.copy(hit.position);
    if (hit.rotation) modelRoot.quaternion.copy(hit.rotation);
    modelRoot.visible = true;
    isPlaced.value = true;
  }
}

function recenter() {
  window.XR8?.XrController?.recenter();
  isPlaced.value = false;
  if (modelRoot) modelRoot.visible = false;
}

onMounted(async () => {
  try {
    const XR8 = await XR8Promise;

    XR8.addCameraPipelineModules([
      XR8.GlTextureRenderer.pipelineModule(),
      XR8.Threejs.pipelineModule(),
      XR8.XrController.pipelineModule(),
      window.XRExtras.FullWindowCanvas.pipelineModule(),
      createSpeciesPipelineModule(XR8),
    ]);

    XR8.run({
      canvas: cameraFeed.value,
      allowedDevices: XR8.XrConfig.device().MOBILE_AND_HEADSETS,
      cameraConfig: { direction: XR8.XrConfig.camera().BACK },
    });

    cameraFeed.value?.addEventListener('touchstart', handleTap);
    cameraFeed.value?.addEventListener('click', handleTap);
  } catch (error) {
    loadError.value = 'AR failed to start.';
    console.error('XR8 init error:', error);
  }
});

onBeforeUnmount(() => {
  cameraFeed.value?.removeEventListener('touchstart', handleTap);
  cameraFeed.value?.removeEventListener('click', handleTap);
  window.XR8?.stop?.();
});
</script>

<template>
  <canvas ref="cameraFeed" class="w-screen h-screen block fixed inset-0"></canvas>

  <div
    v-if="isLoading"
    class="fixed inset-0 z-10 flex items-center justify-center bg-black/50 text-white"
  >
    Loading AR experience… {{ loadingProgress.toFixed(0) }}%
  </div>

  <div
    v-if="loadError"
    class="fixed inset-0 z-10 flex items-center justify-center bg-black/70 text-white"
  >
    {{ loadError }}
  </div>

  <div
    v-if="!isLoading && !isPlaced"
    class="fixed top-16 left-1/2 -translate-x-1/2 z-10 bg-black/60 text-white px-4 py-2 rounded-full text-sm"
  >
    Tap a surface to place the model
  </div>

  <button
    @click="recenter"
    class="fixed bottom-8 left-1/2 -translate-x-1/2 z-10 bg-black/60 text-white px-4 py-2 rounded-full text-sm"
  >
    Recenter
  </button>
</template>
