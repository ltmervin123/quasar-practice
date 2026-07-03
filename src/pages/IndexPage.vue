<!-- eslint-disable @typescript-eslint/no-explicit-any
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

// function isLowEndDevice(): boolean {
//   const memory = (navigator as any).deviceMemory as number | undefined;
//   const cores = navigator.hardwareConcurrency;

//   // deviceMemory is in GB, only available on Chromium-based browsers (most Android)
//   if (typeof memory === 'number' && memory <= 4) return true;
//   if (typeof cores === 'number' && cores <= 4) return true;

//   return false;
// }

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
</template> -->

<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from 'vue';
import { XR8Promise } from '@8thwall/engine-binary';
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import {
  FilesetResolver,
  HandLandmarker,
  type HandLandmarkerResult,
} from '@mediapipe/tasks-vision';

declare global {
  interface Window {
    THREE: typeof THREE;
    XR8: any;
    XRExtras: any;
  }
}

window.THREE = THREE;

type TrackingMode = 'surface' | 'hand';

const cameraFeed = ref<HTMLCanvasElement | null>(null);
const handVideo = ref<HTMLVideoElement | null>(null);
const isLoading = ref(true);
const loadingProgress = ref(0);
const loadError = ref<string | null>(null);
const isPlaced = ref(false);
const currentMode = ref<TrackingMode>('surface');
const isSwitchingMode = ref(false);

const MODEL_URL = '/assets/lake_titicaca_water_frog.glb';
const MODEL_SCALE = 0.2;
const PALM_LANDMARK_INDEX = 9; // middle-finger MCP joint — stable "palm center" point
const HAND_MODEL_DEPTH = 0.4; // meters in front of camera; tune to taste

// Hand tracking stability tuning
const HAND_SMOOTHING_ALPHA = 0.3; // 0–1: lower = smoother but laggier, higher = snappier but jitterier
const HAND_LOST_GRACE_MS = 500; // keep last known position briefly if detection drops a few frames

let modelRoot: THREE.Object3D | null = null;
let handLandmarker: HandLandmarker | null = null;
let handDetectRAF = 0;
let attachedToHand = false;
let sceneReady = false;
let pendingSurfaceSync = false;
let isMounted = false;
let modeGeneration = 0;
let modelLoadTimeoutId: ReturnType<typeof setTimeout> | null = null;

let lastPalmNormalized: { x: number; y: number } | null = null; // raw, latest detection — used for tap-to-attach
let smoothedPalm: { x: number; y: number } | null = null; // smoothed, used for per-frame placement
let lastDetectionTime = 0;
let lastHandSeenTime = 0;
let handDetectionIntervalMs = 100;

let xrControllerModuleName = 'xrcontroller'; // overwritten with the real name once XR8 loads
const ndcVector = new THREE.Vector3();

function isLowEndDevice(): boolean {
  const memory = (navigator as any).deviceMemory as number | undefined;
  const cores = navigator.hardwareConcurrency;

  if (typeof memory === 'number' && memory <= 4) return true;
  if (typeof cores === 'number' && cores <= 4) return true;

  return false;
}

function createSpeciesPipelineModule(XR8: any) {
  return {
    name: 'speciesar',
    onStart: () => {
      const { scene } = XR8.Threejs.xrScene();

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
        },
        (error) => {
          loadError.value = 'Failed to load 3D model.';
          console.error('GLTF load failed:', error);
        },
      );

      modelLoadTimeoutId = setTimeout(() => {
        if (isLoading.value) {
          console.warn('Model still loading after 15s — check Network tab for the .glb request');
        }
      }, 15000);

      sceneReady = true;
      if (pendingSurfaceSync) {
        syncCameraProjectionMatrix();
        pendingSurfaceSync = false;
      }
    },
  };
}

// --- Surface (SLAM) mode ---

function enableSurfaceMode() {
  const XR8 = window.XR8;
  if (!XR8) return;

  XR8.addCameraPipelineModule(XR8.XrController.pipelineModule());
  XR8.XrController.configure({ scale: 'absolute', enableLighting: true });

  if (sceneReady) {
    syncCameraProjectionMatrix();
  } else {
    pendingSurfaceSync = true;
  }
}

function disableSurfaceMode() {
  window.XR8?.removeCameraPipelineModules([xrControllerModuleName]);
  pendingSurfaceSync = false;
}

function syncCameraProjectionMatrix() {
  const XR8 = window.XR8;
  if (!XR8) return;
  const { camera } = XR8.Threejs.xrScene();
  XR8.XrController.updateCameraProjectionMatrix({
    origin: camera.position,
    facing: camera.quaternion,
  });
}

// --- Hand tracking mode ---

