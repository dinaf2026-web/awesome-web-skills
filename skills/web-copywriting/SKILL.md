---
name: web-copywriting
description: Web copywriting patterns for developers — headlines, CTAs, microcopy, value propositions, error messages, and onboarding copy. Converts visitors into users.
origin: community
tags: [copywriting, landing-page, conversion, ux-writing, microcopy]
---

# Web Copywriting

Developers ship features. Copywriters ship meaning. This skill does both.
Good copy is not decoration — it is part of the product. Weak copy tanks
conversion even when the product is excellent.

---

## 1. When to Use

Use this skill when working on:

- Landing pages and marketing sites
- SaaS onboarding flows
- Pricing pages
- Error messages and empty states
- Button labels and form microcopy
- Email subject lines and welcome sequences
- Tooltips and in-app guidance
- Product announcements and changelogs

Do NOT use this skill for long-form content (blog posts, documentation,
technical writing). Those follow different rules.

---

## 2. The Copywriting Hierarchy

Every page or screen has a hierarchy. Users scan before they read.
Structure copy in this order — each layer earns the next.

```
HEADLINE        → Stops the scroll. States the promise.
SUBHEADLINE     → Explains how or who it's for.
BODY            → Proves the promise. Handles objections.
CTA             → Tells them exactly what to do next.
```

### Rule: The headline does 80% of the work

If the headline fails, nothing below it gets read. Write the headline last,
after you understand the full value proposition.

### Rule: One idea per screen

Each section earns one promise. When a section tries to say three things,
it says nothing.

### Rule: The reader is always asking "so what?"

After every sentence, ask it. If you cannot answer it, cut the sentence.

---

## 3. Headline Formulas

### Problem-Solution

Format: `[Painful problem] — [Your product] fixes it.`
Or: `Stop [doing painful thing]. Start [doing desired thing].`

Examples:
- "Stop chasing late invoices. Get paid on time, automatically."
- "Your spreadsheet is lying to you. Claros gives you numbers you can trust."
- "Tired of rebuilding the same auth every project? Ship auth in 4 minutes."

When to use: Audience is already aware of the problem. They just haven't
found the right solution yet.

---

### Outcome-Based

Format: `[Specific result] in [timeframe] — without [common obstacle].`

Examples:
- "Rank on page one in 90 days — without hiring an SEO agency."
- "Cut your AWS bill by 40% this month, without touching your architecture."
- "Go from idea to live app in a weekend, even if you've never shipped before."

When to use: Result is clear and desirable. Timeframe makes it feel real.
The "without" clause removes the biggest objection.

---

### Curiosity

Format: `What [unexpected group] knows about [topic] that you don't.`
Or: `The [thing everyone does] that's actually [costing/hurting] you.`

Examples:
- "What solo consultants charge that agencies never will."
- "The deployment habit that's silently killing your uptime."
- "Why your best engineers keep leaving — and it's not the salary."

When to use: Audience needs to be shaken out of complacency. Use sparingly.
Curiosity headlines must pay off immediately in the body or trust collapses.

---

### Specificity

Format: Use real numbers, real names, real timeframes. Vague claims are
invisible. Specific claims are credible.

Examples:
- "Used by 14,200 developers at companies like Stripe, Linear, and Vercel."
- "Reduced deployment time from 47 minutes to 6."
- "The tool Figma's design team uses to QA every release."

When to use: Always. Even other headline types get stronger with specifics.
"Save time" is weak. "Save 3 hours a week" is a headline.

---

## 4. Value Proposition Framework

A value proposition is one sentence that answers three questions:

- **Who** is this for?
- **What** does it do?
- **Why** is it different from alternatives?

### Formula

```
[Product] helps [specific audience] [achieve outcome] by [differentiator].
```

### Examples

**Weak:** "A better project management tool for teams."
**Strong:** "Plane helps engineering teams ship features faster by replacing
scattered Jira boards with a single source of truth that actually stays
up to date."

---

**Weak:** "AI writing assistant."
**Strong:** "Wordsmith helps solo founders write launch emails that convert
by generating copy in their own voice — not ChatGPT's."

---

**Weak:** "Analytics for your website."
**Strong:** "Fathom gives privacy-first companies a complete picture of what's
working on their site — without GDPR headaches or cookie banners."

---

### Positioning Test

Read your value prop aloud. Then ask:

