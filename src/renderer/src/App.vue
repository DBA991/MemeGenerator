<template>
  <div class="meme-generator">
    <div class="meme-container">
      <button class="reset-btn" @click="resetImage" v-if="imageUrl">
        <span class="icon">↺</span> Reset
      </button>

      <div
        class="meme-box"
        ref="memeBox"
        :style="{
          aspectRatio: '1/1',
          gridTemplateAreas: gridTemplateAreas
        }"
      >
        <div v-if="!imageUrl" class="placeholder" @click="triggerFileInput">
          <div class="placeholder-content">
            <span class="plus-icon">+</span>
            <span>Carica Immagine</span>
          </div>
        </div>

        <img v-else :src="imageUrl" alt="Meme" class="meme-image" />

        <div
          v-if="leftText && imageUrl && !topText && !bottomText"
          class="text-overlay left"
          :style="sideSectionStyle"
          @click="openLightbox('left')"
        >
          {{ leftText ? leftText : '+' }}
        </div>

        <div
          v-if="topText && imageUrl && !rightText && !leftText"
          class="text-overlay top"
          :style="horizontalBarStyle"
          @click="openLightbox('top')"
        >
          {{ topText ? topText : '+' }}
        </div>

        <div
          v-if="bottomText && imageUrl && !rightText && !leftText"
          class="text-overlay bottom"
          :style="horizontalBarStyle"
          @click="openLightbox('bottom')"
        >
          {{ bottomText ? bottomText : '+' }}
        </div>

        <div
          v-if="rightText && imageUrl && !topText && !bottomText"
          class="text-overlay right"
          :style="sideSectionStyle"
          @click="openLightbox('right')"
        >
          {{ rightText ? rightText : '+' }}
        </div>

        <div
          v-if="imageUrl && !topText && !rightText && !leftText"
          class="clickable-zone top-zone"
          @click="openLightbox('top')"
        ></div>
        <div
          v-if="imageUrl && !bottomText && !rightText && !leftText"
          class="clickable-zone bottom-zone"
          @click="openLightbox('bottom')"
        ></div>

        <div
          v-if="imageUrl && !rightText && !topText && !bottomText"
          class="clickable-zone right-zone"
          @click="openLightbox('right')"
        ></div>

        <div
          v-if="imageUrl && !leftText && !topText && !bottomText"
          class="clickable-zone left-zone"
          @click="openLightbox('left')"
        ></div>
      </div>

      <input
        type="file"
        ref="fileInput"
        accept="image/*"
        @change="handleImageUpload"
        style="display: none"
      />
    </div>

    <div v-if="showLightbox" class="lightbox" @click.self="closeLightbox">
      <div class="lightbox-content">
        <div class="lightbox-header">
          <h2>Modifica {{ positionLabel }}</h2>
          <button class="close-icon" @click="closeLightbox">×</button>
        </div>

        <textarea
          v-model="currentEditText"
          placeholder="Scrivi qui il tuo testo..."
          rows="3"
        ></textarea>

        <div class="style-controls">
          <div class="control-row">
            <div class="control-group half">
              <label>Colore Testo</label>
              <div class="color-wrapper">
                <input type="color" v-model="textColor" />
                <span>{{ textColor }}</span>
              </div>
            </div>

            <div class="control-group half">
              <label>Colore Bordo</label>
              <div class="color-wrapper">
                <input type="color" v-model="textOutlineColor" />
                <span>{{ textOutlineColor }}</span>
              </div>
            </div>
          </div>

          <div class="control-row">
            <div class="control-group half">
              <label>Colore Sfondo</label>
              <div class="color-wrapper">
                <input type="color" v-model="backgroundColor" />
                <span>{{ backgroundColor }}</span>
              </div>
            </div>

            <div class="control-group half">
              <label>Opacità Sfondo</label>
              <div class="range-wrapper">
                <input type="range" v-model.number="backgroundAlpha" min="0" max="1" step="0.05" />
                <span class="range-value">{{ Math.round(backgroundAlpha * 100) }}%</span>
              </div>
            </div>
          </div>

          <div class="control-group">
            <label>Dimensione Font</label>
            <div class="range-wrapper">
              <input type="range" v-model="fontSize" min="16" max="64" step="2" />
              <span class="range-value">{{ fontSize }}px</span>
            </div>
          </div>

          <div class="control-group">
            <label>Altezza Barra</label>
            <div class="range-wrapper">
              <input type="range" v-model="textHeight" min="40" max="150" step="5" />
              <span class="range-value">{{ textHeight }}px</span>
            </div>
          </div>
        </div>

        <div class="lightbox-buttons">
          <button class="btn btn-secondary" @click="clearText">Cancella Testo</button>
          <button class="btn btn-primary" @click="saveText">Applica Modifiche</button>
        </div>
      </div>
    </div>

    <div v-if="imageUrl" class="save-section">
      <div class="save-settings">
        <div class="control-group">
          <label>Qualità</label>
          <div class="range-wrapper">
            <input type="range" v-model.number="saveQuality" min="0.1" max="1" step="0.1" />
            <span class="range-value">{{ Math.round(saveQuality * 100) }}%</span>
          </div>
        </div>
        <div class="control-group">
          <label>Formato</label>
          <select v-model="saveFormat">
            <option value="png">PNG</option>
            <option value="jpeg">JPEG</option>
          </select>
        </div>
      </div>
      <button class="btn btn-primary btn-large btn-save" @click="saveMeme">Scarica</button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { toPng, toJpeg } from 'html-to-image'

