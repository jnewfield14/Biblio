# Biblio: from prototype to the App Store

This doc covers three things:

1. What I changed in the HTML prototype and why.
2. How I'd build the real iOS/Android app.
3. What has to be true before Biblio ships.

The prototype (`index.html`) is still one self-contained file, so it stays easy to share and click through. It is a **design and behaviour spec** for the native app, not the app itself.

---

## 1. Prototype review: what was wrong and what changed

### Bugs fixed
| Issue in previous `index.html` | Effect | Fix |
|---|---|---|
| `setupDrag` added `mousemove`/`touchmove`/`mouseup` listeners to `document` on **every card render** and never removed them | Listener leak. Each swipe added more handlers, so the app slowed down over a session | One set of pointer-event handlers per card, using pointer capture |
| Cards could only be dragged by the small title header | Most of the card ignored swipes | The whole card is draggable. Axis-locking lets a vertical drag scroll the profile and a horizontal drag swipe it |
| The quiz offered "Sci-Fi" but the data uses "Science Fiction" | Sci-Fi fans never got a genre boost | Alias map: Sci-Fi → Science Fiction/Dystopia; Non-Fiction → Memoir, History & Biography; and so on |
| "Already Read" and "Skip" behaved the same way, and nothing was recorded | No personalisation signal. Skipped books were lost | **Read** marks the book as read and prompts for a rating. **Skip** moves the book to the back of the deck |
| "Edit My Preferences" wiped every existing selection | Users had to redo the whole quiz | The quiz opens pre-filled. Saving re-ranks the deck |
| Nothing persisted | A refresh lost all matches | State is saved to `localStorage` (the native app would use a server) |
| Book data was written into `innerHTML` unescaped | XSS risk once there's user-generated content | Every dynamic string goes through `esc()` |
| Account stats were hard-coded (2 reading, 7 finished, 7/24 goal), and the Currently Reading, Finished and Declined rows did nothing | Fake UI | Stats, goals and lists are computed from real state |
| Fixed 390×780 frame | Unusable on an actual phone | Phone frame on desktop. On phones it goes full-screen with safe-area insets |
| Spice scale had 4 levels | The PRD specifies 5 | 5 levels with the PRD's copy ("Definitely NSFW, but we don't judge") |
| Tapping a book on Home sent you to the swipe deck | No way to view a specific book | Opens the full **Book details** page |

### PRD features that were missing and are now built
- **Welcome, reader!** screen with **Get Started** (primary) and **Log in** (secondary).
- **Sign-up** with display name (labelled public), email, password and **date of birth with an 18+ block**, plus terms consent.
- **Home modules** in PRD order: Matches (excludes books marked Read), **Trending**, **Top in [genre 1]** (default Fiction), **Top in [genre 2]** (default Fantasy).
- **Spoiler settings** behind the gear icon at the top right of Discover. You can toggle the plot one-liner, quotes and first lines, tags, and reviews. Fan art is stubbed as "coming soon".
- **Full Book details page**:
  - Goodreads average and rating count, plus a Biblio average.
  - Your rating from 0 to 5 in **0.5 steps**.
  - Status: **Want to read / Currently reading / Read / Did not finish**.
  - Expandable sections for Tags, Highlights, Reviews, About the author, Discussion (coming soon) and Videos & fan art (future scope).
- **My Books** with tabs for each status, plus **Previously passed** books and a **Reconsider** button that puts a book back in the deck.
- **Reading goals**: a yearly target you can edit, plus "Read N in [Genre]".
- **Avatar colour picker**. The avatar defaults to the first letter of your handle.

### UX changes I'm recommending (also built)
- **Let people swipe before they sign up.** The PRD asks for sign-up straight after onboarding. I'd move it to after the first few swipes:
  - Guests swipe right away. After **5 swipes** a "Save your matches" sheet appears and can be dismissed. After **12 swipes** an account is required.
  - Why: users sign up once they've seen the value and have something to lose. It also matches Apple's guidance not to force registration before people can see what an app does (Guideline 5.1.1).
  - Both thresholds are constants (`GATE_SOFT`, `GATE_HARD`), so you can tune them, or set them to 0 to match the PRD exactly.
