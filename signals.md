# Signals

Signals are the raw evidence of user confusion. Every time a user does something that indicates friction, the SDK emits a signal. The scoring engine weighs signals by type, frequency, and recency to compute page confusion scores and cluster repeated patterns into issues.

There are 128 built-in signal types across eight tiers, all detecting on a default install. Any signal can be switched off per site from the served SDK config.

## Tier 1: Direct friction

High-confidence indicators. A single occurrence is meaningful. You don't need volume to act. These move your page score the most and surface issues fastest.

If you see tier 1 signals on a critical element (a checkout button, a signup form, an upgrade CTA), investigate before waiting for a cluster.

| Signal | What it means |
|---|---|
| `rage_click` | User clicked the same element 3 or more times rapidly. Usually means the element appears clickable but isn't responding as expected. |
| `dead_click` | User clicked a non-interactive element. Often a label, image, or decorative element the user thought was a button. |
| `layout_shift_rage` | A rage click followed by a significant layout shift. The interface moved under the user's cursor. |
| `active_funnel_rejection` | User initiated a multi-step flow and exited before the first meaningful step. Not a casual bounce: they started, then left. |
| `dead_end_submit` | User submitted a form and received no visible feedback. No success state, no error message, nothing. |
| `accidental_click_bounce` | User clicked an element and immediately hit the back button. Likely an accidental tap or misclick. |
| `error_recovery_loop` | User encountered an error and tried the same action multiple times in quick succession without changing their input. |
| `silent_failure_retry` | User repeated an action that appeared to succeed but produced no visible result. Common with async operations that fail silently. |
| `form_abandonment` | User focused one or more form fields and left without submitting. Threshold: at least one meaningful field interaction. |
| `error_encounter` | An unhandled JavaScript error, rejected promise, failed network request, or console-logged error occurred during the session. Console capture matters because most production errors never reach `window.onerror`: error boundaries and `.catch(console.error)` swallow them, and the console is where they land. Messages are scrubbed of tokens, secrets, and PII in the browser; identical messages emit once per session with a hard per-session cap. Together with interaction context this is the flight recorder: what the code was doing at the moment users struggled, attached to issues as evidence and fed to the AI diagnosis and Autofix briefs, with no session recording anywhere. |
| `form_validation_loop` | User submitted a form, hit a validation error, edited the flagged field, and got the same error again. Three or more cycles on a single field within two minutes fires the signal. |
| `invisible_required_field` | User submitted a form and validation failed on a required field they can't see. The field is offscreen, inside a closed `<details>`, behind an inactive tab, or hidden by a parent with `display:none`. |
| `offline_interaction_blackhole` | The device is offline and the user keeps clicking interactive elements that cannot respond, with nothing on the page saying the connection is gone. |
| `spa_back_button_dead` | Browser Back presses produce no visible reaction: no route change, no DOM update, no scroll restore within the settle window. |
| `otp_entry_struggle` | A pasted one-time code lands in only the first box of a segmented verification input, or the user gives up and clicks resend. Only the paste length is reported, never the code. |
| `captcha_challenge_rage` | The CAPTCHA challenge frame is inserted repeatedly within a minute: users keep failing or re-triggering the challenge before they can proceed. |
| `payment_widget_failure` | A payment provider iframe (Stripe, PayPal, Braintree, Adyen, Square) never loaded or rendered at zero size on a visible page: checkout shows a blank payment area. Only the provider host is reported, never the frame URL. |
| `auth_roundtrip_limbo` | The user sits on a check-your-email screen, leaves for their inbox and returns 2+ times over 90+ seconds, and the verification never completes. |
| `deep_link_dead_end` | An external or campaign arrival landed on a page whose title or top heading reads as not-found. Only the referrer host is reported, never the full referrer. |
| `ghost_toggle` | User clicked a toggle, checkbox, or switch and the state either never changed (swallowed click) or changed then silently reverted within 2 seconds. Requires 2+ ghost events on the same control within 30 seconds. |
| `premature_commit_reversal` | User clicked a primary CTA (submit, buy, confirm, add to cart) and reversed the action within 2 seconds via browser back, Ctrl+Z, Escape, or clicking a cancel element. The speed of the reversal indicates regret or accidental commitment. |
| `cls_interaction_corruption` | A layout shift moved the user's click target between mousedown and mouseup. They pressed on the right element, but the page yanked it away and the click landed on something else. |
| `paste_blocked` | User pasted into an input field but the value didn't change. The site is blocking paste, usually on password confirmation or email fields. |
| `autocomplete_fight` | Browser autofill populated multiple form fields and site JavaScript immediately cleared or shortened the values. Common on checkout, login, and address forms where custom input masking fights the browser. |

