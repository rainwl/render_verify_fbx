<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'
import { KTX2Loader } from 'three/examples/jsm/loaders/KTX2Loader.js'

const viewerRef = ref<HTMLDivElement | null>(null)
const loading = ref(true)
const progress = ref(0)
const errorMessage = ref('')
type MetricItem = {
  durationMs: number | null
  sizeBytes: number | null
}
const loadMetrics = ref<{ total: MetricItem; model: MetricItem; textures: MetricItem }>({
  total: { durationMs: null, sizeBytes: null },
  model: { durationMs: null, sizeBytes: null },
  textures: { durationMs: null, sizeBytes: null }
})

const bootstrapEpoch =
  typeof performance !== 'undefined' && typeof performance.timeOrigin === 'number'
    ? performance.timeOrigin
    : Date.now()
const nowRelative = () => {
  if (typeof performance !== 'undefined' && typeof performance.now === 'function') {
    return performance.now()
  }
  return Date.now() - bootstrapEpoch
}

const textureBasePath = 'https://cdn.rainnnn.com/ktx/v3'
const modelUrl = 'https://cdn.rainnnn.com/glb/model.glb'
const envMapPath = 'https://cdn.rainnnn.com/hdr/studio_small_08_1k.hdr'

type TextureChannel = 'BaseColor' | 'Normal' | 'ORM'
type TextureColorSpace = 'srgb' | 'linear' | 'none'
type TextureMetaItem = {
  suffix: TextureChannel
  targets: Array<'map' | 'normalMap' | 'roughnessMap' | 'metalnessMap' | 'aoMap'>
  colorSpace?: TextureColorSpace
}

const normalizeKey = (value: string) => value.replace(/[^a-z0-9_]/gi, '').toLowerCase()

const textureMeta: TextureMetaItem[] = [
  { suffix: 'BaseColor', targets: ['map'], colorSpace: 'srgb' },
  { suffix: 'Normal', targets: ['normalMap'], colorSpace: 'none' },
  { suffix: 'ORM', targets: ['aoMap', 'roughnessMap', 'metalnessMap'], colorSpace: 'none' }
]

const extractVariantName = (value?: string | null) => {
  if (!value) return null
  const match = value.match(/mat[a-z0-9]+/i)
  return match ? match[0] : null
}

const sanitizeName = (value: string) =>
  normalizeKey(value.replace(/\s+/g, '_').replace(/_low\b/g, ''))

const extractPartKeyFromCandidate = (value?: string | null) => {
  if (!value) return null
  const sanitized = sanitizeName(value)
  const tokens = sanitized.split('_').filter(Boolean)
  const index = tokens.findIndex((t) => /^\d+$/.test(t))
  if (index === -1) return null
  const collected: string[] = []
  for (let i = index; i < tokens.length; i++) {
    const token = tokens[i]
    if (!token) continue
    if (token.startsWith('mat')) break
    collected.push(token)
  }
  if (collected.length >= 2) {
    // 去掉末尾编号（例如 4_screw_1 -> 4_screw）
    const joined = collected.join('_')
    return joined.replace(/_\d+$/, '')
  }
  return null
}

const findPartKeyFromObject = (
  object?: THREE.Object3D | null,
  materialName?: string | null
): string | null => {
  const candidates: (string | undefined | null)[] = []
  if (object?.name) candidates.push(object.name)
  if (materialName) candidates.push(materialName)
  let current = object?.parent
  while (current) {
    if (current.name) candidates.push(current.name)
    current = current.parent
  }
  for (const candidate of candidates) {
    const part = extractPartKeyFromCandidate(candidate)
    if (part) return part
  }
  return null
}

const variantFileNameMap: Record<string, string> = {
  matmetal: 'MatMetal',
  matpaint: 'MatPaint',
  matrubber: 'matRubber'
}
const resolveVariantKey = (variant?: string | null) => {
  const normalized = normalizeKey(variant ?? '')
  return normalized || 'matmetal'
}
const buildSharedTextureFileName = (variantKey: string, suffix: TextureChannel): string => {
  const variantName = variantFileNameMap[variantKey] ?? variantKey
  return `model_${variantName}_${suffix}.ktx2`
}

