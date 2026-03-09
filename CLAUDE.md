# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Git Workflow

**Repository:** `https://github.com/ryan-heck/ClaudeCode`

This project uses GitHub as the source of truth. Follow these rules on every work session:

- **Commit after every meaningful change** — don't batch unrelated work into one commit
- **Push to GitHub immediately after every commit** — never leave commits only local
- **Use clean, descriptive commit messages** — present tense, imperative style (e.g. `Add AI theme streaming endpoint`, `Fix profile RLS policy for friends visibility`)
- **Never leave work in an uncommitted state** at the end of a session

```bash
# Standard flow after any change:
git add <specific files>
git commit -m "Brief description of what changed and why"
git push origin main
```

This ensures the project is always recoverable, progress is visible, and nothing is ever lost.

---

# Context — CLAUDE.md

## What We Are Building

Context is a verified digital identity platform. Every user gets a fully customizable profile that works like a beautifully designed personal website inside a social network. It is the place someone goes when they want to genuinely understand who another person is — not their highlight reel, not their resume, not their hot takes. Them.

The name is intentional. Context is what you give someone when you want them to actually know you. It is what every other platform strips away.

**The value proposition in one sentence:** Be genuinely known online.

---

## The One Rule That Overrides Everything

Every line of code, every component, every interaction, every empty state must serve one of two things:

1. Making someone feel that their profile genuinely reflects who they are
2. Making someone feel that they are discovering a real, interesting human being

If a feature does not serve one of those two things, do not build it.

---

## Tech Stack

```
Framework:    Next.js 14 (App Router) + TypeScript — strict mode always
Styling:      Tailwind CSS + CSS Custom Properties for per-user theming
Animations:   Framer Motion — slow, intentional, never frantic
Rich Text:    Tiptap editor
Database:     Supabase (Postgres + Auth + Realtime + Storage)
AI:           Anthropic Claude API (claude-sonnet-4-20250514) — always streaming
Payments:     Stripe (Pro subscriptions + Creator payouts via Stripe Connect)
Fonts:        next/font with Google Fonts — loaded per user theme
Images:       Supabase Storage + next/image
Deployment:   Vercel
```

---

## Project Structure

```
context/
├── app/
│   ├── (auth)/
│   │   ├── sign-up/
│   │   └── sign-in/
│   ├── (onboarding)/
│   │   ├── username/
│   │   ├── questions/
│   │   └── preview/
│   ├── [username]/
│   │   ├── page.tsx              ← THE most important page
│   │   └── [slug]/
│   │       └── page.tsx
│   │   ├── connections/
│   │   ├── memories/
│   │   ├── wrapped/
│   │   └── settings/
│   ├── discover/
│   │   └── [tag]/
│   └── api/
│       ├── ai/
│       │   ├── theme/
│       │   └── writing-prompt/
│       ├── wrapped/
│       │   └── generate/
│       ├── profile/
│       │   └── visit/
│       └── stripe/
│           └── webhook/
├── components/
│   ├── profile/
│   │   ├── ProfileHero.tsx
│   │   ├── ProfileBio.tsx
│   │   ├── ProfileContent.tsx
│   │   ├── CurrentlySection.tsx
│   │   ├── IdentityHub.tsx
│   │   └── ReadingMode.tsx
│   ├── editor/
│   │   ├── AIThemeChat.tsx
│   │   ├── ThemePreview.tsx
│   │   └── ProfileEditor.tsx
│   └── composer/
│       ├── PostComposer.tsx
│       └── VisibilitySelector.tsx
├── lib/
│   ├── supabase/
│   │   ├── client.ts
│   │   ├── server.ts
│   │   └── middleware.ts
│   ├── anthropic.ts
│   ├── stripe.ts
│   └── theme.ts
├── types/
│   ├── theme.ts
│   ├── database.ts
│   └── index.ts
└── CLAUDE.md                     ← this file
```

---

## Database Schema