## Tier 2: Navigation and flow friction

Patterns that emerge across a session. Each single occurrence is moderate-confidence. Clusters are high-confidence.

If a tier 2 signal appears 20+ times on the same page, or you're seeing 5+ different tier 2 types on the same page simultaneously, treat it like tier 1.

| Signal | What it means |
|---|---|
| `u_turn_navigation` | User went forward in a flow, then immediately went back. Repeated occurrences suggest unclear copy or a missing expectation-setter. |
| `information_maze` | User visited the same page or section multiple times without completing anything. They're looking for something they can't find. |
| `load_abandonment` | User waited for a page to load and left before it finished. Indicates a performance problem on a high-friction path. |
| `multi_step_reset` | User restarted a multi-step form or wizard from the beginning. Could be data entry confusion or unclear field validation. |
| `query_thrashing` | User made multiple search or filter queries in rapid succession. They're not finding what they're looking for. |
| `filter_spiral` | User applied and removed filters repeatedly without settling. Likely an expectation mismatch between filters and results. |
| `post_action_anxiety` | User completed an action but immediately checked for confirmation or left the page. Suggests missing confirmation feedback. |
| `settings_hop` | User visited settings or preferences multiple times in a session without finding what they needed. |
| `copy_paste_rework` | User selected text and then immediately re-typed it. Often indicates a pre-fill or autocomplete that didn't work. |
| `repeat_form_input` | User cleared and re-entered the same field more than once. Usually a validation message that's unclear or delayed. |
| `bulk_action_miss` | User committed the same value into 4 or more structurally identical sibling fields, one row at a time, within 3 minutes. A copy-to-all control went unnoticed, or doesn't exist. The value itself never leaves the browser: only counts and field identity are reported. |
| `decision_paralysis` | User's cursor oscillated between two or more interactive elements (pricing tiers, plan selectors, CTAs) with 800ms+ dwell on each, repeating the back-and-forth at least twice within 12 seconds. They can't decide. |
| `friction_cascade` | A tier 1 signal fired and was followed by 2+ distinct tier 2 or tier 3 signal types within 90 seconds on the same page. The tier 1 signal is the root cause; the followers are symptoms. |
| `keyboard_trap` | Focus can't escape a container. The user pressed Tab 6+ consecutive times and focus stayed within the same parent region without the user clicking elsewhere. |
| `modal_stack` | Two or more modal, dialog, or overlay elements are visible at the same time. Stacked modals confuse users: dismiss buttons close the wrong layer, keyboard focus traps conflict, and users lose track of which layer they're interacting with. |
| `dead_click_trap_zone` | Three or more dead clicks landed in the same viewport grid zone (the screen is divided into a 4x4 grid) within 90 seconds. A hot spot of non-interactive content that users keep mistaking for a button. |
| `navigation_confusion` | User visited 5+ distinct pages within 30 seconds, spending less than 5 seconds on each. They're lost and clicking through pages trying to find something. |
| `scroll_to_click_confusion` | User scrolled down, reversed direction by at least 30% of the viewport, and clicked an element within 3 seconds of the reversal. They overshot something they wanted and had to scroll back to find it. |
| `tab_thrash` | User pressed Tab 10+ times within 5 seconds. Rapid tabbing with direction changes signals they're hunting for a focusable element they can't reach. |
| `focus_trap` | User pressed Tab or Escape 5+ times within 3 seconds inside a dialog, menu, or modal container without escaping it. The trap isn't releasing focus when the user expects it to. |
| `keyboard_nav_frustration` | User pressed arrow keys 15+ times within 5 seconds inside a listbox, menu, or tree widget. They're scrolling through options without finding what they want. |

