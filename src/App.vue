<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { FBXLoader } from 'three/examples/jsm/loaders/FBXLoader.js'
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'
import textureManifest from './assets/model-texture-manifest.json'

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

const textureBasePath = 'https://cdn.rainnnn.com/textures'
const modelUrl = 'https://cdn.rainnnn.com/models/gearpump20251111.fbx'
const envMapPath = 'https://cdn.rainnnn.com/hdr/studio_small_08_1k.hdr'

type TextureChannel = 'BaseColor' | 'Normal' | 'ORM'
type TextureManifest = Record<
  string,
  Record<string, Partial<Record<TextureChannel, string>>>
>
type TextureMetaItem = {
  suffix: TextureChannel
  targets: Array<'map' | 'normalMap' | 'roughnessMap' | 'metalnessMap' | 'aoMap'>
  srgb?: boolean
}

const manifestData = textureManifest as TextureManifest
const normalizeKey = (value: string) => value.replace(/[^a-z0-9_]/gi, '').toLowerCase()
const manifestPartMeta = Object.keys(manifestData).map((key) => ({
  original: key,
  normalized: normalizeKey(key)
}))

const textureMeta: TextureMetaItem[] = [
  { suffix: 'BaseColor', targets: ['map'], srgb: true },
  { suffix: 'Normal', targets: ['normalMap'] },
  { suffix: 'ORM', targets: ['aoMap', 'roughnessMap', 'metalnessMap'] }
]

const findPartKeyFromObject = (object?: THREE.Object3D | null): string | null => {
  let current: THREE.Object3D | null | undefined = object
  while (current) {
    const name = current.name?.trim()
    if (name) {
      const normalizedName = normalizeKey(name)
      const match = manifestPartMeta.find(
        ({ normalized }) =>
          normalizedName.startsWith(normalized) || normalizedName.includes(normalized)
      )
      if (match) {
        return match.original
      }
    }
    current = current.parent
  }
  return null
}

const extractVariantName = (value?: string | null) => {
  if (!value) return null
  const match = value.match(/mat[a-z0-9]+/i)
  return match ? match[0] : null
}

const resolveManifestInfo = (
  mesh: THREE.Mesh,
  materialName?: string
): { partKey: string; textures: Partial<Record<TextureChannel, string>> } | null => {
  const partKey = findPartKeyFromObject(mesh)
  if (!partKey) return null
  const partEntry = manifestData[partKey]
  if (!partEntry) return null
  const variantName =
    extractVariantName(materialName) ?? extractVariantName(mesh.name) ?? null
  const variantKey =
    (variantName &&
      Object.keys(partEntry).find(
        (key) => normalizeKey(key) === normalizeKey(variantName)
      )) ||
    Object.keys(partEntry)[0]
  if (!variantKey) return null
  const textures = partEntry[variantKey]
  if (!textures) return null
  return {
    partKey,
    textures
  }
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

const textureCache = new Map<string, THREE.Texture>()
const assetSizeCache = new Map<string, number>()
const recordedTextureAssets = new Set<string>()
let textureLoadsInFlight = 0
let textureLoadStart: number | null = null
let modelLoadStart: number | null = null
let modelSizeRecorded = false
let hasRenderedFirstFrame = false

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
    (await fetchContentLength(absoluteUrl))
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
  textureCache.forEach((texture) => texture.dispose())
  textureCache.clear()
  renderer?.dispose()
  scene = null
  camera = null
  controls = null
  pmremGenerator = null
  renderer = null
}

const loadTexture = (path: string, srgb = false) =>
  new Promise<THREE.Texture>((resolve, reject) => {
    if (textureCache.has(path)) {
      resolve(textureCache.get(path) as THREE.Texture)
      return
    }
    const loader = textureLoader
    if (!loader) {
      reject(new Error('Texture loader is not ready'))
      return
    }
    markTextureLoadStart()
    loader.load(
      path,
      (texture: THREE.Texture) => {
        texture.colorSpace = srgb ? THREE.SRGBColorSpace : THREE.LinearSRGBColorSpace
        texture.anisotropy = renderer?.capabilities.getMaxAnisotropy() ?? 1
        textureCache.set(path, texture)
        markTextureLoadEnd(path)
        resolve(texture)
      },
      undefined,
      (event: ErrorEvent | unknown) => {
        markTextureLoadEnd(path)
        reject(event instanceof ErrorEvent ? event.error : event)
      }
    )
  })

const buildMaterialForMesh = async (
  mesh: THREE.Mesh,
  materialName?: string
): Promise<THREE.MeshStandardMaterial | null> => {
  const manifestInfo = resolveManifestInfo(mesh, materialName)
  if (!manifestInfo) return null

  const candidate = new THREE.MeshStandardMaterial({
    metalness: 1,
    roughness: 1,
    envMapIntensity: 1.2
  })

  await Promise.all(
    textureMeta.map(async ({ suffix, targets, srgb }) => {
      const fileName = manifestInfo.textures[suffix]
      if (!fileName) return
      const texturePath = `${textureBasePath}/${manifestInfo.partKey}/${fileName}`
      try {
        const tex = await loadTexture(texturePath, srgb ?? false)
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

const loadFbx = () =>
  new Promise<THREE.Group>((resolve, reject) => {
    if (!loadingManager) {
      reject(new Error('Loading manager is not initialized'))
      return
    }
    const loader = new FBXLoader(loadingManager)
    const modelPath = modelUrl
    markModelLoadStart()
    loader.load(
      modelPath,
      async (fbx: THREE.Group) => {
        const applyTasks: Promise<void>[] = []
        fbx.traverse((child: THREE.Object3D) => {
          if ((child as THREE.Mesh).isMesh) {
            applyTasks.push(applyMaterialsToMesh(child as THREE.Mesh))
          }
        })
        try {
          await Promise.all(applyTasks)
          markModelLoadEnd(modelPath)
          resolve(fbx)
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

  renderer = new THREE.WebGLRenderer({
    antialias: true,
    alpha: true
  })
  renderer.outputColorSpace = THREE.SRGBColorSpace
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.15

  const container = viewerRef.value
  container.appendChild(renderer.domElement)

  pmremGenerator = new THREE.PMREMGenerator(renderer)
  pmremGenerator.compileEquirectangularShader()

  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x050509)

  camera = new THREE.PerspectiveCamera(45, 1, 0.1, 500)
  camera.position.set(4, 2, 6)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05
  controls.minDistance = 2
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

  const ground = new THREE.Mesh(
    new THREE.CircleGeometry(30, 64).rotateX(-Math.PI / 2),
    new THREE.MeshStandardMaterial({
      color: 0x111119,
      roughness: 0.95,
      metalness: 0.05
    })
  )
  ground.receiveShadow = true
  ground.position.y = -0.01
  scene.add(ground)

  ensureRendererSize()
  startResizeHandling()

  const envMap = await loadEnvironmentMap()
  scene.environment = envMap

  const fbxModel = await loadFbx()
  scene.add(fbxModel)
  fitCameraToObject(fbxModel)

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
</style>
