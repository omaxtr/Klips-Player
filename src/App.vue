<template>
  <div class="app-container">
    <!-- WOW Loading / Error / Interaction Layer -->
    <div v-if="isPlayer && (loading || error || needsInteraction)" class="interaction-layer">
      <div v-if="error" class="interaction-card error-card">
        <div class="error-icon">⚠️</div>
        <h2>Fetch Error</h2>
        <p>{{ error }}</p>
        <button @click="goToSetup" class="start-btn">Return to Config</button>
      </div>
      
      <div v-else-if="loading" class="wow-loader">
        <div class="loader-bg-text">K L I P S</div>
        <div class="loader-content">
          <div class="loader-logo">🚀</div>
          <h2>Fetching Awesome Clips...</h2>
          <div class="progress-box">
            <div class="progress-fill" :style="{ width: loadProgress + '%' }"></div>
          </div>
          <p>Connecting to Twitch Proxy...</p>
        </div>
      </div>

      <div v-else-if="needsInteraction" class="interaction-card play-card">
        <div class="play-icon">▶</div>
        <h2>Click to Play</h2>
        <p>Browser blocked automated playback.</p>
        <button @click="startPlayer(true)" class="start-btn">Start Player</button>
      </div>
    </div>

    <div v-if="isPlayer" class="player-mode">
    <div id="view" :class="[{ 'vhs-look': cfg.flt === 'vhs' }, activeAnimClass]">
      <video 
        ref="vid1" 
        playsinline 
        :muted="isMuted" 
        :class="[vidClass1, { 'vid-sliding': cfg.anim === 'slide' }]"
        :style="videoStyle"
        @ended="onVideoEnded"
        @timeupdate="onTimeUpdate"
        @loadedmetadata="onMetadataLoaded"
      ></video>
      <video 
        ref="vid2" 
        playsinline 
        :muted="isMuted" 
        :class="[vidClass2, { 'vid-sliding': cfg.anim === 'slide' }]"
        :style="videoStyle"
        @ended="onVideoEnded"
        @timeupdate="onTimeUpdate"
        @loadedmetadata="onMetadataLoaded"
      ></video>
      <!-- Improved Timer -->
      <div id="timer" v-if="cfg.tm && activeClip" :style="timerStyle">
        <div v-if="['circular', 'both'].includes(cfg.tType)" class="timer-circle-wrap" 
             :style="{ width: (cfg.tSize || 48) + 'px', height: (cfg.tSize || 48) + 'px' }">
          <svg viewBox="0 0 36 36" class="circular-chart">
            <path class="circle-bg" 
              d="M18 2.0845 a 15.9155 15.9155 0 0 1 0 31.831 a 15.9155 15.9155 0 0 1 0 -31.831" 
            />
            <path class="circle-fg" 
              :stroke-dasharray="timerDashArray" 
              :style="{ stroke: cfg.hc }"
              d="M18 2.0845 a 15.9155 15.9155 0 0 1 0 31.831 a 15.9155 15.9155 0 0 1 0 -31.831" 
            />
          </svg>
        </div>
        <span v-if="['numeric', 'both'].includes(cfg.tType)" :style="{ fontSize: (cfg.tSize || 48) + 'px', marginLeft: cfg.tType==='both'?'15px':0 }">{{ timeLeft }}</span>
      </div>
      
      <div id="overlay" :style="overlayStyle">
        <div id="info" :style="infoStyle" v-if="showInfo">
          <div id="title" v-if="cfg.sT">{{ activeClip?.title }}</div>
          <div id="meta">
            <span id="clipper" v-if="cfg.sC">By <span :style="{ color: cfg.hc }">{{ activeClip?.creator_name || activeClip?.broadcaster_name }}</span></span>
            <span id="date" v-if="cfg.sD"> • {{ formatDate(activeClip?.created_at) }}</span>
            <span id="game" v-if="cfg.sG"> • {{ activeClip?.game_name }}</span>
          </div>
        </div>
      </div>

      <div id="flash" :style="{ opacity: flashOpacity }"></div>
      
      <!-- VHS Noise Layer -->
      <!-- Always show if Filter is VHS OR Animation is VHS -->
      <div v-if="(cfg.flt === 'vhs' || cfg.anim === 'vhs')" class="vhs-osd">
        <div class="osd-top-left">PLAY ▶</div>
        <div class="osd-top-right">REC <span class="blink-dot">●</span></div>
        <div class="osd-bottom-left" style="max-width: 60%; white-space: nowrap; overflow: hidden; text-overflow: ellipsis;">
           {{ activeClip ? activeClip.title : 'SP' }}
        </div>
        <div class="osd-bottom-right">{{ activeClip ? formatDate(activeClip.created_at) : currentDate }}</div>
      </div>

      <div v-if="(cfg.flt === 'vhs' || cfg.anim === 'vhs')" class="vhs-scanlines"></div>
      <div v-if="(cfg.flt === 'vhs' || cfg.anim === 'vhs')" class="vhs-tracking-glitch"></div>
      <!-- Persistent Static when VHS Filter is ON, or during Rewind Anim -->
      <div v-if="(cfg.flt === 'vhs' || vhsNoiseActive)" class="vhs-static"></div>

      <!-- Bg Music -->
      <audio ref="bgAudio" loop :src="cfg.bgMusicUrl" v-if="cfg.bgMusicUrl"></audio>

       <div class="door door-l" :class="{ closing: doorsClosing }" :style="doorStyle"></div>
      <div class="door door-r" :class="{ closing: doorsClosing }" :style="doorStyle"></div>
    </div>
  </div>

  <div v-else class="setup-mode">
    <div class="container">
      <aside class="panel">
        <div class="side-scroll">
          <h1>KlipsPlayer Config</h1>
          
          <div class="group">
            <h3>1. Data Source</h3>
            <label>Mode</label>
            <select v-model="form.dataSource">
              <option value="auto">Auto (Proxy Fetch)</option>
              <option value="manual">Manual (Streamer.bot Import)</option>
            </select>
            
            <div v-if="form.dataSource === 'auto'">
              <label>Channel Name</label>
              <input type="text" v-model="form.channelName" placeholder="Twitch Channel">
              <div class="chk-row">
                <input type="checkbox" v-model="form.onlyCurrentGame" id="ocg">
                <label for="ocg">Only Current Game</label>
              </div>
              <label>Specific Game</label>
              <input type="text" v-model="form.specificGame">
              <label>Mode</label>
              <select v-model="form.mode">
                <option value="random">Random</option>
                <option value="top">Top Viewed</option>
              </select>
              <label>Period</label>
              <select v-model="form.period">
                <option value="all">All Time</option>
                <option value="30d">30 Days</option>
                <option value="7d">7 Days</option>
              </select>
              <div class="chk-row">
                <input type="checkbox" v-model="form.shuffle" id="shfl">
                <label for="shfl">Shuffle Clips</label>
              </div>
              <label>Max Length (s)</label>
              <input type="number" v-model="form.maxClipLength">
            </div>

            <div v-else>
              <label>Paste Clips JSON (from Streamer.bot)</label>
              <textarea v-model="form.manualClips" placeholder='[{"title":"Clip","playable_url":"..."}]' rows="8"></textarea>
              <p class="hint">Paste the array from your C# script's clips.js</p>
            </div>
          </div>

          <div class="group">
            <h3>2. Info Bar & Appearance</h3>
            
            <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 15px;">
              <div>
                <label>Position</label>
                <select v-model="form.infoPos">
                  <option value="bottom-left">Bottom Left</option>
                  <option value="bottom-right">Bottom Right</option>
                  <option value="top-left">Top Left</option>
                  <option value="top-right">Top Right</option>
                </select>
              </div>
              <div>
                <label>Date Format</label>
                <select v-model="form.dateFormat">
                  <option value="US">US (MM/DD/YYYY)</option>
                  <option value="EU">EU (DD/MM/YYYY)</option>
                  <option value="ISO">ISO (YYYY-MM-DD)</option>
                </select>
              </div>
            </div>

            <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 15px;">
              <div>
                <label>Style</label>
                <select v-model="form.infoShape">
                  <option value="box">Box</option>
                  <option value="rounded">Rounded</option>
                  <option value="glass">Glass</option>
                  <option value="minimal">Minimal</option>
                </select>
              </div>
            </div>

            <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 15px;">
              <div>
                <label>Visibility</label>
                <select v-model="form.infoMode">
                  <option value="always">Always On</option>
                  <option value="5">Fade 5s</option>
                  <option value="10">Fade 10s</option>
                </select>
              </div>
              <div>
                 <label>Font Size</label>
                 <input type="number" v-model="form.fontSize" min="10" max="60">
              </div>
            </div>

            <div style="margin-bottom: 15px;">
              <label>Elements</label>
              <div style="display: flex; gap: 15px; flex-wrap: wrap;">
                <label style="display:inline-flex; align-items:center; gap:5px"><input type="checkbox" v-model="form.showTitle"> Title</label>
                <label style="display:inline-flex; align-items:center; gap:5px"><input type="checkbox" v-model="form.showClipper"> Clipper</label>
                <label style="display:inline-flex; align-items:center; gap:5px"><input type="checkbox" v-model="form.showDate"> Date</label>
                <label style="display:inline-flex; align-items:center; gap:5px"><input type="checkbox" v-model="form.showGame"> Game</label>
              </div>
            </div>

            <div style="border-top: 1px solid #333; margin: 15px 0; padding-top: 15px;">
              <label>Timer Settings</label>
              <div style="display: flex; gap: 10px; margin-bottom: 10px;">
                 <label style="display:inline-flex; align-items:center; gap:5px"><input type="checkbox" v-model="form.showTimer"> Show Timer</label>
              </div>
              <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 10px;">
                <div>
                  <label>Type</label>
                  <select v-model="form.timerType">
                    <option value="numeric">Numeric</option>
                    <option value="circular">Circular</option>
                    <option value="both">Both</option>
                  </select>
                </div>
                <div>
                   <label>Position</label>
                   <select v-model="form.timerPos">
                     <option value="top-right">Top Right</option>
                     <option value="top-left">Top Left</option>
                     <option value="bottom-right">Bottom Right</option>
                     <option value="bottom-left">Bottom Left</option>
                   </select>
                </div>
              </div>
              <div style="margin-top: 10px;">
                  <label>Timer Size (px)</label>
                  <input type="number" v-model="form.timerSize" min="20" max="150">
              </div>
            </div>

            <div style="border-top: 1px solid #333; margin: 15px 0; padding-top: 15px;">
              <label>Theme Colors</label>
              <div style="display: flex; gap: 10px;">
                <div style="flex:1">
                  <label>Accent</label>
                  <input type="color" v-model="form.highlightColor" style="height:40px">
                </div>
                <div style="flex:1">
                  <label>Background</label>
                  <input type="color" v-model="form.bgColor" style="height:40px">
                </div>
              </div>
              <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-top: 10px;">
                 <div>
                    <label>Opacity</label>
                    <input type="range" v-model="form.bgOpacity" min="0" max="1" step="0.1">
                 </div>
                 <div>
                    <label>Blur</label>
                    <input type="range" v-model="form.blurStr" min="0" max="30">
                 </div>
              </div>
              
              <div style="margin-top: 15px; padding-top: 15px; border-top: 1px dashed #333;">
                 <label>Background Music (URL)</label>
                 <input type="text" v-model="form.bgMusicUrl" placeholder="https://example.com/lofi.mp3">
                 <label>Music Volume ({{ form.bgMusicVol }}%)</label>
                 <input type="range" v-model="form.bgMusicVol" min="0" max="100">
              </div>
            </div>
          </div>

          <div class="group">
            <h3>3. Animation & Effects</h3>
            <label>Transition Animation</label>
            <select v-model="form.anim">
              <option value="slide">Slide (Gapless)</option>
              <option value="doors">Doors</option>
              <option value="fade">Fade</option>
               <option value="flash">Flash</option>
              <option value="shake">Shake</option>
              <option value="blackhole">Black Hole</option>
              <option value="vhs">VHS Rewind</option>
            </select>
            
            <label>Transition Sound Effect</label>
            <select v-model="form.transSound">
              <option value="none">None</option>
              <option value="vhs">VHS Rewind</option>
              <option value="whoosh">Whoosh</option>
              <option value="ding">Ding</option>
              <option value="glitch">Glitch</option>
              <option value="custom">Custom URL</option>
            </select>
            <div v-if="form.transSound === 'custom'" style="margin-bottom: 15px;">
              <input type="text" v-model="form.transUrl" placeholder="https://.../sound.mp3">
            </div>

            <label>Duration (s)</label>
            <input type="number" v-model="form.dur" step="0.1">

            <label>Video Filter</label>
            <select v-model="form.filter">
              <option value="none">None</option>
              <option value="vhs">VHS Look</option>
              <option value="sepia">Sepia</option>
              <option value="bw">Black & White</option>
            </select>
            
            <div v-if="form.anim === 'doors'" class="door-options">
              <label>Door Color</label>
              <input type="color" v-model="form.doorColor">
              <label>Door Texture</label>
              <select v-model="form.doorTex">
                <option value="none">Solid Color</option>
                <option value="glass">Glass (Blur)</option>
                <option value="carbon">Carbon Fiber</option>
                <option value="wood">Wood Panel</option>
              </select>
              <label>Door Image URL</label>
              <input type="text" v-model="form.doorImg" placeholder="https://...">
            </div>
          </div>
        </div>

        <div class="actions">
          <button @click="openPlayer" class="apply-btn">Launch Player</button>
          <button @click="testAnim" class="test-btn">Test Animation</button>
        </div>
      </aside>

      <section class="panel preview-panel">
        <h2>Live Preview</h2>
        <div class="preview-box" id="pv" :class="previewAnimClass">
          <div class="preview-overlay" :style="previewStyle">
            <div v-if="form.showTitle"><b>Clip Title</b></div>
            <div style="font-size: 80%">
              <span v-if="form.showClipper">By <span :style="{color: form.highlightColor}">User</span></span>
              <span v-if="form.showDate"> • Date</span>
              <span v-if="form.showGame"> • Game</span>
            </div>
          </div>
          <!-- Preview Timer -->
          <div class="preview-timer" :style="previewTimerStyle" 
               v-if="form.showTimer">
               
               <div v-if="['circular', 'both'].includes(form.timerType)" class="timer-circle-preview"
                    :style="{ width: form.timerSize + 'px', height: form.timerSize + 'px' }">
                  <svg viewBox="0 0 36 36" class="circular-chart">
                    <path class="circle-bg" d="M18 2.0845 a 15.9155 15.9155 0 0 1 0 31.831 a 15.9155 15.9155 0 0 1 0 -31.831" />
                    <path class="circle-fg" 
                          :style="{ stroke: form.highlightColor }"
                          stroke-dasharray="75, 100" 
                          d="M18 2.0845 a 15.9155 15.9155 0 0 1 0 31.831 a 15.9155 15.9155 0 0 1 0 -31.831" />
                  </svg>
               </div>

               <span v-if="['numeric', 'both'].includes(form.timerType)" :style="{ fontSize: form.timerSize + 'px', marginLeft: form.timerType==='both'?'10px':0 }">30</span>
          </div>
          <div id="p-flash" class="flash-overlay" :style="{ opacity: previewFlashOpacity }"></div>
          <div v-if="vhsNoisePreview" class="vhs-static"></div>
          <div class="door door-l" :class="{ closing: doorsClosingPreview }" :style="previewDoorStyle"></div>
          <div class="door door-r" :class="{ closing: doorsClosingPreview }" :style="previewDoorStyle"></div>
        </div>
        
        <h3>OBS Browser Source URL</h3>
        <div class="url-box">
          {{ obsUrl }}
          <button @click="copyUrl" class="copy-btn">Copy</button>
        </div>
        
        <div class="note">
          <p><b>Note:</b> In "Auto" mode, we use a public CORS proxy to fetch clips. This may take a few seconds after the Start button is clicked.</p>
        </div>
      </section>
    </div>
    </div>
  </div>
