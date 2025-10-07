<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'
import * as THREE from 'three'

const canvas = ref<HTMLCanvasElement>()
let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let renderer: THREE.WebGLRenderer
let animationFrameId: number

onMounted(() => {
  if (!canvas.value) return

  // Scene setup
  scene = new THREE.Scene()

  // Camera setup
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.z = 5

  // Renderer setup
  renderer = new THREE.WebGLRenderer({ canvas: canvas.value, antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(window.devicePixelRatio)

  // Load star texture and create particle system
  const textureLoader = new THREE.TextureLoader()
  const starTexture = textureLoader.load('/star-sprite.png')

  const starsGeometry = new THREE.BufferGeometry()
  const starCount = 2000
  const positions = new Float32Array(starCount * 3)
  const colors = new Float32Array(starCount * 3)
  const sizes = new Float32Array(starCount)

  for (let i = 0; i < starCount; i++) {
    const i3 = i * 3

    // Generate position with minimum distance from camera
    let x = (Math.random() - 0.5) * 20
    let y = (Math.random() - 0.5) * 20
    let z = (Math.random() - 0.5) * 20

    // Ensure minimum distance of 5 units from camera at origin
    const minDistance = 2
    const distance = Math.sqrt(x * x + y * y + z * z)
    if (distance < minDistance) {
      const scale = minDistance / distance
      x *= scale
      y *= scale
      z *= scale
    }

    positions[i3] = x
    positions[i3 + 1] = y
    positions[i3 + 2] = z

    // Random RGB colors
    colors[i3] = Math.random() // r
    colors[i3 + 1] = Math.random() // g
    colors[i3 + 2] = Math.random() // b

    // Random sizes
    sizes[i] = Math.random() * 0.3 + 0.1
  }

  starsGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3))
  starsGeometry.setAttribute('color', new THREE.BufferAttribute(colors, 3))
  starsGeometry.setAttribute('size', new THREE.BufferAttribute(sizes, 1))

  // Custom shader material
  const starsMaterial = new THREE.ShaderMaterial({
    uniforms: {
      uTime: { value: 0 },
      uTexture: { value: starTexture },
    },
    vertexShader: `
      attribute float size;
      attribute vec3 color;
      varying vec3 vColor;
      uniform float uTime;

      void main() {
        vColor = color;
        vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);

        // Pulsating effect based on position and time
        float pulse = sin(uTime + position.x * 10.0) * 0.5 + 0.5;
        gl_PointSize = size * (0.7 + pulse * 0.3) * (300.0 / -mvPosition.z);

        gl_Position = projectionMatrix * mvPosition;
      }
    `,
    fragmentShader: `
      uniform sampler2D uTexture;
      varying vec3 vColor;

      void main() {
        vec4 texColor = texture2D(uTexture, gl_PointCoord);

        // Apply color tint with brightness boost
        vec3 brightColor = vColor * 2.0; // Boost vibrance
        gl_FragColor = vec4(brightColor * texColor.rgb, texColor.a);
      }
    `,
    transparent: true,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
  })

  const stars = new THREE.Points(starsGeometry, starsMaterial)
  scene.add(stars)

  // Handle window resize
  const handleResize = () => {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth, window.innerHeight)
  }
  window.addEventListener('resize', handleResize)

  // Animation loop
  const animate = () => {
    animationFrameId = requestAnimationFrame(animate)
    stars.rotation.y += 0.0002

    // Update shader time uniform for animation
    if (starsMaterial.uniforms) {
      starsMaterial.uniforms.uTime.value += 0.01
    }

    renderer.render(scene, camera)
  }
  animate()
})

onUnmounted(() => {
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId)
  }
  renderer?.dispose()
})
</script>

<template>
  <canvas ref="canvas"></canvas>
  <div class="name-overlay">
    <h1>Erik Gimbergsson</h1>
  </div>
</template>

<style scoped>
canvas {
  display: block;
  width: 100%;
  height: 100%;
  background: #000;
}

.name-overlay {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  pointer-events: none;
}

.name-overlay h1 {
  font-size: 4rem;
  font-weight: 700;
  color: white;
  text-align: center;
  margin: 0;
  text-shadow: 0 0 20px rgba(255, 255, 255, 0.5);
}
</style>
