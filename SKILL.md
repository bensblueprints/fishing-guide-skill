# Fishing Guide Website Builder

Build a complete fishing guide/charter website with admin dashboard from scratch.

## Input: $ARGUMENTS

If no arguments provided, use the conversation context to gather: business name, captain name, phone, email, dock address, pricing, existing website URL, and logo file location.

## Template Repository
Clone from: `bensblueprints/sml-wicked-striper` (the proven blueprint)
Local template: `C:\Users\admin\Desktop\Github Repo Backup 2-18-2026\sml-wicked-striper\`

## Execution Steps

### Phase 1: Gather Info
Extract from arguments or conversation:
- **Business name** (e.g. "SML Wicked Striper Guide Service LLC")
- **Captain/guide name** (e.g. "Captain Tommy Moore")
- **Phone number**
- **Email address**
- **Dock/launch address** (for Google Maps directions link)
- **Pricing tiers** (trip types, hours, base price, per-person add-on, max guests)
- **Existing website URL** (to scrape photos from)
- **Logo file path** (check Downloads folder for logo.png or similar)
- **Google Maps Place ID** (search via Snapchat place page or Google)
- **Facebook page URL** (for additional photos)
- **Target species** (what fish they go after — needed for species guide section)
- **Body of water** (lake, bay, ocean — determines saltwater vs freshwater license)
- **Nearby cities** (for area guide section — get drive times from dock address)

### Phase 1b: Detect Site Platform
Before scraping, detect what platform the site runs on. Try these in order:
1. **WordPress** — Try `{site}/wp-json/wp/v2/media?per_page=1`. If 200 response → WordPress. Use WP REST API for everything.
2. **Wix** — Check page source for `wix.com`, `static.wixstatic.com`, `_api/v2`. Wix images are on `static.wixstatic.com/media/` CDN.
3. **Squarespace** — Check for `squarespace.com`, `static1.squarespace.com`, `/api/`. Images on `images.squarespace-cdn.com`.
4. **Shopify** — Check for `cdn.shopify.com`, `myshopify.com`.
5. **GoDaddy/Website Builder** — Check for `godaddy.com`, `img1.wsimg.com`.
6. **CloudFront/WAF Protected** — If you get 403 from both direct fetch AND Playwright, the site has bot protection. Fall back to third-party sources.
7. **Static/Custom** — If none of the above, it's a custom site. Use Playwright only.

**Platform-specific scraping strategies:**
- **WordPress**: Use REST API (`/wp-json/wp/v2/media`, `/wp-json/wp/v2/pages`, `/wp-json/wp/v2/posts`). Fastest and most reliable.
- **Wix**: Use Playwright. Images are lazy-loaded. Scroll the full page. Image URLs contain `static.wixstatic.com/media/` — append `/v1/fill/w_1920,h_1080` to get full-size versions.
- **Squarespace**: Use Playwright. Images use `data-src` for lazy loading. Append `?format=2500w` to image URLs for max resolution.
- **CloudFront-blocked sites**: Skip the site entirely. Scrape from FishingBooker, Guidesly, FishAnywhere, Google Maps, Facebook, Yelp, TripAdvisor. These third-party listing sites often have the same photos uploaded by the business.
- **Any platform**: ALWAYS also scrape FishingBooker, Guidesly, FishAnywhere, Facebook, and Google Images as backup sources. Some sites block bots but their photos are on these platforms at full resolution.

**Getting photos by any means necessary (priority order):**
1. WordPress REST API (fastest, full-res)
2. Playwright browse with scroll + lazy-load triggers
3. FishingBooker listing (often has 10-20 gallery photos)
4. Guidesly listing (has profile + trip photos on CloudFront CDN, often full-res)
5. FishAnywhere listing (AWS S3 hosted gallery)
6. Facebook page photos tab (scroll to load more)
7. Google Maps listing photos
8. Yelp listing photos
9. TripAdvisor listing photos
10. Google Image Search as absolute last resort

### Phase 2: Scrape Photos & Identify Key Images
Use the platform detection from Phase 1b to choose the right scraping strategy:
1. Try platform-specific API first (WordPress REST, Wix API, etc.)
2. Fall back to Playwright with aggressive scrolling and lazy-load detection
3. ALWAYS also scrape third-party listing sites for additional/backup photos
4. Download all images >10KB to `images/` folder using Node.js (NOT Python — not installed)
5. Also scrape Google Maps listing and Facebook page for additional photos
6. Check Downloads folder for any pre-downloaded photos (webp files etc.)
7. Remove business cards, flyers, tracking pixels, tiny icons, map screenshots
8. Find and copy logo to `images/logo.png`
9. **Hero image MUST be >100KB** — if no scraped image is large enough, keep scraping other sources until you get at least one high-quality photo

### Phase 2b: Browse Site with Playwright to Identify Captain & Boat Photos
**CRITICAL**: Use Playwright to browse the actual website and identify:
1. **Captain photo** — find the "Meet the Captain" or "About" page, look for the captain portrait. Save as `captain-{name}.jpg`. Be very selective — this MUST be the actual captain, not a customer.
2. **Boat photo** — find images showing the actual charter boat (name visible, boat in water, stern shot). Save as `boat-on-water.jpg`, `smokin-gun-ii.jpg`, etc.
3. **Social media links** — scrape ALL social links (Facebook, Instagram, YouTube, TikTok, Twitter/X) from every page. Store for Phase 3.

**Playwright browse script pattern:**
```javascript
import { chromium } from 'playwright';
// Install locally first: npm install playwright
const browser = await chromium.launch();
const page = await browser.newPage();
await page.goto(url, { waitUntil: 'networkidle' });
await page.waitForTimeout(3000); // let Divi/JS frameworks render