- **"Why this book" chips** on each card (✨ Fantasy · ✨ Enemies to lovers). They make the recommendations feel personal, and they show us which signals drive matches.
- **Undo** (↶) for the last swipe. Accidental swipes are the most common complaint in swipe UIs.
- **First-run coachmark** that explains the four actions.
- **Recommendations update as you swipe.** Matches, reads, ratings and passes feed a genre affinity score that re-ranks the deck after every swipe. The next two cards stay put so nothing jumps around.
- **Accessibility**:
  - Real `<button>` elements everywhere.
  - Keyboard shortcuts on desktop: ← pass, → match, ↓ skip, R read, Z undo.
  - `aria-pressed` and `aria-live` where they matter.
  - Focus moves into and out of sheets correctly.
  - Reduced-motion support.
  - 44pt touch targets.
- **Account deletion** inside the app (an App Store requirement, see §3).
- **Analytics hooks**: a `track()` call on every event that maps to the PRD success metrics (see §2.6).

### Still prototype-only (flagged in the UI)
- **Auth is fake.** Any email logs you in, and nothing leaves the browser. Passwords and dates of birth are never stored.
- **The review quotes are sample content.** The modules are labelled "· sample". See the data warning in §3.
- **Formats aren't used in matching yet.** The data has no edition or format information, so the format answer is collected but doesn't affect results.

---

## 2. How I'd build the production app

### 2.1 Stack (tuned for a small team or solo founder)
| Layer | Recommendation | Why |
|---|---|---|
| App | **React Native + Expo** (TypeScript) | One codebase for iOS and Android. EAS Build and Submit handle signing and store uploads, and EAS Update ships JS fixes without a store review |
| Swipe gestures | `react-native-gesture-handler` + `react-native-reanimated` | Swipes run at 60–120fps on the UI thread. The prototype's axis-lock and velocity thresholds carry straight over |
| Images | `expo-image` (disk cache, blurhash placeholders) | Fast cover loading |
| Backend | **Supabase**: Postgres, Auth, Storage, Edge Functions | The data is relational (books ↔ tags ↔ users ↔ statuses). It includes Auth with Apple/Google sign-in, and row-level security keeps each user's data private |
| Search | Postgres full-text search with `pg_trgm` to start; Typesense/Algolia once the catalogue is large | Enough for launch |
| Recommendations | A Postgres function or Edge Function that ports `scoreBook()`. Later, `pgvector` embeddings over tags and descriptions, then collaborative filtering once there's swipe data | Starts simple and grows without a rewrite |
| Analytics | **PostHog** or Amplitude | Funnels and retention for the PRD goals |
| Crash reporting | **Sentry** | Required for a production app |
| Feature flags | PostHog flags | Tune the sign-up gate, deck size and module order without a release |

A fully native build (SwiftUI + Jetpack Compose) gives the very best gesture feel, but it means two codebases. I'd only go that way with dedicated iOS and Android engineers.

### 2.2 Data model (first pass)
```
books(id, title, author_id, year, pages, audio_minutes, cover_url, isbn13, ol_work_id, series, series_complete)
authors(id, name, bio)
genres(id, name) · book_genres(book_id, genre_id)
tags(id, name, kind: vibe|trope|content) · book_tags(book_id, tag_id, source: editorial|derived)
book_modules(book_id, kind: plot|opening|quote|review, body, source, verified_at, sensitivity_skip)
editions(book_id, format: novel|novella|comic|manga|audiobook|ebook)   ← makes the "formats" answer usable
users(id, handle, avatar_color, created_at, age_verified_at)            ← store a verified flag, not the date of birth
user_prefs(user_id, genres[], tropes[], story_tags[], formats[], max_spice, spoiler_settings jsonb)
user_books(user_id, book_id, status: want|reading|read|dnf|passed, source: swipe|manual, updated_at)
ratings(user_id, book_id, value numeric(2,1) check (value*2 = floor(value*2)))
swipes(user_id, book_id, action, deck_position, reasons[], created_at)  ← raw event log for training recommendations
goals(user_id, year, kind: total|genre|local_author, target, genre_id)
```
`book_modules.verified_at` and `sensitivity_skip` turn the content policy that's currently hard-coded in the HTML (`VERIFIED_OPENING_IDS`, `SENSITIVE_PLOT_SKIP_IDS`) into data, with an admin review step.

### 2.3 Data pipeline
- **Catalogue and covers:**
  - Open Library first, Google Books as the fallback. This is the approach in the PRD's script.
  - Run it as a scheduled job.
  - Copy covers into your own storage and CDN instead of hot-linking `openlibrary.org` and `wsrv.nl`, which are rate-limited and not built for production traffic.
- **Ratings:** Goodreads **closed its public API in 2020**, and scraping it breaks its terms. For launch:
  - Show the 2025 snapshot as a clearly dated figure (or drop it).
  - Lead with the **Biblio rating** from our own users.
  - If an external rating is a must, look at licensed sources (e.g. ISBNdb or publisher feeds).
