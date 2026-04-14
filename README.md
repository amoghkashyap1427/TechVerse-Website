# Tech Verse — Website

A community platform connecting builders and startups. Built with plain HTML, CSS, and JavaScript — no framework, no build step.

---

## File Structure

```
Tech Verse/
├── index.html
├── css/
│   ├── base.css          ← design tokens & reset
│   ├── navbar.css        ← fixed nav, hamburger, mobile drawer
│   ├── hero.css          ← hero section styles
│   ├── stats.css         ← stat counter cards
│   ├── ticker.css        ← scrolling company strip
│   ├── impact.css        ← 3D carousel
│   ├── courses.css       ← courses + coming soon block
│   ├── testimonials.css  ← student review cards
│   ├── footer.css        ← footer
│   └── responsive.css    ← mobile breakpoints
└── js/
    ├── navbar.js         ← scroll behaviour + hamburger logic
    ├── animations.js     ← fade-in on scroll + hero parallax
    ├── ticker.js         ← pause ticker on hover
    ├── counters.js       ← animated number counters
    ├── slider.js         ← 3D ring carousel engine
    └── auth.js           ← Firebase Google sign-in + Firestore
```

---

## Complexities I Tackled

### 1. Infinite Scrolling Ticker

The ticker strip (company names scrolling across the screen) needs to loop forever with no visible jump or glitch. The trick is to duplicate all the items in HTML and only animate the track by exactly half its width. When the animation resets to the start, the second half looks identical to the first, so the eye never notices the cut.

```css
/* ticker.css */
@keyframes ticker {
    0%   { transform: translateX(0); }
    100% { transform: translateX(-50%); }
}
```

The edges also fade out using `::before` and `::after` pseudo-elements with a gradient that blends into the page background — so there's no harsh visible start or end, it just feels like it goes on forever.

---

### 2. 3D Ring Carousel

This was the hardest part of the whole site. The "Core Members" section shows photo cards arranged in a rotating 3D ring. The challenge is placing 12 cards at exact angles around a circle in 3D space and then rotating the whole ring smoothly, not individual cards.

Each card's position is calculated with trigonometry: the radius of the ring is derived from the card width and the number of cards. Once placed, the cards never move — only the stage rotates.

```js
// slider.js
const radius = Math.round((CARD_W * 1.5) / (2 * Math.tan(Math.PI / N)));

slides.forEach((slide, i) => {
    slide.style.transform = `rotateY(${STEP * i}deg) translateZ(${radius}px)`;
});

// to rotate forward, just subtract from a running angle
cumulativeAngle -= STEP;
stage.style.transform = `rotateY(${cumulativeAngle}deg)`;
```

Working out which card is currently at the front (for the active highlight/glow) requires modular arithmetic that always gives a positive result, even when the cumulative angle goes deeply negative:

```js
(Math.round(-cumulativeAngle / STEP) % N + N) % N;
```