## Tier 3: Passive confusion

Lower individual weight, but meaningful in aggregate. These often surface chronic friction that lacks the dramatic edge of rage clicks, the kind of friction users tolerate rather than abandon over.

A page with dense tier 3 signals and no tier 1 or 2 signals is worth investigating. It usually means users are managing confusion rather than hitting a wall.

| Signal | What it means |
|---|---|
| `confidence_collapse` | User paused on a form field for an unusually long time without typing. Hesitation before a field that's unclear. |
| `target_near_miss` | User's tap or click landed very close to an interactive element but missed it. Common on mobile with small touch targets. |
| `user_confusion_idle` | User stopped interacting entirely for longer than expected. Could be reading carefully, or could be stuck. |
| `thrash_hover` | User rapidly moved the cursor back and forth over an element without clicking. Indecision or unclear affordance. |
| `text_selection_thrash` | User selected text multiple times rapidly. May be trying to copy something that's being blocked. |
| `jerky_scrolling` | User's scroll pattern was erratic: fast then stopped, or reversed frequently. Usually indicates a busy page layout. |
| `thrash_cursor` | Rapid cursor movement across the page with no clear direction. A general confusion indicator. |
| `viewport_thrashing` | User zoomed in and out multiple times. On mobile, this usually means text is too small or layout is not responsive. |
| `visibility_thrashing` | User repeatedly scrolled elements in and out of view, possibly trying to compare content across sections. |
| `passive_drift` | User stayed on a page longer than average but generated very few events. Could be reading carefully or completely lost. |
| `help_hunt` | User opened FAQ, support, documentation, or help pages during an active conversion flow. |
| `trust_hesitation` | User focused a sensitive field (password, email, credit card) and then clicked through to a privacy, terms, security, or about page before typing more than 2 characters. They didn't trust the form enough to fill it in. |
| `tooltip_dependency` | User hovered over the same tooltip trigger 3+ times within 5 minutes, spending 500ms+ on each hover. The label is unclear and they keep re-reading the explanation. |
| `phantom_progress` | A progress indicator (stepper, progress bar, step counter) is visible but the user has been stuck on the same step for 60+ seconds while actively clicking and typing. The wizard isn't advancing. |
| `scroll_to_nowhere` | User scrolled to the bottom of the page and immediately scrolled back up within 3 seconds. They expected more content below, or couldn't find what they were looking for. |
| `notification_fatigue` | Browser notification permission prompt was denied within 800ms. A sub-second denial means the prompt was unwanted or the site asked for permissions before establishing trust. |
| `video_autoplay_escape` | A video or audio element autoplayed and the user immediately paused it, muted it, or scrolled past it within 3 seconds. Autoplaying media is annoying enough that they took evasive action. |
| `scroll_hijack` | User's scroll reversed direction 4+ times within 3 seconds with large-stroke reversals. The page is overriding native scroll behavior (smooth-scroll libraries, scroll-snap fights, parallax effects). |
| `input_correction` | User deleted 3+ characters and retyped a field at least twice within 60 seconds. They keep correcting their input, likely because a validation rule or format requirement is unclear. |
| `hover_dwell` | User hovered over an interactive element for 2+ seconds without clicking. They're reading it, evaluating it, or unsure what it does. Cooldown of 30 seconds per element prevents noise. |
| `exit_intent` | User's cursor left the browser viewport toward the top of the screen (toward the tab bar or address bar). Classic indicator they're about to close the tab or leave. Fires after 3+ seconds on page, once per 60 seconds. |
| `copy_frustration` | User triggered the copy action 3+ times within 15 seconds. Repeated copy attempts suggest the selection isn't working as expected or copy is being blocked. |
| `scroll_depth_abandon` | Tracks the deepest scroll point reached before the user leaves the page. Fires on page hide or visibility change with the max scroll depth as a percentage. Useful for identifying where users give up on long pages. |

## Mobile and touch friction

Gesture-level friction that mostly shows up on touch devices, though `false_swipe` also fires for mouse and pen drags.