let renderer: THREE.WebGLRenderer | null = null
let scene: THREE.Scene | null = null
let camera: THREE.PerspectiveCamera | null = null
let controls: OrbitControls | null = null
let pmremGenerator: THREE.PMREMGenerator | null = null
let animationFrame = 0
let resizeObserver: ResizeObserver | null = null
let handleResize: (() => void) | null = null
let loadingManager: THREE.LoadingManager | null = null
let textureLoader: THREE.TextureLoader | null = null
let ktx2Loader: KTX2Loader | null = null

const textureCache = new Map<string, THREE.Texture>()
const assetSizeCache = new Map<string, number>()
const recordedTextureAssets = new Set<string>()
let textureLoadsInFlight = 0
let textureLoadStart: number | null = null
let modelLoadStart: number | null = null
let modelSizeRecorded = false
let hasRenderedFirstFrame = false
const enableSizeProbeFallback = false
const showDebugPanel = ref(false)
let glbRoot: THREE.Object3D | null = null
const originalPositions = new Map<THREE.Object3D, THREE.Vector3>()
const exploded = ref(false)
type TextureDebugEntry = {
  path: string
  status: 'success' | 'fail'
  part?: string | null
  variant?: string | null
}
const textureDebug = ref<TextureDebugEntry[]>([])
const pushTextureDebug = (entry: TextureDebugEntry) => {
  textureDebug.value = [...textureDebug.value.slice(-50), entry]
}
const resetLoadTracking = () => {
  loadMetrics.value = {
    total: { durationMs: null, sizeBytes: null },
    model: { durationMs: null, sizeBytes: null },
    textures: { durationMs: null, sizeBytes: null }
  }
  textureLoadsInFlight = 0
  textureLoadStart = null
  modelLoadStart = null
  modelSizeRecorded = false
  recordedTextureAssets.clear()
  hasRenderedFirstFrame = false
}

const updateTotalSizeMetric = () => {
  const modelBytes = loadMetrics.value.model.sizeBytes ?? 0
  const textureBytes = loadMetrics.value.textures.sizeBytes ?? 0
  const combined = modelBytes + textureBytes
  loadMetrics.value.total.sizeBytes = combined > 0 ? combined : null
}

const markFirstFrameRendered = () => {
  if (hasRenderedFirstFrame || loading.value) {
    return
  }
  hasRenderedFirstFrame = true
  loadMetrics.value.total.durationMs = nowRelative()
  updateTotalSizeMetric()
}

const toAbsoluteAssetUrl = (path: string) => {
  if (typeof window === 'undefined') {
    return path
  }
  return new URL(path, window.location.origin).href
}

const readPerformanceTransferSize = (absoluteUrl: string): number | null => {
  if (typeof performance === 'undefined' || typeof performance.getEntriesByName !== 'function') {
    return null
  }
  const entries = performance.getEntriesByName(absoluteUrl) as PerformanceResourceTiming[]
  const resource = entries[entries.length - 1]
  if (resource) {
    if (resource.transferSize && resource.transferSize > 0) {
      return resource.transferSize
    }
    // 当 transferSize 为 0（缓存命中）时，不使用 encoded/decoded 体积，改由 HEAD 结果获取准确文件大小
  }
  return null
}

