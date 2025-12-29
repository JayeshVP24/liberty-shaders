# 🎬 Webflow Video Shader Integration Guide

## 📦 What You Need
- The `frames` folder (228 WebP images, 3.6MB total)
- The JavaScript code below
- A Webflow project with Designer access

---

## 🚀 Step-by-Step Setup

### Step 1: Upload Frames to Webflow

1. **Go to your Webflow project** → **Assets panel**
2. **Create a folder** called `frames` (optional but organized)
3. **Upload all 228 WebP files** from your frames folder
   - Tip: You can select all and drag-drop them at once
4. **Note the URL structure** - it will be something like:
   ```
   https://uploads-ssl.webflow.com/YOUR_SITE_ID/...
   ```

---

### Step 2: Add HTML Structure in Webflow

1. **Add a Div Block** to your page (where you want the video shader)
2. **Set the div's custom attributes**:
   - Click the div → Settings (⚙️) icon
   - Add Custom Attribute:
     - **Name**: `shader`
     - **Value**: `pixel`

3. **Add an Image inside the div**:
   - Drag an Image element into the shader div
   - Set image src to: `./frames/frame_0001.webp` (or full Webflow URL)
   - Add Custom Attribute to the image:
     - **Name**: `crossorigin`
     - **Value**: `anonymous`

---

### Step 3: Style the Shader Container

In Webflow Designer, select your shader div and set these styles:

```
Position: Fixed
Top: 0
Left: 0
Width: 100vw
Height: 100vh
Background: Black (#000000)
Overflow: Hidden
Z-index: (set as needed for your layout)
```

**For the image inside**:
```
Position: Absolute
Opacity: 0
Pointer Events: None
```

---

### Step 4: Add the JavaScript Code

1. **Go to Project Settings** → **Custom Code** tab
2. **In "Footer Code"** (before `</body>` tag), paste this:

```html
<script type="module">
  import {
    ShaderMount,
    HalftoneDotsGrids,
    HalftoneDotsTypes,
    ShaderFitOptions,
    getShaderColorFromString,
    halftoneDotsFragmentShader,
    defaultObjectSizing
  } from "https://esm.sh/@paper-design/shaders@0.0.68";

  import { animate } from "https://esm.sh/motion@12.23.26";

  /* =====================================================
     VIDEO FRAMES CONFIG
  ===================================================== */
  const TOTAL_FRAMES = 228;
  const FPS = 30;
  
  // ⚠️ IMPORTANT: Update this path to match your Webflow asset path
  const FRAMES_DIR = "https://uploads-ssl.webflow.com/YOUR_SITE_ID/frames";
  
  const frameImages = [];
  let framesLoaded = 0;

  // Preload all frames
  console.log("Preloading video frames...");
  for (let i = 1; i <= TOTAL_FRAMES; i++) {
    const img = new Image();
    img.crossOrigin = "anonymous";
    const paddedIndex = String(i).padStart(4, "0");
    img.src = `${FRAMES_DIR}/frame_${paddedIndex}.webp`;
    
    img.onload = () => {
      framesLoaded++;
      if (framesLoaded === TOTAL_FRAMES) {
        console.log("All frames loaded! Starting animation...");
        startVideoAnimation();
      }
    };
    
    img.onerror = (e) => {
      console.error(`Failed to load frame ${i}:`, e);
    };
    
    frameImages.push(img);
  }

  /* =====================================================
     INIT SHADER
  ===================================================== */
  const canvas = document.querySelector('[shader="pixel"]');
  const img = canvas?.querySelector("img");
  
  if (!canvas || !img) {
    console.error("Shader container or image not found");
  }

  img.style.opacity = "0";
  img.style.pointerEvents = "none";

  const image = img.src;

  /* ---------- PRESET ---------- */
  const preset = {
    ...defaultObjectSizing,
    fit: "cover",
    speed: 0,
    frame: 0,
    colorBack: "#000000",
    colorFront: "#ffffff",
    size: 0.6,
    radius: 2,
    contrast: 0.01,
    originalColors: true,
    inverted: false,
    grainMixer: 0,
    grainOverlay: 0,
    grainSize: 0.5,
    grid: "square",
    type: "holes",
    image
  };

  /* ---------- UNIFORMS ---------- */
  const uniformsProp = {
    u_image: preset.image,
    u_size: preset.size,
    u_radius: preset.radius,
    u_contrast: preset.contrast,
    u_originalColors: preset.originalColors,
    u_inverted: preset.inverted,
    u_grainMixer: preset.grainMixer,
    u_grainOverlay: preset.grainOverlay,
    u_grainSize: preset.grainSize,
    u_rotation: preset.rotation,
    u_scale: preset.scale,
    u_offsetX: preset.offsetX,
    u_offsetY: preset.offsetY,
    u_originX: preset.originX,
    u_originY: preset.originY,
    u_worldWidth: preset.worldWidth,
    u_worldHeight: preset.worldHeight,
    u_colorFront: getShaderColorFromString(preset.colorFront),
    u_colorBack: getShaderColorFromString(preset.colorBack),
    u_grid: HalftoneDotsGrids[preset.grid],
    u_type: HalftoneDotsTypes[preset.type],
    u_fit: ShaderFitOptions[preset.fit]
  };

  async function processUniforms(props) {
    const out = {};
    const promises = [];

    Object.entries(props).forEach(([k, v]) => {
      if (typeof v === "string") {
        const i = new Image();
        i.crossOrigin = "anonymous";
        const p = new Promise((res, rej) => {
          i.onload = () => { out[k] = i; res(); };
          i.onerror = rej;
        });
        i.src = v;
        promises.push(p);
      } else {
        out[k] = v;
      }
    });

    await Promise.all(promises);
    return out;
  }

  // Initialize shader
  let shader;
  
  (async () => {
    const uniforms = await processUniforms(uniformsProp);

    shader = new ShaderMount(
      canvas,
      halftoneDotsFragmentShader,
      uniforms,
      {},
      preset.speed,
      preset.frame,
      preset.minPixelRatio,
      preset.maxPixelCount,
      preset.mipmaps
    );

    // Force initial dotted state
    shader.setUniforms({ u_size: 1 });

    /* ---------- CONTROLLER ---------- */
    const controller = {
      started: false,
      shaderInstance: shader,
      start() {
        if (this.started) return;
        this.started = true;

        // Animate from dots (1.0) to clear (0.1)
        animate(1, 0.1, {
          duration: 6,
          ease: [0.36, -0.01, 0.41, 1.01],
          onUpdate: (u_size) => {
            shader.setUniforms({ u_size });
          }
        });
      }
    };

    // Auto-start when element is visible (25% threshold)
    const selfObserver = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            controller.start();
            selfObserver.unobserve(entry.target);
          }
        });
      },
      { threshold: 0.25 }
    );

    selfObserver.observe(canvas);
  })();

  /* =====================================================
     VIDEO ANIMATION LOOP
  ===================================================== */
  function startVideoAnimation() {
    let currentFrame = 0;
    const interval = 1000 / FPS;

    setInterval(() => {
      if (frameImages.length < TOTAL_FRAMES || !shader) return;

      const nextImage = frameImages[currentFrame];
      shader.setUniforms({ u_image: nextImage });

      currentFrame = (currentFrame + 1) % TOTAL_FRAMES;
    }, interval);
  }
</script>
```