// Refs per l'immagine
const imageUrl = ref('')
const fileInput = ref(null)
const memeBox = ref(null)

// Refs per i testi (Aggiunto leftText)
const topText = ref('')
const bottomText = ref('')
const rightText = ref('')
const leftText = ref('')

// Refs per la lightbox
const showLightbox = ref(false)
const currentPosition = ref('')
const currentEditText = ref('')

// Refs per lo stile
const textColor = ref('#ffffff')
const backgroundColor = ref('#000000')
const backgroundAlpha = ref(0.8)
const textOutlineColor = ref('#000000')
const fontSize = ref(32)
const textHeight = ref(80)

// Refs per il salvataggio
const saveQuality = ref(1)
const saveFormat = ref('png')

// Helper per convertire hex a rgba
const hexToRgba = (hex, alpha) => {
  const r = parseInt(hex.slice(1, 3), 16)
  const g = parseInt(hex.slice(3, 5), 16)
  const b = parseInt(hex.slice(5, 7), 16)
  return `rgba(${r}, ${g}, ${b}, ${alpha})`
}

// Computed per lo stile base del testo
const baseTextStyle = computed(() => ({
  color: textColor.value,
  backgroundColor: hexToRgba(backgroundColor.value, backgroundAlpha.value),
  fontSize: fontSize.value + 'px',
  WebkitTextStroke: `2px ${textOutlineColor.value}`,
  textStroke: `2px ${textOutlineColor.value}`
}))

// Computed per lo stile delle barre orizzontali (top/bottom)
const horizontalBarStyle = computed(() => ({
  ...baseTextStyle.value,
  height: textHeight.value + 'px',
  lineHeight: textHeight.value + 'px'
}))

// Computed per lo stile delle porzioni laterali (Right e Left condividono questo stile)
const sideSectionStyle = computed(() => ({
  ...baseTextStyle.value
}))

// Computed per label posizione (Aggiunto left)
const positionLabel = computed(() => {
  const labels = {
    top: 'testo superiore',
    bottom: 'testo inferiore',
    right: 'didascalia destra',
    left: 'didascalia sinistra'
  }
  return labels[currentPosition.value] || ''
})

// Computed per le grid-template-areas dinamiche
const gridTemplateAreas = computed(() => {
  if (rightText.value || leftText.value) {
    return `'left top right'
            'left center right'
            'left bottom right'`
  } else {
    return `'top top top'
            'left center right'
            'bottom bottom bottom'`
  }
})

// Funzione per triggerare input file
const triggerFileInput = () => {
  fileInput.value.click()
}

// Funzione per gestire upload immagine
const handleImageUpload = (event) => {
  const file = event.target.files[0]
  if (file) {
    const reader = new FileReader()
    reader.onload = (e) => {
      imageUrl.value = e.target.result
    }
    reader.readAsDataURL(file)
  }
}

// Funzione per reset immagine (Aggiunto reset leftText)
const resetImage = () => {
  imageUrl.value = ''
  topText.value = ''
  bottomText.value = ''
  rightText.value = ''
  leftText.value = ''
  if (fileInput.value) {
    fileInput.value.value = ''
  }
}

// Funzione per aprire lightbox (Aggiunto gestione left)
const openLightbox = (position) => {
  currentPosition.value = position

  if (position === 'top') {
    currentEditText.value = topText.value
  } else if (position === 'bottom') {
    currentEditText.value = bottomText.value
  } else if (position === 'right') {
    currentEditText.value = rightText.value
  } else if (position === 'left') {
    currentEditText.value = leftText.value
  }

  showLightbox.value = true
}

// Funzione per chiudere lightbox
const closeLightbox = () => {
  showLightbox.value = false
  currentPosition.value = ''
  currentEditText.value = ''
}