const fetchContentLength = async (absoluteUrl: string): Promise<number | null> => {
  if (typeof fetch !== 'function') {
    return null
  }
  try {
    const response = await fetch(absoluteUrl, { method: 'HEAD', cache: 'no-store' })
    if (response.ok) {
      const headerValue = response.headers.get('content-length')
      if (headerValue) {
        return Number(headerValue)
      }
    }
  } catch (error) {
    console.warn('HEAD request failed while probing asset size', error)
  }
  try {
    const rangeResponse = await fetch(absoluteUrl, {
      method: 'GET',
      headers: { Range: 'bytes=0-0' },
      cache: 'no-store'
    })
    if (rangeResponse.ok) {
      const rangeHeader = rangeResponse.headers.get('content-range')
      if (rangeHeader) {
        const match = rangeHeader.match(/\/(\d+)$/)
        if (match) {
          await rangeResponse.body?.cancel?.()
          return Number(match[1])
        }
      }
    }
  } catch (error) {
    console.warn('Range request failed while probing asset size', error)
  }
  return null
}

const recordAssetSize = async (type: 'model' | 'texture', path: string) => {
  const absoluteUrl = toAbsoluteAssetUrl(path)
  if (type === 'texture' && recordedTextureAssets.has(absoluteUrl)) {
    return
  }
  if (type === 'model' && modelSizeRecorded) {
    return
  }
  const knownSize =
    assetSizeCache.get(absoluteUrl) ??
    readPerformanceTransferSize(absoluteUrl) ??
    (enableSizeProbeFallback ? await fetchContentLength(absoluteUrl) : null)
  if (typeof knownSize === 'number' && knownSize > 0) {
    assetSizeCache.set(absoluteUrl, knownSize)
    if (type === 'model') {
      loadMetrics.value.model.sizeBytes = knownSize
      modelSizeRecorded = true
    } else {
      recordedTextureAssets.add(absoluteUrl)
      const existing = loadMetrics.value.textures.sizeBytes ?? 0
      loadMetrics.value.textures.sizeBytes = existing + knownSize
    }
    updateTotalSizeMetric()
  }
}

const markModelLoadStart = () => {
  modelLoadStart = nowRelative()
  loadMetrics.value.model.durationMs = null
}

const markModelLoadEnd = (path: string) => {
  if (modelLoadStart !== null) {
    loadMetrics.value.model.durationMs = nowRelative() - modelLoadStart
  }
  modelLoadStart = null
  void recordAssetSize('model', path)
}

const markTextureLoadStart = () => {
  if (textureLoadStart === null) {
    textureLoadStart = nowRelative()
    loadMetrics.value.textures.durationMs = null
  }
  textureLoadsInFlight += 1
}

const markTextureLoadEnd = (path: string) => {
  textureLoadsInFlight = Math.max(0, textureLoadsInFlight - 1)
  if (textureLoadsInFlight === 0 && textureLoadStart !== null) {
    loadMetrics.value.textures.durationMs = nowRelative() - textureLoadStart
    textureLoadStart = null
  }
  void recordAssetSize('texture', path)
}

const formatDuration = (value: number | null) => {
  if (value === null) return '--'
  if (value < 1000) {
    return value.toFixed(0) + ' ms'
  }
  return (value / 1000).toFixed(2) + ' s'
}

const formatBytes = (value: number | null) => {
  if (value === null) return '--'
  const units = ['B', 'KB', 'MB', 'GB']
  let size = value
  let unitIndex = 0
  while (size >= 1024 && unitIndex < units.length - 1) {
    size /= 1024
    unitIndex += 1
  }
  const precision = unitIndex === 0 ? 0 : 2
  return `${size.toFixed(precision)} ${units[unitIndex]}`
}

const isCriticalAsset = (url: string) => {
  const normalized = url.toLowerCase()
  return (
    normalized.includes('gearpump20251111.fbx') || normalized.includes('studio_small_08_1k.hdr')
  )
}

const toggleExplode = (state: boolean) => {
  if (!glbRoot) return
  const box = new THREE.Box3().setFromObject(glbRoot)
  const center = box.getCenter(new THREE.Vector3())
  const magnitude = box.getSize(new THREE.Vector3()).length() * 0.08
  glbRoot.traverse((child: THREE.Object3D) => {
    if (!(child as THREE.Mesh).isMesh) return
    if (!originalPositions.has(child)) {
      originalPositions.set(child, child.position.clone())
    }
    const original = originalPositions.get(child)
    if (!original) return
    if (!state) {
      child.position.copy(original)
      return
    }
    const dir = child.position.clone().sub(center)
    if (dir.lengthSq() === 0) {
      dir.set(Math.random() * 0.5 + 0.1, Math.random() * 0.5 + 0.1, Math.random() * 0.5 + 0.1)
    }
    dir.normalize().multiplyScalar(magnitude)
    child.position.copy(original.clone().add(dir))
  })
  exploded.value = state
}

