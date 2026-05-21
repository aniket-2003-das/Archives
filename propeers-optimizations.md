# `/propeers` Page — Performance & Code Quality Optimizations

  > **Bundle impact:** First Load JS reduced from **520 kB → 394 kB** (−126 kB, ~24% reduction)
  > Route size reduced from **94.5 kB → 81.2 kB**

  ---

  ## Table of Contents

  1. [Remove Unused Code](#step-1-remove-unused-code)
  2. [Image Optimization](#step-2-image-optimization)
  3. [Lazy Loading](#step-3-lazy-loading)
  4. [Re-render Fixes](#step-4-re-render-fixes)
  5. [requestIdleCallback for Non-Critical Tasks](#step-5-requestidlecallback-for-non-critical-tasks)
  6. [SSR Upgrade](#step-6-ssr-upgrade)
  7. [API / React Query Fix](#step-7-api--react-query-fix)
  8. [Tree Shaking & Import Optimization](#step-8-tree-shaking--import-optimization)
  9. [UX Fix — Initial Loading Skeleton](#step-9-ux-fix--initial-loading-skeleton)
  10. [NPM Package Updates](#npm-package-updates)
  11. [Future Scope](#future-scope)

  ---

  ## Step 1 — Remove Unused Code

  | # | File | Issue |
  |---|------|-------|
  | 1 | `loading.tsx` | `import React from "react"` — unused; Next.js App Router auto-imports React |
  | 2 | `loading.tsx` | `import SupportCard from "@components/common/supportCard"` — imported but never used in JSX |
  | 3 | `MentorsListingClient.tsx` | `usePathname` import + `const pathname = usePathname()` — declared but never referenced |
  | 4 | `layout.tsx` | Duplicate `window.addEventListener("resize", ...)` — both `layout.tsx` (lines 14–23) and `MentorsListingClient.tsx` (lines 93–105) attach a
  resize listener calling `setIsMobile(window.innerWidth < 1024)` from the same Zustand store. The one in `layout.tsx` also misses the initial value set on mount —
  removed entirely since `MentorsListingClient` already handles both |

  ---

  ## Step 2 — Image Optimization

  **`LeftWavyBar.tsx:7` — Remove redundant `xmlns` from inline SVG**

  ```diff
  - <svg ... xmlns="http://www.w3.org/2000/svg" ...>
  + <svg ... ...>
  xmlns is unnecessary in JSX; the browser already knows the SVG namespace.

  ---
  SuggestedCompanies.tsx:27 — Use imported AmazonIcon SVG instead of external S3 URL

  - logo: "https://codergram-profilepic.s3.ap-south-1.amazonaws.com/AmazonLogo.png"
  + logo: AmazonIcon,
  AmazonIcon was already imported (line 9) but unused. This matches the pattern used for Google & Microsoft and removes an external network dependency.

  ---
  avatar.tsx:40 — Drop quality={100} on <Image>

  - <Image ... quality={100} ...>
  + <Image ... quality={80} ...>
  quality={100} disables Next.js compression entirely. 80 is a safe balance for avatars. avatar.tsx is a shared component, so this improves image performance
  app-wide.

  ---
  Step 3 — Lazy Loading

  ┌────────────────────┬──────────────────────────┬──────────────────────────────────────────────────────────────────────────────────────────────────┬──────────┐
  │     Component      │           File           │                                              Reason                                              │   UX     │
  │                    │                          │                                                                                                  │  Impact  │
  ├────────────────────┼──────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ MentorsFilterModal │ MentorsListingClient.tsx │ Only needed after user interaction; excluded from initial bundle                                 │ None     │
  ├────────────────────┼──────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ MentorCardLong     │ SearchedMentors.tsx      │ Rendered only after async mentor data arrives via React Query; skeleton UI already masks load    │ None     │
  │                    │                          │ time                                                                                             │          │
  ├────────────────────┼──────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ MentorCardLong2    │ SearchedMentors.tsx      │ AI-flow variant — same behavior as above                                                         │ None     │
  ├────────────────────┼──────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ SortedTopMentors   │ SearchedMentors.tsx      │ Only shown in the "no results" edge case; not needed in common flows                             │ None     │
  └────────────────────┴──────────────────────────┴──────────────────────────────────────────────────────────────────────────────────────────────────┴──────────┘

  Goal: Reduce initial JavaScript bundle size by deferring non-critical or conditionally rendered components until they are actually needed.

  ---
  Step 4 — Re-render Fixes

  SearchedMentors.tsx
  - useMentorFiltersStore → useShallow selector for 11 fields — opening the filter modal no longer triggers a re-render of the mentor list
  - filterApplied added to mentors useMemo deps (was missing)
  - showMore boolean (hasMore || (hasMore2 && !!token && !filterApplied)) memoized into a single place
  - handleLoadMore (InfiniteScroll's next callback) wrapped in useCallback

  MentorsListingClient.tsx
  - Two separate useMentorFiltersStore() calls consolidated into one useShallow selector for 7 fields — filter selection changes in SearchedMentors no longer
  re-render the header/sort bar

  MentorFilterButton.tsx
  - useMentorFiltersStore → useShallow selector for 7 fields — opening the modal (showModal changing) no longer re-renders the filter button itself

  ---
  Step 5 — requestIdleCallback for Non-Critical Tasks

  Defers work that has zero user-visible impact until the browser is idle after the mentor cards render.

  ▎ Safari note: requestIdleCallback landed in Safari 16.4 (2023). A setTimeout(cb, 1) fallback is included.

  Candidates

  PropeersAnalytics.tsx — PostHog analytics event
  // Before — fires immediately on mount, competing with initial mentor list render
  useEffect(() => {
    createAnalyticEvent("MentorsSearchPageVisited", { path: window.location.pathname })
  }, [])
  Analytics tracking has zero user-visible impact and shouldn't compete with rendering mentor cards on mount.

  ---
  SearchedMentors.tsx — Coupon fetching
  // Before — runs immediately on mount, but coupons aren't needed to show the mentor list
  useEffect(() => {
    fetchCoupons()
  }, [getPriceCodeForFetch])
  Coupons are supplementary UI (discount badges on cards). The mentor list is fully usable without them.

  Non-Candidates

  ┌─────────────────────────────┬────────────────────────────────────────────────────────────────────┐
  │            Task             │                               Reason                               │
  ├─────────────────────────────┼────────────────────────────────────────────────────────────────────┤
  │ AI mentor fetching          │ User is actively waiting for results — must be prompt              │
  ├─────────────────────────────┼────────────────────────────────────────────────────────────────────┤
  │ URL coupon validation       │ Only fires when coupon_code is in the URL; already infrequent      │
  ├─────────────────────────────┼────────────────────────────────────────────────────────────────────┤
  │ URL params → filter parsing │ Affects initial filter state for deep links — must run immediately │
  └─────────────────────────────┴────────────────────────────────────────────────────────────────────┘

  ---
  Step 6 — SSR Upgrade

  layout.tsx — Converted to a Server Component

  After removing the resize listener in Step 1, layout.tsx has zero client-side code. The flex wrapper div now renders in the server's HTML response, so the browser
   gets the page structure in the first byte — before the JS bundle for MentorsListingClient or SearchMentorFilter has even started downloading. Both of those still
   load on the client exactly as before via their dynamic() imports.

  ---
  Step 7 — API / React Query Fix

  useSearchMentors.tsx:132 — useSearchTopMentors missing staleTime and refetchOnWindowFocus

  useSearchTopMentors is used by SortedTopMentors (shown in the "no results" state). Unlike useSearchMentors and useSuggestedMentors, it had no staleTime (defaults
  to 0 — stale immediately) and no refetchOnWindowFocus: false, causing a fresh fetch on every mount and every window focus.

  + staleTime: 5 * 60 * 1000,
  + refetchOnWindowFocus: false,

  Now consistent with the other hooks.

  ---
  Step 8 — Tree Shaking & Import Optimization

  MentorCardLong.tsx + MentorCardLong2.tsx — Replace moment with native Date API

  moment (~300 KB unpacked / ~67 KB gzipped) was imported in both card components and used only once each — to format a single date as DD/MM/YYYY. Replaced with a
  zero-dependency native helper:

  - import moment from "moment"

  + const formatDateDMY = (date: string) => {
  +   if (!date) return ""
  +   const d = new Date(date)
  +   return `${String(d.getDate()).padStart(2, "0")}/${String(d.getMonth() + 1).padStart(2, "0")}/${d.getFullYear()}`
  + }

  - {moment(mentor?.nextAvailableSessionDate).format("DD/MM/YYYY")}
  + {formatDateDMY(mentor?.nextAvailableSessionDate)}

  moment is now fully removed from both card components. Since these are dynamic()-loaded chunks, this directly reduces the weight of each card's JS chunk.

  ---
  MentorCardLong.tsx + MentorCardLong2.tsx — Merge duplicate lucide-react imports

  Two separate named-import lines from the same package prevent some bundlers from deduplicating the import graph entry:

  - import { Calendar1 } from "lucide-react"
  - import { StarIcon, UserCircleIcon } from "lucide-react"
  + import { Calendar1, StarIcon, UserCircleIcon } from "lucide-react"

  Same fix applied to MentorCardLong2.tsx where Calendar1 was imported at line 46 (far from the other lucide imports at line 25).

  ---
  PriceRangeSlider.tsx — Named imports instead of import * as

  import * as X (namespace import) pulls the entire module object and disables tree-shaking in most bundlers — only 4 of the slider package's exports are used:

  - import * as Slider from "@radix-ui/react-slider"
  + import { Root, Track, Range, Thumb } from "@radix-ui/react-slider"

  - <Slider.Root ...>  <Slider.Track ...>  <Slider.Range ...>  <Slider.Thumb ...>
  + <Root ...>         <Track ...>         <Range ...>          <Thumb ...>

  The bundler can now statically analyse which exports are used and drop the rest.

  ---
  Step 9 — UX Fix — Initial Loading Skeleton

  SearchedMentors.tsx — Always show multi-card skeleton on page load

  Root cause of the two-phase flicker:

  ┌───────┬─────────────────────────────────────────────────────────────────────────────────────────────────┬───────────────────────────────────────────────────┐
  │ Phase │                                          What happened                                          │                  Visible output                   │
  ├───────┼─────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────┤
  │ 1     │ isLoading = true (API in-flight)                                                                │ 1 card skeleton — <LoadingSkeleton /> rendered    │
  │       │                                                                                                 │ once                                              │
  ├───────┼─────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────┤
  │ 2     │ isLoading = false, data arrives, N MentorCardLong cards mount — but each is dynamic()-loaded so │ N card skeletons — each dynamic() fallback fires  │
  │       │  the chunk has not landed yet                                                                   │ independently                                     │
  └───────┴─────────────────────────────────────────────────────────────────────────────────────────────────┴───────────────────────────────────────────────────┘

  Phase 1 always showed exactly one skeleton regardless of how many results would arrive, causing a jarring layout shift when phase 2 expanded it to many cards.

  Fix: Render 5 <LoadingSkeleton /> cards in phase 1, matching the multi-card layout of phase 2:

  - {isLoading ? (
  -   <LoadingSkeleton />
  - ) : (
  + {isLoading ? (
  +   <div className="mb-6 flex flex-col gap-3">
  +     {Array.from({ length: 5 }).map((_, i) => (
  +       <LoadingSkeleton key={i} />
  +     ))}
  +   </div>
  + ) : (

  LoadingSkeleton itself is unchanged — the single-card version is still used correctly as the per-card loading fallback inside each dynamic() import.

  ---
  NPM Package Updates

  ┌─────────────┬────────────────────────────────────────────────────────┐
  │  Priority   │                        Packages                        │
  ├─────────────┼────────────────────────────────────────────────────────┤
  │ 🔴 Critical │ next, swiper                                           │
  ├─────────────┼────────────────────────────────────────────────────────┤
  │ 🟠 High     │ pdfjs-dist, tar, undici                                │
  ├─────────────┼────────────────────────────────────────────────────────┤
  │ 🟡 Moderate │ dompurify, follow-redirects, postcss, prismjs, tmp, ws │
  └─────────────┴────────────────────────────────────────────────────────┘

  ---
  Bundle Size Summary

  ┌───────────────┬─────────┬─────────┬────────────────┐
  │    Metric     │ Before  │  After  │     Delta      │
  ├───────────────┼─────────┼─────────┼────────────────┤
  │ Route size    │ 94.5 kB │ 81.2 kB │ −13.3 kB       │
  ├───────────────┼─────────┼─────────┼────────────────┤
  │ First Load JS │ 520 kB  │ 394 kB  │ −126 kB (−24%) │
  └───────────────┴─────────┴─────────┴────────────────┘

  ▎ Additional gain from Step 8: moment removal (~67 kB gzipped) is not reflected in the numbers above — those were captured before Steps 8–9 were applied.

  ---
  Future Scope

  - Redis caching for coupons — coupon data is read-heavy and changes infrequently; caching at the edge would eliminate redundant API calls across users
