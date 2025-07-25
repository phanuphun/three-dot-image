<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue";
import * as THREE from "three";

const scene        = new THREE.Scene();
const containerRef = ref<HTMLDivElement|null>(null);
const coordLabel   = ref<HTMLDivElement|null>(null);
const overlay      = ref<HTMLCanvasElement|null>(null);

let overlayCtx!: CanvasRenderingContext2D;
let camera!: THREE.OrthographicCamera;
let renderer!: THREE.WebGLRenderer;
let animationId: number;

let raycaster = new THREE.Raycaster();
let pointer   = new THREE.Vector2();
let planeMesh!: THREE.Mesh;
let imgW = 0, imgH = 0;

// เก็บจุดพร้อมทั้ง display และ world coords
type Pt = { dx:number; dy:number; wx:number; wy:number; };
const points: Pt[] = [];

// ระยะ snap (world units)
const SNAP_DIST = 20;

function onPointerMove(ev: PointerEvent) {
  if (!containerRef.value || !coordLabel.value) return;
  const r = containerRef.value.getBoundingClientRect();
  const px = ev.clientX - r.left;
  const py = ev.clientY - r.top;

  // update coord label ด้วย Raycaster+UV
  pointer.x = (px/r.width)*2 - 1;
  pointer.y = -(py/r.height)*2 + 1;
  raycaster.setFromCamera(pointer, camera);
  const hit = raycaster.intersectObject(planeMesh)[0];
  if (!hit) {
    coordLabel.value.textContent = "—";
    return;
  }

  const uv = hit.uv!;
  const ix = uv.x * imgW;
  const iy = (1 - uv.y) * imgH;
  coordLabel.value.style.left = `${px}px`;
  coordLabel.value.style.top  = `${py - 20}px`;
  coordLabel.value.textContent = `(${ix.toFixed(0)}, ${iy.toFixed(0)})`;
}

function onPointerDown(ev: PointerEvent) {
  if (!containerRef.value) return;
  const r = containerRef.value.getBoundingClientRect();
  const dx = ev.clientX - r.left;
  const dy = ev.clientY - r.top;

  // ยิง Raycaster หา hit.point (world)
  pointer.x = (dx/r.width)*2 - 1;
  pointer.y = -(dy/r.height)*2 + 1;
  raycaster.setFromCamera(pointer, camera);
  const h = raycaster.intersectObject(planeMesh)[0];
  if (!h) return;
  const wp = h.point; // THREE.Vector3

  // ถ้าเก็บ ≥3 จุด และจุดนี้ใกล้จุดแรก → ปิดรูปร่าง
  if (points.length >= 3) {
    const first = new THREE.Vector2(points[0].wx, points[0].wy);
    const now   = new THREE.Vector2(wp.x, wp.y);
    if (first.distanceTo(now) < SNAP_DIST) {
      createShape();
      return;
    }
  }

  // ไม่ปิด ก็เก็บ point ใหม่
  points.push({ dx, dy, wx: wp.x, wy: wp.y });
  drawOverlay();
}

function drawOverlay() {
  if (!overlayCtx || !overlay.value) return;
  const W = overlay.value.width, H = overlay.value.height;
  overlayCtx.clearRect(0,0,W,H);

  // เส้น
  overlayCtx.strokeStyle = "lime";
  overlayCtx.lineWidth   = 2;
  if (points.length > 1) {
    overlayCtx.beginPath();
    overlayCtx.moveTo(points[0].dx, points[0].dy);
    for (let p of points.slice(1)) overlayCtx.lineTo(p.dx, p.dy);
    overlayCtx.stroke();
  }

  // จุด
  overlayCtx.fillStyle = "cyan";
  for (let p of points) {
    overlayCtx.beginPath();
    overlayCtx.arc(p.dx, p.dy, 5, 0, Math.PI*2);
    overlayCtx.fill();
  }
}

function createShape() {
  // สร้าง THREE.Shape จาก world coords
  const shape = new THREE.Shape(
    points.map(p => new THREE.Vector2(p.wx, p.wy))
  );
  const geom = new THREE.ShapeGeometry(shape);
  const mat  = new THREE.MeshBasicMaterial({
    color: 0xff0000, side: THREE.DoubleSide, opacity: 0.5, transparent: true
  });
  scene.add(new THREE.Mesh(geom, mat));

  // ล้าง overlay + reset points
  overlayCtx.clearRect(0,0,overlay.value!.width, overlay.value!.height);
  points.length = 0;
}

function onFileChange(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0];
  if (!file) return;
  const url = URL.createObjectURL(file);
  new THREE.TextureLoader().load(url, tex => {
    imgW = tex.image.width;
    imgH = tex.image.height;
    planeMesh = new THREE.Mesh(
      new THREE.PlaneGeometry(imgW, imgH),
      new THREE.MeshBasicMaterial({ map: tex })
    );
    scene.add(planeMesh);
    URL.revokeObjectURL(url);
  });
}

function resizeAll() {
  if (!containerRef.value) return;
  const w = containerRef.value.clientWidth;
  const h = containerRef.value.clientHeight;
  // renderer + camera
  renderer.setSize(w, h);
  camera.left   = -w/2; camera.right = w/2;
  camera.top    =  h/2; camera.bottom= -h/2;
  camera.updateProjectionMatrix();
  // overlay canvas
  overlay.value!.width  = w;
  overlay.value!.height = h;
}

onMounted(() => {
  // camera + renderer
  const w = containerRef.value!.clientWidth;
  const h = containerRef.value!.clientHeight;
  camera = new THREE.OrthographicCamera(-w/2, w/2, h/2, -h/2, 0.1, 1000);
  camera.position.set(0,0,10); camera.lookAt(0,0,0);
  renderer = new THREE.WebGLRenderer({ antialias:true });
  renderer.setPixelRatio(window.devicePixelRatio);
  containerRef.value!.appendChild(renderer.domElement);

  // init overlay
  overlayCtx = overlay.value!.getContext("2d")!;
  resizeAll();
  window.addEventListener("resize", resizeAll);

  // loop
  const loop = () => {
    animationId = requestAnimationFrame(loop);
    renderer.render(scene, camera);
  };
  loop();
});

onUnmounted(() => {
  window.removeEventListener("resize", resizeAll);
  cancelAnimationFrame(animationId);
});
</script>

<template>
  <main>
    <div class="flex justify-center items-center h-16 bg-blue-500 text-white">
      <h1 class="text-xl font-bold">Blue Print Import</h1>
      <input type="file" accept=".png , .jpg , .jpeg" class="ml-4 p-2 bg-white text-blue-500 rounded"
        @change="onFileChange" />
    </div>


    <div ref="containerRef" class="relative w-full h-screen bg-gray-200 cursor-crosshair" @pointermove="onPointerMove"
      @pointerdown="onPointerDown">
      <!-- Rulers เหมือนเดิม… -->
      <canvas ref="rulerH" class="absolute top-0 left-0 w-full h-6 pointer-events-none"></canvas>
      <canvas ref="rulerV" class="absolute top-0 right-0 w-6 h-full pointer-events-none"></canvas>

      <!-- overlay canvas สำหรับวาดจุด–เส้น -->
      <canvas ref="overlay" class="absolute top-0 left-0 w-full h-full pointer-events-none"></canvas>

      <!-- ป้ายพิกัด -->
      <div ref="coordLabel" class="absolute pointer-events-none text-xs px-1.5 py-0.5 bg-black/60 text-white rounded"
        style="transform: translate(-50%, -100%);"></div>
    </div>
  </main>
</template>