const ensureRendererSize = () => {
  if (!viewerRef.value || !renderer || !camera) return
  const { clientWidth, clientHeight } = viewerRef.value
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.setSize(clientWidth, clientHeight)
  camera.aspect = clientWidth / Math.max(clientHeight, 1)
  camera.updateProjectionMatrix()
}

const startResizeHandling = () => {
  if (!viewerRef.value) return
  handleResize = () => ensureRendererSize()
  window.addEventListener('resize', handleResize)
  resizeObserver = new ResizeObserver(() => ensureRendererSize())
  resizeObserver.observe(viewerRef.value)
}

const disposeResources = () => {
  cancelAnimationFrame(animationFrame)
  window.removeEventListener('resize', handleResize || (() => undefined))
  resizeObserver?.disconnect()
  controls?.dispose()
  pmremGenerator?.dispose()
  ktx2Loader?.dispose?.()
  textureCache.forEach((texture) => texture.dispose())
  textureCache.clear()
  renderer?.dispose()
  scene = null
  camera = null
  controls = null
  pmremGenerator = null
  renderer = null
}

const resolveColorSpace = (type?: TextureColorSpace) => {
  if (type === 'srgb') return THREE.SRGBColorSpace
  if (type === 'linear') return THREE.LinearSRGBColorSpace
  return THREE.NoColorSpace
}

const loadTexture = (
  path: string,
  colorSpace: TextureColorSpace = 'linear',
  partKey?: string | null,
  variant?: string | null
) =>
  new Promise<THREE.Texture>((resolve, reject) => {
    if (textureCache.has(path)) {
      resolve(textureCache.get(path) as THREE.Texture)
      return
    }
    const isKtx2 = path.toLowerCase().endsWith('.ktx2')
    const loader = isKtx2 ? ktx2Loader : textureLoader
    if (!loader) {
      reject(new Error('Texture loader is not ready'))
      return
    }
    markTextureLoadStart()
    loader.load(
      path,
      (texture: THREE.Texture) => {
        texture.colorSpace = resolveColorSpace(colorSpace)
        texture.anisotropy = renderer?.capabilities.getMaxAnisotropy() ?? 1
        if (!isKtx2) {
          texture.flipY = false
        }
        textureCache.set(path, texture)
        markTextureLoadEnd(path)
        pushTextureDebug({ path, status: 'success', part: partKey, variant })
        resolve(texture)
      },
      undefined,
      (event: ErrorEvent | unknown) => {
        markTextureLoadEnd(path)
        pushTextureDebug({ path, status: 'fail', part: partKey, variant })
        reject(event instanceof ErrorEvent ? event.error : event)
      }
    )
  })

const buildMaterialForMesh = async (
  mesh: THREE.Mesh,
  materialName?: string
): Promise<THREE.MeshStandardMaterial | null> => {
  const partKey = findPartKeyFromObject(mesh, materialName)
  const variantRaw =
    extractVariantName(materialName) ?? extractVariantName(mesh.name) ?? extractVariantName(partKey)
  const variantKey = resolveVariantKey(variantRaw)
  const variantLabel = variantFileNameMap[variantKey] ?? variantKey

  const candidate = new THREE.MeshStandardMaterial({
    metalness: 1,
    roughness: 1,
    envMapIntensity: 1.2
  })
  // Flip Y to match baked normal/height orientation
  candidate.normalScale.set(1, -1)

  await Promise.all(
    textureMeta.map(async ({ suffix, targets, colorSpace }) => {
      const fileName = buildSharedTextureFileName(variantKey, suffix)
      const texturePath = `${textureBasePath}/${fileName}`
      try {
        const tex = await loadTexture(texturePath, colorSpace ?? 'linear', partKey, variantLabel)
        targets.forEach((target) => {
          ;(candidate as THREE.MeshStandardMaterial)[target] = tex
        })
        if (targets.includes('aoMap')) {
          candidate.aoMapIntensity = 1
        }
      } catch (error) {
        console.warn(`Texture load failed: ${texturePath}`, error)
      }
    })
  )

  return candidate
}