async function createHandLandmarker(vision: any): Promise<HandLandmarker> {
  const baseOptionsCommon = {
    modelAssetPath:
      'https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task',
  };

  try {
    return await HandLandmarker.createFromOptions(vision, {
      baseOptions: { ...baseOptionsCommon, delegate: 'GPU' },
      runningMode: 'VIDEO',
      numHands: 1,
      minHandDetectionConfidence: 0.6,
      minHandPresenceConfidence: 0.6,
      minTrackingConfidence: 0.6,
    });
  } catch (gpuError) {
    console.warn('GPU delegate unavailable for hand tracking, falling back to CPU:', gpuError);
    return await HandLandmarker.createFromOptions(vision, {
      baseOptions: { ...baseOptionsCommon, delegate: 'CPU' },
      runningMode: 'VIDEO',
      numHands: 1,
      minHandDetectionConfidence: 0.6,
      minHandPresenceConfidence: 0.6,
      minTrackingConfidence: 0.6,
    });
  }
}

async function enableHandMode(generation: number) {
  const lowEnd = isLowEndDevice();
  handDetectionIntervalMs = lowEnd ? 150 : 80; // throttle inference more aggressively on weak hardware
  const idealWidth = lowEnd ? 320 : 640;
  const idealHeight = lowEnd ? 240 : 480;

  let stream: MediaStream | null = null;

  try {
    stream = await navigator.mediaDevices.getUserMedia({
      video: {
        facingMode: 'environment',
        width: { ideal: idealWidth },
        height: { ideal: idealHeight },
      },
      audio: false,
    });

    // Bail out cleanly if the mode changed or the component unmounted while awaiting the camera
    if (!isMounted || generation !== modeGeneration || !handVideo.value) {
      stream.getTracks().forEach((t) => t.stop());
      return;
    }

    handVideo.value.srcObject = stream;
    await handVideo.value.play();

    if (!isMounted || generation !== modeGeneration) {
      stream.getTracks().forEach((t) => t.stop());
      return;
    }

    if (!handLandmarker) {
      const vision = await FilesetResolver.forVisionTasks(
        'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.3/wasm',
      );

      if (!isMounted || generation !== modeGeneration) {
        stream.getTracks().forEach((t) => t.stop());
        return;
      }

      handLandmarker = await createHandLandmarker(vision);
    }

    if (!isMounted || generation !== modeGeneration) return;

    lastDetectionTime = 0;
    lastHandSeenTime = 0;
    smoothedPalm = null;
    handTrackingLoop();
  } catch (error) {
    stream?.getTracks().forEach((t) => t.stop());
    loadError.value = 'Hand tracking unavailable on this device.';
    console.error('Hand tracking init failed:', error);
  }
}

function disableHandMode() {
  cancelAnimationFrame(handDetectRAF);
  (handVideo.value?.srcObject as MediaStream | null)?.getTracks().forEach((t) => t.stop());
  if (handVideo.value) handVideo.value.srcObject = null;
  attachedToHand = false;
  lastPalmNormalized = null;
  smoothedPalm = null;
}

function handTrackingLoop() {
  if (currentMode.value !== 'hand') return; // stop recursing once switched away

  if (handLandmarker && handVideo.value && handVideo.value.readyState >= 2) {
    const now = performance.now();

    // Throttle the actual ML inference — the render/smoothing step below still
    // runs every frame, so motion still looks smooth even at a lower detection rate.
    if (now - lastDetectionTime >= handDetectionIntervalMs) {
      lastDetectionTime = now;

      const result: HandLandmarkerResult = handLandmarker.detectForVideo(handVideo.value, now);

      if (result.landmarks.length > 0) {
        const palm = result.landmarks[0]![PALM_LANDMARK_INDEX]!;
        lastHandSeenTime = now;
        lastPalmNormalized = { x: palm.x, y: palm.y };

        smoothedPalm = smoothedPalm
          ? {
              x: smoothedPalm.x + (palm.x - smoothedPalm.x) * HAND_SMOOTHING_ALPHA,
              y: smoothedPalm.y + (palm.y - smoothedPalm.y) * HAND_SMOOTHING_ALPHA,
            }
          : { x: palm.x, y: palm.y };
      } else if (now - lastHandSeenTime > HAND_LOST_GRACE_MS) {
        // Only clear after a real gap — avoids single dropped-frame flicker
        // making the model vanish and snap back.
        lastPalmNormalized = null;
        smoothedPalm = null;
      }
    }
  }

  if (attachedToHand && smoothedPalm) {
    updateModelToPalm(smoothedPalm);
  }

  handDetectRAF = requestAnimationFrame(handTrackingLoop);
}

function updateModelToPalm(normalized: { x: number; y: number }) {
  if (!modelRoot || !sceneReady || !window.XR8) return;

  const { camera } = window.XR8.Threejs.xrScene();

  const ndcX = normalized.x * 2 - 1;
  const ndcY = -(normalized.y * 2 - 1);

  ndcVector.set(ndcX, ndcY, 0.5).unproject(camera);
  const direction = ndcVector.sub(camera.position).normalize();
  const targetPosition = camera.position.clone().add(direction.multiplyScalar(HAND_MODEL_DEPTH));

  modelRoot.position.copy(targetPosition);

  // Yaw-only "billboard": face the camera horizontally without inheriting
  // its pitch/roll, so the model stays upright instead of twisting.
  const dx = camera.position.x - modelRoot.position.x;
  const dz = camera.position.z - modelRoot.position.z;
  modelRoot.rotation.set(0, Math.atan2(dx, dz), 0);

  modelRoot.visible = true;
}