// Collect all image src URLs
const images = await page.evaluate(() =>
  [...document.querySelectorAll('img')].map(img => ({
    src: img.src, alt: img.alt, classes: img.className,
    width: img.naturalWidth, height: img.naturalHeight
  }))
);

// Collect all social media links
const socialLinks = await page.evaluate(() =>
  [...document.querySelectorAll('a[href]')].map(a => a.href)
    .filter(h => /facebook|instagram|youtube|tiktok|twitter|x\.com/i.test(h))
);

// Also check CSS background-image for hero/banner images
const bgImages = await page.evaluate(() => {
  const els = document.querySelectorAll('*');
  const bgs = [];
  els.forEach(el => {
    const bg = getComputedStyle(el).backgroundImage;
    if (bg && bg !== 'none' && bg.includes('url(')) {
      bgs.push(bg.match(/url\("?(.+?)"?\)/)?.[1]);
    }
  });
  return bgs.filter(Boolean);
});
```

Look for captain-specific pages: `/about`, `/meet-the-captain`, `/captain`, `/our-captain`, `/the-crew`
Look for boat pages: `/the-boat`, `/our-boat`, `/vessel`, `/fleet`
Download the identified captain and boat images with descriptive filenames.

### Phase 2c: Visual Dedup with Playwright Perceptual Hashing
**CRITICAL**: WordPress sites often have duplicate images uploaded with different filenames. SHA-256 only catches exact copies. You MUST run visual dedup using Playwright perceptual hashing after all downloads.

**How it works:**
1. Install Playwright locally: `npm install playwright`
2. Load each image as a base64 data URI into a Playwright page
3. Draw each image to a 16x16 canvas, extract grayscale pixel data
4. Compute average hash (pixel > average = 1, else 0)
5. Compare all pairs using hamming distance
6. 90%+ similarity = duplicate → keep highest resolution, delete others

**Visual dedup script** (save as `visual-dedup.mjs` and run with `node visual-dedup.mjs`):
```javascript
import { chromium } from 'playwright';
import fs from 'fs';
import path from 'path';

const DIR = './images';
const THUMB_SIZE = 16;
const SIMILARITY_THRESHOLD = 0.90;

function getBase64DataUri(filePath) {
  const ext = path.extname(filePath).toLowerCase();
  const mime = ext === '.png' ? 'image/png' : ext === '.webp' ? 'image/webp' : 'image/jpeg';
  return 'data:' + mime + ';base64,' + fs.readFileSync(filePath).toString('base64');
}

