# APEX Arc — Technical Specification

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| react | ^19.0.0 | UI framework |
| react-dom | ^19.0.0 | DOM renderer |
| three | ^0.172.0 | 3D engine for helix gallery |
| @react-three/fiber | ^9.0.0 | React renderer for Three.js |
| @react-three/drei | ^10.0.0 | Three.js helpers (useTexture, etc.) |
| gsap | ^3.12.0 | Animation engine |
| @gsap/react | ^2.1.0 | GSAP React integration (useGSAP) |
| @studio-freight/lenis | ^1.0.0 | Smooth scroll |
| imagesloaded | ^5.0.0 | Image load detection for fold effect |
| @types/three | ^0.172.0 | Three.js type definitions |
| tailwindcss | ^4.0.0 | Utility CSS |
| @tailwindcss/vite | ^4.0.0 | Tailwind Vite plugin |
| typescript | ^5.7.0 | Type checking |
| vite | ^6.0.0 | Build tool |
| @vitejs/plugin-react | ^4.3.0 | React Vite plugin |

**Fonts** (loaded via Google Fonts CDN in index.html):
- Playfair Display (400)
- Inter (400, 500)
- JetBrains Mono (400)

## Component Inventory

### Layout

| Component | Source | Notes |
|-----------|--------|-------|
| Navigation | Custom | Fixed header, scroll-aware background transition |
| Footer | Custom | Dark-themed, two-row layout |

### Sections

| Component | Source | Notes |
|-----------|--------|-------|
| HeroSection | Custom | Contains Canvas + overlay content |
| AboutSection | Custom | Two-column with stats grid |
| TrustedBySection | Custom | Kinetic logo scroll band |
| ServicesSection | Custom | Horizontal scrollable card row |
| PortfolioSection | Custom | 3D fold grid with 5 projects |
| StatisticsSection | Custom | Dark band with slot-machine counters |
| ContactSection | Custom | Two-column form layout |

### Reusable Components

| Component | Source | Used By | Notes |
|-----------|--------|---------|-------|
| HelixGallery | Custom (R3F) | HeroSection | Three.js double-helix of images |
| FoldGridItem | Custom | PortfolioSection | Single image split into 5 folding strips |
| SlotMachineStat | Custom | StatisticsSection | Digit-column counter with roll animation |
| ServiceCard | Custom | ServicesSection | Image + number + title + description |

### Hooks

| Hook | Purpose |
|------|---------|
| useLenis | Initialize and expose Lenis instance for scroll velocity |

## Animation Implementation Table

| Animation | Library | Implementation Approach | Complexity |
|-----------|---------|------------------------|------------|
| 3D Helix Gallery | Three.js + R3F | useFrame loop updates mesh positions along helical paths; pointer events control rotation speed with inertia damping | **High** 🔒 |
| 3D Image Fold Scroll | GSAP ScrollTrigger | Per-image timeline: strip containers animate from offset/scaled state, then rotationY fold based on strip position; scrubbed to scroll | **High** 🔒 |
| Kinetic Logo Scroll | GSAP + Lenis | Two infinite-looping gsap.to tweens; Lenis velocity added/subtracted from tween time each frame | **Medium** |
| Slot Machine Counter | GSAP ScrollTrigger | Per-digit: animate yPercent from -1000 to -(digit*10) with random digit cycling via onRepeat callback; blur filter tween simultaneous | **Medium** |
| Section Entrance Animations | GSAP ScrollTrigger | Batch pattern: translateY(40)+opacity, stagger 0.1s, trigger at 80% viewport | **Low** |
| Nav Scroll Transition | CSS + scroll listener | Toggle class at 100px scroll; CSS handles background/blur/border transition | **Low** |
| Service Cards Entrance | GSAP ScrollTrigger | translateX(60)+opacity, stagger 0.12s | **Low** |
| Contact Section Entrance | GSAP ScrollTrigger | Left col translateX(-40), right col translateX(40), simultaneous | **Low** |
| Scroll Indicator Bounce | CSS | translateY keyframe animation, 2s infinite | **Low** |
| Image Hover Scale | CSS | transform scale(1.03), transition 0.5s, overflow hidden on parent | **Low** |

## State & Logic Plan

### Lenis ↔ GSAP Ticker Sync

Lenis must be connected to GSAP's ticker so ScrollTrigger and all scroll-driven animations use the same time source. On every GSAP ticker tick, call `lenis.raf(time * 1000)`. This is the single most critical integration — all scroll-dependent effects (fold, kinetic logo, entrance triggers) depend on it.

### Helix Gallery — Shared Geometry Strategy

All 36 mesh instances share a single `PlaneGeometry(0.8, 1.2)` to minimize GPU memory. Each mesh gets its own `MeshBasicMaterial` with a unique texture. Position/rotation data is precomputed once and stored in a ref array; the `useFrame` loop reads this data and applies transforms. The `rotationSpeed` ref is mutated directly (not via React state) for 60fps performance.

### Fold Grid — Image Load Dependency

The fold effect requires all portfolio images to be fully loaded before initializing ScrollTrigger measurements. Wrap fold initialization in `imagesloaded(container, callback)`. After load completes, calculate strip widths, set background-position-x offsets, then create timelines. Call `ScrollTrigger.refresh()` after setup.

### Kinetic Logo — Width Measurement Timing

Logo scroll speeds depend on the total width of each row's inner wrapper. These widths must be measured after DOM mount and image/logo load. Use `useLayoutEffect` with `getBoundingClientRect()` on the inner wrappers. Store widths in refs (not state) to avoid re-renders.

### Slot Machine — Digit Strip DOM Structure

Each digit column contains a strip div with 10 digit spans (0-9). The strip's final `yPercent` position is `-(finalDigit * 10)` — this maps digit 0 to yPercent 0, digit 1 to yPercent -10, etc. The GSAP animation tweens from `yPercent: -1000` (well above) to the calculated target. A `repeat: -1` with `onRepeat` callback randomizes the strip's innerText during the roll phase.

## Other Key Decisions

### Three.js via R3F (not vanilla)

The helix gallery uses React Three Fiber for declarative Three.js integration within React. This avoids manual canvas lifecycle management and integrates cleanly with React's render cycle. The `<Canvas>` component is placed inside HeroSection with absolute positioning.

### No shadcn/ui Components

This design is fully custom with no standard UI patterns (forms, dialogs, tables, etc.) that would benefit from shadcn/ui. All components are bespoke. The contact form uses native HTML inputs with custom Tailwind styling.

### Asset Strategy

18 helix images + 4 service images + 5 portfolio images = 27 images total. All are AI-generated and placed in `public/images/`. Helix images are portrait (3:4) to match the PlaneGeometry aspect ratio. Portfolio and service images are landscape (4:3).

### Performance Considerations

- Helix images: compress to WebP, target <150KB each
- Use `will-change: transform` on fold strips and logo items
- The fold effect creates 25 DOM elements per image (5 strips × 5 images) — acceptable for 5 portfolio items
- Lenis smooth scroll with lerp 0.1 provides smooth feel without excessive lag