- **Tags and modules:** keep the spreadsheet workflow for the first catalogue. Move it into a small admin tool (Supabase Studio is fine at first) that has a "needs review" state.

### 2.4 App structure
```
app/(auth)/welcome · login · signup
app/(onboarding)/genres · romance · formats · story-tags
app/(tabs)/home · discover · account
app/book/[id]            ← full details, reachable from anywhere
app/library/[status]
app/settings/spoilers
```
State: TanStack Query for server data, with optimistic updates on swipes. Swipes are queued while offline, so swiping still works on the subway.

### 2.5 Performance targets
- Swipe animation never drops below 60fps on an iPhone 11 or Pixel 6a.
- Time to the first card is under 1.5s on a cold start.
- The next 5 cards and their covers are prefetched.
- The deck endpoint returns 20 cards per call and excludes anything the user has already seen.

### 2.6 Analytics (maps to the PRD success criteria)
| PRD goal | Event(s) already in the prototype |
|---|---|
| >5K registrants | `onboarding_started` → `onboarding_completed` → `signup_prompt_shown` → `signup_completed` |
| >2K MAU | `session_start` |
| 25% of accounts reach more than 5 matches within their first 5 sessions | `match` (with `total_matches`) + `session_start` (with `n`) |
| Personalisation quality | `swipe` with `reasons[]`, `rating_set`, `status_changed`, `undo`, `passed_restored` |

---

## 3. Before you submit: App Store and Play Store checklist

**Required by the stores:**
- [ ] **Account deletion inside the app** (Apple 5.1.1(v), Google Play). *Prototype has it.*
- [ ] **Sign in with Apple** if you offer any other third-party login such as Google (Apple 4.8).
- [ ] **Privacy policy URL**, Apple **privacy nutrition labels**, Google **Data safety form**, and an iOS **privacy manifest** (PrivacyInfo.xcprivacy), including one for each SDK you add.
- [ ] **Moderating user content** (reviews, forums, fan art) per Apple 1.2. You need four things:
  - a filter for objectionable content;
  - a way to report content;
  - a way to block users;
  - published contact info.
- [ ] **Age rating.** With 18+ accounts and a "5 peppers = NSFW" scale, answer the age-rating questionnaire honestly. Expect an **18+** rating on iOS and Mature on Play.
  - The 18+ gate should be enforced on the server, not just in the app.
  - Store an "age verified" flag rather than the date of birth.
- [ ] **Toolchain requirements.** The PRD's figures (iOS 16 SDK / Xcode 14.1; Android API 35) look out of date. Apple and Google raise these every year. Check Apple's "Upcoming requirements" page and Play's "Target API level requirements" page before you set up the build.

**Content and legal (I'd treat these as launch blockers):**
- [ ] **Replace the sample reviews.** The "What readers are saying" quotes are credited to named Goodreads handles, but they haven't been verified as real reviews. Publishing invented quotes under real-looking names is a trust problem and a legal one. Two options:
  - Use reviews from Biblio users, or properly licensed review content.
  - Write editorial copy and label it clearly as Biblio's.
- [ ] **Verify quotes.** Keep the existing "verified-only" policy for opening lines and quotes, and build it into the admin tool.
- [ ] **Cover image use.** Covers shown to identify a book are standard practice, but check the terms of whichever source you use and give attribution (Open Library asks for it).
- [ ] **Remove "Goodreads" branding** wherever it isn't a factual, dated attribution.

**Quality:**
- [ ] Test with VoiceOver and TalkBack, including the swipe button alternatives. Test Dynamic Type up to the largest sizes, and test dark mode.
- [ ] Run a TestFlight and Play Internal Testing beta with 20–50 BookTok-style readers before launch.

---

## 4. Suggested build order
1. **Weeks 1–2:** Set up Expo, auth and the database schema. Import the 100-book catalogue.
2. **Weeks 3–4:** Onboarding, the Discover deck (gestures, undo, the deck API) and match persistence.
3. **Week 5:** Home modules, search and book details (ratings, statuses).
4. **Week 6:** My Books, goals, spoiler settings and account deletion.
5. **Week 7:** Analytics, Sentry, accessibility pass and store assets (screenshots, description, privacy labels).
6. **Week 8:** TestFlight and Play internal beta, fixes, then submission.

Discussion forums, fan art and video reviews come after launch. Forums are the first of these because they need the moderation tooling in §3 anyway.