async function run() {
  const browser = await chromium.launch();
  const page = await (await browser.newContext()).newPage();
  await page.setContent('<html><body></body></html>');

  const files = fs.readdirSync(DIR)
    .filter(f => /\.(jpg|jpeg|png|webp)$/i.test(f) && f !== 'logo.png').sort();

  const hashes = {};
  for (const file of files) {
    const dataUri = getBase64DataUri(path.join(DIR, file));
    const pixelData = await page.evaluate(async ({ dataUri, size }) => {
      return new Promise((resolve) => {
        const img = new Image();
        img.onload = () => {
          const canvas = document.createElement('canvas');
          canvas.width = size; canvas.height = size;
          const ctx = canvas.getContext('2d');
          ctx.drawImage(img, 0, 0, size, size);
          const data = ctx.getImageData(0, 0, size, size).data;
          const gray = [];
          for (let i = 0; i < data.length; i += 4)
            gray.push(Math.round(data[i]*0.299 + data[i+1]*0.587 + data[i+2]*0.114));
          resolve({ gray, width: img.naturalWidth, height: img.naturalHeight });
        };
        img.onerror = () => resolve(null);
        img.src = dataUri;
      });
    }, { dataUri, size: THUMB_SIZE });
    if (!pixelData) continue;
    const avg = pixelData.gray.reduce((a,b)=>a+b,0) / pixelData.gray.length;
    hashes[file] = {
      hash: pixelData.gray.map(g => g > avg ? '1' : '0').join(''),
      width: pixelData.width, height: pixelData.height,
      size: fs.statSync(path.join(DIR, file)).size
    };
  }

  // Compare all pairs
  const fileList = Object.keys(hashes);
  const inGroup = new Set();
  for (let i = 0; i < fileList.length; i++) {
    if (inGroup.has(fileList[i])) continue;
    const group = [fileList[i]];
    for (let j = i+1; j < fileList.length; j++) {
      if (inGroup.has(fileList[j])) continue;
      let matches = 0;
      for (let k = 0; k < hashes[fileList[i]].hash.length; k++)
        if (hashes[fileList[i]].hash[k] === hashes[fileList[j]].hash[k]) matches++;
      if (matches / hashes[fileList[i]].hash.length >= SIMILARITY_THRESHOLD) {
        group.push(fileList[j]); inGroup.add(fileList[j]);
      }
    }
    if (group.length > 1) {
      inGroup.add(fileList[i]);
      const sorted = group.sort((a,b) =>
        (hashes[b].width*hashes[b].height) - (hashes[a].width*hashes[a].height) || hashes[b].size - hashes[a].size
      );
      console.log('KEEP: ' + sorted[0]);
      for (const r of sorted.slice(1)) {
        fs.unlinkSync(path.join(DIR, r));
        console.log('  REMOVED: ' + r);
      }
    }
  }
  await browser.close();
  console.log('Remaining:', fs.readdirSync(DIR).filter(f => /\.(jpg|jpeg|png|webp)$/i.test(f)).length);
}
run();
```

**Run order:**
1. First run SHA-256 exact dedup (fast, catches identical files)
2. Then run Playwright visual dedup (catches renamed/re-encoded copies)

### Phase 3: Create Project
1. Create project directory in `C:\Users\admin\Desktop\Github Repo Backup 2-18-2026\{slug}\`
2. Add `.gitignore` with: `node_modules/`, `package-lock.json`, `package.json`, `*.mjs`, `*.png` screenshot patterns
3. Copy `index.html` from template, customize ALL of:
   - Business name, captain name throughout
   - Phone number, email, dock address
   - Pricing cards (trip types, hours, prices, add-on per person)
   - **Captain photo**: Use the REAL captain photo from Phase 2b (`captain-{name}.jpg`) in the About section. Do NOT use a random fish/customer photo as the captain image.
   - **Boat photo**: Use the REAL boat photo from Phase 2b (`boat-on-water.jpg`) in the Boat section and as hero background. The hero and OG image should show the actual charter boat.
   - **CTA banner background**: Use second captain photo or action shot
   - Gallery photo array (all local `images/` paths — only non-duplicate files after Phase 2c dedup)
   - Google Maps directions URL (dock address)
   - Fishing license link (state-appropriate: saltwater for bay/ocean, freshwater for lakes)
   - OG meta tags (title, description, image — use boat photo for OG image)
   - Testimonials based on Google review sentiment
   - **Social media links**: Add ALL social links found in Phase 2b to:
     - Footer (social icon circles with SVG icons + hover color per platform)
     - Contact section (card for each platform found)
     - Schema.org `sameAs` array
   - Footer text with social icon row
   - **ALL SEO content sections** (see Phase 3b below)
4. Copy `admin.html` from template, customize:
   - Generate new admin password, update SHA-256 hash
   - SITE_ID and NETLIFY_TOKEN will be updated after Netlify site creation
5. Copy all photos into `images/` folder
6. Copy logo to `images/logo.png`

**Image placement rules:**
- `About section` → Captain portrait photo (from /meet-the-captain or /about page)
- `Hero background` → Best boat-on-water or scenic fishing photo
- `Boat section` → Boat photo showing the vessel clearly (stern shot or on water)
- `CTA banner` → Second captain photo or fishing action shot
- `OG/Twitter image` → Same as hero (boat photo)
- `Gallery` → ALL remaining non-duplicate photos (captain, boat, catches, customers)

**Social media integration:**
- Scrape ALL social links from website using Playwright (Facebook, Instagram, YouTube, TikTok, Twitter/X)
- Add SVG icon for each found platform in footer with hover effect (FB=#1877F2, IG=#E4405F, YT=#FF0000, TT=#000000, X=#000000)
- Add to Schema.org `sameAs` array
- Add as contact card if only 1-2 social platforms found
- Common page locations to check for social links: header, footer, sidebar, contact page, about page

### Phase 3b: SEO Content Sections (MANDATORY)
Every site MUST include these content-heavy SEO sections in addition to the base template sections. These are critical for ranking:

**1. Species Guide Section** (`#species`)
- Grid of cards for each target species (4-column grid)
- Each card: emoji icon, species name, season range, 3-4 sentence description with fishing techniques and local context
- Below the grid: 2-column text block (3-4 paragraphs) explaining why this body of water is a world-class fishery, the local ecosystem, structure/habitat, and how the captain's experience translates to finding fish
- Target keywords: "[species] fishing [location]", "[species] charter [state]"