The carousel also supports: keyboard arrows, touch swipe (with a 40px dead-zone so normal scrolling doesn't accidentally trigger it), dot navigation, auto-play that pauses on hover, and a "jump to" function for clicking a specific dot.

---

### 3. Frosted Glass Navbar on Scroll

The navbar starts fully transparent over the hero and only becomes a dark frosted-glass bar once the user scrolls down. A scroll listener watches how far the page has moved and directly toggles inline styles to apply the blur and background.

```js
// navbar.js
window.addEventListener('scroll', () => {
    if (window.scrollY > 60) {
        navbarWrapper.style.background       = 'rgba(8,8,8,0.85)';
        navbarWrapper.style.backdropFilter   = 'blur(24px)';
        navbarWrapper.style.webkitBackdropFilter = 'blur(24px)';
    } else {
        navbarWrapper.style.background = 'transparent';
        navbarWrapper.style.backdropFilter = 'none';
    }
});
```

The `-webkit-` prefix is there specifically for Safari, which still requires it. The CSS `transition` on the wrapper makes the change animate in smoothly rather than snapping.

---

### 4. Scroll Fade-In with Staggered Delays

Cards and headings fade up into view as the user scrolls down to them — with each card in a group appearing slightly after the previous one (the "waterfall" effect). This uses the browser's `IntersectionObserver` API, which fires a callback the moment an element enters the viewport, rather than running a loop on every scroll event.

```js
// animations.js
const fadeObserver = new IntersectionObserver(
    (entries) => {
        entries.forEach((entry) => {
            if (entry.isIntersecting) entry.target.classList.add('visible');
        });
    },
    { threshold: 0.12 }
);
```

The stagger is done by setting a computed `transitionDelay` per card index after the observer is attached:

```js
document.querySelectorAll('.stats-grid .stat-card')
    .forEach((el, i) => { el.style.transitionDelay = `${i * 0.08}s`; });
```

The actual animation is just two CSS classes — invisible by default, visible when the class is added:

```css
/* base.css */
.fade-in         { opacity: 0; transform: translateY(30px); transition: 0.6s ease; }
.fade-in.visible { opacity: 1; transform: translateY(0); }
```

---

### 5. Hero Parallax + Fade on Scroll

As the user starts scrolling past the hero, the text drifts upward and fades out simultaneously. This is done by multiplying `scrollY` by a small coefficient so it moves slower than the page scroll (parallax), and separately calculating an opacity value that reaches zero before the next section begins.

```js
// animations.js
window.addEventListener('scroll', () => {
    const scrollY = window.scrollY;
    if (heroContent && scrollY < window.innerHeight) {
        heroContent.style.transform = `translateY(${scrollY * 0.12}px)`;
        heroContent.style.opacity   = `${1 - scrollY / 650}`;
    }
});
```

The `0.12` keeps the drift subtle. If it were `1.0` the content would scroll at the same speed as the page and the effect would be invisible.

---

### 6. Animated Number Counters

The stats section shows numbers that count up from zero when they come into view. Instead of using `setInterval` (which fires at irregular intervals and can't be synchronized with the screen refresh rate), the counter uses `requestAnimationFrame`, which fires exactly once per frame — perfectly smooth.

The count-up also uses an ease-out curve so the number rushes through the low values and slows down as it approaches the target, which feels natural.

```js
// counters.js
function animateCounter(el, target, suffix, duration = 1800) {
    let startTime = null;
    function update(currentTime) {
        if (!startTime) startTime = currentTime;
        const progress = Math.min((currentTime - startTime) / duration, 1);
        const eased    = 1 - Math.pow(1 - progress, 3); // ease-out cubic
        el.textContent = Math.floor(eased * target) + suffix;
        if (progress < 1) requestAnimationFrame(update);
    }
    requestAnimationFrame(update);
}
```

An `IntersectionObserver` triggers this only once per element — it calls `unobserve` immediately so the counter doesn't restart if the user scrolls back up.

---

### 7. Firebase Google Sign-In + Firestore

Users can sign in with Google through a popup. The tricky part was making sure the Firestore user record is handled correctly for both new and returning users — a new user needs a full document created (including a skill-level map), while a returning user should only have their profile info refreshed without overwriting any stored skill data.

```js
// auth.js
async function saveUserToFirestore(user) {
    const userRef = db.collection('users').doc(user.uid);
    const doc     = await userRef.get();

    if (!doc.exists) {
        // First time — create the full record
        await userRef.set({
            displayName: user.displayName,
            email:       user.email,
            photoURL:    user.photoURL,
            createdAt:   firebase.firestore.FieldValue.serverTimestamp(),
            skills:      { dsa: 0, htmlcss: 0, javascript: 0, react: 0, nodejs: 0, python: 0 }
        });
    } else {
        // Returning — only refresh profile, keep skills intact
        await userRef.update({
            displayName: user.displayName || doc.data().displayName,
            email:       user.email       || doc.data().email,
            photoURL:    user.photoURL    || doc.data().photoURL
        });
    }
}
```

The whole auth flow is driven by `onAuthStateChanged`, which is a persistent listener — it fires on page load if the user is already logged in (restoring their session) and also fires immediately after any sign-in or sign-out. A double-init guard (`if (!firebase.apps.length)`) stops any "already initialized" errors.

---

### 8. Glassmorphism Modal + Backdrop Dismiss

The login modal uses `backdrop-filter: blur(12px)` for the frosted overlay. Closing by clicking outside the card is handled without any library — the overlay listens for clicks and checks whether the click target was inside the card or not:

```js
// auth.js
modal.addEventListener('click', (e) => {
    if (!card.contains(e.target)) modal.style.display = 'none';
});
```

---

### 9. Hamburger → X Animation

The hamburger is three `<span>` bars inside a button. When the menu opens, CSS transforms each bar independently — top bar rotates 45°, middle bar disappears, bottom bar rotates -45° — producing the X shape through pure CSS, no icon swap needed.

```css
/* navbar.css */
.hamburger.is-open span:nth-child(1) { transform: translateY(7px)  rotate(45deg);  }
.hamburger.is-open span:nth-child(2) { opacity: 0; width: 0; }
.hamburger.is-open span:nth-child(3) { transform: translateY(-7px) rotate(-45deg); }
```

The drawer closes in four different ways — button click, tapping a link, scrolling the page, and clicking anywhere outside the navbar — all handled as separate cases in `navbar.js` to prevent any edge case where the drawer stays stuck open.

---

### 10. Gradient Text

Some words (like *"Openly"* in the hero and "Coming Soon") use a white-to-blue gradient painted directly on the text characters. This requires a CSS trick because `color` can only be a single value:

```css
/* hero.css */
background: linear-gradient(110deg, #ffffff 30%, #2F8BD4 100%);
-webkit-background-clip: text;
-webkit-text-fill-color: transparent;
background-clip: text;
```

Setting `text-fill-color` to transparent makes the text invisible so the background gradient behind it shows through. Both the prefixed and unprefixed versions are declared for full browser support.

---

### 11. Responsive Without a Framework

Every font size that needs to scale fluidly between mobile and desktop uses `clamp()`, which removes the need for breakpoints just for font sizes:

```css
/* hero.css */
font-size: clamp(2.2rem, 4.8vw, 4.2rem);
```

The 3D carousel on mobile required extra work — in 3D space the cards extend beyond their bounding box, which causes horizontal scroll on phones. The fix is `overflow: hidden` on the section wrapper, which clips the out-of-bounds 3D content cleanly.

---

### 12. CSS Variable Design System

All colors, the font stack, and the border style are defined once in `base.css` as CSS custom properties. Every other CSS file uses only these variables — never raw hex values.

```css
/* base.css */
:root {
    --orange      : #2F8BD4;
    --orange-hover: #2070B0;
    --black       : #080808;
    --card-bg     : #141414;
    --gray        : #9a9a9a;
    --border      : rgba(255,255,255,0.08);
    --font        : 'Inter', sans-serif;
}
```

This means the entire site's colour scheme can be changed by editing a single file.

---

## Running Locally

No install needed. Just open `Tech Verse/index.html` in a browser.

For live-reload during editing, use VS Code's Live Server extension, or:

```bash
npx http-server "Tech Verse" -p 3000
```

> Google Sign-In only works when served over `localhost` or HTTPS — not from a `file://` path.

---

© 2026 Tech Verse · techverse505@gmail.com
