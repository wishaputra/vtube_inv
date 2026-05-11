<script setup>
import { ref, reactive, watch, onMounted, nextTick } from 'vue'

// ============================================================
// STATE: Pengaturan Teks (Text Settings)
// ============================================================

// Template teks utama dengan placeholder {name}
const textTemplate = ref('Hai {name}, kamu diundang ke debut ku!')

// Nama font kustom yang sedang aktif
const customFontName = ref('')
const customFontLoaded = ref(false)

// Warna font
const fontColor = ref('#ffffff')

// Ukuran font (px)
const fontSize = ref(48)

// Posisi teks (persentase dari dimensi canvas)
const posX = ref(50)
const posY = ref(50)

// Perataan teks: 'left' | 'center' | 'right'
const textAlign = ref('center')

// ============================================================
// STATE: Gambar Latar Belakang
// ============================================================

const backgroundImage = ref(null) // HTMLImageElement
const backgroundFileName = ref('')

// ============================================================
// STATE: Daftar Tamu (Guest List) dengan LocalStorage
// ============================================================

const guestName = ref('')
const guestList = reactive(loadGuestListFromStorage())

// ============================================================
// STATE: Canvas
// ============================================================

const canvasRef = ref(null)
const canvasWidth = ref(1080)
const canvasHeight = ref(1080)

// ============================================================
// FUNGSI: LocalStorage untuk Daftar Tamu
// ============================================================

/**
 * Memuat daftar tamu dari LocalStorage.
 * Mengembalikan array kosong jika tidak ada data.
 */
function loadGuestListFromStorage() {
  try {
    const stored = localStorage.getItem('vtuber_guest_list')
    return stored ? JSON.parse(stored) : []
  } catch {
    return []
  }
}

/**
 * Menyimpan daftar tamu ke LocalStorage.
 */
function saveGuestListToStorage() {
  localStorage.setItem('vtuber_guest_list', JSON.stringify(guestList))
}

/**
 * Menambahkan tamu baru ke daftar.
 */
function addGuest() {
  const name = guestName.value.trim()
  if (!name) return

  guestList.push({
    id: crypto.randomUUID(),
    name: name,
  })
  guestName.value = ''
  saveGuestListToStorage()
}

/**
 * Menghapus tamu dari daftar berdasarkan ID.
 */
function deleteGuest(id) {
  const index = guestList.findIndex((g) => g.id === id)
  if (index !== -1) {
    guestList.splice(index, 1)
    saveGuestListToStorage()
  }
}

// ============================================================
// FUNGSI: Unggah Gambar Latar Belakang
// ============================================================

/**
 * Menangani unggahan gambar latar belakang.
 * Memuat gambar ke dalam HTMLImageElement dan menyesuaikan dimensi canvas.
 */
function handleBackgroundUpload(event) {
  const file = event.target.files[0]
  if (!file) return

  backgroundFileName.value = file.name

  const reader = new FileReader()
  reader.onload = (e) => {
    const img = new Image()
    img.onload = () => {
      backgroundImage.value = img
      // Sesuaikan dimensi canvas dengan gambar
      canvasWidth.value = img.naturalWidth
      canvasHeight.value = img.naturalHeight
      nextTick(() => drawCanvas())
    }
    img.src = e.target.result
  }
  reader.readAsDataURL(file)
}

// ============================================================
// FUNGSI: Unggah Font Kustom
// ============================================================

/**
 * Menangani unggahan font kustom (.ttf, .otf, .woff).
 * Menyuntikkan font ke halaman menggunakan CSS @font-face secara dinamis.
 */
