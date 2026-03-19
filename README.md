# Fishing Guide Website Builder - Claude Code Skill

A Claude Code slash command (`/fishing-guide`) that builds complete fishing guide/charter websites from scratch in minutes.

## What It Does

Give it a fishing charter's existing website URL and it will:

1. **Research** the business - scrape captain info, pricing, species, reviews, photos
2. **Build** a complete SEO-optimized website (2,500+ lines of HTML/CSS/JS)
3. **Deploy** to Netlify with custom domain support
4. **Set up** an admin dashboard with booking management, trip calendar, and payment processing

## Features

### Public Website
- Full-viewport hero with boat photo + zoom animation
- Captain bio with real photos (auto-identified from scraped images)
- 8+ species cards with seasonal availability
- Monthly species availability chart (12 species x 12 months)
- 4-season fishing calendar
- Boat/fleet section with specs and per-boat booking links
- 3-tier pricing cards with live price calculator
- Photo gallery with lightbox (auto-deduplicated)
- Booking form with Netlify form handling
- Testimonials from real Google/FishingBooker reviews
- "What to Bring" checklist
- Area guide with 10+ nearby cities and drive times
- 12-item FAQ accordion
- Fishing license banner
- Schema.org structured data (LocalBusiness + FAQPage, 15+ questions)
- 30+ long-tail SEO keywords
- Fully responsive with mobile-first fixes

### Admin Dashboard
- Password-protected login
- Booking management (confirm, cancel, edit, calendar sync)
- Multi-boat fleet management with color-coded calendars
- Per-boat date blocking and trip scheduling
- Stripe deposit payment requests
- PayPal deposit payment links
- Branded deposit email notifications (via Resend API)
- Traffic analytics (sessions, sources, devices)
- Google Calendar OAuth integration
- Google Analytics setup

### Netlify Functions
- `create-checkout.js` - Stripe Checkout session creation
- `send-deposit-email.js` - Branded HTML deposit emails via Resend

## Usage

### As a Claude Code Slash Command

1. Copy `SKILL.md` to `~/.claude/commands/fishing-guide.md`
2. In Claude Code, run: `/fishing-guide https://example-charter.com`
3. Wait for the full build, deploy, and verification

### What You Need

- Node.js (for Playwright photo scraping)
- GitHub CLI (`gh`) for repo creation
- Netlify account with API token
- Git configured

## Example Sites Built With This Skill

- [SML Wicked Striper](https://smlwickedstriper.com) - Smith Mountain Lake, VA
- [Captain Hogg's Charters](https://captain-hoggs-charters.netlify.app) - Hampton, VA
- [KJ's Outdoor Adventures](https://kjs-outdoor-adventures.netlify.app) - King George, VA
- [Orange Beach Fishing Charters](https://orange-beach-fishing-charters.netlify.app) - Orange Beach, AL

## Built By

[Advanced Marketing](https://advancedmarketing.co) - Web design and digital marketing for fishing charters and outdoor businesses.