Run these migrations in Supabase in order.

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- USERS
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  auth_id UUID UNIQUE REFERENCES auth.users(id) ON DELETE CASCADE,
  username TEXT UNIQUE NOT NULL CHECK (length(username) >= 2 AND length(username) <= 30),
  display_name TEXT,
  email TEXT UNIQUE NOT NULL,
  avatar_url TEXT,
  verified_level TEXT DEFAULT 'none' CHECK (verified_level IN ('none', 'oauth', 'government')),
  verified_at TIMESTAMPTZ,
  connected_platforms JSONB DEFAULT '[]',
  is_pro BOOLEAN DEFAULT FALSE,
  pro_since TIMESTAMPTZ,
  custom_domain TEXT UNIQUE,
  stripe_customer_id TEXT,
  stripe_connect_id TEXT,
  last_active_at TIMESTAMPTZ DEFAULT NOW(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- PROFILES
CREATE TABLE profiles (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID UNIQUE REFERENCES users(id) ON DELETE CASCADE,
  tagline TEXT CHECK (length(tagline) <= 120),
  bio TEXT CHECK (length(bio) <= 1000),
  location TEXT,
  currently JSONB DEFAULT '{"reading":null,"listening":null,"working_on":null,"thinking_about":null}',
  links JSONB DEFAULT '[]',
  theme JSONB DEFAULT '{
    "fontDisplay": "Cormorant Garamond",
    "fontBody": "Libre Baskerville",
    "fontScale": 1,
    "colorBackground": "#FAFAF8",
    "colorSurface": "#F0EFE9",
    "colorPrimary": "#1A1A1A",
    "colorText": "#2D2D2D",
    "colorMuted": "#8A8A8A",
    "colorAccent": "#C4A882",
    "colorBorder": "#E5E2D9",
    "layout": "editorial",
    "contentWidth": "standard",
    "sectionSpacing": "comfortable",
    "backgroundType": "solid",
    "backgroundGradient": null,
    "backgroundTexture": null,
    "borderRadius": "subtle",
    "shadowStyle": "soft",
    "animationStyle": "subtle"
  }',
  theme_history JSONB DEFAULT '[]',
  is_public BOOLEAN DEFAULT TRUE,
  completion_score INTEGER DEFAULT 0,
  view_count INTEGER DEFAULT 0,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- POSTS
CREATE TABLE posts (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  title TEXT,
  slug TEXT,
  content TEXT NOT NULL,
  content_text TEXT,
  cover_image_url TEXT,
  media JSONB DEFAULT '[]',
  tags TEXT[] DEFAULT '{}',
  post_type TEXT DEFAULT 'essay' CHECK (post_type IN ('essay', 'note', 'photo_essay')),
  read_time_minutes INTEGER,
  visibility TEXT DEFAULT 'public' CHECK (visibility IN ('public', 'friends', 'private', 'subscribers')),
  comments_mode TEXT DEFAULT 'off' CHECK (comments_mode IN ('off', 'private', 'public')),
  view_count INTEGER DEFAULT 0,
  save_count INTEGER DEFAULT 0,
  resonance_count INTEGER DEFAULT 0,
  is_published BOOLEAN DEFAULT FALSE,
  published_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, slug)
);

-- REACTIONS (private to creator always — never shown publicly)
CREATE TABLE reactions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  reaction_type TEXT CHECK (reaction_type IN ('resonated', 'saved', 'moved', 'inspired')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(post_id, user_id, reaction_type)
);