</template>

<script>
const GQL_URL = "https://gql.twitch.tv/gql";
const CLIENT_ID = "kimne78kx3ncx6brgo4mv6wki5h1ko";
const PROXY = "https://corsproxy.io/?";

export default {
  data() {
    return {
      isPlayer: false,
      needsInteraction: true,
      audioInitialized: false,
      audioCtx: null,
      loading: false,
      loadProgress: 0,
      currentDate: '',
      error: null,
      showInfo: true,
      activeAnimClass: '',
      previewAnimClass: '',
      previewFlashOpacity: 0,
      isMuted: false,
      flashOpacity: 0,
      vhsNoiseActive: false,
      vhsNoisePreview: false,
      doorsClosing: false,
      doorsClosingPreview: false,
      activeVRef: 'vid1',
      nextVRef: 'vid2',
      vidClass1: 'vid-center',
      vidClass2: 'vid-right',
      timeLeft: 0,
      currentDuration: 0,
      index: 0,
      clips: [],
      activeClip: null,
      
      form: {
        dataSource: 'auto',
        channelName: '',
        manualClips: '',
        onlyCurrentGame: false,
        specificGame: '',
        mode: 'random',
        period: 'all',
        shuffle: true,
        maxClipLength: 60,
        exclude: '',
        showTitle: true,
        showClipper: true,
        showDate: true,
        showGame: true,
        infoShape: 'box',
        infoPos: 'bottom-left',
        infoMode: 'always',
        showTimer: true,
        timerPos: 'top-right',
        timerType: 'numeric',
        timerSize: 48,
        fontSize: 16,
        highlightColor: '#9146FF',
        bgColor: '#000000',
        bgOpacity: 0.7,
        blurStr: 10,
        volume: 80,
        bgMusicUrl: '',
        bgMusicVol: 20,
        anim: 'doors',
        transSound: 'vhs',
        transUrl: '',
        dur: 0.8,
        filter: 'none',
        doorColor: '#111111',
        doorTex: 'none',
        doorTex: 'none',
        doorImg: '',
        dateFormat: 'US'
      },

      cfg: {}
    }
  },

  computed: {
    obsUrl() {
      const base = window.location.origin + window.location.pathname;
      const params = new URLSearchParams();
      Object.keys(this.form).forEach(key => {
        if (this.form[key] !== '') params.set(key, this.form[key]);
      });
      return `${base}?${params.toString()}`;
    },

    videoStyle() {
      const flt = (this.isPlayer ? this.cfg.flt : this.form.filter) || 'none';
      let filter = 'none';
      if (flt === 'vhs') {
        // Clean VHS look: just contrast/brightness + scanlines/aberration (handled by CSS)
        // Removed sepia/saturate to avoid purple tint
        filter = 'blur(0.5px) contrast(1.2) brightness(1.1)';
      }
      if (flt === 'sepia') filter = 'sepia(1) brightness(0.9) contrast(1.1)';
      if (flt === 'bw') filter = 'grayscale(1) contrast(1.2) brightness(0.9)';
      return { filter };
    },

    timerDashArray() {
      if (!this.activeClip || !this.currentDuration) return "100, 100";
      const pct = (this.timeLeft / this.currentDuration) * 100;
      return `${pct}, 100`;
    },

    overlayStyle() {
      if (!this.cfg) return {};
      return {
        top: this.cfg.pos?.includes('top') ? '30px' : 'auto',
        bottom: this.cfg.pos?.includes('bottom') ? '30px' : 'auto',
        left: this.cfg.pos?.includes('left') ? '30px' : 'auto',
        right: this.cfg.pos?.includes('right') ? '30px' : 'auto',
        fontSize: (this.cfg.fs || 16) + 'px'
      };
    },

    infoStyle() {
      if (!this.cfg || !this.cfg.bg) return {};
      const bg = this.cfg.bg;
      const r = parseInt(bg.slice(1,3), 16), g = parseInt(bg.slice(3,5), 16), b = parseInt(bg.slice(5,7), 16);
      let style = { transition: 'opacity 1s' };
      if (this.cfg.shp === 'minimal') { style.background = 'transparent'; }
      else if (this.cfg.shp === 'glass') {
        style.backdropFilter = `blur(${this.cfg.bl}px)`;
        style.background = `rgba(${r},${g},${b},0.3)`;
        style.borderLeft = `4px solid ${this.cfg.hc}`;
      } else {
        style.background = `rgba(${r},${g},${b},${this.cfg.op})`;
        if (this.cfg.shp === 'box') style.borderLeft = `4px solid ${this.cfg.hc}`;
        if (this.cfg.shp === 'rounded') style.borderRadius = '20px';
      }
      return style;
    },

    timerStyle() {
      if (!this.cfg) return {};
      return {
        top: this.cfg.tp?.includes('top') ? '30px' : 'auto',
        bottom: this.cfg.tp?.includes('bottom') ? '30px' : 'auto',
        left: this.cfg.tp?.includes('left') ? '30px' : 'auto',
        right: this.cfg.tp?.includes('right') ? '30px' : 'auto'
      };
    },

    doorStyle() {
      if (!this.cfg) return {};
      let background = this.cfg.dc || '#111';
      let blur = 'none';
      let backgroundColor = this.cfg.dc; // Default solid color

      if (this.cfg.dt === 'carbon') background = `radial-gradient(black 15%,transparent 16%) 0 0,radial-gradient(black 15%,transparent 16%) 8px 8px, ${this.cfg.dc}`;
      if (this.cfg.dt === 'wood') background = 'linear-gradient(90deg, #5D4037, #3E2723)';
      if (this.cfg.dt === 'blur') { 
        // Fix: Use 10% white for glass effect + blur. FORCE transparency.
        background = 'rgba(255, 255, 255, 0.05)'; 
        backgroundColor = 'rgba(0,0,0,0) !important'; // Force override
        blur = `blur(${this.cfg.bl || 10}px)`; 
      }
      if (this.cfg.di) background = `url(${this.cfg.di}) center/cover no-repeat`;
      return { 
        background, 
        backgroundColor, 
        transitionDuration: (this.cfg.dur || 0.8) + 's',
        backdropFilter: blur,
        webkitBackdropFilter: blur,
        border: this.cfg.dt === 'blur' ? '1px solid rgba(255,255,255,0.1)' : 'none' // Add subtle border for glass
      };
    },

    previewStyle() {
      const st = this.form;
      const r = parseInt(st.bgColor.slice(1,3), 16), g = parseInt(st.bgColor.slice(3,5), 16), b = parseInt(st.bgColor.slice(5,7), 16);
      let bg = `rgba(${r},${g},${b},${st.bgOpacity})`;
      let borderLeft = 'none';
      let radius = '5px';
      let blur = 'none';

      if (st.infoShape === 'glass') { bg = `rgba(${r},${g},${b},0.3)`; blur = `blur(${st.blurStr}px)`; borderLeft = `4px solid ${st.highlightColor}`; }
      else if (st.infoShape === 'box') borderLeft = `4px solid ${st.highlightColor}`;
      else if (st.infoShape === 'rounded') radius = '20px';
      else if (st.infoShape === 'minimal') bg = 'transparent';

      return {
        background: bg, backdropFilter: blur, borderLeft: borderLeft, borderRadius: radius,
        fontSize: st.fontSize + 'px',
        top: st.infoPos.includes('top') ? '20px' : 'auto',
        bottom: st.infoPos.includes('bottom') ? '20px' : 'auto',
        left: st.infoPos.includes('left') ? '20px' : 'auto',
        right: st.infoPos.includes('right') ? '20px' : 'auto',
      };
    },

    previewTimerStyle() {
      return {
        top: this.form.timerPos.includes('top') ? '20px' : 'auto',
        bottom: this.form.timerPos.includes('bottom') ? '20px' : 'auto',
        left: this.form.timerPos.includes('left') ? '20px' : 'auto',
        right: this.form.timerPos.includes('right') ? '20px' : 'auto',
      };
    },

    previewDoorStyle() {
      const st = this.form;
      let background = st.doorColor;
      let backgroundColor = st.doorColor;
      let blur = 'none';
      
      if (st.doorTex === 'carbon') background = `radial-gradient(black 15%,transparent 16%) 0 0,radial-gradient(black 15%,transparent 16%) 8px 8px, ${st.doorColor}`;
      if (st.doorTex === 'wood') background = 'linear-gradient(90deg, #5D4037, #3E2723)';
      if (st.doorTex === 'blur') { 
         // Fix: Transparent bg for preview too
         background = 'rgba(255, 255, 255, 0.15)'; 
         backgroundColor = 'transparent';
         blur = `blur(10px)`; 
      }
      if (st.doorImg) background = `url(${st.doorImg}) center/cover no-repeat`;
      return { 
        background, 
        backgroundColor, 
        transitionDuration: st.dur + 's',
        backdropFilter: blur,
        webkitBackdropFilter: blur
      };
    }
  },

  mounted() {
    const params = new URLSearchParams(window.location.search);
    if (params.has('channelName') || params.has('manualClips')) {
      this.isPlayer = true;
      this.parseConfig(params);
      if (this.cfg.src === 'manual') {
        try {
          this.clips = JSON.parse(decodeURIComponent(params.get('manualClips')));
          this.setupFirstVideos();
        } catch(e) { console.error("Manual JSON Error:", e); }
      } else {
        this.fetchAutoClips();
      }
    }
    this.updateDate();
    setInterval(this.updateDate, 1000);
  },

  methods: {
    updateDate() {
      const now = new Date();
      this.currentDate = now.toLocaleString('en-US', { 
        month: 'short', 
        day: '2-digit', 
        year: 'numeric',
        hour: '2-digit',
        minute: '2-digit',
        hour12: false 
      }).toUpperCase();
    },
    initAudio() {
      if (this.audioInitialized) return;
      try {
        this.audioCtx = new (window.AudioContext || window.webkitAudioContext)();
        this.audioInitialized = true;
      } catch (e) {
        console.warn("Web Audio API not supported", e);
      }
    },
    playTransitionFX(config = this.cfg) {
      if (config.transSound === 'none') return;

      // Custom URL Handling
      if (config.transSound === 'custom' && config.transUrl) {
         try {
           const audio = new Audio(config.transUrl);
           audio.volume = 0.5; // Default volume for fx
           audio.play().catch(e => console.warn("Custom FX blocked", e));
         } catch(e) { console.warn("Invalid Custom FX URL"); }
         return;
      }

      // Synthesis Handling
      if (!this.audioCtx) this.initAudio(); // Auto-init if needed
      if (!this.audioCtx) return;
      if (this.audioCtx.state === 'suspended') this.audioCtx.resume();

      const t = this.audioCtx.currentTime;
      const osc = this.audioCtx.createOscillator();
      const gain = this.audioCtx.createGain();
      osc.connect(gain);
      gain.connect(this.audioCtx.destination);

      if (config.transSound === 'vhs') {
        // Rewind Whir
        osc.type = 'sawtooth';
        osc.frequency.setValueAtTime(100, t);
        osc.frequency.exponentialRampToValueAtTime(300, t + 0.5);
        osc.frequency.exponentialRampToValueAtTime(50, t + 1.2);
        gain.gain.setValueAtTime(0.1, t);
        gain.gain.exponentialRampToValueAtTime(0.001, t + 1.5);
        osc.start();
        osc.stop(t + 1.5);
        this.playStaticNoise(1.5);
      } 
      else if (config.transSound === 'whoosh') {
        // Noise Sweep
        const bufferSize = this.audioCtx.sampleRate * 0.8;
        const buffer = this.audioCtx.createBuffer(1, bufferSize, this.audioCtx.sampleRate);
        const data = buffer.getChannelData(0);
        for (let i = 0; i < bufferSize; i++) data[i] = Math.random() * 2 - 1;
        
        const noise = this.audioCtx.createBufferSource();
        noise.buffer = buffer;
        
        const filter = this.audioCtx.createBiquadFilter();
        filter.type = 'lowpass';
        filter.frequency.setValueAtTime(100, t);
        filter.frequency.exponentialRampToValueAtTime(3000, t + 0.4);
        filter.frequency.exponentialRampToValueAtTime(100, t + 0.8);

        const nGain = this.audioCtx.createGain();
        nGain.gain.setValueAtTime(0.3, t);
        nGain.gain.exponentialRampToValueAtTime(0.001, t + 0.8);

        noise.connect(filter);
        filter.connect(nGain);
        nGain.connect(this.audioCtx.destination);
        noise.start();
      }
      else if (config.transSound === 'ding') {
        // Simple Sine Decay
        osc.type = 'sine';
        osc.frequency.setValueAtTime(800, t);
        gain.gain.setValueAtTime(0.2, t);
        gain.gain.exponentialRampToValueAtTime(0.001, t + 1.0);
        osc.start();
        osc.stop(t + 1);
      }
      else if (config.transSound === 'glitch') {
        // Random Frequency Jumps
        osc.type = 'square';
        osc.frequency.setValueAtTime(200, t);
        osc.frequency.setValueAtTime(800, t + 0.1);
        osc.frequency.setValueAtTime(150, t + 0.2);
        osc.frequency.setValueAtTime(1200, t + 0.3);
        
        gain.gain.setValueAtTime(0.1, t);
        gain.gain.setTargetAtTime(0, t, 0.4);
        
        osc.start();
        osc.stop(t + 0.5);
      }
    },
    playStaticNoise(duration) {
      if (!this.audioCtx) return;
      const bufferSize = this.audioCtx.sampleRate * duration;
      const buffer = this.audioCtx.createBuffer(1, bufferSize, this.audioCtx.sampleRate);
      const data = buffer.getChannelData(0);
      for (let i = 0; i < bufferSize; i++) {
        // White noise
        data[i] = Math.random() * 2 - 1;
      }
      const noise = this.audioCtx.createBufferSource();
      noise.buffer = buffer;

      // Add Lowpass filter for "thicker" sound
      const filter = this.audioCtx.createBiquadFilter();
      filter.type = 'lowpass';
      filter.frequency.value = 800; // Muffles it to sound like bad hardware

      const noiseGain = this.audioCtx.createGain();
      noiseGain.gain.setValueAtTime(0.08, this.audioCtx.currentTime);
      noiseGain.gain.exponentialRampToValueAtTime(0.001, this.audioCtx.currentTime + duration);
      
      noise.connect(filter);
      filter.connect(noiseGain);
      noiseGain.connect(this.audioCtx.destination);
      noise.start();
    },
    parseConfig(p) {
      this.cfg = {
        src: p.get('dataSource') || 'auto',
        ch: p.get('channelName'),
        fs: parseInt(p.get('fontSize') || 16),
        hc: p.get('highlightColor') || '#9146FF',
        bg: p.get('bgColor') || '#000',
        op: p.get('bgOpacity') || '0.7',
        bl: p.get('blurStr') || '10',
        anim: p.get('anim') || 'fade',
        dur: parseFloat(p.get('dur') || 0.8),
        pos: p.get('infoPos') || 'bottom-left',
        tp: p.get('timerPos') || 'top-right',
        mode: p.get('infoMode') || 'always',
        // Timer
        tType: p.get('timerType') || 'numeric',
        tSize: parseInt(p.get('timerSize') || 48),
        tm: p.get('showTimer') !== 'false',
        
        dc: p.get('doorColor') || '#111',
        dt: p.get('doorTex') || 'none',
        di: p.get('doorImg') || '',
        vol: parseInt(p.get('volume') || 80),
        sT: p.get('showTitle') !== 'false',
        sC: p.get('showClipper') !== 'false',
        sD: p.get('showDate') !== 'false',
        sG: p.get('showGame') !== 'false',
        shp: p.get('infoShape') || 'box',
        onlyCur: p.get('onlyCurrentGame') === 'true',
        shfl: p.get('shuffle') !== 'false',
        specG: p.get('specificGame') || '',
        flt: p.get('filter') || 'none',
        bgMusicUrl: p.get('bgMusicUrl') || '',
        bgMusicVol: parseInt(p.get('bgMusicVol') || 20),
        transSound: p.get('transSound') || 'vhs',
        transUrl: p.get('transUrl') || '',
        dateFormat: p.get('dateFormat') || 'MM/DD/YYYY HH:mm'
      };
      // Legacy mapping
      if (p.has('showTimer')) this.cfg.tm = p.get('showTimer') !== 'false';
      if (p.has('showTitle')) this.cfg.sT = p.get('showTitle') !== 'false';
      if (p.has('showClipper')) this.cfg.sC = p.get('showClipper') !== 'false';
      if (p.has('showDate')) this.cfg.sD = p.get('showDate') !== 'false';
      if (p.has('showGame')) this.cfg.sG = p.get('showGame') !== 'false';
      if (p.has('shuffle')) this.cfg.shfl = p.get('shuffle') !== 'false';
      if (p.has('onlyCurrentGame')) this.cfg.onlyCur = p.get('onlyCurrentGame') === 'true';
      if (p.has('transSound')) this.cfg.transSound = p.get('transSound');
      if (p.has('transUrl')) this.cfg.transUrl = p.get('transUrl');
      if (p.has('dateFormat')) this.cfg.dateFormat = p.get('dateFormat');
      
      document.documentElement.style.setProperty('--dur', this.cfg.dur + 's');
    },

    async fetchAutoClips() {
      if (!this.cfg.ch || this.cfg.ch === 'undefined') {
        this.loading = false;
        return;
      }
      this.loading = true;
      this.error = null;
      try {
        console.log("Fetching User & Clips for:", this.cfg.ch);
        const combinedQ = [{ 
          operationName: "GetClips", 
          variables: { login: this.cfg.ch },
          query: `query GetClips($login: String) {
            user(login: $login) {
              id
              broadcastSettings { game { displayName } }
              stream { game { displayName } }
              clips(first: 50) {
                edges {
                  node {
                    slug
                    title
                    createdAt
                    game { displayName }
                    curator { displayName }
                    broadcaster { displayName }
                  }
                }
              }
            }
          }`
        }];
        
        const res = await fetch(PROXY + encodeURIComponent(GQL_URL), { 
          method: 'POST', 
          headers: { 'Client-ID': CLIENT_ID, 'Content-Type': 'application/json' }, 
          body: JSON.stringify(combinedQ) 
        });
        
        const data = await res.json();
        console.log("Twitch Combined Response:", data);
        
        const result = data[0];
        if (result.errors) throw new Error(result.errors[0].message);
        
        const user = result?.data?.user;
        if (!user) throw new Error(`User "${this.cfg.ch}" not found.`);
        
        const edges = user.clips?.edges || [];
        if (edges.length === 0) throw new Error("No clips found for this user.");

        const currentGame = user.stream?.game?.displayName || user.broadcastSettings?.game?.displayName;
        const targetGame = this.cfg.specG || currentGame;
        console.log("Game Detection -> Stream:", user.stream?.game?.displayName, "| Last:", user.broadcastSettings?.game?.displayName, "| Target:", targetGame);

        let clipList = edges.map(e => e.node);
        
        // Filter by current game if requested
        const onlyCur = this.cfg.onlyCur === true;
        if (onlyCur && targetGame) {
          console.log("Filtering by Game:", targetGame);
          const normTarget = targetGame.toLowerCase().trim();
          clipList = clipList.filter(c => {
            const gName = c.game?.displayName || "";
            return gName.toLowerCase().trim() === normTarget;
          });
          console.log(`Filter result: ${clipList.length} clips found for "${targetGame}"`);
        }

        if (clipList.length === 0) {
          console.warn("No clips found for detected game, falling back...");
          clipList = edges.map(e => e.node); // Fallback to all
        }

        // Shuffle logic (Fisher-Yates)
        if (this.cfg.shfl !== false) {
          console.log("Shuffling playlist...");
          for (let i = clipList.length - 1; i > 0; i--) {
            const j = Math.floor(Math.random() * (i + 1));
            [clipList[i], clipList[j]] = [clipList[j], clipList[i]];
          }
        }

        const fetchedClips = [];
        this.loadProgress = 10;

        const fetchToken = async (c, idx) => {
          try {
            const tokenQ = [{ 
              operationName: "VideoAccessToken_Clip", 
              variables: { slug: c.slug },
              extensions: { persistedQuery: { version: 1, sha256Hash: "36b89d2507fce29e5ca551df756d27c1cfe079e2609642b4390aa4c35796eb11" }}
            }];
            const tRes = await fetch(PROXY + encodeURIComponent(GQL_URL), { 
              method: 'POST', 
              headers: { 'Client-ID': CLIENT_ID, 'Content-Type': 'application/json' }, 
              body: JSON.stringify(tokenQ) 
            });
            const tData = await tRes.json();
            const clipToken = tData[0]?.data?.clip;
            
            if (clipToken) {
              const source = clipToken.videoQualities?.[0]?.sourceURL;
              const sig = clipToken.playbackAccessToken?.signature;
              const val = clipToken.playbackAccessToken?.value;
              fetchedClips.push({
                title: c.title,
                creator_name: c.curator?.displayName || c.broadcaster?.displayName,
                broadcaster_name: c.broadcaster?.displayName,
                game_name: c.game?.displayName,
                created_at: c.createdAt,
                playable_url: `${source}?sig=${sig}&token=${encodeURIComponent(val)}`
              });
            }
            this.loadProgress = Math.min(90, 10 + (idx / clipList.length) * 80);
          } catch (te) { console.warn("Token fetch failed for clip:", c.slug); }
        };

        // Fetch tokens in parallel (limited batches of 5 for stability)
        for (let i = 0; i < clipList.length; i += 5) {
          const batch = clipList.slice(i, i + 5);
          await Promise.all(batch.map((c, j) => fetchToken(c, i + j)));
        }
        
        this.loadProgress = 100;
        this.clips = fetchedClips;
        if (this.clips.length === 0) throw new Error("Could not find playable URLs for clips.");
        
        // Close doors over the loader before showing player
        // Immediately start player after loading finishes
        this.loading = false;
        this.setupFirstVideos();
      } catch (e) {
        console.error("Fetch Error:", e);
        this.error = e.message;
      } finally {
        this.loading = false;
      }
    },

    goToSetup() {
      window.location.search = '';
    },

    setupFirstVideos() {
      if (this.clips.length > 0) {
        this.activeClip = this.clips[0];
        this.loadVideo(this.$refs.vid1, this.activeClip);
        if (this.clips.length > 1) this.loadVideo(this.$refs.vid2, this.clips[1]);
        
        // Auto-start immediately
        this.startPlayer();
      }
    },

    startPlayer(userClick = false) {
      this.initAudio();
      if (this.loading) return;
      this.needsInteraction = false;
      const vid = this.$refs[this.activeVRef];
      if (vid) {
        // Remove the extra doors delay for initial start if user wants direct video
        vid.play().catch(e => {
          console.warn("Autoplay blocked, attempting muted fallback.", e);
          this.isMuted = true;
          vid.muted = true;
          vid.play().catch(e2 => {
              console.error("Muted autoplay failed too.", e2);
              this.needsInteraction = true;
            });
          });
          this.handleInfoVisibility();
          
          // Start BG Music
          if (this.cfg.bgMusicUrl && this.$refs.bgAudio) {
            this.$refs.bgAudio.volume = (this.cfg.bgMusicVol / 100);
            this.$refs.bgAudio.play().catch(e => console.warn("BG Music Autoplay blocked", e));
          }
      }
    },

    handleInfoVisibility() {
      if (this.cfg.mode === 'always') { this.showInfo = true; }
      else {
        this.showInfo = true;
        setTimeout(() => { this.showInfo = false; }, parseInt(this.cfg.mode) * 1000);
      }
    },

    loadVideo(el, clip) {
      if (!el || !clip) return;
      el.src = clip.playable_url;
      el.volume = this.cfg.vol / 100;
      el.load();
    },

    onVideoEnded() {
      if (this.clips.length === 1) {
        const vid = this.$refs[this.activeVRef];
        vid.currentTime = 0;
        vid.play();
        return;
      }
      const nextIdx = (this.index + 1) % this.clips.length;
      const futureIdx = (this.index + 2) % this.clips.length;
      this.triggerSwap(nextIdx, futureIdx);
    },

    triggerSwap(nextIdx, futureIdx) {
      const activeV = this.$refs[this.activeVRef];
      const nextV = this.$refs[this.nextVRef];
      const dur = this.cfg.dur * 1000;

      // Play Sound Check
      this.playTransitionFX();

      if (this.cfg.anim === 'doors') {
        this.doorsClosing = true;
        setTimeout(() => {
          this.performSwap(activeV, nextV, nextIdx, futureIdx);
          setTimeout(() => this.doorsClosing = false, 500);
        }, dur);
      } else if (this.cfg.anim === 'flash') {
        this.flashOpacity = 1;
        setTimeout(() => {
          this.performSwap(activeV, nextV, nextIdx, futureIdx);
          setTimeout(() => this.flashOpacity = 0, 300);
        }, 150);
      } else if (this.cfg.anim === 'shake') {
        this.activeAnimClass = 'anim-shake';
        setTimeout(() => {
          this.performSwap(activeV, nextV, nextIdx, futureIdx);
          setTimeout(() => this.activeAnimClass = '', 500);
        }, 200);
      } else if (this.cfg.anim === 'blackhole') {
        this.activeAnimClass = 'anim-blackhole';
        setTimeout(() => {
          this.performSwap(activeV, nextV, nextIdx, futureIdx);
          setTimeout(() => this.activeAnimClass = '', dur);
        }, dur * 0.8);
      } else if (this.cfg.anim === 'vhs') {
        this.vhsNoiseActive = true;
        this.activeAnimClass = 'anim-vhs-rewind';
        setTimeout(() => {
          this.performSwap(activeV, nextV, nextIdx, futureIdx);
          setTimeout(() => {
            this.vhsNoiseActive = false;
            this.activeAnimClass = '';
          }, 300);
        }, dur * 0.7);
      } else {
        this.performSwap(activeV, nextV, nextIdx, futureIdx);
      }
      this.handleInfoVisibility();
    },

    performSwap(activeV, nextV, nextIdx, futureIdx) {
      if (this.cfg.anim === 'slide') {
        if (this.activeVRef === 'vid1') { this.vidClass1 = 'vid-left'; this.vidClass2 = 'vid-center'; }
        else { this.vidClass2 = 'vid-left'; this.vidClass1 = 'vid-center'; }
      } else {
        if (this.activeVRef === 'vid1') { this.vidClass1 = 'vid-hidden'; this.vidClass2 = 'vid-center'; }
        else { this.vidClass2 = 'vid-hidden'; this.vidClass1 = 'vid-center'; }
      }
      this.activeClip = this.clips[nextIdx];
      nextV.play();
      const temp = this.activeVRef;
      this.activeVRef = this.nextVRef;
      this.nextVRef = temp;
      this.index = nextIdx;
      setTimeout(() => {
        if (this.activeVRef === 'vid1') this.vidClass2 = 'vid-right'; else this.vidClass1 = 'vid-right';
        this.loadVideo(this.$refs[this.nextVRef], this.clips[futureIdx]);
      }, this.cfg.dur * 1000);
    },

    onMetadataLoaded(e) {
      if (e.target && e.target.duration) {
         this.currentDuration = e.target.duration;
      }
    },

    onTimeUpdate(e) {
      const v = e.target;
      if (v.duration) {
        this.timeLeft = Math.ceil(v.duration - v.currentTime);
        this.currentDuration = v.duration; // Ensure it's in sync
      }
    },

    formatDate(date) {
      if (!date) return '';
      const d = new Date(date);
      const fmt = (this.isPlayer ? this.cfg.dateFormat : this.form.dateFormat) || 'US';
      
      if (fmt === 'EU') {
        return d.toLocaleDateString('en-GB'); // DD/MM/YYYY
      } else if (fmt === 'ISO') {
        return d.toISOString().split('T')[0]; // YYYY-MM-DD
      }
      return d.toLocaleDateString('en-US'); // MM/DD/YYYY
    },

    testAnim() {
      this.initAudio();
      this.playTransitionFX(this.form);
      const a = this.form.anim;
      if (a === 'doors') { this.doorsClosingPreview = true; setTimeout(() => this.doorsClosingPreview = false, 1500); }
      else if (a === 'flash') { this.previewFlashOpacity = 1; setTimeout(() => this.previewFlashOpacity = 0, 300); }
      else if (a === 'shake') { this.previewAnimClass = 'anim-shake'; setTimeout(() => this.previewAnimClass = '', 600); }
      else if (a === 'slide') { this.previewAnimClass = 'anim-slide-test'; setTimeout(() => this.previewAnimClass = '', 1000); }
      else if (a === 'blackhole') { this.previewAnimClass = 'anim-blackhole'; setTimeout(() => this.previewAnimClass = '', 1000); }
      else if (a === 'vhs') { 
        this.vhsNoisePreview = true;
        this.previewAnimClass = 'anim-vhs-rewind'; 
        setTimeout(() => {
          this.vhsNoisePreview = false;
          this.previewAnimClass = '';
        }, 1200); 
      }
    },

    openPlayer() { window.open(this.obsUrl, '_blank'); },
    copyUrl() { navigator.clipboard.writeText(this.obsUrl); alert('URL Copied! Paste into OBS.'); }
  }
}
</script>

