# Klips Player 📼

A highly customizable, aesthetic Twitch Clip player for OBS overlays. Featuring retro VHS effects, seamless transitions, and dynamic background atmosphere.

![Klips Player](https://placehold.co/600x400/1a1a1a/9146FF?text=Klips+Player+Preview)

## ✨ Features

*   **📺 VHS Retro Mode**: Authentic CRT scanlines, static noise, and chromatic aberration effects.
*   **🔁 Seamless Transitions**: Support for "Gapless" sliding, Doors, Flash, and VHS Rewind animations.
*   **🔊 Audio System**:
    *   **Transition SFX**: Customizable sounds (Whoosh, Ding, Glitch) on clip swaps.
    *   **Background Music**: Loop your own Lofi/Background tracks via URL.
*   **🎨 Fully Customizable**:
    *   Adjust accent colors, transparency, and blur.
    *   Position and style the Info Bar and Timer.
    *   **Circular Timer**: Visual progress indicator that syncs with your theme.
*   **🔌 Twitch Integration**: Auto-fetches clips from any channel, filters by Game, and handles "Live" status.

## 🚀 Usage

### 1. Setup
Open the player in your browser. You will see the **Configuration Panel**.
*   **Channel**: Enter the Twitch username to fetch clips from.
*   **Mode**: Choose 'Trending', 'Top 7 Days', etc.
*   **Look & Feel**: Tweak the visual style, enable VHS mode, or add background music.

### 2. OBS Setup
1.  Configure your settings in the panel.
2.  Click **"Copy URL"** at the bottom.
3.  In OBS, add a **Browser Source**.
4.  Paste the URL.
5.  Set Width: `1920`, Height: `1080` (or your canvas size).
6.  **Important**: Enable "Control audio via OBS" if you want to manage volume there.

## 🛠️ Deployment

### Self-Hosting (Vercel/Netlify)
This is a standard Vue 3 + Vite app.
1.  Fork this repo.
2.  Import to Vercel/Netlify.
3.  Deploy!

### GitHub Pages
This repo includes a GitHub Action to auto-deploy to GitHub Pages.
1.  Go to Settings -> Pages.
2.  Select **GitHub Actions** as the source.