-- COMMENTS
CREATE TABLE comments (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
  author_id UUID REFERENCES users(id) ON DELETE CASCADE,
  content TEXT NOT NULL CHECK (length(content) <= 2000),
  is_public BOOLEAN DEFAULT FALSE,
  parent_id UUID REFERENCES comments(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- CONNECTIONS
CREATE TABLE connections (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  follower_id UUID REFERENCES users(id) ON DELETE CASCADE,
  following_id UUID REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(follower_id, following_id)
);

-- INTRODUCTIONS
CREATE TABLE introductions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  introducer_id UUID REFERENCES users(id) ON DELETE CASCADE,
  user_a_id UUID REFERENCES users(id) ON DELETE CASCADE,
  user_b_id UUID REFERENCES users(id) ON DELETE CASCADE,
  note TEXT CHECK (length(note) <= 500),
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'accepted', 'declined')),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- AI THEME GENERATIONS
CREATE TABLE ai_theme_generations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  prompt TEXT NOT NULL,
  generated_theme JSONB NOT NULL,
  explanation TEXT,
  applied BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- WRAPPED
CREATE TABLE wrapped (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  year INTEGER NOT NULL,
  data JSONB NOT NULL,
  is_public BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, year)
);

-- CREATOR SUBSCRIPTIONS
CREATE TABLE creator_subscriptions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  creator_id UUID REFERENCES users(id) ON DELETE CASCADE,
  subscriber_id UUID REFERENCES users(id) ON DELETE CASCADE,
  stripe_subscription_id TEXT UNIQUE,
  price_monthly NUMERIC(10,2),
  status TEXT DEFAULT 'active' CHECK (status IN ('active', 'cancelled', 'paused')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(creator_id, subscriber_id)
);