| Signal | What it means |
|---|---|
| `false_swipe` | User performed a clear horizontal swipe or drag over an element that looks swipeable (a carousel, slider, scroll-snap rail, or anything with draggable/grab styling), but the element and its scrollable ancestors did not move and no page transition occurred. A false affordance: it looked swipeable, the user tried, and nothing responded. |
| `swipe_miss` | User swiped horizontally on an element that isn't swipeable, but the page has swipeable elements elsewhere (a carousel or slider exists). They swiped in the wrong spot. |
| `orientation_thrash` | User rotated their device 3+ times within 30 seconds. They're trying to find an orientation where the content is readable or the layout makes sense. |

## Revenue and close behavior

Signals that correlate directly with lost conversions. These feed the revenue impact estimate.

| Signal | What it means |
|---|---|
| `close_click_reversal` | User moved toward closing, dismissing, or canceling something, then stopped and stayed. They were about to leave but didn't. Use this to find moments where users reconsider. |
| `value_collapse_downgrade` | User selected a higher plan or tier, then downgraded before completing purchase. Usually indicates a pricing page clarity problem. |
| `layout_exhaustion_settling` | User scrolled through the full page, returned to the top, and selected an option. They were overwhelmed and had to start over to decide. |

## Opt-in interaction signals

Three newer detectors target obstruction and blocked-interaction friction. They are **off by default** while we calibrate their thresholds on real traffic, so enable the ones you want explicitly:

```ts
import { init } from 'flusterduck'

init({
  key: 'fd_pub_...',
  signals: {
    disabledElementAttempt: { enabled: true },
    overlayDismissStruggle: { enabled: true },
    stickyObstructionClick: { enabled: true },
  },
})
```

| Signal | What it means |
|---|---|
| `disabled_element_attempt` | User clicked a disabled (or disabled-looking) control more than once with no response. Either the disabled state is wrong, or there's no explanation of why the control is unavailable. |
| `overlay_dismiss_struggle` | User pressed Escape and jabbed at the backdrop or close affordance repeatedly while a modal, dialog, or popup stayed open. The dismiss controls aren't working or aren't where users expect. |
| `sticky_obstruction_click` | User's click landed on a sticky or fixed element (a header, floating bar, or cookie banner) that's covering the interactive target directly beneath it. |

Every fire carries a `cf` confidence value (0-1); the scoring engine weighs each fire by it, so a borderline detection counts for less than an unambiguous one.

## Performance

Signals tied to page speed and responsiveness. These correlate with user frustration but originate from browser performance metrics rather than user behavior.

| Signal | What it means |
|---|---|
| `long_task_interaction_block` | User clicked, tapped, or pressed a key during or immediately after a long task (50ms+ main thread block). The browser's event loop was frozen and their input was delayed. Payload includes the task duration and estimated delay. |
| `loading_interaction` | User clicked an element that was in a loading state (marked with `aria-busy`, disabled, or a spinner CSS class) and clicked it again before the loading finished. They're waiting and losing patience. |
| `lcp` | Largest Contentful Paint measurement for the page. Rated good (under 2500ms), needs improvement (2500-4000ms), or poor (over 4000ms). |
| `inp` | Interaction to Next Paint measurement. Rated good (under 200ms), needs improvement (200-500ms), or poor (over 500ms). Captures how responsive the page feels after it loads. |
| `cls` | Cumulative Layout Shift score. Rated good (under 0.1), needs improvement (0.1-0.25), or poor (over 0.25). Measures how much the page jumps around during loading. |

## Aggregate

Session-level and cross-signal patterns. These don't detect a single friction moment. They summarize friction across an entire session or correlate multiple signals into a higher-level finding.

| Signal | What it means |
|---|---|
| `session_toxicity_score` | A running 0-100 score reflecting cumulative friction across the session. Computed from the weighted sum of all signals, the number of distinct pages with friction, and the number of distinct signal types observed. Levels: calm (0-20), building (21-40), frustrated (41-60), toxic (61-80), meltdown (81-100). Emitted every 60 seconds and on page unload. |
| `frustration_burst` | Multiple friction signals fired within a 60-second window and the weighted total crossed a severity threshold. Levels: low, medium, high, critical. Includes the top contributing signal types and detected behavioral sequences (like rage click followed by form validation loop). |
| `element_impression` | Tracks when a watched element (marked with `[data-fd-impression]` or a custom selector) enters the viewport. Records first-seen time and visibility percentage. Not a friction signal, but useful for measuring whether users actually see a specific element before abandoning. |