function handleFontUpload(event) {
  const file = event.target.files[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = async (e) => {
    const fontName = `CustomFont_${Date.now()}`
    const fontFace = new FontFace(fontName, `url(${e.target.result})`)

    try {
      const loadedFont = await fontFace.load()
      document.fonts.add(loadedFont)
      customFontName.value = fontName
      customFontLoaded.value = true
      // Redraw canvas setelah font dimuat
      nextTick(() => drawCanvas())
    } catch (err) {
      console.error('Gagal memuat font:', err)
      alert('Gagal memuat font. Pastikan file font valid.')
    }
  }
  reader.readAsDataURL(file)
}

// ============================================================
// FUNGSI: Menggambar Canvas (Draw Canvas)
// ============================================================

/**
 * Menggambar canvas dengan latar belakang dan teks.
 * @param {string} nameOverride - Nama tamu untuk menggantikan {name}. 
 *   Jika null, gunakan placeholder '[Nama Tamu]'.
 * @returns {HTMLCanvasElement|null} - Elemen canvas atau null jika gagal.
 */
function drawCanvas(nameOverride = null) {
  const canvas = canvasRef.value
  if (!canvas) return null

  const ctx = canvas.getContext('2d')

  // Bersihkan canvas
  ctx.clearRect(0, 0, canvas.width, canvas.height)

  // Gambar latar belakang jika tersedia
  if (backgroundImage.value) {
    ctx.drawImage(backgroundImage.value, 0, 0, canvas.width, canvas.height)
  } else {
    // Tampilkan placeholder abu-abu jika belum ada gambar
    ctx.fillStyle = '#2d2d2d'
    ctx.fillRect(0, 0, canvas.width, canvas.height)

    // Teks petunjuk
    ctx.fillStyle = '#666666'
    ctx.font = '24px sans-serif'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(
      'Unggah gambar latar belakang untuk memulai',
      canvas.width / 2,
      canvas.height / 2
    )
  }

  // Gambar teks undangan
  const displayName = nameOverride !== null ? nameOverride : '[Nama Tamu]'
  const finalText = textTemplate.value.replace(/\{name\}/g, displayName)

  // Tentukan font
  const fontFamily = customFontLoaded.value ? customFontName.value : 'sans-serif'
  ctx.font = `${fontSize.value}px "${fontFamily}"`
  ctx.fillStyle = fontColor.value
  ctx.textAlign = textAlign.value
  ctx.textBaseline = 'middle'

  // Hitung posisi berdasarkan persentase
  const x = (posX.value / 100) * canvas.width
  const y = (posY.value / 100) * canvas.height

  // Gambar teks dengan word-wrap sederhana
  drawWrappedText(ctx, finalText, x, y, canvas.width * 0.85, fontSize.value * 1.3)

  return canvas
}

/**
 * Menggambar teks dengan word-wrap.
 */
function drawWrappedText(ctx, text, x, y, maxWidth, lineHeight) {
  const words = text.split(' ')
  let line = ''
  const lines = []

  for (let i = 0; i < words.length; i++) {
    const testLine = line + words[i] + ' '
    const metrics = ctx.measureText(testLine)
    if (metrics.width > maxWidth && i > 0) {
      lines.push(line.trim())
      line = words[i] + ' '
    } else {
      line = testLine
    }
  }
  lines.push(line.trim())

  // Hitung offset Y agar teks terpusat secara vertikal
  const totalHeight = lines.length * lineHeight
  const startY = y - totalHeight / 2 + lineHeight / 2

  for (let i = 0; i < lines.length; i++) {
    ctx.fillText(lines[i], x, startY + i * lineHeight)
  }
}

// ============================================================
// FUNGSI: Download Undangan
// ============================================================

/**
 * Mengunduh undangan yang dipersonalisasi untuk tamu tertentu.
 */
function downloadInvitation(guest) {
  // Buat canvas offscreen dengan dimensi penuh
  const offCanvas = document.createElement('canvas')
  offCanvas.width = canvasWidth.value
  offCanvas.height = canvasHeight.value
  const ctx = offCanvas.getContext('2d')

  // Gambar latar belakang
  if (backgroundImage.value) {
    ctx.drawImage(backgroundImage.value, 0, 0, offCanvas.width, offCanvas.height)
  }

  // Gambar teks dengan nama tamu
  const finalText = textTemplate.value.replace(/\{name\}/g, guest.name)
  const fontFamily = customFontLoaded.value ? customFontName.value : 'sans-serif'
  ctx.font = `${fontSize.value}px "${fontFamily}"`
  ctx.fillStyle = fontColor.value
  ctx.textAlign = textAlign.value
  ctx.textBaseline = 'middle'

  const x = (posX.value / 100) * offCanvas.width
  const y = (posY.value / 100) * offCanvas.height

  drawWrappedText(ctx, finalText, x, y, offCanvas.width * 0.85, fontSize.value * 1.3)

  // Konversi ke Data URL dan unduh
  const dataURL = offCanvas.toDataURL('image/png')
  const link = document.createElement('a')
  link.download = `Undangan_Debut_${guest.name.replace(/\s+/g, '_')}.png`
  link.href = dataURL
  document.body.appendChild(link)
  link.click()
  document.body.removeChild(link)
}

// ============================================================
// WATCHERS: Redraw canvas saat pengaturan berubah
// ============================================================

watch(
  [textTemplate, fontColor, fontSize, posX, posY, textAlign, customFontName],
  () => {
    drawCanvas()
  }
)

// ============================================================
// LIFECYCLE: Gambar canvas saat komponen dimuat
// ============================================================

onMounted(() => {
  nextTick(() => drawCanvas())
})
</script>

<template>
  <div class="min-h-screen bg-gray-900 text-gray-100 p-4 md:p-8">
    <!-- Header -->
    <header class="text-center mb-8">
      <h1 class="text-3xl md:text-4xl font-bold bg-gradient-to-r from-purple-400 to-pink-400 bg-clip-text text-transparent">
        ✨ VTuber Debut Invitation Generator ✨
      </h1>
      <p class="text-gray-400 mt-2">Buat undangan debut yang dipersonalisasi untuk setiap tamu</p>
    </header>

    <!-- Layout Utama: 2 Kolom -->
    <div class="grid grid-cols-1 lg:grid-cols-2 gap-6 max-w-7xl mx-auto">
      
      <!-- ============================================ -->
      <!-- KOLOM KIRI: Panel Kontrol & Daftar Tamu -->
      <!-- ============================================ -->
      <div class="space-y-6">

        <!-- Panel: Unggah Gambar Latar Belakang -->
        <section class="bg-gray-800 rounded-xl p-5 shadow-lg">
          <h2 class="text-lg font-semibold text-purple-300 mb-3">🖼️ Gambar Latar Belakang</h2>
          <label
            class="flex flex-col items-center justify-center w-full h-24 border-2 border-dashed border-gray-600 rounded-lg cursor-pointer hover:border-purple-400 transition-colors"
          >
            <span class="text-gray-400 text-sm">
              {{ backgroundFileName || 'Klik untuk unggah gambar template (.png, .jpg)' }}
            </span>
            <input
              type="file"
              accept="image/*"
              class="hidden"
              @change="handleBackgroundUpload"
            />
          </label>
        </section>

        <!-- Panel: Pengaturan Teks -->
        <section class="bg-gray-800 rounded-xl p-5 shadow-lg space-y-4">
          <h2 class="text-lg font-semibold text-purple-300 mb-3">🎨 Pengaturan Teks</h2>

          <!-- Template Teks -->
          <div>
            <label class="block text-sm text-gray-300 mb-1">Template Teks (gunakan {'{name}'} untuk nama tamu)</label>
            <textarea
              v-model="textTemplate"
              rows="3"
              class="w-full bg-gray-700 border border-gray-600 rounded-lg px-3 py-2 text-sm text-gray-100 focus:outline-none focus:ring-2 focus:ring-purple-500 resize-none"
              placeholder="Hai {name}, kamu diundang ke debut ku!"
            ></textarea>
          </div>

          <!-- Unggah Font Kustom -->
          <div>
            <label class="block text-sm text-gray-300 mb-1">Font Kustom (.ttf, .otf, .woff)</label>
            <input
              type="file"
              accept=".ttf,.otf,.woff"
              @change="handleFontUpload"
              class="w-full text-sm text-gray-400 file:mr-3 file:py-1.5 file:px-3 file:rounded-lg file:border-0 file:text-sm file:font-medium file:bg-purple-600 file:text-white hover:file:bg-purple-500 file:cursor-pointer"
            />
            <p v-if="customFontLoaded" class="text-xs text-green-400 mt-1">
              ✓ Font dimuat: {{ customFontName }}
            </p>
          </div>

          <!-- Warna Font & Ukuran Font -->
          <div class="grid grid-cols-2 gap-4">
            <div>
              <label class="block text-sm text-gray-300 mb-1">Warna Font</label>
              <input
                type="color"
                v-model="fontColor"
                class="w-full h-10 rounded-lg border border-gray-600 cursor-pointer bg-gray-700"
              />
            </div>
            <div>
              <label class="block text-sm text-gray-300 mb-1">Ukuran Font: {{ fontSize }}px</label>
              <input
                type="range"
                v-model.number="fontSize"
                min="12"
                max="120"
                class="w-full accent-purple-500"
              />
            </div>
          </div>

          <!-- Posisi X & Y -->
          <div class="grid grid-cols-2 gap-4">
            <div>
              <label class="block text-sm text-gray-300 mb-1">Posisi X: {{ posX }}%</label>
              <input
                type="range"
                v-model.number="posX"
                min="0"
                max="100"
                class="w-full accent-purple-500"
              />
            </div>
            <div>
              <label class="block text-sm text-gray-300 mb-1">Posisi Y: {{ posY }}%</label>
              <input
                type="range"
                v-model.number="posY"
                min="0"
                max="100"
                class="w-full accent-purple-500"
              />
            </div>
          </div>

          <!-- Perataan Teks -->
          <div>
            <label class="block text-sm text-gray-300 mb-2">Perataan Teks</label>
            <div class="flex gap-2">
              <button
                @click="textAlign = 'left'"
                :class="[
                  'flex-1 py-2 rounded-lg text-sm font-medium transition-colors',
                  textAlign === 'left'
                    ? 'bg-purple-600 text-white'
                    : 'bg-gray-700 text-gray-300 hover:bg-gray-600',
                ]"
              >
                ← Kiri
              </button>
              <button
                @click="textAlign = 'center'"
                :class="[
                  'flex-1 py-2 rounded-lg text-sm font-medium transition-colors',
                  textAlign === 'center'
                    ? 'bg-purple-600 text-white'
                    : 'bg-gray-700 text-gray-300 hover:bg-gray-600',
                ]"
              >
                ↔ Tengah
              </button>
              <button
                @click="textAlign = 'right'"
                :class="[
                  'flex-1 py-2 rounded-lg text-sm font-medium transition-colors',
                  textAlign === 'right'
                    ? 'bg-purple-600 text-white'
                    : 'bg-gray-700 text-gray-300 hover:bg-gray-600',
                ]"
              >
                Kanan →
              </button>
            </div>
          </div>
        </section>

        <!-- Panel: Daftar Tamu -->
        <section class="bg-gray-800 rounded-xl p-5 shadow-lg">
          <h2 class="text-lg font-semibold text-purple-300 mb-3">👥 Daftar Tamu</h2>

          <!-- Input Tambah Tamu -->
          <div class="flex gap-2 mb-4">
            <input
              v-model="guestName"
              type="text"
              placeholder="Masukkan nama tamu..."
              class="flex-1 bg-gray-700 border border-gray-600 rounded-lg px-3 py-2 text-sm text-gray-100 focus:outline-none focus:ring-2 focus:ring-purple-500"
              @keyup.enter="addGuest"
            />
            <button
              @click="addGuest"
              class="px-4 py-2 bg-purple-600 hover:bg-purple-500 text-white text-sm font-medium rounded-lg transition-colors"
            >
              + Add
            </button>
          </div>

          <!-- Tabel Daftar Tamu -->
          <div v-if="guestList.length === 0" class="text-center text-gray-500 py-4 text-sm">
            Belum ada tamu. Tambahkan nama di atas.
          </div>
          <div v-else class="max-h-64 overflow-y-auto space-y-2">
            <div
              v-for="guest in guestList"
              :key="guest.id"
              class="flex items-center justify-between bg-gray-700 rounded-lg px-3 py-2"
            >
              <span class="text-sm text-gray-200 truncate flex-1">{{ guest.name }}</span>
              <div class="flex gap-2 ml-2 shrink-0">
                <button
                  @click="downloadInvitation(guest)"
                  class="px-3 py-1 bg-green-600 hover:bg-green-500 text-white text-xs font-medium rounded-md transition-colors"
                  :disabled="!backgroundImage"
                  :class="{ 'opacity-50 cursor-not-allowed': !backgroundImage }"
                  :title="!backgroundImage ? 'Unggah gambar latar belakang terlebih dahulu' : 'Download undangan'"
                >
                  ⬇ Download
                </button>
                <button
                  @click="deleteGuest(guest.id)"
                  class="px-3 py-1 bg-red-600 hover:bg-red-500 text-white text-xs font-medium rounded-md transition-colors"
                >
                  ✕ Delete
                </button>
              </div>
            </div>
          </div>

          <p class="text-xs text-gray-500 mt-3">
            Total: {{ guestList.length }} tamu • Data tersimpan di LocalStorage
          </p>
        </section>
      </div>

      <!-- ============================================ -->
      <!-- KOLOM KANAN: Pratinjau Canvas -->
      <!-- ============================================ -->
      <div class="space-y-4">
        <section class="bg-gray-800 rounded-xl p-5 shadow-lg">
          <h2 class="text-lg font-semibold text-purple-300 mb-3">👁️ Pratinjau (Preview)</h2>
          <div class="relative w-full overflow-hidden rounded-lg border border-gray-700">
            <canvas
              ref="canvasRef"
              :width="canvasWidth"
              :height="canvasHeight"
              class="w-full h-auto"
            ></canvas>
          </div>
          <p class="text-xs text-gray-500 mt-2 text-center">
            Dimensi: {{ canvasWidth }} × {{ canvasHeight }}px
          </p>
        </section>
      </div>
    </div>

    <!-- Footer -->
    <footer class="text-center text-gray-600 text-xs mt-8 pb-4">
      VTuber Debut Invitation Generator • Dibuat dengan Vue 3 + Tailwind CSS + HTML5 Canvas
    </footer>
  </div>
</template>