const ensureUv2ForAo = (mesh: THREE.Mesh) => {
  const geometry = mesh.geometry as THREE.BufferGeometry | undefined
  if (!geometry) return
  const uv = geometry.getAttribute('uv') as THREE.BufferAttribute | undefined
  const uv2 = geometry.getAttribute('uv2')
  if (uv && !uv2) {
    geometry.setAttribute('uv2', uv.clone())
  }
}

const applyMaterialsToMesh = async (
  mesh: THREE.Mesh<THREE.BufferGeometry, THREE.Material | THREE.Material[]>
) => {
  ensureUv2ForAo(mesh)
  mesh.castShadow = true
  mesh.receiveShadow = true
  if (Array.isArray(mesh.material)) {
    const updated = await Promise.all(
      mesh.material.map(async (existing: THREE.Material) => {
        const created = await buildMaterialForMesh(mesh, existing?.name)
        return created ?? existing
      })
    )
    mesh.material = updated
  } else {
    const created = await buildMaterialForMesh(mesh, mesh.material?.name)
    if (created) {
      mesh.material = created
    } else if (!(mesh.material instanceof THREE.MeshStandardMaterial)) {
      mesh.material = new THREE.MeshStandardMaterial({
        color: 0x777777,
        metalness: 0.5,
        roughness: 0.6
      })
    }
  }
}

const loadEnvironmentMap = () =>
  new Promise<THREE.Texture>((resolve, reject) => {
    if (!renderer || !pmremGenerator || !loadingManager) {
      reject(new Error('Renderer not ready for environment map'))
      return
    }
    const rgbeLoader = new RGBELoader(loadingManager)
    rgbeLoader.load(
      envMapPath,
      (hdrTexture: THREE.DataTexture) => {
        const envTexture = pmremGenerator!.fromEquirectangular(hdrTexture).texture
        hdrTexture.dispose()
        resolve(envTexture)
      },
      undefined,
      reject
    )
  })

const loadGlb = () =>
  new Promise<THREE.Group>((resolve, reject) => {
    if (!loadingManager) {
      reject(new Error('Loading manager is not initialized'))
      return
    }
    const loader = new GLTFLoader(loadingManager)
    const modelPath = modelUrl
    markModelLoadStart()
    loader.load(
      modelPath,
      async (gltf) => {
        const sceneObj = gltf.scene
        const applyTasks: Promise<void>[] = []
        sceneObj.traverse((child: THREE.Object3D) => {
          if ((child as THREE.Mesh).isMesh) {
            applyTasks.push(applyMaterialsToMesh(child as THREE.Mesh))
          }
        })
        try {
          await Promise.all(applyTasks)
          markModelLoadEnd(modelPath)
          resolve(sceneObj)
        } catch (error) {
          reject(error)
        }
      },
      undefined,
      reject
    )
  })

const fitCameraToObject = (target: THREE.Object3D) => {
  if (!camera || !controls) return
  const boundingBox = new THREE.Box3().setFromObject(target)
  const size = boundingBox.getSize(new THREE.Vector3())
  const center = boundingBox.getCenter(new THREE.Vector3())
  const maxDim = Math.max(size.x, size.y, size.z)
  const fitDistance = maxDim / (2 * Math.tan((Math.PI * camera.fov) / 360))
  const direction = new THREE.Vector3(1.2, 0.8, 1.4).normalize()
  camera.position.copy(direction.multiplyScalar(fitDistance * 2)).add(center)
  controls.target.copy(center)
  controls.update()
}

