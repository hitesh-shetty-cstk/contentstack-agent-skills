# The Onboarding Check

Every visual experience product ships a setup check that runs automatically when its panel opens.
There are **three** of them, and they differ in both gates and shape: Live Preview shows one card
with the first failing gate; Visual Editor shows a six-item list with a status per item; Timeline
shows a small status card while it checks, then a full-pane "Set Up Timeline" overlay listing three
items with a status each once one fails. Confirm which product the user is in before reading anything below. It is the highest-yield thing to ask
about, because it has already run the diagnosis the user is asking you to do.

**Always ask for a screenshot of this card before asking anything else.** The step name on it
localises the problem to one gate, and because of how the check is structured, it also tells you
everything that already passed.

## The property that makes the Live Preview and Timeline checks powerful

For Live Preview the gates are evaluated as a single ordered chain that stops at the first failure,
so only one step is ever shown. Timeline's three items are chained the same way, each evaluated only
after the previous one passed, so on its overlay the first empty circle is the failure and the ones
below it never ran. **Visual Editor is different**: every item is evaluated
independently and shown with its own status, so read the whole list rather than one card. The
inference below applies to the two chained checks, Live Preview and Timeline, not to Visual Editor.

So a card reading "Preview Service Not Enabled" is not just one fact. It proves the website loaded
in the frame, the SDK initialised, and the SDK version is supported. Three contracts are already
ruled out. Do not re-ask about them.

The inverse also holds. A card stuck on the first gate tells you nothing about any later gate,
because none of them ran. Do not theorise about the SDK when the site never loaded.

## Live Preview and Visual Editor

Five gates, in this order.

| # | Card while checking | Card on failure | What the failure means |
|---|---|---|---|
| 1 | Website Loading | **Could Not Connect to Website** | The site never rendered in the frame. Frame headers (`X-Frame-Options`, CSP `frame-ancestors`), an auth gate or password protection, a wrong or unreachable Base URL, HTTP (mixed content, no bypass) or an untrusted certificate (bypass: open the URL in its own tab, accept the warning, reload the pane), or the browser blocking localhost. Nothing after this ran. |
| 2 | Verifying Live Preview SDK | **Live Preview SDK Not Initialized** | The frame loaded but no init handshake arrived. `init()` is in server-only code, the enable flag did not parse as a boolean in the deployed build, the init module was tree-shaken out, or init runs after the check window. |
| 3 | Verifying Live Preview SDK | **Outdated Live Preview SDK Version** | The handshake arrived from a version below the supported minimum. Upgrade Live Preview Utils. |
| 4 | Verifying Preview Service | **Preview Service Not Enabled** | The SDK is fine and the site is fine, but content is not coming from the Preview Service. This is the fetch layer never switching host and headers, which is the most common failure of all. |
| 5 | — | **Default Environment Not Set** | Everything works. Setup is recorded as complete. The stack simply has no default preview environment, set in stack settings. |
| — | — | **Setup Complete** | All gates passed. |

The exact body text is worth quoting back to users, because they often paraphrase it into
something ambiguous. Gate 1 reads "Ensure the website is live and accessible via Contentstack
origins." Gate 4 reads "Please enable the Preview Service for a seamless live preview experience."

## Visual Editor

Visual Editor runs its own check, not Live Preview's. Six items, shown as a list with a status per
item. Every item is evaluated independently (`steps.every(isComplete)`), so a partially green list is
the normal way it looks while something is wrong, and the failing items are the ones to read. Card
text as observed in the UI; the identifier is the implementation's step id.

| # | Item (card text) | Step id | Passes when |
|---|---|---|---|
| 1 | Configure environment | `ENVIRONMENT` | both sub-steps below pass |
| 1a | Default Environment | `LP_DEFAULT_ENV` | the environment in the preview URL's parameters exists on the stack |
| 1b | Base URL | `BASE_URL` | that environment has a Base URL for the current locale, and its **origin matches the origin of the page being previewed**. A Base URL on a different host or scheme fails here even though the page loads |
| 2 | Install SDK | `LP_SDK_VERSION` | Live Preview Utils major version is 3 or higher |
| 3 | Verify Mode for Live Preview | `LP_SDK_INIT_MODE` | `init()` was called with `mode: "builder"`. **`mode: "preview"` fails this gate.** Edit tags alone do not get you a canvas |
| 4 | Preview Token | `LP_SERVICE` | after the init handshake, the editor polls the Preview Service and it responds; the site is fetching through the Preview Service, not the delivery CDN |

Two consequences worth stating:

- **Visual Editor needs three things, not two.** Working Live Preview, edit tags, and `mode: "builder"`
  in `init()`. The product table in SKILL.md says so; a customer who followed a Live Preview guide has
  `mode: "preview"` and will fail item 3 with everything else green.
- **Item 1b is origin-exact.** The Base URL must match the previewed page's scheme and host. A Base
  URL of `https://www.example.com` with the site served at `https://example.com`, or `http` versus
  `https`, fails the check while the frame still renders.

