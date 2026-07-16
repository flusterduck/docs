# Flusterduck Guide

Flusterduck already knows where your users get stuck. Guide tells them what to do about it.

It's a user-triggered help layer that lives inside your product. When someone gets confused, they press a button or hit a keyboard shortcut, and Guide explains the element they're looking at: what it does, what happens if they click it, what comes next. Plain language, generated from real friction data, specific to the exact screen and state they're in right now.

Not a tooltip. Not a chatbot. Not a product tour someone authored six months ago. A live explanation of whatever the user is pointing at, informed by what Flusterduck knows about where people actually struggle.

## Why this exists

Powerful software loses users at the moment of confusion. Someone opens a dashboard, a settings panel, an admin console, and freezes. They don't know what a button does. They're scared to click because they can't tell whether the next action charges their card, deletes something, or triggers a notification they can't undo.

That fear is the thing. The task isn't hard. The not-knowing is.

Flusterduck's detection engine already identifies exactly where this happens. It watches 34 behavioral signals across thousands of sessions and clusters them into ranked friction points. Guide takes that data and does something about it on the spot: explains the confusing thing before the user gives up.

## How it fits into what you already run

Flusterduck's existing cycle:

1. **Detect** friction from real user behavior
2. **Cluster** repeated signals into issues
3. **Rank** by revenue at risk
4. Your team fixes it
5. **Verify** the fix recovered revenue

Guide inserts a new step between rank and fix. Many friction points don't need a redesign and a sprint. They need four sentences: "This button creates a new environment. It won't affect your production data. You can delete it later from Settings. Most people start here."

Instead of filing a ticket and waiting, you deploy an explanation at that exact spot. The verify step you already have proves whether it worked. That collapses the loop from weeks to minutes for every friction point that's really a confusion problem, not a true bug.

## The read-only constraint

Guide never touches anything itself. It only explains. It can't click, can't fill forms, can't run actions, can't change state. This is deliberate.

A wrong action inside a customer's product is a support incident. A wrong explanation is a minor annoyance you can fix with a cache clear. The risk profile is completely different, and it means Guide can ship without the reliability and liability burden that "do it for me" agents carry.

Users keep full control. They just stop being afraid, because they can see what's behind a button before pressing it.

## What users see

The user triggers Guide mode. Their cursor changes. They hover over an element. A small card appears near the cursor with:

- What the element does
- What would happen if they clicked it
- What the next step typically looks like
- If Flusterduck has friction data: a note like "87 users clicked this expecting it to save, but it only previews. Hit the green button below to actually save."

The card disappears when they move to a different element or exit Guide mode. No overlays. No modals. No interruptions.

## What makes it different from tooltips

WalkMe, Pendo, Appcues, and every onboarding tool on the market show guidance someone wrote ahead of time. That guidance is finite, goes stale, and only covers what the author thought to write. The weird button in the advanced settings that confuses 12% of users? Nobody wrote a tooltip for that.

Guide generates explanations on the fly from the actual interface state and the friction data Flusterduck already has. It has no edge. It can explain the one control nobody anticipated, because it reads the screen and knows from behavioral data what specifically confuses people about it.

A pre-written tooltip tells you what the author thought you'd need to know. Guide tells you what real users actually struggled with.