-- INVITES
CREATE TABLE invites (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  inviter_id UUID REFERENCES users(id) ON DELETE CASCADE,
  invite_code TEXT UNIQUE NOT NULL DEFAULT encode(gen_random_bytes(6), 'hex'),
  used_by UUID REFERENCES users(id),
  used_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- PROFILE VISITS
CREATE TABLE profile_visits (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  profile_user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  visitor_id UUID REFERENCES users(id) ON DELETE SET NULL,
  visited_at TIMESTAMPTZ DEFAULT NOW()
);

-- ROW LEVEL SECURITY
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;
ALTER TABLE reactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE comments ENABLE ROW LEVEL SECURITY;
ALTER TABLE connections ENABLE ROW LEVEL SECURITY;
ALTER TABLE ai_theme_generations ENABLE ROW LEVEL SECURITY;
ALTER TABLE wrapped ENABLE ROW LEVEL SECURITY;
ALTER TABLE creator_subscriptions ENABLE ROW LEVEL SECURITY;
ALTER TABLE invites ENABLE ROW LEVEL SECURITY;
ALTER TABLE profile_visits ENABLE ROW LEVEL SECURITY;

-- Public profiles readable by all
CREATE POLICY "Public profiles viewable" ON users FOR SELECT USING (TRUE);
CREATE POLICY "Users update own record" ON users FOR UPDATE USING (auth.uid() = auth_id);

-- Profiles respect is_public
CREATE POLICY "Public profiles viewable by all" ON profiles
  FOR SELECT USING (
    is_public = TRUE OR
    user_id IN (SELECT id FROM users WHERE auth_id = auth.uid())
  );
CREATE POLICY "Owners manage own profile" ON profiles
  FOR ALL USING (user_id IN (SELECT id FROM users WHERE auth_id = auth.uid()));

-- Posts respect visibility
CREATE POLICY "Posts visibility" ON posts
  FOR SELECT USING (
    (visibility = 'public' AND is_published = TRUE)
    OR user_id IN (SELECT id FROM users WHERE auth_id = auth.uid())
    OR (visibility = 'friends' AND is_published = TRUE AND
        user_id IN (
          SELECT following_id FROM connections
          WHERE follower_id IN (SELECT id FROM users WHERE auth_id = auth.uid())
        ))
  );
CREATE POLICY "Owners manage own posts" ON posts
  FOR ALL USING (user_id IN (SELECT id FROM users WHERE auth_id = auth.uid()));

-- Reactions visible only to post creator
CREATE POLICY "Creators see reactions on their posts" ON reactions
  FOR SELECT USING (
    post_id IN (
      SELECT id FROM posts WHERE user_id IN (
        SELECT id FROM users WHERE auth_id = auth.uid()
      )
    )
    OR user_id IN (SELECT id FROM users WHERE auth_id = auth.uid())
  );
CREATE POLICY "Authenticated users can react" ON reactions
  FOR INSERT WITH CHECK (
    user_id IN (SELECT id FROM users WHERE auth_id = auth.uid())
  );
```

---

## Theme System

Per-user themes are applied via CSS custom properties injected at the profile root. This is how every profile looks completely different without separate stylesheets.

```typescript
// types/theme.ts
export interface ContextTheme {
  fontDisplay: string;
  fontBody: string;
  fontScale: number;
  colorBackground: string;
  colorSurface: string;
  colorPrimary: string;
  colorText: string;
  colorMuted: string;
  colorAccent: string;
  colorBorder: string;
  layout: 'editorial' | 'minimal' | 'magazine' | 'grid' | 'stream';
  contentWidth: 'narrow' | 'standard' | 'wide';
  sectionSpacing: 'tight' | 'comfortable' | 'airy';
  backgroundType: 'solid' | 'gradient' | 'texture' | 'mesh';
  backgroundGradient: string | null;
  backgroundTexture: string | null;
  borderRadius: 'none' | 'subtle' | 'rounded';
  shadowStyle: 'none' | 'soft' | 'sharp';
  animationStyle: 'none' | 'subtle' | 'expressive';
}

// lib/theme.ts
export function themeToCSS(theme: ContextTheme): Record<string, string> {
  return {
    '--font-display': theme.fontDisplay,
    '--font-body': theme.fontBody,
    '--font-scale': String(theme.fontScale),
    '--color-bg': theme.colorBackground,
    '--color-surface': theme.colorSurface,
    '--color-primary': theme.colorPrimary,
    '--color-text': theme.colorText,
    '--color-muted': theme.colorMuted,
    '--color-accent': theme.colorAccent,
    '--color-border': theme.colorBorder,
  };
}
```

```css
/* In globals.css */
.profile-root {
  background-color: var(--color-bg);
  color: var(--color-text);
  font-family: var(--font-body), Georgia, serif;
}

.profile-root h1,
.profile-root h2,
.profile-root h3 {
  font-family: var(--font-display), 'Times New Roman', serif;
  color: var(--color-primary);
}
```

---

## AI Theme Customizer

This is the signature feature. Stream the response. Update the profile preview in real time. This is the moment someone understands what Context is.

```typescript
// app/api/ai/theme/route.ts
const THEME_SYSTEM_PROMPT = `You are Context's design intelligence.
You translate how a person feels, thinks, or wants to be perceived
into a precise visual identity for their profile.

You understand typography deeply:
- Cormorant Garamond: aristocratic, literary, timeless
- Playfair Display: editorial, confident, modern
- DM Serif Display: warm, approachable, contemporary
- Fraunces: nostalgic, tender, characterful
- Syne: architectural, bold, geometric
- Crimson Pro: scholarly, intimate, serious
- Libre Baskerville: trustworthy, readable, grounded

You translate vibes into precise hex values:
- "rainy evening in Tokyo" → deep slate, muted ink blues, soft amber accent
- "old library with good light" → warm ivory, aged paper, deep walnut text
- "minimal architect's studio" → pure white, concrete grey, single black accent
- "mediterranean summer" → warm terracotta, faded linen, deep olive
- "late night jazz club" → near-black, warm amber surface, brass accent
- "someone who thinks carefully" → cream, serious navy, considered spacing

RULES:
- Never suggest Inter, Roboto, Arial, or system fonts
- Only use Google Fonts loadable via next/font
- Always return valid hex colors only
- Explanation: one evocative sentence, never technical
- Always return a COMPLETE theme object

Return ONLY this JSON, no preamble, no markdown:
{
  "theme": { ...complete ContextTheme object... },
  "explanation": "One evocative sentence."
}`;
```

The AI chat component maintains conversation history across turns so follow-up requests like "make it darker" or "keep the fonts but change the colors" work naturally.

---

## Onboarding Flow

Seven steps. Each one matters. The goal: user has a profile they are proud to share within 10 minutes.

```
Step 1 — Invite gate (beta only)
         Beautiful landing page + invite code input
         No invite? → email waitlist capture

Step 2 — Account creation
         Email + password OR LinkedIn/Spotify/GitHub OAuth
         OAuth auto-populates connected_platforms and profile links

Step 3 — Username
         Single input, full screen
         Real-time availability check
         Show them: context.so/[username]
         This should feel significant

Step 4 — Three questions (one at a time, full screen each)
         Q1: "How would you describe yourself in one sentence?"
             → tagline (hint: "Not your job title. You.")
         Q2: "What do you want people to understand about you
              that your other profiles don't show?"
             → bio seed
         Q3: "Pick the vibe that feels most like you"
             → 6 visual profile previews to choose from
             → selection triggers AI theme generation

Step 5 — Profile preview
         Show their real profile with AI-generated theme applied
         Two options: [This feels right] or [Change the look]
         Do NOT push them to the editor immediately
         Let them feel something first

Step 6 — Currently section (optional, inline, 60 seconds)
         "What are you currently reading, listening to, working on?"
         Four small fields, all optional
         Makes the profile feel alive immediately

Step 7 — Verification prompt
         "Connect LinkedIn, Spotify, or Strava to show you're real"
         Framed as: "Verified profiles get discovered first"
         Never as: a requirement
         [Connect] [Do this later]

Engagement hooks in onboarding:
- Every step saves to Supabase immediately
- Skip always available on steps 4-7
- End screen: shareable profile link, prominent
- "Share your Context" → copies to clipboard
```

---

## Profile Page

This is the product. It receives 80% of design attention. Everything else serves it.

**Design direction: Editorial Luxury meets Human Warmth.**
Think: tactile quality of a beautifully printed book. Intimacy of a handwritten letter. Confidence of a well-designed magazine. A room designed for one specific person.

**Typography is the soul.** Platform UI uses Cormorant Garamond + Libre Baskerville as defaults. Never Inter, Arial, or system fonts anywhere a user can see.

**Motion feels like breathing.** Slow, intentional. Page transitions like turning a page. No bouncing, no spinning. Dignity in every animation.

### Component Structure
```typescript
// app/[username]/page.tsx

<ProfileRoot theme={profile.theme}>           // applies CSS vars
  <ProfileHero>
    <Avatar />
    <DisplayName />                           // large, display font
    <Tagline />                               // italic, accent color
    <VerifiedBadge />                         // subtle, never overwhelming
    <CurrentlySection />                      // alive signal
    <IdentityHub />                           // links to other platforms
  </ProfileHero>

  <ProfileBio>
    <BioText />                               // up to 1000 chars
    <LocationBadge />
  </ProfileBio>

  <ProfileContent>
    <PostGrid />                              // or stream/list per layout
  </ProfileContent>

  <ReadingModeButton />                       // fixed, subtle
</ProfileRoot>
```

### What Is Never Shown on a Profile
```
- Follower count
- Following count
- Like count
- View count
- Any engagement metric
- Join date
- Ads (ever)
- Suggested profiles
```

### Profile Visit Tracking
```typescript
// On every authenticated visit:
// 1. Record in profile_visits
// 2. Increment profiles.view_count
// 3. Queue batched notification (max once per day):
//    "Your profile was visited X times this week"
// Never reveal who visited
```

### Reading Mode
```
Activated by reader or automatically on individual post pages.
All profile chrome disappears.
Content reflows: centered, max-width 65ch, generous line-height (1.8).
Font size increases slightly.
Only content and a minimal back button remain.
Feels like opening a book.
```

---

## Post Composer

Full screen. Distraction-free. The toolbar appears only when text is selected.

```
Layout:
[Title field]          ← large, display font, no border, hint: "Give it a title, or don't."
[Body — Tiptap]        ← full screen, hint: "Say something true."

Bottom bar (always visible):
[Essay ▾] [🌐 Public ▾] [Tags...] [Save Draft] [Publish →]

Auto-save: every 20 seconds with subtle "saved" indicator
```

### Post Types (three only — no more)
```
Essay       → long-form, title required, cover image optional, read time shown
Note        → short-form, no title required, max 500 words (soft limit), feels like a thought
Photo Essay → images primary, text secondary, drag-and-drop upload
```

### Visibility (must be clear at all times)
```
🌐 Public      → anyone on the internet
👥 Friends     → only mutual connections
🔒 Only Me     → private journal

Color coding: Public = green, Friends = blue, Private = soft amber
Changing visibility after publishing: always allowed
```

### Writing Prompt (re-engagement)
```
Appears when: composer opened AND no post in 7+ days
One question, dismissible, never pushy
Examples:
  "What's something you've changed your mind about recently?"
  "Describe a place that made you feel like yourself."
  "What are you in the middle of figuring out?"
One tap to use as starting point. One tap to dismiss forever.
```

---

## Dashboard

Private. Feels like a study, not a control panel.

```
Navigation: Left sidebar (desktop) / Bottom nav (mobile, 5 items max)

Dashboard Home shows (in order):
1. Memories card — if today has memories, ALWAYS first
2. Currently updater — one-tap inline update
3. Writing prompt — if no post in 7+ days
4. Analytics preview — profile views this week, recent reactions
5. Recent comments/messages
6. Recent new connections

What dashboard never shows:
- A social feed of other people's content
- Trending anything
- Other users' metrics
- Ads or upsells (except one tasteful Pro card if not Pro)
```

---

## Memories

```typescript
// Query: posts published on this calendar day in prior years
const query = `
  SELECT * FROM posts
  WHERE user_id = $1
    AND is_published = TRUE
    AND visibility != 'private'
    AND TO_CHAR(published_at, 'MM-DD') = TO_CHAR(NOW(), 'MM-DD')
    AND DATE_PART('year', published_at) < DATE_PART('year', NOW())
  ORDER BY published_at DESC
`;

// Dashboard notification card:
// "On this day, [X] years ago:"
// [Post preview]
// [Reshare publicly] [Keep as memory] [Dismiss]
// Resharing adds: "Originally written on [date]"
```

---

## Currently Section

```typescript
interface Currently {
  reading: string | null;
  listening: string | null;
  working_on: string | null;
  thinking_about: string | null;
}

// Update flow: Dashboard → inline edit (no navigation) → saves with one tap
// Appears on every public profile
// Makes profiles feel alive between posts
// 30 seconds to update = lowest friction re-engagement in the product
```

---

## Discovery

Feels like browsing a beautifully curated bookshop. Not scrolling a feed.

```
Three sections:

1. Featured Profiles
   - Hand-curated by Context team
   - 6 profiles, refreshed weekly
   - Shows: avatar, name, tagline, latest post snippet

2. By Interest
   - User-defined tag cloud
   - Clicking tag → profiles and posts with that tag

3. Recently Verified
   - New verified profiles in last 30 days
   - Max 12, refreshed daily
   - Signal: real person, new here, go say hello

What is NOT on discovery:
- Trending posts
- Most followed profiles
- Viral content
- Any metric-based ranking
```

---

## Engagement Architecture

Every feature has a re-engagement hook.

```
IMMEDIATE (first session):
✓ Onboarding ends with shareable profile link
✓ Currently section fills profile with life in 60 seconds
✓ AI customizer creates investment immediately
✓ Writing prompt removes blank page barrier

SHORT-TERM (first 30 days):
✓ "Your profile was visited X times this week"
✓ "Someone resonated with [post]"
✓ "X is now following your work"
✓ Writing prompt after 7 days of inactivity

LONG-TERM (month 2+):
✓ Memories: perpetual, on-this-day, costs nothing
✓ Wrapped: annual reflection in December
✓ Introductions: new connections through trusted people

PASSIVE (no creation required):
✓ Discovery browsing
✓ Library — revisiting saved posts
✓ Currently section updates
✓ Wrapped / analytics viewing

NOTIFICATION RULES:
- Maximum one push notification per day
- Never notify about other people's metrics
- Never notify about platform features
- Only things that directly involve the user
- All notifications opt-in after first week
```

---

## Monetization

```typescript
// Context Pro — $8/month or $72/year
// Positioned as: status + tools, not features paywall

const PRO_FEATURES = {
  verifiedBadge: true,           // prominent verified indicator
  customDomain: true,            // yourname.com
  priorityDiscovery: true,       // featured in discovery first
  advancedThemes: true,          // full theme library
  analyticsDepth: true,          // detailed private analytics
  wrappedSharing: true,          // shareable Wrapped cards
  governmentVerification: true,  // unlocks ID verification flow
};

// Free tier: fully functional
// Pro: upgrade, not a gate

// Creator Subscriptions (requires government verification):
// Creator sets price: $3–$50/month
// Context fee: 7% (lower than Substack's 10%)
// Stripe Connect for payouts
// Subscriber-only posts: visibility = 'subscribers'
```

---

## Context Wrapped

```typescript
interface WrappedData {
  year: number;
  totalWordsWritten: number;
  totalPostsPublished: number;
  longestPost: { title: string; wordCount: number; slug: string };
  mostResonatedPost: { title: string; resonanceCount: number; slug: string };
  topTags: string[];
  themesChanged: number;
  firstPostOfYear: { title: string; published_at: string } | null;
  lastPostOfYear: { title: string; published_at: string } | null;
  mostActiveMonth: string;
  yearInOneSentence: string;    // AI reads all posts, summarizes the year
  dominantThemes: string[];     // 3 themes from their writing
  profileViewsThisYear: number;
  newConnectionsThisYear: number;
}

// Slide-based presentation (like Spotify Wrapped)
// Shareable as OG image (server-side generated)
// Can be posted publicly on their profile
// Archived forever — accessible every prior year
```

---

## Environment Variables

```env
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
ANTHROPIC_API_KEY=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
LINKEDIN_CLIENT_ID=
LINKEDIN_CLIENT_SECRET=
SPOTIFY_CLIENT_ID=
SPOTIFY_CLIENT_SECRET=
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXT_PUBLIC_APP_NAME=Context
```

---

## Build Order — Do Not Deviate

```
WEEK 1 — The Profile (the only thing that matters right now)
  Day 1-2:  Supabase setup, schema, auth
  Day 3-4:  /[username] profile page — all 5 layout variants
            Make it extraordinary before making it functional
  Day 5:    Theme system (CSS variables + theme object)
  Day 6-7:  AI theme customizer (streaming, real-time preview)
            This is the demo. Make it feel like magic.

WEEK 2 — Content
  Day 1-2:  Post composer (Tiptap + auto-save + visibility)
  Day 3:    /[username]/[slug] post page with reading mode
  Day 4:    Dashboard home
  Day 5-7:  Onboarding flow (all 7 steps)

WEEK 3 — Social Layer
  Day 1-2:  Connections (no public counts ever)
  Day 3:    Reactions + private comments
  Day 4-5:  Discovery page
  Day 6-7:  Profile visits + notifications

WEEK 4 — Engagement Features
  Day 1-2:  Memories
  Day 3-4:  Currently section updater
  Day 5:    Writing prompt API
  Day 6-7:  Private analytics dashboard

WEEK 5 — Monetization
  Day 1-2:  Stripe Pro subscription
  Day 3-4:  Creator subscriptions (Stripe Connect)
  Day 5:    Verification upgrade flow
  Day 6-7:  Wrapped

WEEK 6 — Polish
  Day 1-2:  Mobile optimization
  Day 3:    Performance (LCP < 2s, no CLS on load)
  Day 4:    OG meta tags for profiles and posts
  Day 5:    Invite system
  Day 6-7:  Landing page
```

---

## Design Rules — Never Break These

```
1. Never use Inter, Roboto, Arial, or system fonts anywhere visible
2. Never show follower counts, like counts, or view counts publicly
3. Never show a feed of other people's content on the dashboard
4. Never add background music to profiles
5. Never add custom CSS editing for users
6. Never build more than three post types
7. Never make verification required before seeing value
8. Never send more than one push notification per day
9. Never show engagement metrics publicly — reactions are private to creators
10. Always make visibility setting visible and changeable on every post
```

---

## The Question That Overrides All Others

Before shipping any page, component, or feature, open it on your phone and ask:

**Does this feel like somewhere a real person lives?**

Not a template. Not a social media profile. Not a portfolio.

Somewhere a person lives. If the answer is anything other than an immediate yes — redesign before writing another line.

That question is the product.

---

# Engineering Principles

## 1. Clarity Over Cleverness

Prioritize readability and maintainability. Avoid overly clever implementations, dense one-liners, and unnecessary abstractions. Prefer simple logic, explicit control flow, and descriptive naming. Future engineers should be able to understand code quickly.

## 2. Small, Focused Components

Functions and modules should have a single responsibility. Functions should generally be under ~40 lines, files should rarely exceed ~300 lines, and complex logic should be split into reusable helpers. Each module should do one thing well.

## 3. Consistent Patterns

Match existing patterns in the codebase before introducing new approaches. Search for existing implementations, reuse established utilities, and maintain architectural consistency. Avoid introducing multiple patterns for the same problem.

## 4. Safe Changes

Minimize risk when modifying the repository. Do not rewrite large sections unnecessarily, avoid touching unrelated files, prefer incremental improvements over large refactors, and preserve existing behavior unless explicitly instructed. When uncertain, choose the least disruptive change.

## 5. Deterministic Code

Prefer predictable and testable behavior. Avoid hidden side effects, global state mutations, and implicit dependencies. Prefer pure functions when possible, explicit parameters, and clear data flow.

## 6. Error Handling

All external interactions must include error handling — API requests, database queries, file system operations, and user input. Errors should fail gracefully, produce useful messages, and avoid crashing the application.

## 7. Testing Discipline

New functionality should include tests when appropriate. Core business logic must be testable, edge cases should be considered, and tests should be deterministic. Do not introduce code that cannot reasonably be tested.

## 8. Iterative Development Loop

When implementing features: plan the approach → implement the code → run tests → fix failures → repeat until stable. Continue iterating until the change satisfies the Definition of Done.

## 9. Minimal Dependencies

Avoid adding new dependencies unless necessary. Before introducing a library, confirm the problem cannot be solved with existing tools, evaluate maintenance and security risk, and prefer well-maintained libraries. Favor simplicity.

## 10. Documentation

Public modules and complex logic should be documented. Explain *why* the code exists, clarify non-obvious logic, and document public interfaces. Avoid redundant comments that restate obvious code.

## 11. Performance Awareness

Be mindful of performance. Avoid unnecessary re-renders, N+1 database queries, and excessive memory allocation. Use efficient algorithms when possible. Optimization should not sacrifice readability.

## 12. Debugging Discipline

When encountering failures: identify the root cause → fix the minimal necessary code → verify with tests → ensure no regressions were introduced. Avoid guess-based debugging.

## 13. Code Stability

Production-critical paths should prioritize stability. Avoid introducing experimental patterns, unnecessary architectural changes, or breaking interface changes. Stability is preferred over novelty.