**2. Seasonal Fishing Calendar** (`#seasons`)
- 4 season cards (Spring, Summer, Fall, Winter) in a grid
- Each card: colored season name, month range, 3-4 sentence description of what fishing is like that season, species tag pills
- Below: 1-2 paragraphs about year-round fishing opportunities and a booking tip with phone CTA
- Navy background section (matches pricing/testimonials aesthetic)

**3. The Boat Section** (`#boat`)
- 2-column layout: boat photo (left) + text/specs (right)
- Boat name as heading, 2 paragraphs about the vessel's capabilities
- Spec grid: length, passengers, certification, home port
- Feature list with check icons: electronics, cabin, shade, restroom, safety gear, fishing equipment
- Target keywords: "[boat name] charter boat", "charter boat [location]"

**4. What to Bring Section** (`#prepare`)
- 3-column grid: Essentials, Food & Comfort, Nice to Have
- Each column: 6 items with check icons
- Navy background section
- Practical items: fishing license, sunscreen, sunglasses, hat, shoes, food/drinks, rain gear, motion sickness meds, cooler, camera, cash for gratuity, waterproof phone case

**5. Area Guide / Service Area** (`#area`)
- Grid of city cards (4-column) for every nearby city within 2 hours
- Each card: city name, drive time, 3-4 sentences about why visitors from that city should book this charter
- Below: 2-column text block about the broader region, why it's great for fishing, tourism tie-ins
- Target keywords: "fishing charter near [city]", "charter fishing [city] VA"