1. Could a competitor say this exact sentence? (If yes, it is not differentiated.)
2. Does it name a specific person? (If not, it speaks to no one.)
3. Does it describe an outcome, or just a feature? (Features do not convert.)

---

## 5. CTA Patterns

### The Verb + Outcome Formula

A CTA is not a label. It is a tiny promise.

Format: `[Verb] + [what they get or where they go]`

| Weak CTA | Strong CTA |
|---|---|
| Submit | Get my free report |
| Sign up | Start building for free |
| Learn more | See how it works |
| Download | Download the 2025 benchmark |
| Buy now | Claim your spot |
| Get started | Ship your first integration |

---

### What to Avoid

**Friction words:** Submit, Register, Purchase, Subscribe — these make the
user think about the cost, not the benefit.

**Vague words:** Click here, Learn more, Continue — these tell the user
nothing about what happens next.

**First-person mismatch:** If your body copy says "you", your CTA should
say "my". Example: "Start my free trial" outperforms "Start your free trial"
in most A/B tests because it completes the user's thought.

---

### CTA Examples by Context

**SaaS — free trial:**
- "Start my 14-day free trial — no credit card"
- "Try [Product] free for two weeks"
- "Get instant access"

**SaaS — demo:**
- "Book a 20-minute demo"
- "See [Product] in action"
- "Watch a 3-minute walkthrough"

**Ecommerce:**
- "Add to bag"
- "Reserve yours — ships in 3 days"
- "Get it by Friday — order in the next 4 hours"

**Content / lead magnet:**
- "Send me the guide"
- "Get the free checklist"
- "Join 6,800 developers — it's free"

**Pricing page:**
- "Start free, upgrade when ready"
- "Pick this plan"
- "Talk to sales — get a custom quote"

---

## 6. Microcopy

Microcopy is the small text that makes or breaks trust: button labels,
placeholder text, helper text, empty states, loading messages. It is the
most underrated lever in product design.

---

### Button Labels

Every button label should complete the sentence: "I want to ___."

| Context | Weak | Strong |
|---|---|---|
| Save draft | Save | Save draft |
| Delete account | Delete | Delete my account permanently |
| Send message | Send | Send message |
| Upload file | Upload | Upload your resume (PDF, max 5MB) |
| Confirm purchase | Confirm | Pay $49 and start building |

---

### Placeholder Text

Placeholders disappear when the user starts typing. Do not put instructions
there — put them in helper text below the field.

**Wrong:** `Enter your email address here`
**Right:** `you@company.com`

**Wrong:** `Write a short description of your project`
**Right:** `e.g. A mobile app that helps runners track their training`

---

### Helper Text

Helper text lives below the field and stays visible. Use it to:
- Set format expectations: "Use the format MM/DD/YYYY"
- Reduce anxiety: "We'll never share your email"
- Explain why you're asking: "Your phone number is only used for 2FA"

---

### Empty States

Empty states are a missed onboarding opportunity. They appear at the most
critical moment — when the user has committed to signing up but hasn't
gotten value yet.

**Wrong:** "No data yet."

**Right:** Three parts — what's missing, why it matters, what to do next.

Examples:
- "You haven't connected any accounts yet. Connect your first account and
  we'll start pulling in your data automatically. [Connect an account]"

- "Your inbox is empty. When teammates mention you or assign you a task,
  it'll show up here. [Browse open tasks]"

- "No reports yet. Create your first report and share it with your team
  in one click. [Create a report]"

---

### Loading Messages

Loading messages reduce perceived wait time. Instead of a spinner and silence:

- "Analyzing your repository... this takes about 20 seconds"
- "Generating your report — crunching 3 months of data"
- "Almost there — connecting to your database"
- "Setting up your workspace... we're installing your integrations"

For long waits (>10 seconds), add reassurance:
- "Still working — large repositories take a little longer"
- "Hang tight — we're pulling data from 6 sources"

---

## 7. Error Messages

Error messages are the most-hated and least-designed copy in software.
Two rules govern all of them:

**Rule 1: Never blame the user.**
**Rule 2: Always tell them what to do next.**

### The Error Message Formula

```
[What happened] + [Why it happened, if it helps] + [What to do next]
```

---

### 10 Error Message Examples

**1. Invalid email format**
- Wrong: "Invalid email."
- Right: "That email address doesn't look right. Double-check for typos and try again."

**2. Password too short**
- Wrong: "Password must be at least 8 characters."
- Right: "Your password needs to be at least 8 characters. Add a few more and you're good."