const animate = () => {
  if (!renderer || !scene || !camera) return
  animationFrame = requestAnimationFrame(animate)
  controls?.update()
  renderer.render(scene, camera)
  markFirstFrameRendered()
}

const initScene = async () => {
  if (!viewerRef.value) {
    throw new Error('Viewer container is not mounted')
  }

  resetLoadTracking()
  loadingManager = new THREE.LoadingManager()
  loadingManager.onProgress = (_url: string, loaded: number, total: number) => {
    progress.value = Math.round((loaded / total) * 100)
  }
  loadingManager.onLoad = () => {
    loading.value = false
  }
  loadingManager.onError = (url: string) => {
    if (isCriticalAsset(url)) {
      errorMessage.value = `Critical asset failed: ${url}`
      loading.value = false
    } else {
      console.warn(`Non-critical asset failed: ${url}`)
    }
  }

  textureLoader = new THREE.TextureLoader(loadingManager)
  textureLoader.setCrossOrigin('anonymous')

  renderer = new THREE.WebGLRenderer({
    antialias: true,
    alpha: true
  })
  renderer.outputColorSpace = THREE.SRGBColorSpace
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.15

  ktx2Loader = new KTX2Loader(loadingManager)
    .setTranscoderPath('https://cdn.jsdelivr.net/npm/three@0.159/examples/jsm/libs/basis/')
    .detectSupport(renderer)
  ktx2Loader.setCrossOrigin?.('anonymous')

  const container = viewerRef.value
  container.appendChild(renderer.domElement)

  pmremGenerator = new THREE.PMREMGenerator(renderer)
  pmremGenerator.compileEquirectangularShader()

  scene = new THREE.Scene()
  // Light gray white background for a clean render backdrop
  scene.background = new THREE.Color(0xf7f7f7)

  camera = new THREE.PerspectiveCamera(45, 1, 0.1, 500)
  camera.position.set(4, 2, 6)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05
  controls.minDistance = 0.6
  controls.maxDistance = 30
  controls.maxPolarAngle = Math.PI * 0.48

  const hemi = new THREE.HemisphereLight(0xffffff, 0x1a1a33, 0.5)
  scene.add(hemi)

  const dirLight = new THREE.DirectionalLight(0xffffff, 1.2)
  dirLight.position.set(8, 10, 6)
  dirLight.castShadow = true
  dirLight.shadow.mapSize.set(2048, 2048)
  dirLight.shadow.bias = -0.0005
  scene.add(dirLight)

  const fillLight = new THREE.SpotLight(0x88aaff, 0.7, 40, Math.PI / 6, 0.3, 2)
  fillLight.position.set(-6, 5, -4)
  scene.add(fillLight)

  ensureRendererSize()
  startResizeHandling()

  const envMap = await loadEnvironmentMap()
  scene.environment = envMap

  const glbModel = await loadGlb()
  glbRoot = glbModel
  // Load model with original orientation
  scene.add(glbModel)
  fitCameraToObject(glbModel)

  animate()
}

onMounted(() => {
  initScene().catch((error) => {
    loading.value = false
    errorMessage.value = error instanceof Error ? error.message : 'Unknown error, check console'
    console.error(error)
  })
})

onBeforeUnmount(() => {
  disposeResources()
})
</script>