**6. FAQ Accordion** (`#faq`)
- 10-12 expandable FAQ items with toggle animation
- Must cover: pricing, fishing license, target species, what's included, family friendly, best season, booking timeline, what to bring, keeping fish, weather policy, directions, corporate/events
- Each answer: 3-5 sentences with internal links to other sections (#seasons, #prepare, etc.)
- Duplicate key FAQs in Schema.org FAQ structured data (15+ questions)

**7. Expanded Meta & Schema:**
- Keywords meta: 30+ long-tail keywords covering species, locations, occasion types
- FAQ Schema: 15+ questions (more than the visible FAQ — add booking, weather, boat, tipping, etc.)
- LocalBusiness schema with full address, geo, rating, offers, area served

**Section order in HTML:**
Nav → Hero → About (expanded 3+ paragraphs) → Species Guide → Seasonal Calendar → The Boat → Pricing → Gallery → Booking → Testimonials → What to Bring → Area Guide → FAQ → Fishing License Banner → CTA Banner → Contact → Footer

### Phase 4: Design Customization
- Default: Navy/gold/teal color scheme (works great for fishing/water)
- If client has different brand colors, update CSS `:root` variables
- Logo displays in nav (white bg on scroll) with text fallback
- Hero uses best boat/water photo (MUST be >100KB, not a thumbnail)
- About section uses captain with catch photo
- Boat section uses best boat photo
- CTA banner uses an action/fishing photo

**CRITICAL NAV SCROLL FIX** — The scrolled nav turns white background. MUST add these CSS rules or logo/hamburger become invisible:
```css
.nav.scrolled .nav-logo { color: var(--navy); text-shadow: none; }
.nav.scrolled .nav-logo span, .nav.scrolled .nav-logo em { color: var(--gold); }
.nav.scrolled .mobile-toggle span { background: var(--navy); }
```

**CRITICAL MOBILE HERO STATS FIX** — Hero stats (rating, years, count) look messy on mobile. MUST add inside `@media (max-width: 768px)`:
```css
.hero h1 { margin-bottom: 1rem; }
.hero-stats { gap: 1.5rem; flex-wrap: wrap; justify-content: center; margin-top: 2rem; padding-top: 1.5rem; }
.hero-stat-num { font-size: 1.5rem; }
.hero-stat-label { font-size: 0.65rem; letter-spacing: 0.5px; }
```

**CRITICAL IMAGE EXTENSION CHECK** — After downloading photos, verify that the file extensions in the gallery array and all `src=` / `url()` references MATCH the actual file extensions on disk. WordPress often serves `.png` files with `.jpg` in the URL or vice versa. Cross-check every reference.

### Phase 5: Deploy
1. `git init` + `git config` (user.email=ben@advancedmarketing.co, user.name=Benjamin Boyce)
2. `gh repo create bensblueprints/{slug} --public --source=. --remote=origin --push`
3. Create Netlify site via API:
   ```
   POST https://api.netlify.com/api/v1/sites
   Authorization: Bearer {NETLIFY_TOKEN}
   Body: { "name": "{slug}", "repo": { "provider": "github", "repo": "bensblueprints/{slug}", "branch": "master" } }
   ```
4. **CRITICAL**: Patch site to enable form processing:
   ```
   PUT https://api.netlify.com/api/v1/sites/{site_id}
   Body: { "processing_settings": { "html": { "pretty_urls": true }, "ignore_html_forms": false } }
   ```
5. Update `admin.html` with new SITE_ID
6. Deploy via file upload API (handles files with spaces via URL encoding):
   - Scan all files, compute SHA-1 digests
   - POST deploy with file digests
   - Upload required files (URL-encode paths with spaces/parens)
7. Commit + push to GitHub
8. Verify site is live

### Phase 6: Verify
1. Take screenshots (hero, pricing, gallery, booking, mobile)
2. Submit test booking
3. Check admin dashboard loads
4. Verify all photos display
5. Test "Get Directions" link
6. Test "Get Your License" link

## What The System Includes

### Public Website (index.html)
- Full-viewport hero with boat photo + zoom animation
- Sticky nav with logo (transparent → white on scroll)
- About section with captain bio (3+ paragraphs) + feature grid (equipment, bait, PFDs, max guests)
- **Species guide** — card grid for each target species with season, description, and fishing techniques + long-form text about the fishery
- **Seasonal fishing calendar** — 4 season cards with species tags and month ranges
- **The Boat** — vessel specs, features, photo, and detailed description
- 3-tier pricing cards with live price calculator
- Photo gallery (12 shown, "Show All" expands) with lightbox — **deduplicated**
- Booking form with trip selector, guest count, date picker, price estimate
- Testimonials (3 cards, 5 stars) based on real Google review sentiment
- **What to Bring** — 3-column checklist (essentials, comfort, nice to have)
- **Area Guide** — city cards with drive times + long-form regional fishing content
- **FAQ accordion** — 10-12 expandable questions with internal cross-links
- Fishing license banner with direct purchase link
- CTA banner
- Contact cards (phone, Facebook/email, location with "Get Directions")
- Footer with admin link + section links
- Built-in session tracker (1 pageview per session via Netlify Forms)
- Scroll animations, fully responsive
- 30+ long-tail SEO keywords in meta
- 15+ FAQ Schema questions in structured data

### Admin Dashboard (admin.html)
- Password-protected login
- **Bookings tab**: stats, search/filter, expandable rows, confirm/cancel/edit date/calendar
- **Trip Calendar tab**: monthly view with **per-boat filtering** — dropdown to select boat, each boat gets its own color. Click day → add trip (select boat) or block day (select boat). Color-coded by boat.
- **Fleet Management** in Settings: add/remove boats with name, type, capacity. Stored in localStorage. Calendar and bookings reference boats by name.
- **Traffic tab**: sessions, daily chart, top pages/sources/devices
- **Settings tab**: email notifications, Google Calendar OAuth setup, **Stripe deposit payments**, **PayPal deposits**, deposit email settings, **Fleet/Boat management**
- Status workflow: New → Confirmed → Completed / Cancelled (with restore)
- "Add to Calendar" works with zero setup (Google Calendar URL)
- **"Send Deposit" button** on each booking — creates Stripe Checkout session or PayPal.me link, sends branded email via Resend API
- Copy admin.html from `C:\Users\admin\Desktop\Github Repo Backup 2-18-2026\kjs-outdoor-adventures\admin.html` as the base template (has Stripe/PayPal built in)
- Customize: title, branding text, password hash, SITE_ID, default business name, default phone number

**Multi-Boat Calendar Requirements:**
- Settings tab has a "Fleet Management" section where captain can add boats (name, type, color)
- Trip Calendar has a boat selector dropdown at the top — "All Boats" or filter by specific boat
- When adding a manual trip or blocking dates, must select which boat it applies to
- Each boat's events are color-coded differently on the calendar
- Bookings show which boat was requested (from form trip type selector)
- Default boats: populate from the pricing tiers if applicable

### Netlify Functions (REQUIRED)
Always create `netlify/functions/` with these two functions:
1. **create-checkout.js** — Creates Stripe Checkout sessions for deposit payments
2. **send-deposit-email.js** — Sends branded deposit request emails via Resend API (update default business name and phone)
Copy from: `C:\Users\admin\Desktop\Github Repo Backup 2-18-2026\kjs-outdoor-adventures\netlify\functions\`
Also create **netlify.toml** with `[build] functions = "netlify/functions"`

### Multi-Boat Charters
For businesses with multiple boats, each boat MUST have:
- Its own photo in the fleet card
- Its own "Book [Boat Name]" button linking to the boat's booking system (e.g., FareHarbor item URL)
- The fleet section should use a grid of cards with: boat image, name, type/capacity, description, and direct booking CTA

### Post-Deploy Mobile QA (MANDATORY)
After EVERY deploy, run a Playwright agent at 375x812 viewport to check:
- Hero stats not overlapping/wrapping badly
- Nav logo + hamburger visible on scrolled white background
- Pricing cards stacking properly
- Gallery grid on small screens
- No text overflow or elements cut off
Fix any issues found before considering deploy complete.

## Admin Password
Generate a random password for each new client. Use:
```javascript
// To generate SHA-256 hash for admin.html
crypto.subtle.digest('SHA-256', new TextEncoder().encode('thepassword'))
```
Default template password: `wickedstriper2026`
Hash: `cf6940b85ac6fc5f8f2eeb97651f916fce61da18d843c172fc4f7c5d936c47f2d`

## CSS for New SEO Sections
Add these CSS blocks to the `<style>` section (see captain-hoggs-charters/index.html for the proven implementation):

**Required CSS blocks:**
- `.species` / `.species-grid` / `.species-card` — 4-column card grid with icon, title, season badge, description
- `.seasons` / `.season-grid` / `.season-card` — 4-column navy-bg cards with colored headings and species pills
- `.boat` / `.boat-grid` / `.boat-specs` / `.boat-spec` — 2-column layout with spec cards
- `.area-guide` / `.area-cities` / `.area-city` — 4-column city cards with drive times
- `.faq` / `.faq-list` / `.faq-item` / `.faq-q` / `.faq-a` — accordion with max-height toggle
- `.bring` / `.bring-grid` / `.bring-card` — 3-column navy-bg checklist cards
- All sections need `@media (max-width: 768px)` responsive overrides to single column

**Reference implementation:** `C:\Users\admin\Desktop\Github Repo Backup 2-18-2026\captain-hoggs-charters\index.html`

## Key Gotchas
- Netlify API site creation sets `ignore_html_forms: true` — MUST patch to false
- Files with spaces in names need URL-encoding when uploading via API
- Git branch is `master` not `main`
- Deploy via API file upload (not git-linked) since GitHub permissions are unreliable
- Admin statuses/manual trips/blocked dates stored in localStorage (per-browser)
- Always push to GitHub after every change
- Always deploy after code changes
- **ALWAYS deduplicate photos** after downloading — WordPress media libraries often have duplicate uploads
- WordPress REST API (`/wp-json/wp/v2/media`) is the fastest way to get all photos — no need for Puppeteer on WordPress sites
- Use Node.js for all scripts (Python is NOT installed)
- `tail`/`head` commands not available in Windows bash
- **CRITICAL `</title>` BUG**: When editing the admin.html title, ALWAYS ensure the closing `</title>` tag is present. Missing it breaks the ENTIRE page (all CSS/JS gets eaten as title text). Template has this bug — always verify after customizing.
- **ALWAYS DEPLOY admin.html** — After every build, verify admin.html is included in the Netlify deploy and loads at `{site}/admin.html`. WebFetch the admin URL after deploy to confirm it shows a login screen. The admin backend is a core deliverable, not optional.
- **ALWAYS DEPLOY Netlify Functions** — The `netlify/functions/create-checkout.js` and `netlify/functions/send-deposit-email.js` files must be in the deploy. The `netlify.toml` must point to them.
- **Reference admin backend**: `smlwickedstriper.com/admin.html` — this is the proven admin template. Every new site must match this functionality (bookings, calendar, traffic, settings with Stripe/PayPal).
- **Post-deploy checklist**: After EVERY deploy, verify: (1) index.html loads with hero image, (2) admin.html shows login screen, (3) gallery images not broken, (4) mobile nav works on scroll. Do NOT consider a build complete until all 4 pass.

## Auto-Add to Client Portal (MANDATORY)
After every new site build, ALWAYS add the client to the Advanced Marketing client portal at `clients.advancedmarketing.co`.

**API Details:**
- Portal API: `https://portal-api.advancedmarketing.co`
- Login as admin first: `POST /api/auth/login` with `{ email: "ben@advancedmarketing.co", password: "JEsus777$$!" }`
- Then create client: `POST /api/auth/signup` with the admin token in Authorization header

**Signup payload:**
```json
{
  "name": "Captain Name",
  "email": "client@email.com",
  "password": "same-as-admin-dashboard-password",
  "business_name": "Business Name",
  "phone": "(XXX) XXX-XXXX",
  "service_type": "web_design"
}
```

**Field mapping:**
- `name` = Captain/owner name
- `email` = Client email (from research) or placeholder like `info@domain.com`
- `password` = Same password used for the admin.html dashboard
- `business_name` = Business name (no special chars that break JSON)
- `phone` = Business phone number
- `service_type` = Always `web_design` for fishing guide sites

This auto-populates web_design task templates for the client. The client appears in the admin view and can log in to see their project status.

## Post-Build Playwright QA (MANDATORY — ~20 minutes)
After the site is deployed, run a comprehensive Playwright QA session testing EVERYTHING on both desktop (1920x1080) and mobile (375x812). This is NOT optional. Spend up to 20 minutes on QA. Fix every issue found before considering the build complete.

**Create a Playwright QA script (`qa.mjs`) that tests:**

### Desktop (1920x1080)
1. **Hero section** — screenshot, verify background image loads (not blank/solid color), badge text visible, CTA buttons present
2. **Nav on scroll** — scroll down 500px, screenshot, verify: logo visible and readable, nav links visible, hamburger hidden on desktop, background is white/opaque
3. **All sections exist** — scroll to each section anchor (#about, #species, #seasons, #boat, #pricing, #gallery, #booking, #testimonials, #prepare, #area, #faq, #contact) and verify each has content (not empty)
4. **Gallery** — verify at least 8 images load (no broken images), click one to test lightbox opens
5. **Pricing cards** — verify 3 cards visible with dollar amounts
6. **Booking form** — fill out test data, verify price calculator updates, do NOT submit
7. **Footer** — verify admin link exists, social icons present, copyright text
8. **Social media links** — click each, verify they open (don't follow, just check href exists)
9. **"Get Directions" link** — verify it points to Google Maps
10. **Fishing license link** — verify href is correct state fishing license URL
11. **FAQ accordion** — click 3 items, verify they expand/collapse
12. **Species cards** — verify correct count (8+), each has icon/title/description

### Mobile (375x812)
13. **Hero** — screenshot, verify text doesn't overflow, stats wrap cleanly
14. **Hamburger menu** — verify visible at top, click it, verify menu opens fullscreen with readable links
15. **Hamburger on scroll** — scroll down, verify hamburger still visible (navy on white bg)
16. **Close menu** — click hamburger again or a link, verify menu closes
17. **Pricing cards** — verify they stack single-column
18. **Gallery** — verify 2-column grid on mobile
19. **Booking form** — verify form fields don't overflow
20. **Top bar (if present)** — verify phone number visible, long text hidden on mobile
21. **Contact cards** — verify single-column stack

### Admin Dashboard
22. **Login page loads** — navigate to /admin.html, verify password input visible
23. **Login works** — enter password, verify dashboard appears
24. **Bookings tab** — verify it shows (even if empty)
25. **Calendar tab** — switch to calendar, verify month grid renders
26. **Settings tab** — switch to settings, verify Stripe/PayPal sections exist
27. **Fleet management (if multi-boat)** — verify boats are listed with colors

### Per-Boat Booking (if applicable)
28. **Fleet cards** — verify each boat has an image and "Book" button
29. **Booking links** — verify each button has a valid href (FareHarbor or other booking URL)

**QA script pattern:**
```javascript
import { chromium } from 'playwright';
const browser = await chromium.launch();

// Desktop
const desktop = await browser.newContext({ viewport: { width: 1920, height: 1080 } });
const dp = await desktop.newPage();
await dp.goto(SITE_URL, { waitUntil: 'networkidle' });
// ... run desktop tests, take screenshots ...

// Mobile
const mobile = await browser.newContext({ viewport: { width: 375, height: 812 }, isMobile: true });
const mp = await mobile.newPage();
await mp.goto(SITE_URL, { waitUntil: 'networkidle' });
// ... run mobile tests, take screenshots ...

// Admin
const admin = await browser.newContext({ viewport: { width: 1920, height: 1080 } });
const ap = await admin.newPage();
await ap.goto(SITE_URL + '/admin.html', { waitUntil: 'networkidle' });
// ... test login, tabs, settings ...

await browser.close();
```

**On ANY failure:** Fix the issue immediately, redeploy, and re-run the failing test. Do not skip failures.