// Funzione per salvare testo (Logica aggiornata per le esclusioni)
const saveText = () => {
  if (currentPosition.value === 'top') {
    topText.value = currentEditText.value
    // Se inserisce top, cancella i laterali
    if (currentEditText.value) {
      rightText.value = ''
      leftText.value = ''
    }
  } else if (currentPosition.value === 'bottom') {
    bottomText.value = currentEditText.value
    // Se inserisce bottom, cancella i laterali
    if (currentEditText.value) {
      rightText.value = ''
      leftText.value = ''
    }
  } else if (currentPosition.value === 'right') {
    rightText.value = currentEditText.value
    // Se inserisce right, cancella top e bottom
    if (currentEditText.value) {
      topText.value = ''
      bottomText.value = ''
    }
  } else if (currentPosition.value === 'left') {
    leftText.value = currentEditText.value
    // Se inserisce left, cancella top e bottom
    if (currentEditText.value) {
      topText.value = ''
      bottomText.value = ''
    }
  }

  closeLightbox()
}

// Funzione per cancellare testo
const clearText = () => {
  currentEditText.value = ''
  saveText()
}

// Funzione per salvare meme
const saveMeme = async () => {
  if (!memeBox.value) return

  try {
    let dataUrl

    if (saveFormat.value === 'png') {
      dataUrl = await toPng(memeBox.value, {
        quality: saveQuality.value,
        pixelRatio: 1
      })
    } else {
      dataUrl = await toJpeg(memeBox.value, {
        quality: saveQuality.value,
        pixelRatio: 1
      })
    }

    const link = document.createElement('a')
    link.download = `meme.${saveFormat.value}`
    link.href = dataUrl
    link.click()
  } catch (error) {
    console.error('Errore durante il salvataggio:', error)
    alert('Errore durante il salvataggio del meme')
  }
}
</script>

<style scoped>
/* Layout Generale */
.meme-generator {
  width: 100%;
}

h1 {
  text-align: center;
  color: var(--text-main);
  font-size: 2.5rem;
  font-weight: 800;
  margin-bottom: 0.5rem;
  letter-spacing: -1px;
}

.subtitle {
  text-align: center;
  color: var(--text-muted);
  margin-bottom: 40px;
  font-size: 1.1rem;
}

/* Container principale */
.meme-container {
  position: relative;
  max-width: 600px;
  margin: 0 auto 40px;
  background: var(--surface);
  border-radius: var(--radius);
  box-shadow: var(--shadow-lg);
  padding: 10px;
}

/* Pulsante Reset flottante */
.reset-btn {
  position: absolute;
  top: -50px;
  right: 0;
  left: auto;
  padding: 8px 16px;
  background: transparent;
  color: var(--danger);
  border: 1px solid var(--danger);
  border-radius: 20px;
  cursor: pointer;
  font-size: 13px;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 6px;
  z-index: 10;
}

.reset-btn:hover {
  background: var(--danger);
  color: white;
}

.reset-btn .icon {
  font-size: 16px;
}

/* Area Meme */
.meme-box {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  grid-template-rows: 1fr 1fr 1fr;
  width: 100%;
  background: #f1f5f9;
  border-radius: 8px;
  overflow: hidden;
  position: relative;
}

/* Placeholder */
.placeholder {
  grid-area: 1 / 1 / -1 / -1;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f1f5f9;
  color: var(--primary);
  cursor: pointer;
  user-select: none;
  aspect-ratio: 1/1;
  z-index: 1;
  transition: background 0.2s;
  border: 2px dashed var(--border);
  border-radius: 8px;
  margin: 10px;
}

.placeholder:hover {
  background: #e2e8f0;
  border-color: var(--primary);
}

.placeholder-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  font-weight: 600;
}

.plus-icon {
  font-size: 40px;
  line-height: 1;
}

.meme-image {
  grid-area: 1 / 1 / -1 / -1;
  width: 100%;
  height: 100%;
  object-fit: contain;
  display: block;
  z-index: 1;
}

/* Testi Overlay */
.text-overlay {
  font-weight: bold;
  text-align: center;
  cursor: pointer;
  word-wrap: break-word;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0 20px;
  z-index: 10;
  transition: opacity 0.2s;
}

.text-overlay:hover {
  opacity: 0.85;
}

.text-overlay.top {
  align-self: start;
  grid-area: top;
}

.text-overlay.bottom {
  grid-area: bottom;
  align-self: end;
}

.text-overlay.right {
  grid-area: right;
}

/* NUOVO STILE LEFT */
.text-overlay.left {
  grid-area: left;
}

/* Zone cliccabili */
.clickable-zone {
  cursor: pointer;
  background: rgba(255, 255, 255, 0);
  transition: background 0.2s;
  z-index: 5;
}

.clickable-zone:hover {
  background: rgba(99, 102, 241, 0.1);
  box-shadow: inset 0 0 0 2px var(--primary);
}

