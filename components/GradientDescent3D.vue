<template>
  <div class="gd3d">
    <div ref="mountEl" class="gd3d-canvas" aria-label="3D 梯度下降动画"></div>
    <div v-if="failed" class="gd3d-fallback">当前浏览器未能初始化 3D 画面（WebGL 不可用）</div>
    <div class="gd3d-label top">高损失山峰</div>
    <div class="gd3d-label bottom">局部低谷</div>
    <svg class="gd3d-axes" viewBox="0 0 130 80" aria-hidden="true">
      <defs>
        <marker id="gd3d-arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="5" markerHeight="5" orient="auto-start-reverse">
          <path d="M0,0 L10,5 L0,10 Z" fill="#475569" />
        </marker>
        <marker id="gd3d-arrow-orange" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="5" markerHeight="5" orient="auto-start-reverse">
          <path d="M0,0 L10,5 L0,10 Z" fill="#ea580c" />
        </marker>
      </defs>
      <line x1="18" y1="66" x2="72" y2="66" stroke="#475569" stroke-width="1.1" marker-end="url(#gd3d-arrow)" />
      <line x1="18" y1="66" x2="54" y2="44" stroke="#475569" stroke-width="1.1" marker-end="url(#gd3d-arrow)" />
      <line x1="18" y1="66" x2="18" y2="22" stroke="#ea580c" stroke-width="1.1" marker-end="url(#gd3d-arrow-orange)" />
      <text x="75" y="69" class="gd3d-axis-label">参数 1</text>
      <text x="57" y="40" class="gd3d-axis-label">参数 2</text>
      <text x="22" y="20" class="gd3d-axis-label gd3d-axis-label-orange">损失</text>
    </svg>
  </div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'
import * as THREE from 'three'

const mountEl = ref<HTMLDivElement | null>(null)
const failed = ref(false)

let renderer: THREE.WebGLRenderer | undefined
let frameId = 0
let resizeObserver: ResizeObserver | undefined
let intersectionObserver: IntersectionObserver | undefined
let contextLostHandler: ((e: Event) => void) | undefined
let contextRestoredHandler: (() => void) | undefined
const lossHeight = 0.9

function gaussian(x: number, z: number, cx: number, cz: number, sx: number, sz: number, amp: number) {
  const dx = (x - cx) / sx
  const dz = (z - cz) / sz
  return amp * Math.exp(-0.5 * (dx * dx + dz * dz))
}

function loss(x: number, z: number) {
  const base = 1.1 + 0.1 * x + 0.08 * z + 0.07 * x * x + 0.06 * z * z
  const hills =
    gaussian(x, z, -1.95, 1.55, 0.95, 0.7, 1.35) +
    gaussian(x, z, 0.25, 0.45, 0.65, 0.85, 1.05) +
    gaussian(x, z, 1.75, -1.0, 0.9, 0.75, 0.95) +
    gaussian(x, z, -1.35, -1.35, 0.75, 0.8, 0.58)
  const valleys =
    gaussian(x, z, -0.65, 0.35, 0.85, 0.7, 0.82) +
    gaussian(x, z, 1.05, 1.25, 0.85, 0.8, 0.62)
  const smallRidges = 0.08 * Math.sin(2.0 * x + 0.7) * Math.sin(1.7 * z - 0.4)

  return base + hills - valleys + smallRidges
}

function gradient(x: number, z: number) {
  const h = 0.01
  return {
    x: (loss(x + h, z) - loss(x - h, z)) / (2 * h),
    z: (loss(x, z + h) - loss(x, z - h)) / (2 * h),
  }
}

function makeSurface() {
  const size = 5
  const segments = 96
  const half = size / 2
  const vertices: number[] = []
  const colors: number[] = []
  const indices: number[] = []
  const color = new THREE.Color()

  for (let zi = 0; zi <= segments; zi += 1) {
    const z = -half + (zi / segments) * size
    for (let xi = 0; xi <= segments; xi += 1) {
      const x = -half + (xi / segments) * size
      const rawLoss = loss(x, z)
      const y = rawLoss * lossHeight
      vertices.push(x, y, z)

      const t = Math.min(1, Math.max(0, (rawLoss - 0.65) / 2.7))
      color.setHSL(0.58 - t * 0.44, 0.72, 0.56)
      colors.push(color.r, color.g, color.b)
    }
  }

  for (let zi = 0; zi < segments; zi += 1) {
    for (let xi = 0; xi < segments; xi += 1) {
      const a = zi * (segments + 1) + xi
      const b = a + 1
      const c = a + segments + 1
      const d = c + 1
      indices.push(a, c, b, b, c, d)
    }
  }

  const geometry = new THREE.BufferGeometry()
  geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3))
  geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3))
  geometry.setIndex(indices)
  geometry.computeVertexNormals()

  const material = new THREE.MeshStandardMaterial({
    side: THREE.DoubleSide,
    vertexColors: true,
    roughness: 0.72,
    metalness: 0.04,
    transparent: true,
    opacity: 0.92,
  })

  return new THREE.Mesh(geometry, material)
}