## UX anti-patterns

Signals that detect specific anti-patterns in page design. These are distinct from general friction indicators because they point at a concrete design problem, not just user confusion.

| Signal | What it means |
|---|---|
| `viewport_dead_zone` | User clicked an interactive element in the bottom 10% of the viewport, but a fixed or sticky overlay (cookie banner, floating bar, sticky header, chat widget) intercepted the click. The intended target never received it. |
| `input_format_roulette` | User cleared and retyped a formatted field (phone, date, credit card, zip code) 3+ times with different formats within 60 seconds. The expected format is ambiguous and the field isn't telling them what it wants. |
| `undo_panic` | User clicked a destructive action (delete, remove, archive, send, publish) and panicked within 3 seconds: hit Ctrl+Z, thrashed the cursor, or went back. Tracks whether an undo UI (toast, snackbar) appeared in response. |
| `scroll_hijack_rage` | Programmatic scroll override moved the viewport and the user fought back with 3+ rapid scroll inputs in the opposite direction within 2 seconds. Distinct from `scroll_hijack` (reversal bursts): this fires when the user is visibly resisting the override. |
| `text_select_frustration` | User tried to select text 3+ times rapidly on the same element. Often indicates copy protection, CSS `user-select: none`, or an element that swallows selection events. |

## Conversion friction

Signals specific to pricing, checkout, and purchase flows. These fire where money is on the line, so even low-volume occurrences are worth investigating.

| Signal | What it means |
|---|---|
| `pricing_comparison_stall` | User's cursor or focus oscillated between pricing tiers for 15+ seconds without selecting one. They're stuck comparing and can't tell which plan fits. |
| `checkout_field_retreat` | User filled in a checkout field, moved to the next field, then went back and cleared or edited the previous one. Repeated retreats on the same field indicate confusing labels or unexpected validation. |
| `coupon_code_frustration` | User attempted to apply a coupon or promo code 3+ times within 60 seconds. The code isn't working and there's no clear explanation why. |
| `price_shock_abandon` | User reached a payment or order summary screen showing a total and left within 5 seconds without interacting. The final price surprised them. |
| `payment_method_thrash` | User switched between payment methods (credit card, PayPal, Apple Pay, etc.) 3+ times without completing payment. Either a method is failing silently or the user can't find the one they want. |
| `billing_shipping_confusion` | User copied or re-typed values between billing and shipping address fields, or toggled the "same as billing" checkbox 2+ times. The relationship between the two forms is unclear. |
| `plan_comparison_scroll` | User scrolled a pricing comparison table or feature matrix back and forth horizontally 4+ times within 30 seconds. The table is too wide or the differentiators aren't visible without scrolling. |
| `pricing_toggle_regret` | User toggled between monthly and annual billing, selected a plan, then toggled back and selected again. They second-guessed the billing interval after committing. |
| `faq_bounce` | User opened an FAQ or help section during a purchase flow, read it, then left the flow entirely instead of returning to checkout. The FAQ didn't resolve their concern. |
| `setup_step_abandon` | User completed at least one step of a post-purchase setup wizard and left before finishing. They paid but didn't activate the product. |
| `integration_retry_failure` | User attempted to connect a third-party integration (OAuth flow, API key entry, webhook test) and the connection failed 2+ times within 5 minutes. Broken integrations during onboarding kill activation. |
| `first_value_delay` | More than 60 seconds elapsed between account creation and the user's first meaningful action (creating a project, uploading a file, running a query). They signed up but couldn't figure out what to do next. |

## Retention friction

Signals tied to settings, cancellation, and feature adoption. These surface churn risk before it shows up in your billing dashboard.