## Timeline

Timeline runs its own check with three items, not Live Preview's five or Visual Editor's six. Do not
map any of them onto another table. It has two surfaces:

1. **A small status card** at the bottom of the pane while the check runs. Its header names the item
   under test: "Default Environment", then "Live Preview SDK", then "Preview Token", then "All Set!".
   When everything passes it shows All Set! and disappears about two seconds later.
2. **A full-pane "Set Up Timeline" overlay** that replaces the card when an item fails. It lists all
   three items under "Get Started", each with a filled or empty circle. This is the screenshot to ask
   for.

| # | Overlay item | Passes when | Empty circle means |
|---|---|---|---|
| 1 | Configure the environment | an environment resolves for the session: the `environment` on the Timeline URL, falling back to the stack's default preview environment (`live_preview.default-env`) | no environment on the URL and no default preview environment in stack settings |
| 2 | Install the latest Live Preview SDK | item 1 passed and the SDK's `init()` handshake reached Timeline with a major version of 2 or higher | `init()` never ran inside the frame, or the SDK is 1.x |
| 3 | Generate and use Preview Token | item 2 passed and, inside the polling window, the tracker for this session's hash reports a REST or GraphQL Preview Service version, meaning a content request hit the Preview Service | the site fetched from the delivery CDN, or fetched nothing in time |

The items are chained: 2 is only evaluated after 1 passes, 3 only after 2. Read the **first** empty
circle. The ones below it never ran and tell you nothing.

**Timing.** The overlay appears ten seconds after the last status change if anything is still
unchecked, or after one second when no environment resolves at all. Item 3 polls the tracker up to
five times, first after one second and then every two, roughly a nine-second window. A site whose
first preview request lands later than that shows item 3 empty on a correct setup. Have the user
reopen the panel on a warm cache before believing it.

**Links on the overlay.** "Configure" opens Settings → Live Preview for the stack. "Install" and
"Preview Token" open help articles. The footer links to the Set Up Timeline documentation page.

**Closing it.** The X on the full overlay hides it for the current view only; nothing is stored. The
X on the small status card is where suppression happens: owners, admins and developers get a modal
offering "this session" (browser storage, per stack) or "for everyone" (writes
`timeline.onboarding-setup-visible: false` to stack settings); any other role gets the session
suppression silently.

## What the check does not cover

This is where the check earns its keep as a diagnostic, and where people over-trust it. "Setup
Complete" and a broken experience is a real and informative combination.

The check verifies reachability, the SDK handshake, SDK version, and that the Preview Service is in
use. It does not look at any of the following:

- **Edit tags.** Nothing in the chain inspects `data-cslp`. If the card says Setup Complete and
  nothing on the canvas is editable, the answer is edit-tag generation, every time. This is the
  single most useful inference the check supports.
- **Preview URL resolution.** The site loading is not the same as the correct entry loading. A card
  reading Setup Complete on the wrong page is a routing problem.
- **Locale, variants, and Timeline timestamps.**
- **Caching.** Any cache on the preview path can serve published content while every gate passes.
- **Roles.** The check runs as the signed-in user but does not report permission gaps, which is why
  a role problem looks like "works for everyone except one person" rather than a failed gate.

## When the card does not appear at all

Absence is not a pass. Before treating a missing card as "the check succeeded", rule out:

- No default environment is set and none is remembered locally for that locale, in which case the
  overlay is deliberately not shown.
- The user, or someone on their team, dismissed it. The preference persists per stack.
- The stack setting suppresses it.

**Do not reason about the default.** Each product keeps its own stack setting, and they disagree on
what an unset value means:

| Product | Setting | Unset resolves to |
|---|---|---|
| Timeline | `timeline.onboarding-setup-visible` | visible |
| Live Preview | `live_preview.lp-onboarding-setup-visible` | hidden |
| Visual Editor | `visual_builder.onboarding-setup-visible` | depends on whether the settings object exists |

The Visual Experience settings screen renders every one of these as **on** when unset, so a stack
that never explicitly saved the setting can show the toggle on while the card never appears.

The fix is one action: open Settings → Visual Experience and Save, without changing anything. That
form submits all three products' keys together and an unset value arrives as on, so one save writes
an explicit value for all three and they stop disagreeing.

If it then appears for colleagues but not for the reporter, they dismissed it. That suppression
lives per stack in their own browser storage and no stack-level save clears it.

If the user cannot produce the card, fall back to the four contracts in SKILL.md and verify them by
hand.

## How to ask for it

Ask for the screenshot rather than the wording. Users paraphrase the step name into a different
one, and since each name maps to a specific gate, a paraphrase can send you down the wrong branch.

If they cannot screenshot it, ask for the exact step name and body text as displayed, and confirm
which product they are in. Live Preview shows one card, Timeline a three-item overlay, Visual Editor
a six-item list, and none of them share gates; a paraphrased step name from the wrong product sends
you down the wrong branch.