function makePath() {
  let x = -2.15
  let z = 1.45
  const points: THREE.Vector3[] = []

  for (let i = 0; i < 30; i += 1) {
    points.push(new THREE.Vector3(x, loss(x, z) * lossHeight + 0.12, z))
    const g = gradient(x, z)
    const length = Math.hypot(g.x, g.z) || 1
    const step = 0.12
    x -= step * (g.x / length)
    z -= step * (g.z / length)
  }

  return points
}

function makeCircleTexture(inner: string, outer: string) {
  const canvas = document.createElement('canvas')
  canvas.width = 96
  canvas.height = 96
  const context = canvas.getContext('2d')
  if (!context) return undefined

  const gradient = context.createRadialGradient(36, 32, 8, 48, 48, 44)
  gradient.addColorStop(0, inner)
  gradient.addColorStop(0.42, inner)
  gradient.addColorStop(1, outer)
  context.fillStyle = gradient
  context.beginPath()
  context.arc(48, 48, 44, 0, Math.PI * 2)
  context.fill()

  const texture = new THREE.CanvasTexture(canvas)
  texture.needsUpdate = true
  return texture
}

onMounted(() => {
  const el = mountEl.value
  if (!el) return

  try {
    renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false })
  } catch (err) {
    console.warn('[GradientDescent3D] WebGL unavailable:', err)
    failed.value = true
    return
  }

  const scene = new THREE.Scene()
  scene.background = new THREE.Color(0xf8fafc)

  const camera = new THREE.PerspectiveCamera(39, 1, 0.1, 100)
  camera.position.set(5.1, 4.2, 6.4)
  camera.lookAt(0, 1.0, 0)

  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.setSize(Math.max(1, el.clientWidth), Math.max(1, el.clientHeight))
  el.appendChild(renderer.domElement)

  scene.add(new THREE.HemisphereLight(0xffffff, 0xdbeafe, 1.8))
  const keyLight = new THREE.DirectionalLight(0xffffff, 1.6)
  keyLight.position.set(3, 5, 4)
  scene.add(keyLight)

  const group = new THREE.Group()
  group.rotation.y = -0.18
  scene.add(group)

  const surface = makeSurface()
  group.add(surface)

  const wire = new THREE.LineSegments(
    new THREE.WireframeGeometry(surface.geometry),
    new THREE.LineBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.2 }),
  )
  group.add(wire)

  const pathPoints = makePath()
  const pathCurve = new THREE.CatmullRomCurve3(pathPoints)
  const pathLine = new THREE.Line(
    new THREE.BufferGeometry().setFromPoints(pathCurve.getPoints(36)),
    new THREE.LineBasicMaterial({
      color: 0x8b5cf6,
      transparent: true,
      opacity: 0.7,
      depthTest: false,
    }),
  )
  pathLine.renderOrder = 3
  group.add(pathLine)

  const marker = new THREE.Sprite(
    new THREE.SpriteMaterial({
      map: makeCircleTexture('#7c3aed', 'rgba(124, 58, 237, 0)'),
      depthTest: false,
      depthWrite: false,
      transparent: true,
    }),
  )
  marker.scale.set(0.34, 0.34, 1)
  marker.renderOrder = 4
  group.add(marker)

  for (let i = 0; i <= 5; i += 1) {
    const isStart = i === 0
    const isEnd = i === 5
    const bead = new THREE.Mesh(
      new THREE.SphereGeometry(isStart || isEnd ? 0.055 : 0.04, 16, 10),
      new THREE.MeshBasicMaterial({
        color: isStart ? 0xa78bfa : 0x8b5cf6,
        transparent: true,
        opacity: isEnd ? 0 : 0.72,
        depthTest: false,
        depthWrite: false,
      }),
    )
    bead.position.copy(pathCurve.getPointAt(i / 5))
    bead.renderOrder = 3
    group.add(bead)
  }

  const markerHalo = new THREE.Sprite(
    new THREE.SpriteMaterial({
      map: makeCircleTexture('rgba(196, 181, 253, 0.6)', 'rgba(196, 181, 253, 0)'),
      transparent: true,
      depthTest: false,
      depthWrite: false,
    }),
  )
  markerHalo.scale.set(0.38, 0.38, 1)
  markerHalo.renderOrder = 5
  group.add(markerHalo)

  const floorGrid = new THREE.GridHelper(5.4, 8, 0xcbd5e1, 0xe2e8f0)
  floorGrid.position.y = -0.02
  group.add(floorGrid)

  function resize() {
    if (!renderer || !mountEl.value) return
    const { clientWidth, clientHeight } = mountEl.value
    if (clientWidth === 0 || clientHeight === 0) return
    camera.aspect = clientWidth / clientHeight
    camera.updateProjectionMatrix()
    renderer.setSize(clientWidth, clientHeight)
  }

  resizeObserver = new ResizeObserver(resize)
  resizeObserver.observe(el)
  resize()

  if (typeof IntersectionObserver !== 'undefined') {
    intersectionObserver = new IntersectionObserver((entries) => {
      for (const entry of entries) {
        if (entry.isIntersecting) resize()
      }
    })
    intersectionObserver.observe(el)
  }

  const animate = (time: number) => {
    try {
      const t = (time * 0.00012) % 1
      marker.position.copy(pathCurve.getPointAt(t))
      marker.position.y += 0.03 + Math.sin(time * 0.006) * 0.025
      markerHalo.position.copy(marker.position)
      markerHalo.position.y += 0.01
      group.rotation.y = -0.2 + Math.sin(time * 0.00035) * 0.16
      if (renderer && el.clientWidth > 0 && el.clientHeight > 0) {
        const canvas = renderer.domElement
        const dpr = renderer.getPixelRatio()
        const targetW = Math.floor(el.clientWidth * dpr)
        const targetH = Math.floor(el.clientHeight * dpr)
        if (canvas.width !== targetW || canvas.height !== targetH) {
          resize()
        }
        renderer.render(scene, camera)
      }
      frameId = requestAnimationFrame(animate)
    } catch (err) {
      console.warn('[GradientDescent3D] render failed:', err)
      failed.value = true
    }
  }

  const canvasEl = renderer.domElement
  contextLostHandler = (event: Event) => {
    event.preventDefault()
    cancelAnimationFrame(frameId)
  }
  contextRestoredHandler = () => {
    resize()
    frameId = requestAnimationFrame(animate)
  }
  canvasEl.addEventListener('webglcontextlost', contextLostHandler as EventListener)
  canvasEl.addEventListener('webglcontextrestored', contextRestoredHandler)

  frameId = requestAnimationFrame(animate)
})