| Signal | What it means |
|---|---|
| `settings_churn` | User visited account, billing, or subscription settings 3+ times within 7 days without making a change. They're thinking about downgrading or canceling but haven't pulled the trigger. |
| `cancellation_flow_confusion` | User entered a cancellation or downgrade flow and clicked back, restarted, or spent 30+ seconds on a single step. The flow is unclear or the retention offer is confusing them further. |
| `downgrade_hesitation` | User selected a lower-tier plan, reached the confirmation step, and abandoned without confirming. They want to downgrade but something on the confirmation screen stopped them. |
| `feature_discovery_failure` | User searched for a term, navigated to 3+ pages, or opened help docs looking for a feature that exists but that they can't find. The feature is buried or named in a way that doesn't match what users expect. |
| `reactivation_attempt` | A user who previously canceled or let their subscription lapse returned and attempted to resubscribe or re-enable their account. Tracks whether the reactivation succeeded or hit an error. |

## Content friction

Signals that fire when users struggle to read, understand, or consume content on the page.

| Signal | What it means |
|---|---|
| `reading_abandonment` | User scrolled past 30% of an article or long-form page, stopped scrolling for 10+ seconds, then left. They started reading and gave up mid-content. |
| `copy_failure` | User attempted to copy text (Ctrl+C, right-click > Copy, or long-press on mobile) and the clipboard didn't receive the expected content. Copy protection, JavaScript interception, or CSS `user-select: none` blocked them. |
| `content_overload_scroll` | User scrolled a content-heavy page at high velocity (covering 50%+ of the viewport per second) for 3+ seconds without stopping. They're skimming past content that's too dense to read. |
| `jargon_hover_loop` | User hovered over the same tooltip, abbreviation tag, or glossary term 3+ times within 2 minutes. The term is unclear and the explanation isn't sticking. |
| `pagination_thrash` | User clicked forward and backward through paginated content 4+ times within 60 seconds. They're hunting for something specific and the pagination doesn't help them find it. |

## Error handling friction

Signals that surface when error states, permissions, or validation timing confuse users.

| Signal | What it means |
|---|---|
| `error_blindness` | An error message appeared on the page but the user continued interacting without acknowledging it for 10+ seconds. They didn't see the error, or the error wasn't visually prominent enough to interrupt their flow. |
| `silent_logout_surprise` | User attempted an authenticated action and was redirected to a login page without warning. Their session expired silently and the redirect broke their workflow. |
| `validation_timing_mismatch` | A form field showed a validation error while the user was still actively typing in it. The validation fired before the user finished their input, creating a false error that disappears on its own. |
| `error_recovery_abandon` | User encountered an error, attempted 1-2 recovery actions (refresh, re-submit, navigate back), then left the page entirely. The error was unrecoverable or the recovery path was unclear. |
| `permission_denial_break` | User attempted an action that requires elevated permissions (admin, owner, paid tier) and received a denial. They either didn't know the restriction existed or expected to have access. |

## Mobile and touch friction (expanded)

Additional gesture-level and device-specific friction signals. These complement the existing `false_swipe`, `swipe_miss`, and `orientation_thrash` signals in the base mobile tier.

| Signal | What it means |
|---|---|
| `gesture_mismatch` | User performed a recognized gesture (pinch, rotate, two-finger scroll) on an element that doesn't support that gesture type. The element's visual affordance suggested a gesture vocabulary it doesn't actually handle. |
| `orientation_thrash` | User rotated their device and the layout broke or reflowed in a way that caused immediate scroll correction or a rage tap. Distinct from the base `orientation_thrash` signal: this variant fires only when the rotation triggers a follow-up frustration signal within 3 seconds. |
| `thumb_zone_miss` | User tapped a target in the top 20% of the screen (outside the natural thumb zone) and missed on the first attempt. They had to stretch or shift grip to reach it. Fires only when a second tap on the same target succeeds within 2 seconds. |
| `mobile_keyboard_dismiss` | User tapped outside an input to dismiss the on-screen keyboard and accidentally triggered an action on the element behind it. The keyboard dismissed and a button or link underneath received the tap. |
| `tap_target_adjacency` | User tapped between two adjacent interactive elements (less than 8px gap) and hit the wrong one, then immediately tapped the other. The targets are too close together. |
| `responsive_layout_break` | A media query or container query breakpoint triggered and caused content to overflow its container, text to truncate mid-word, or interactive elements to overlap. Detected by comparing element bounds before and after a resize or orientation change. |

## Performance friction (expanded)