---

## ⚙️ Configuration Options

### Adjust Animation Settings

In the code, you can modify:

```javascript
// Animation duration (currently 6 seconds)
duration: 6,

// Starting dot size (1.0 = full dots, 0.1 = almost invisible)
animate(1, 0.1, {

// Frame rate (30fps, can reduce to 15 for lighter load)
const FPS = 30;

// Colors
colorBack: "#000000",  // Background color
colorFront: "#ffffff", // Dot color (when not using originalColors)

// Keep video colors or use solid color dots
originalColors: true,  // true = keep video colors, false = use colorFront
```

---

## 🎨 Alternative: Use as Section Background

If you want it as a section background (not full screen):

1. Change the div from **Fixed** to **Absolute** positioning
2. Set the parent section to **Relative** positioning
3. Adjust width/height as needed

---

## 🐛 Troubleshooting

### Frames not loading?
- Check browser console for errors
- Verify the `FRAMES_DIR` path matches your Webflow asset URLs
- Check that all 228 frames uploaded successfully
- Make sure CORS is enabled (crossorigin="anonymous")

### Shader not appearing?
- Verify the div has `shader="pixel"` attribute
- Check that the image inside has a valid src
- Open browser console and look for error messages

### Performance issues?
- Reduce FPS from 30 to 15
- Ensure frames are the optimized WebP versions (13KB each)
- Consider lazy loading or only starting animation when in viewport

---

## 📁 File Structure

After uploading to Webflow, your assets should look like:

```
Webflow Assets/
├── frames/
│   ├── frame_0001.webp
│   ├── frame_0002.webp
│   ├── ...
│   └── frame_0228.webp
```

---

## 💡 Tips

1. **Test locally first** - Use the `webflow-video-shader.html` file to test before deploying
2. **Publish to staging** - Test on a staging domain before going live
3. **Monitor performance** - Check load times with 3.6MB of images
4. **Consider CDN** - Webflow uses a CDN, but you can also host on Cloudflare/AWS S3
5. **Add loading state** - Consider showing a loader while frames preload

---

## 🔗 Resources

- Paper Design Shaders: https://www.npmjs.com/package/@paper-design/shaders
- Motion library: https://motion.dev/
- Webflow Custom Code: https://university.webflow.com/lesson/custom-code-in-the-head-and-body-tags

---

Need help? Check the browser console for detailed error messages!