onBeforeUnmount(() => {
  cancelAnimationFrame(frameId)
  resizeObserver?.disconnect()
  intersectionObserver?.disconnect()
  if (renderer && contextLostHandler) {
    renderer.domElement.removeEventListener('webglcontextlost', contextLostHandler as EventListener)
  }
  if (renderer && contextRestoredHandler) {
    renderer.domElement.removeEventListener('webglcontextrestored', contextRestoredHandler)
  }
  renderer?.dispose()
  renderer?.domElement.remove()
})
</script>

<style scoped>
.gd3d {
  position: relative;
  height: 238px;
  overflow: hidden;
  border: 1px solid rgba(148, 163, 184, 0.32);
  border-radius: 6px;
  background: #f8fafc;
}

.gd3d-canvas {
  width: 100%;
  height: 100%;
}

.gd3d-canvas canvas {
  width: 100% !important;
  height: 100% !important;
  display: block;
}

.gd3d-label {
  position: absolute;
  padding: 3px 7px;
  border-radius: 4px;
  background: rgba(255, 255, 255, 0.82);
  color: #334155;
  font-size: 11px;
  line-height: 1.25;
  box-shadow: 0 1px 8px rgba(15, 23, 42, 0.08);
  pointer-events: none;
}

.gd3d-label.top {
  left: 14px;
  top: 12px;
}

.gd3d-label.bottom {
  right: 18px;
  bottom: 14px;
}

.gd3d-axes {
  position: absolute;
  left: 6px;
  bottom: 4px;
  width: 130px;
  height: 80px;
  pointer-events: none;
  overflow: visible;
}

.gd3d-axis-label {
  font-family: ui-sans-serif, system-ui, -apple-system, "Segoe UI", Roboto, "PingFang SC", "Noto Sans CJK SC", "Microsoft YaHei", sans-serif;
  font-size: 8px;
  font-weight: 600;
  fill: #334155;
  paint-order: stroke;
  stroke: rgba(248, 250, 252, 0.95);
  stroke-width: 2.5px;
  stroke-linejoin: round;
}

.gd3d-axis-label-orange {
  fill: #c2410c;
}

.gd3d-fallback {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 12px 20px;
  color: #64748b;
  font-size: 13px;
  text-align: center;
  background: rgba(248, 250, 252, 0.7);
}

</style>