// --- Mode switching ---

async function switchMode(mode: TrackingMode) {
  if (mode === currentMode.value || isSwitchingMode.value) return;
  isSwitchingMode.value = true;
  const generation = ++modeGeneration;

  if (currentMode.value === 'surface') {
    disableSurfaceMode();
  } else {
    disableHandMode();
  }

  if (modelRoot) modelRoot.visible = false;
  isPlaced.value = false;
  currentMode.value = mode;

  if (mode === 'surface') {
    enableSurfaceMode();
  } else {
    await enableHandMode(generation);
  }

  if (generation === modeGeneration) {
    isSwitchingMode.value = false;
  }
}

// --- Tap handling ---

function handleTap(e: TouchEvent | MouseEvent) {
  if (!modelRoot || isLoading.value || isSwitchingMode.value) return;

  if (currentMode.value === 'hand') {
    if (lastPalmNormalized) {
      attachedToHand = true;
      smoothedPalm = { ...lastPalmNormalized }; // snap smoothing to the current position on attach
      updateModelToPalm(smoothedPalm);
      isPlaced.value = true;
    }
    return;
  }

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
  if (currentMode.value === 'hand') {
    attachedToHand = false;
  } else {
    window.XR8?.XrController?.recenter();
  }
  isPlaced.value = false;
  if (modelRoot) modelRoot.visible = false;
}

onMounted(async () => {
  isMounted = true;

  try {
    const XR8 = await XR8Promise;

    const xrControllerModule = XR8.XrController.pipelineModule();
    xrControllerModuleName = xrControllerModule.name;

    const pipelineModules = [
      XR8.GlTextureRenderer.pipelineModule(),
      XR8.Threejs.pipelineModule(),
      window.XRExtras.FullWindowCanvas.pipelineModule(),
      createSpeciesPipelineModule(XR8),
    ];

    if (currentMode.value === 'surface') {
      pipelineModules.push(xrControllerModule);
    }

    XR8.addCameraPipelineModules(pipelineModules);

    XR8.run({
      canvas: cameraFeed.value,
      allowedDevices: XR8.XrConfig.device().MOBILE_AND_HEADSETS,
      cameraConfig: { direction: XR8.XrConfig.camera().BACK },
    });

    if (!isMounted) return; // unmounted while XR8Promise/run were resolving

    if (currentMode.value === 'surface') {
      enableSurfaceMode();
    } else {
      await enableHandMode(modeGeneration);
    }

    cameraFeed.value?.addEventListener('touchstart', handleTap);
    cameraFeed.value?.addEventListener('click', handleTap);
  } catch (error) {
    loadError.value = `AR failed to start: ${(error as Error).message}`;
    console.error('XR8 init error:', error);
  }
});

onBeforeUnmount(() => {
  isMounted = false;
  modeGeneration++; // invalidates any in-flight enableHandMode

  cameraFeed.value?.removeEventListener('touchstart', handleTap);
  cameraFeed.value?.removeEventListener('click', handleTap);

  if (modelLoadTimeoutId) clearTimeout(modelLoadTimeoutId);

  disableHandMode();
  handLandmarker?.close();
  handLandmarker = null;

  if (currentMode.value === 'surface') {
    disableSurfaceMode();
  }

  window.XR8?.stop?.();
});
</script>

<template>
  <canvas ref="cameraFeed" class="w-screen h-screen block fixed inset-0"></canvas>
  <video ref="handVideo" playsinline muted style="display: none"></video>

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
    {{
      currentMode === 'hand'
        ? 'Tap your hand to place the model'
        : 'Tap a surface to place the model'
    }}
  </div>

  <div class="fixed top-4 left-1/2 -translate-x-1/2 z-10 flex gap-2 bg-black/60 rounded-full p-1">
    <button
      @click="switchMode('surface')"
      :disabled="isSwitchingMode"
      class="px-4 py-1.5 rounded-full text-sm text-white"
      :class="currentMode === 'surface' ? 'bg-white/30' : ''"
    >
      Surface
    </button>
    <button
      @click="switchMode('hand')"
      :disabled="isSwitchingMode"
      class="px-4 py-1.5 rounded-full text-sm text-white"
      :class="currentMode === 'hand' ? 'bg-white/30' : ''"
    >
      Hand
    </button>
  </div>

  <button
    @click="recenter"
    class="fixed bottom-8 left-1/2 -translate-x-1/2 z-10 bg-black/60 text-white px-4 py-2 rounded-full text-sm"
  >
    Recenter
  </button>
</template>