**3. Credit card declined**
- Wrong: "Card declined."
- Right: "Your card was declined. Check that the number, expiry, and CVC are correct — or try a different card."

**4. File too large**
- Wrong: "File size exceeded."
- Right: "That file is too large (max 10MB). Compress it or upload a smaller version."

**5. Session expired**
- Wrong: "Session expired. Please login."
- Right: "You've been logged out for security. Sign back in — your work has been saved."

**6. Network error**
- Wrong: "Network error."
- Right: "We couldn't connect to our servers. Check your internet connection and try again. If the problem persists, [check our status page]."

**7. Username already taken**
- Wrong: "Username unavailable."
- Right: "That username is taken. Try adding your last initial or a number — like jsmith42."

**8. Required field missing**
- Wrong: "Please fill in all required fields."
- Right: Highlight the specific field. "Your company name is missing — we need it to set up your account."

**9. Permissions error**
- Wrong: "Access denied."
- Right: "You don't have permission to view this. Ask your workspace admin to grant you access, or [contact support]."

**10. 404 page**
- Wrong: "Page not found."
- Right: "We can't find that page — it may have moved or been deleted. [Go to the homepage] or [search for what you need]."

---

## 8. Onboarding Copy

Onboarding copy has one job: get the user to their first "aha moment"
as fast as possible.

---

### Welcome Emails

Structure: Human greeting → specific value reminder → one clear action.

**Subject line options:**
- "You're in — here's where to start"
- "Welcome to [Product]. Your first step:"
- "[First name], you're set up. One thing to do right now:"

**Body:**
```
Hey [First name],

Welcome to [Product]. You signed up to [solve specific problem they
checked the box for]. Here's the fastest way to get there:

[Single CTA: Complete your first setup step]

This takes about 4 minutes. After that, you'll have [specific outcome].

— [Founder name or team]

P.S. If you run into anything, just reply to this email. I read every one.
```

---

### Empty State Onboarding

See Section 6 for empty state copy. For onboarding specifically, add
a time estimate and a social proof element when possible:

"Most teams complete setup in under 10 minutes. 4,200 teams did it last week."

---

### Tooltips

Tooltips explain things at the moment of confusion, not before.

Rules:
- One sentence max
- Describe what it does, not what it is
- If you need two sentences, the feature needs redesigning

Examples:
- "Saves a snapshot of your current pipeline — you can restore it anytime."
- "Members with Viewer access can see reports but can't edit or share them."
- "Setting this to 'Auto' lets us pick the best time based on your audience's activity."

---

### Progress Indicators

Progress copy reduces abandonment during multi-step flows.

**Step labels:**
- Weak: "Step 1 of 4"
- Strong: "Step 1 of 4 — Connect your account"

**Completion messages:**
- Weak: "Setup complete."
- Strong: "You're all set. Your first report is generating now — it'll be
  ready in about 60 seconds."

**Encouragement at friction points (step 3 of 4):**
- "Almost done — this is the last step."
- "One more thing and you're live."

---

## 9. Pricing Page Copy

Pricing pages fail because they lead with numbers. Numbers without context
are meaningless. Context converts.

---

### Anchoring

Show the most expensive plan first (left to right on desktop).
The first number seen becomes the reference point. Everything else
feels like a deal by comparison.

Label the recommended plan visibly:
- "Most popular"
- "Best for growing teams"
- "What most customers choose"

---

### Loss Aversion

People are more motivated to avoid losing $50 than to gain $50.
Frame pricing around what they lose by not acting.

Examples:
- "Teams on our free plan leave an average of $14,000 in unbilled work
  per year. The Pro plan pays for itself in the first recovered invoice."

- "Every week without [feature] costs the average team 6 hours of manual
  reconciliation. The Pro plan costs less than one hour of that time."

---

### Guarantee Language

Guarantees reverse risk and accelerate decisions.

Strong guarantee copy:
- "30-day money-back guarantee. No questions, no forms, no friction. Just
  email us and we'll refund you the same day."

- "Try it free for 14 days. We won't ask for a card until you're ready.
  And if you're not happy after your first paid month, we'll give it back."

- "Cancel anytime. Seriously — it's one click in your settings. No
  cancellation calls, no retention offers unless you want them."

---

### Plan Name Conventions

Avoid: Starter / Growth / Enterprise (everyone uses these; they mean nothing)

Better: Name plans after the customer's job, not their size.