<template>
  <div class="viewer" ref="viewerRef">
    <div class="ui-buttons">
      <button class="action-btn" @click="toggleExplode(!exploded)">爆炸视图</button>
    </div>
    <div class="overlay" v-if="loading">
      <p>Loading assets...</p>
      <p>{{ progress }}%</p>
    </div>
    <div class="overlay error" v-else-if="errorMessage">
      <p>{{ errorMessage }}</p>
    </div>
    <div class="metrics-panel">
      <div class="metric-block highlight">
        <p class="metric-title">Total</p>
        <p>Time: {{ formatDuration(loadMetrics.total.durationMs) }}</p>
        <p>Size: {{ formatBytes(loadMetrics.total.sizeBytes) }}</p>
      </div>
      <div class="metric-block">
        <p class="metric-title">Model</p>
        <p>Time: {{ formatDuration(loadMetrics.model.durationMs) }}</p>
        <p>Size: {{ formatBytes(loadMetrics.model.sizeBytes) }}</p>
      </div>
      <div class="metric-block">
        <p class="metric-title">Textures</p>
        <p>Time: {{ formatDuration(loadMetrics.textures.durationMs) }}</p>
        <p>Size: {{ formatBytes(loadMetrics.textures.sizeBytes) }}</p>
      </div>
    </div>
    <div class="debug-panel" v-if="showDebugPanel && textureDebug.length">
      <p class="metric-title">Texture Debug (latest {{ textureDebug.length }})</p>
      <ul>
        <li v-for="(item, idx) in textureDebug" :key="idx">
          <span :class="item.status">{{ item.status }}</span>
          <span>{{ item.part || 'part?' }}</span>
          <span>{{ item.variant || 'variant?' }}</span>
          <span class="path">{{ item.path }}</span>
        </li>
      </ul>
    </div>
  </div>
</template>

<style scoped>
.viewer {
  position: relative;
  width: 100%;
  height: 100%;
  min-height: 100vh;
  background: radial-gradient(circle at 40% 30%, #1b1f2d, #050509 60%);
  overflow: hidden;
}

canvas {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  display: block;
}

.overlay {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  background: rgba(5, 5, 9, 0.65);
  color: #e2e8ff;
  font-size: 1rem;
  letter-spacing: 0.1em;
  text-transform: uppercase;
}

.overlay.error {
  background: rgba(78, 11, 11, 0.8);
  color: #ffb4b4;
}

.metrics-panel {
  position: absolute;
  left: 1.5rem;
  bottom: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  padding: 1rem 1.25rem;
  border-radius: 0.75rem;
  background: rgba(6, 7, 16, 0.78);
  color: #dfe3ff;
  font-size: 0.85rem;
  line-height: 1.4;
  backdrop-filter: blur(8px);
  border: 1px solid rgba(143, 171, 255, 0.15);
  pointer-events: none;
  z-index: 2;
}

.metric-block {
  display: flex;
  flex-direction: column;
  gap: 0.1rem;
}

.metric-title {
  font-size: 0.75rem;
  letter-spacing: 0.2em;
  color: #8aa4ff;
}

.ui-buttons {
  position: absolute;
  top: 1rem;
  right: 1rem;
  display: flex;
  gap: 0.5rem;
  z-index: 4;
}
.action-btn {
  padding: 0.45rem 0.85rem;
  border-radius: 0.5rem;
  border: 1px solid rgba(0, 0, 0, 0.08);
  background: rgba(30, 35, 50, 0.88);
  color: #dfe3ff;
  cursor: pointer;
  transition: all 0.2s ease;
}
.action-btn:hover {
  background: rgba(60, 70, 95, 0.95);
}

.debug-panel {
  position: absolute;
  right: 1rem;
  bottom: 1rem;
  max-width: 420px;
  max-height: 50vh;
  overflow: auto;
  background: rgba(0, 0, 0, 0.65);
  color: #dfe3ff;
  padding: 0.75rem;
  border-radius: 0.5rem;
  font-size: 0.78rem;
  line-height: 1.35;
  border: 1px solid rgba(143, 171, 255, 0.25);
  z-index: 3;
}
.debug-panel ul {
  list-style: none;
  margin: 0.25rem 0 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}
.debug-panel li {
  display: grid;
  grid-template-columns: 64px 110px 110px 1fr;
  gap: 0.35rem;
  align-items: center;
}
.debug-panel .success {
  color: #7de38d;
}
.debug-panel .fail {
  color: #ff7b7b;
}
.debug-panel .path {
  color: #cfd8ff;
  word-break: break-all;
}
</style>