<style>
:root { --dur: 0.8s; }
/* Base Layout */
.app-container { width: 100vw; height: 100vh; background: #000; position: relative; overflow: hidden; display: flex; flex-direction: column; }
.player-mode { width: 100%; height: 100%; position: relative; flex: 1; display: flex; }

/* Interaction Card */
.interaction-layer { position: absolute; inset: 0; z-index: 9999; background: #000; display: flex; align-items: center; justify-content: center; }
.interaction-card { background: #1a1a1a; padding: 3rem; border-radius: 20px; text-align: center; border: 2px solid #9146FF; box-shadow: 0 0 50px rgba(145, 70, 255, 0.3); }
.interaction-card h2 { margin-bottom: 1rem; color: #9146FF; font-weight: 900; }
.start-btn { background: #9146FF; border: none; color: white; padding: 1.5rem 3rem; font-size: 1.5rem; font-weight: 800; border-radius: 12px; cursor: pointer; transition: transform 0.2s; }
.start-btn:hover:not(:disabled) { transform: scale(1.05); }
.start-btn:disabled { opacity: 0.5; cursor: wait; }

/* Setup View */
.setup-mode { background: #0e0e10; height: 100vh; padding: 2rem; box-sizing: border-box; overflow: hidden; }
.container { display: grid; grid-template-columns: 450px 1fr; gap: 2rem; height: 100%; max-width: 1600px; margin: 0 auto; }
.panel { background: #18181b; border-radius: 16px; box-shadow: 0 10px 40px rgba(0,0,0,0.5); display: flex; flex-direction: column; overflow: hidden; }
.side-scroll { flex: 1; overflow-y: auto; padding: 2rem; scrollbar-width: thin; scrollbar-color: #9146FF #111; padding-bottom: 2rem; }
.actions { padding: 1.5rem 2rem; background: #1f1f23; border-top: 1px solid #333; display: grid; gap: 0.8rem; z-index: 100; }
.group { margin-bottom: 2.5rem; }
.group h3 { color: #bf94ff; font-size: 0.9rem; text-transform: uppercase; letter-spacing: 2px; margin-bottom: 1.5rem; border-left: 3px solid #9146FF; padding-left: 10px; }
.setup-mode label { display: block; font-size: 0.8rem; color: #adadb8; margin-bottom: 0.5rem; font-weight: 600; }
.setup-mode input, .setup-mode select, .setup-mode textarea {
  width: 100%; background: #0a0a0c; border: 1px solid #303032; color: #efeff1; padding: 0.8rem; border-radius: 8px; margin-bottom: 1.2rem; box-sizing: border-box; font-family: inherit;
}
.setup-mode input:focus { border-color: #9146FF; outline: none; }
.hint { font-size: 0.75rem; color: #666; margin-top: -1rem; margin-bottom: 1rem; }
.apply-btn { background: #9146FF; color: white; border: none; padding: 1.2rem; font-weight: 800; border-radius: 8px; cursor: pointer; font-size: 1.1rem; transition: background 0.2s; }
.apply-btn:hover { background: #772ce8; }
.test-btn { background: #323239; color: white; border: none; padding: 0.8rem; border-radius: 8px; cursor: pointer; }

/* Preview & Timer */
.preview-panel { padding: 2rem; }
.preview-box { width: 100%; aspect-ratio: 16/9; background: #000 url('https://placehold.co/1920x1080/1a1a1a/444?text=Stream+Preview') no-repeat center; background-size: cover; position: relative; overflow: hidden; border-radius: 12px; border: 4px solid #9146FF; }
.preview-overlay { padding: 1.5rem; position: absolute; z-index: 10; color: white; }
.preview-timer { position: absolute; z-index: 12; display:flex; align-items:center; justify-content:center; font-weight: 900; text-shadow: 2px 2px 8px #000; color:white; }
.flash-overlay { position: absolute; inset: 0; background: #fff; z-index: 50; transition: opacity 0.3s; pointer-events: none; }
.url-box { background: #0a0a0c; padding: 1.5rem; border-radius: 10px; border: 1px solid #333; position: relative; word-break: break-all; color: #bf94ff; font-family: 'Fira Code', monospace; font-size: 0.9rem; margin-top: 1rem; }
.copy-btn { position: absolute; right: 10px; top: 50%; transform: translateY(-50%); background: #9146FF; border: none; color: #fff; padding: 8px 15px; border-radius: 6px; cursor: pointer; font-weight: 700; }

/* SVG Timer */
.timer-circle-wrap, .timer-circle-preview { position: relative; margin: 0 5px; }
.circular-chart { display: block; margin: 0 auto; max-width: 100%; max-height: 100%; }
.circle-bg { fill: none; stroke: rgba(255,255,255,0.2); stroke-width: 3.8; }
.circle-fg { fill: none; stroke: #9146FF; stroke-width: 3.8; stroke-linecap: round; transition: stroke-dasharray 0.5s linear; }
/* Note: stroke color is now overridden inline by Vue bindings */

/* Player Animations */
#view { width: 100%; height: 100%; position: relative; overflow: hidden; background: #000; }
video { position: absolute; inset: 0; width: 100%; height: 100%; object-fit: contain; }
.vid-sliding { transition: transform var(--dur) cubic-bezier(0.4, 0, 0.2, 1), opacity var(--dur); }
.vid-center { z-index: 5; opacity: 1; transform: translateX(0); }
.vid-right { z-index: 1; opacity: 1; transform: translateX(100%); }
.vid-left { z-index: 1; opacity: 1; transform: translateX(-100%); }
.vid-hidden { z-index: 0; opacity: 0; }
#overlay { position: absolute; z-index: 300 !important; color: white; }
#info { padding: 1.5rem; border-radius: 8px; }
#title { font-size: 2em; font-weight: 900; margin-bottom: 0.2rem; }
#meta { font-size: 1rem; font-weight: 600; opacity: 0.9; }
#timer { position: absolute; display: flex; align-items: center; justify-content: center; font-weight: 900; color: white; text-shadow: 4px 4px 10px black; z-index: 300 !important; }
#flash { position: absolute; inset: 0; background: white; z-index: 100; pointer-events: none; transition: opacity 0.3s; }
.door { position: absolute; top: 0; width: 50%; height: 100%; z-index: 80; transition: transform var(--dur) ease-in-out; background-size: cover; }
.door-l { left: 0; transform: translateX(-101%); border-right: 2px solid #9146FF; }
.door-r { right: 0; transform: translateX(101%); border-left: 2px solid #9146FF; }
.door.closing { transform: translateX(0); }

@keyframes shake { 0%, 100% { transform: translate(0, 0); } 20% { transform: translate(-20px, 0); } 40% { transform: translate(20px, 0); } 60% { transform: translate(-10px, 0); } 80% { transform: translate(10px, 0); } }
.anim-shake { animation: shake 0.5s ease; }
@keyframes slide-test { 0% { transform: translateX(0); } 50% { transform: translateX(-100%); opacity: 0; } 51% { transform: translateX(100%); opacity: 0; } 100% { transform: translateX(0); opacity: 1; } }
.anim-slide-test { animation: slide-test 1s ease-in-out; }

@keyframes blackhole {
  0% { transform: scale(1) rotate(0deg); filter: brightness(1) blur(0); opacity: 1; }
  60% { transform: scale(0.2) rotate(360deg); filter: brightness(0.5) blur(10px); opacity: 0.8; }
  100% { transform: scale(0) rotate(720deg); filter: brightness(0) blur(20px); opacity: 0; }
}
.anim-blackhole video,
.anim-blackhole .preview-overlay,
.anim-blackhole .preview-timer,
.anim-blackhole #overlay,
.anim-blackhole #timer { animation: blackhole var(--dur) ease-in forwards; }

/* VHS OSD & Effects */
.vhs-osd {
  position: absolute;
  inset: 0;
  z-index: 200;
  padding: 60px;
  color: #eee;
  font-family: 'Courier New', Courier, monospace;
  font-size: 32px;
  font-weight: bold;
  text-shadow: 2px 2px 0px rgba(0,0,0,0.8), -1px -1px 0px rgba(255,255,255,0.2);
  pointer-events: none;
  opacity: 0.9;
  letter-spacing: 2px;
}

.osd-top-left { position: absolute; top: 60px; left: 80px; }
.osd-top-right { position: absolute; top: 60px; right: 80px; }
.osd-bottom-left { position: absolute; bottom: 60px; left: 80px; }
.osd-bottom-right { position: absolute; bottom: 60px; right: 80px; font-size: 24px; }

.blink-dot {
  color: #ff3333;
  animation: vhs-blink 1s steps(2, start) infinite;
}

@keyframes vhs-blink {
  to { visibility: hidden; }
}

.vhs-scanlines {
  position: absolute;
  top: 0; left: 0; width: 100%; height: 100%;
  background: linear-gradient(to bottom, rgba(255,255,255,0), rgba(255,255,255,0) 50%, rgba(0,0,0,0.5) 50%, rgba(0,0,0,0.5));
  background-size: 100% 4px;
  pointer-events: none;
  z-index: 180;
  opacity: 0.7; /* Increased visibility */
}

.vhs-tracking-glitch {
  position: absolute;
  width: 100%;
  height: 15px;
  background: rgba(255, 255, 255, 0.15);
  box-shadow: 0 0 10px rgba(255,255,255,0.2);
  z-index: 185;
  top: 0;
  filter: blur(3px);
  opacity: 0.4;
  animation: vhs-tracking 12s infinite linear;
  pointer-events: none;
}

@keyframes vhs-tracking {
  0% { top: -10%; transform: scaleY(1); }
  2% { top: 110%; transform: scaleY(3); }
  100% { top: 110%; }
}

.vhs-static {
  position: absolute;
  inset: 0;
  z-index: 190;
  background: 
    repeating-radial-gradient(#000 0 0.0001%, #fff 0 0.0002%) 50% 0/2500px 2500px,
    repeating-conic-gradient(#000 0 0.0001%, #fff 0 0.0002%) 50% 50%/2500px 2500px;
  background-blend-mode: difference;
  animation: static-shift 0.1s infinite alternate;
  opacity: 0.3; /* Increased from 0.15 */
  mix-blend-mode: hard-light; /* Stronger blend */
  pointer-events: none;
}

/* REMOVED CHROMATIC ABERRATION OVERLAYS TO KILL PURPLE TINT */
/* The user hates the purple hue, so we remove the global color overlays. */
#view.vhs-look::before, #view.vhs-look::after {
  display: none; 
}

.vhs-filter {
  filter: contrast(1.1) brightness(1.1);
}
@keyframes static-shift {
  0% { transform: scale(1); background-position: 0% 0%; }
  100% { transform: scale(1.1); background-position: 10% 10%; }
}

@keyframes vhs-rewind {
  0% { transform: scale(1) translateY(0); filter: blur(0) contrast(1) brightness(1); }
  20% { transform: scale(1.02) translateY(-5px) skewX(2deg); filter: blur(1px) contrast(1.5) brightness(1.5); }
  40% { transform: scale(1.01) translateY(5px) skewX(-2deg); filter: blur(2px) contrast(2) brightness(2); }
  60% { transform: scale(1.03) translateY(-10px); filter: blur(3px) contrast(2.5) brightness(2.5); }
  80% { transform: scale(1.01) translateY(10px); filter: blur(1px) contrast(1.5) brightness(1.5); }
  100% { transform: scale(1) translateY(0); filter: blur(0) contrast(1) brightness(1); }
}

.anim-vhs-rewind {
  animation: vhs-rewind var(--dur) ease-in-out;
}
.anim-vhs-rewind video,
.anim-vhs-rewind .preview-box,
.anim-vhs-rewind #view {
  animation: vhs-rewind 0.8s steps(10) infinite;
}
.anim-vhs-rewind #overlay,
.anim-vhs-rewind #timer,
.anim-vhs-rewind .preview-overlay,
.anim-vhs-rewind .preview-timer {
  animation: vhs-rewind 0.8s steps(5) infinite;
  opacity: 0.8;
}

/* WOW Loader Styles */
.wow-loader { position: relative; width: 400px; text-align: center; color: white; }
.loader-bg-text { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); font-size: 15rem; font-weight: 900; opacity: 0.03; letter-spacing: 20px; z-index: -1; pointer-events: none; }
.loader-logo { font-size: 4rem; margin-bottom: 1rem; animation: rocket-slide 1.5s infinite ease-in-out; }
@keyframes rocket-slide { 0%, 100% { transform: translateY(0) rotate(5deg); } 50% { transform: translateY(-20px) rotate(-5deg); } }
.wow-loader h2 { font-weight: 900; letter-spacing: 1px; margin-bottom: 2rem; background: linear-gradient(90deg, #fff, #9146FF, #fff); background-size: 200%; animation: text-shimmer 2s linear infinite; -webkit-background-clip: text; -webkit-text-fill-color: transparent; }
@keyframes text-shimmer { 0% { background-position: -200%; } 100% { background-position: 200%; } }
.progress-box { width: 100%; height: 6px; background: rgba(255,255,255,0.1); border-radius: 10px; overflow: hidden; margin-bottom: 1rem; box-shadow: 0 0 20px rgba(145, 70, 255, 0.2); }
.progress-fill { height: 100%; background: linear-gradient(90deg, #9146FF, #ff46b4); transition: width 0.3s ease-out; }
.wow-loader p { font-size: 0.9rem; opacity: 0.6; font-weight: 600; text-transform: uppercase; letter-spacing: 2px; }

.error-card { border: 2px solid #ff4646 !important; box-shadow: 0 0 50px rgba(255, 70, 70, 0.2) !important; }
.error-icon { font-size: 4rem; margin-bottom: 1rem; }

/* Player Overlays FIX */
#view { display: block !important; width: 100vw; height: 100vh; position: relative; background: #000; overflow: hidden; z-index: 1; }
video { width: 100% !important; height: 100% !important; z-index: 5; object-fit: contain; }
video { width: 100% !important; height: 100% !important; z-index: 5; object-fit: contain; }
#overlay { z-index: 300 !important; pointer-events: none; }
#timer { z-index: 300 !important; color: #fff !important; text-shadow: 0 0 10px #000, 0 0 20px #000; }
#info { pointer-events: none; }
#flash { z-index: 150 !important; }
.door { z-index: 200 !important; }

.door-options { margin-top: 10px; padding: 10px; background: rgba(0,0,0,0.2); border-radius: 6px; }
</style>