Examples:
- Solo / Studio / Scale
- Personal / Team / Company
- Builder / Launcher / Leader

---

## 10. Anti-Patterns

### Jargon and Buzzwords

These words have been used so often they carry no signal:

- "World-class", "best-in-class", "industry-leading"
- "Robust", "scalable", "seamless", "intuitive"
- "Next-generation", "cutting-edge", "revolutionary"
- "Leverage", "synergy", "holistic"
- "AI-powered" (without explaining what the AI actually does)

When you catch yourself writing these, ask: what does this actually mean?
Then write that instead.

**Before:** "Our robust, AI-powered platform seamlessly integrates with
your existing workflow."
**After:** "Connect to Slack, Jira, and GitHub in one click. Your team
sees updates in the tools they already use."

---

### Vague Claims

Vague claims are invisible. Specificity is credibility.

| Vague | Specific |
|---|---|
| "Save time" | "Save 3 hours a week" |
| "Trusted by thousands" | "Trusted by 11,400 developers" |
| "Fast setup" | "Live in under 8 minutes" |
| "Improve your workflow" | "Cut your deploy time from 45 minutes to 6" |
| "Affordable pricing" | "Starts at $19/month — less than a team lunch" |

---

### Feature-Led vs. Benefit-Led

Developers write features. Users buy benefits.

| Feature-led (weak) | Benefit-led (strong) |
|---|---|
| "Real-time collaboration" | "Your whole team edits at once — no version conflicts" |
| "Automated backups" | "Your work is always safe, even if something goes wrong" |
| "Role-based permissions" | "Control exactly who can see and change what" |
| "API access" | "Build your own integrations — connect anything" |
| "Audit log" | "See exactly who changed what and when" |

Rule: Features tell you what it is. Benefits tell you what it does for you.
Every feature should map to a benefit. Lead with the benefit.

---

### We-Focused vs. You-Focused

A common pattern in bad SaaS copy: the company talks about itself instead
of the customer.

**We-focused:**
"We built [Product] to help teams collaborate better. We believe in
transparency, and we've spent years building features our customers love."

**You-focused:**
"[Product] gives your team one place to work — no more lost messages,
duplicate files, or missed deadlines. You see what's happening. Your
team stays aligned."

Count the "we" vs. "you" ratio on your landing page. If "we" wins, rewrite.

---

### The Buried Lede

Do not warm up to your point. State it first.

**Buried:**
"In today's fast-moving business environment, teams need tools that can
keep up with the demands of modern work. That's why we built [Product] —
a new kind of project management software for the way teams actually work."

**Direct:**
"[Product] is project management that doesn't require a training session.
Set it up in 8 minutes. Your team will use it without being asked."

---

## 11. The 5-Second Test Checklist

Show your page to someone unfamiliar with your product for 5 seconds.
Then hide it and ask these questions. If they cannot answer them, your
copy is failing.

```
[ ] What does this product do?
[ ] Who is it for?
[ ] What happens when I click the main button?
[ ] Why is this different from what I'm already using?
[ ] Is there any reason NOT to try it?
```

### Supporting Copy Checks

Run these before shipping any page:

```
[ ] Headline states a specific outcome or promise — not a tagline
[ ] Subheadline answers "who is this for?" or "how does it work?"
[ ] At least one piece of social proof is visible above the fold
[ ] The CTA uses a verb and describes what the user gets
[ ] No sentence starts with "We are" or "We believe"
[ ] No placeholder copy remains (no "Lorem ipsum", no "Coming soon")
[ ] Every claim is specific (numbers, names, timeframes)
[ ] Error messages exist and follow the formula in Section 7
[ ] Empty states have copy that explains what to do next
[ ] Mobile viewport: headline still readable, CTA still tappable
```

### The Squint Test

Squint at your page until the text blurs. You should still be able to see:

1. One dominant headline
2. One primary CTA button
3. A clear visual hierarchy from top to bottom

If everything looks equally important, nothing is important.

---

## Quick Reference

| Element | Formula |
|---|---|
| Headline | Specific promise + audience signal |
| Subheadline | How it works or who it's for |
| Value prop | [Product] helps [who] [outcome] by [differentiator] |
| CTA | Verb + what they get |
| Error message | What happened + what to do next |
| Empty state | What's missing + why it matters + one action |
| Tooltip | One sentence: what it does, not what it is |
| Guarantee | Specific terms + zero friction to claim |