.top-zone {
  grid-area: top;
}
.bottom-zone {
  grid-area: bottom;
}
.right-zone {
  grid-area: right;
}
/* NUOVA ZONA LEFT */
.left-zone {
  grid-area: left;
}

/* Lightbox Moderna */
.lightbox {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(15, 23, 42, 0.6);
  backdrop-filter: blur(4px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 20px;
}

.lightbox-content {
  background: var(--surface);
  padding: 30px;
  border-radius: 16px;
  max-width: 500px;
  width: 100%;
  box-shadow: var(--shadow-lg);
  animation: slideUp 0.3s ease-out;
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.lightbox-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 25px;
}

.lightbox-content h2 {
  font-size: 1.5rem;
  color: var(--text-main);
  font-weight: 700;
  margin: 0;
}

.close-icon {
  background: none;
  border: none;
  font-size: 24px;
  color: var(--text-muted);
  cursor: pointer;
  padding: 0 5px;
}

.close-icon:hover {
  color: var(--danger);
}

.info-message {
  padding: 12px 16px;
  margin-bottom: 20px;
  border-radius: 8px;
  font-size: 14px;
  line-height: 1.5;
  display: flex;
  align-items: flex-start;
  gap: 10px;
}

.info-message.warning {
  background: #fff7ed;
  border: 1px solid #fed7aa;
  color: #c2410c;
}

.lightbox-content textarea {
  width: 100%;
  padding: 15px;
  border: 1px solid var(--border);
  border-radius: 8px;
  font-size: 16px;
  margin-bottom: 25px;
  resize: vertical;
  background: #f8fafc;
  transition: border-color 0.2s;
}

.lightbox-content textarea:focus {
  border-color: var(--primary);
  background: white;
}

/* Controlli Stile */
.style-controls {
  display: flex;
  flex-direction: column;
  gap: 20px;
  margin-bottom: 30px;
}

.control-row {
  display: flex;
  gap: 20px;
}

.half {
  flex: 1;
}

.control-group label {
  display: block;
  font-size: 13px;
  font-weight: 600;
  color: var(--text-muted);
  margin-bottom: 8px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.color-wrapper {
  display: flex;
  align-items: center;
  gap: 10px;
  background: #f8fafc;
  padding: 5px 10px;
  border-radius: 8px;
  border: 1px solid var(--border);
}

.color-wrapper input[type='color'] {
  border: none;
  width: 30px;
  height: 30px;
  cursor: pointer;
  background: none;
  padding: 0;
}

.color-wrapper span {
  font-family: monospace;
  font-size: 13px;
  color: var(--text-muted);
}

.range-wrapper {
  display: flex;
  align-items: center;
  gap: 15px;
}

.range-wrapper input[type='range'] {
  flex: 1;
  accent-color: var(--primary);
  height: 6px;
  background: #e2e8f0;
  border-radius: 3px;
  cursor: pointer;
}

.range-value {
  min-width: 45px;
  text-align: right;
  font-size: 14px;
  font-weight: 600;
  color: var(--primary);
}

/* Pulsanti */
.btn {
  padding: 12px 24px;
  border: none;
  border-radius: 8px;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.btn-primary {
  background: var(--primary);
  color: white;
  box-shadow: 0 4px 6px -1px rgba(99, 102, 241, 0.4);
}

.btn-primary:hover {
  background: var(--primary-hover);
  transform: translateY(-1px);
}

.btn-secondary {
  background: transparent;
  color: var(--text-muted);
  border: 1px solid var(--border);
}

.btn-secondary:hover {
  border-color: var(--danger);
  color: var(--danger);
  background: #fff1f2;
}

.lightbox-buttons {
  display: flex;
  gap: 15px;
  justify-content: flex-end;
  border-top: 1px solid var(--border);
  padding-top: 20px;
}

/* Sezione Salvataggio */
.save-section {
  background: white;
  padding: 25px;
  border-radius: var(--radius);
  border: 1px solid var(--border);
  max-width: 600px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.save-settings {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
  padding-bottom: 20px;
  border-bottom: 1px dashed var(--border);
}

.save-settings select {
  width: 100%;
  padding: 8px 12px;
  border: 1px solid var(--border);
  border-radius: 6px;
  background-color: #f8fafc;
  font-size: 14px;
  color: var(--text-main);
}

.btn-save {
  width: 100%;
  font-size: 16px;
  padding: 16px;
  letter-spacing: 0.5px;
}

/* Responsive */
@media (max-width: 600px) {
  .control-row {
    flex-direction: column;
    gap: 15px;
  }

  .save-settings {
    grid-template-columns: 1fr;
  }

  .reset-btn {
    top: -45px;
  }
}
</style>