Additional performance signals beyond the base `long_task_interaction_block`, `loading_interaction`, `lcp`, `inp`, and `cls` metrics.

| Signal | What it means |
|---|---|
| `font_swap_interaction` | A web font loaded and swapped while the user was actively reading or interacting with text. The FOUT (flash of unstyled text) caused a visible text reflow during an active session, not just on initial page load. |
| `third_party_script_block` | A third-party script (analytics, chat widget, ad tag) blocked the main thread for 100ms+ during user interaction. The delay is attributable to an external domain, not the app's own code. |
| `memory_pressure_jank` | The browser signaled memory pressure (via the `Device Memory API` or performance observer long-task clustering) and the user experienced visible jank: dropped frames during scroll, animation stutter, or input delay exceeding 500ms. |
| `image_decode_delay` | A large image decoded on the main thread and blocked interaction for 100ms+. Common with unoptimized PNGs, large JPEGs without progressive encoding, or images decoded synchronously during scroll. |

## Accessibility friction

Signals that detect friction specific to assistive technology users and accessibility-related interaction patterns.

| Signal | What it means |
|---|---|
| `focus_order_confusion` | User pressed Tab and focus moved to an element that's visually distant from the previously focused element. The DOM order and visual order disagree, causing a disorienting jump. Fires when the focus target is more than 500px from the previous focus target. |
| `screen_reader_dead_end` | User moving through the page via `aria-` landmarks or heading shortcuts (`H` key in screen readers) reached a point with no further landmarks or headings, then reversed direction. The page structure has a dead end that forces backtracking. |
| `motion_sensitivity_trigger` | An animation or transition ran while the user has `prefers-reduced-motion: reduce` set. The site ignored the user's motion preference. |
| `contrast_interaction_failure` | User hovered or focused an interactive element and the resulting state change (hover color, focus outline) didn't meet WCAG AA contrast ratio against its background. The interactive feedback is invisible to users with low vision. |

## Custom signals

Emit your own signals with `signal()`:

```ts
import { signal } from 'flusterduck'

// A CTA that looks active but is gated by a required previous step
signal('dead_click', {
  selector: '[data-action="publish"]',
  label: 'Publish',
  blocked_by: 'missing_title',
})

// A custom multi-step flow step abandonment
signal('form_abandonment', {
  form: 'workspace-setup',
  step: 'invite-members',
  fields_touched: 0,
})
```

The first argument is the signal type: any of the 128 built-in types or a custom string. The second is optional metadata. Never include PII, form values, or anything a user typed.

Custom signal types appear in the dashboard and scoring engine alongside built-in types. They don't get tier weights assigned automatically. The scoring engine treats them as tier 2 by default until enough data accumulates for it to assess their actual impact.

## Business events

Wire conversion outcomes with `track()` for revenue impact estimation:

```ts
import { track } from 'flusterduck'

track('plan_intent', {
  plan_id: 'scale',
  amount_cents: 9900,
  billing: 'monthly',
})

track('subscription_started', {
  plan_id: 'scale',
  amount_cents: 9900,
  billing: 'monthly',
})

track('checkout_abandoned', {
  plan_id: 'grow',
  amount_cents: 3900,
})
```

Safe properties: `plan_id`, `amount_cents`, `billing`, `currency`, `quantity`, `product_id`. Never pass names, emails, addresses, or any user-typed text.

## Legacy aliases

These older signal names still work:

| Alias | Resolves to |
|---|---|
| `speed_frustration` | `silent_failure_retry` |
| `loop_nav` | `u_turn_navigation` |
| `scroll_bounce` | `jerky_scrolling` |
| `form_hesitation` | `confidence_collapse` |
| `form_abandon` | `form_abandonment` |
| `tap_miss` | `target_near_miss` |
| `pinch_zoom` | `viewport_thrashing` |

## How scores and issues work

Signals don't directly create issues. The scoring engine clusters signals by element, page, and user population. When a cluster crosses a confidence threshold, it becomes a UX issue. The confusion score for a page reflects the weighted sum of recent signals, normalized for session volume.

Tier weighting means a single `rage_click` moves a page score more than a single `passive_drift`. But 200 `passive_drift` signals on the same element will create an issue regardless of tier: frequency overrides weight at scale.
