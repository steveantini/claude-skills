# UX Writing

| Field | Value |
|---|---|
| **Version** | 1.1.0 |
| **Last Updated** | 2026-09-22 |
| **Applicability** | All user-facing interface text in web and mobile applications |
| **Dependencies** | None (language-agnostic patterns) |

---

## Error Messages

### Structure: What, Why, What Next

Every error message should answer three questions:

1. **What happened** — Describe the problem clearly.
2. **Why it happened** — Give context if helpful (omit if obvious).
3. **What to do** — Provide a concrete next step.

### Examples

| Bad | Good |
|---|---|
| "Error" | "Could not save your changes. Check your connection and try again." |
| "Invalid input" | "Email address must include an @ symbol." |
| "403 Forbidden" | "You don't have permission to view this page. Contact your admin for access." |
| "Something went wrong" | "We couldn't load your projects. Try refreshing the page." |
| "Request failed" | "Payment could not be processed. Please check your card details." |
| "Null reference error" | "Something unexpected happened. Our team has been notified." |

### Error Message Rules

- **Never show technical details** to users (stack traces, error codes, SQL errors).
- **Never blame the user**: Say "That password is too short" not "You entered a bad password."
- **Be specific**: "File must be under 10 MB" not "File too large."
- **Use plain language**: "Could not connect" not "Network request timed out (ETIMEDOUT)."
- **Provide an action**: Every error should suggest what to do next — retry, change input, contact support.
- **Log the technical details** server-side for debugging; show the human message client-side.

### Error Severity Tiers

| Severity | Presentation | Duration | Example |
|---|---|---|---|
| **Field validation** | Inline text below field | Persistent until fixed | "Password must be at least 8 characters." |
| **Form-level** | Alert banner above or below form | Persistent until fixed | "Please fix the 2 errors below." |
| **Action failure** | Toast notification | 8 seconds or persistent | "Could not delete project. Try again." |
| **Page-level** | Full error state replacing content | Until refresh/retry | "This page could not be loaded." |
| **System-wide** | Top banner across entire app | Until resolved | "We're experiencing issues. Some features may be unavailable." |

### Retry Pattern

```
[Error icon] Could not save changes.
[Try again button]  [Dismiss link]
```

If auto-retrying, tell the user: "Having trouble connecting. Retrying..."

---

## Loading States

### Copy by Duration

| Duration | Message | Example |
|---|---|---|
| < 1 second | No message | Skeleton or spinner only |
| 1-5 seconds | Brief label | "Loading..." |
| 5-15 seconds | Contextual label | "Loading your projects..." |
| 15-60 seconds | Progress + explanation | "Setting up your workspace. This may take a moment." |
| > 60 seconds | Progress + time estimate | "Importing 2,400 records. About 2 minutes remaining." |

### Loading Message Rules

- **Be specific about what is loading**: "Loading dashboard..." not just "Loading..."
- **For long operations, explain why**: "Generating your report from 30 days of data."
- **Show progress when possible**: "Uploading... 3 of 12 files" or a progress bar.
- **Use progressive disclosure**: Show partial content as it loads, not a blank screen.
- **Never say "Please wait"** — it adds nothing. Say what is happening instead.

### Post-Action Feedback

| Action | Feedback |
|---|---|
| Save | "Changes saved" (toast, auto-dismiss) |
| Delete | "Project deleted" (toast with Undo action) |
| Send | "Invitation sent to alex@example.com" |
| Create | "New project created" (redirect to it) |
| Copy | "Copied to clipboard" (tooltip near button, 2s) |

---

## Empty States

### Structure

```
[Illustration or icon]
[Headline — what this section is for]
[Description — why it is empty and what they can do]
[Primary action button]
```

### Examples

**No projects yet**:
```
[Folder icon]
No projects yet
Projects help you organize your work. Create your first one to get started.
[+ Create project]
```

**No search results**:
```
[Search icon]
No results for "quarterly report"
Try adjusting your search or filters.
[Clear filters]
```

**No notifications**:
```
[Bell icon]
You're all caught up
When something needs your attention, it will appear here.
```

**No permission**:
```
[Lock icon]
You don't have access to this workspace
Ask your team admin to invite you.
[Request access]
```

### Empty State Rules

- **Always include an action** when the user can create the thing that is missing.
- **Differentiate first-time empty from filtered empty**: "No projects yet" vs. "No projects match your filters."
- **Keep it brief**: 1 headline + 1 sentence max.
- **Use illustration sparingly**: A simple icon is often enough. Decorative illustrations are optional.
- **Never say "No data found"**: That is a developer message, not a user message.

---

## Confirmation Dialogs

### When to Confirm

- **Destructive actions**: Delete, remove, revoke access.
- **Irreversible actions**: Send email, publish, finalize.
- **High-impact actions**: Bulk operations, account changes.

### When NOT to Confirm

- **Easily undone actions**: Archive (can unarchive), soft delete.
- **Routine actions**: Save, create, update — use feedback (toast) instead.
- **Navigation**: Do not confirm leaving a page unless there are unsaved changes.

### Confirmation Dialog Structure

```
[Title — action being confirmed]
[Description — consequence of the action]
[Cancel button]  [Confirm button with specific verb]
```

### Examples

**Delete project**:
```
Title:    Delete "Marketing Site"?
Body:     This will permanently delete the project and all its data.
          This action cannot be undone.
Buttons:  [Cancel]  [Delete project] (destructive style)
```

**Remove team member**:
```
Title:    Remove Alex from this workspace?
Body:     Alex will lose access to all projects in this workspace immediately.
Buttons:  [Cancel]  [Remove]
```

### Confirmation Dialog Rules

- **Title = verb + object**: "Delete project?" not "Are you sure?"
- **Never say "Are you sure?"**: It is vague and adds friction without clarity.
- **Confirm button = specific verb**: "Delete project" not "OK" or "Yes."
- **Destructive confirm buttons are red/destructive variant**.
- **Cancel is always safe and default-focused** (accessible via Escape).
- **Include the name** of the thing being affected: "Delete 'Marketing Site'?" not "Delete this project?"
- **State the consequence**: "This will permanently delete all data" or "Alex will lose access immediately."

---

## Tooltips

### When to Use

- **Icon-only buttons**: Explain what the button does on hover/focus.
- **Truncated text**: Show full text on hover.
- **Supplementary info**: Brief additional context that does not fit in the UI.

### When NOT to Use

- **Essential information**: If users need it, show it in the interface directly.
- **Mobile primary actions**: Tooltips require hover, which does not exist on touch.
- **Long content**: More than 1-2 sentences belongs in a popover or help page.

### Tooltip Content Rules

- **Maximum 1-2 short sentences** (under 150 characters).
- **No titles or headings** inside tooltips.
- **Action tooltips**: Describe the action. "Copy to clipboard" not "Copy button."
- **Info tooltips**: Provide context. "Last updated 3 hours ago" not "Timestamp."
- **Include keyboard shortcut** if available: "Bold (Cmd+B)."

### Examples

| Trigger | Tooltip Text |
|---|---|
| Copy icon button | "Copy to clipboard" |
| Help icon next to field | "Your display name is visible to other members." |
| Truncated project name | Full project name |
| Keyboard shortcut button | "Undo (Ctrl+Z)" |

---

## Onboarding

### Principles

- **Delay until relevant**: Do not explain features before the user needs them.
- **Show, do not tell**: Interactive steps beat text explanations.
- **Skippable always**: Every onboarding flow must have a "Skip" option.
- **Progressive**: Introduce one concept at a time, not all at once.

### Onboarding Patterns

| Pattern | When to Use | Implementation |
|---|---|---|
| **Welcome screen** | First login | Single modal with 1-3 setup steps |
| **Empty state guidance** | First time in a section | Inline within the empty state |
| **Contextual hints** | First use of a feature | Popover near the UI element |
| **Checklist** | Multi-step setup | Persistent sidebar or card |
| **Interactive tour** | Complex interface | Step-by-step with highlights |

### Welcome Flow Example

```
Step 1: "Welcome to [App]. Let's get you set up."
        [Set up profile picture — optional]
        [Continue]  [Skip for now]

Step 2: "Create your first project"
        [Project name input]
        [Create project]

Step 3: "Invite your team"
        [Email input]
        [Send invites]  [I'll do this later]
```

### Onboarding Copy Rules

- **Headline = benefit or outcome**: "Stay on top of your tasks" not "Task management feature."
- **Keep steps short**: 1 sentence of explanation per step max.
- **Use verbs**: "Create a project" not "Project creation."
- **Celebrate completion**: "You're all set!" not "Setup complete."
- **Make skip guilt-free**: "I'll do this later" not just "Skip."

### Returning Users

- Do not repeat onboarding after completion.
- Store completion state per feature, not globally.
- For new features post-onboarding, use contextual hints (one-time popovers).

---

## Tone and Voice Guidelines

### Voice Attributes

| Attribute | Meaning | Example |
|---|---|---|
| **Clear** | No jargon, no ambiguity | "3 projects" not "multiple projects" |
| **Concise** | Fewest words needed | "Saved" not "Your changes have been saved successfully" |
| **Helpful** | Guides toward resolution | "Try a shorter name" not "Invalid input" |
| **Confident** | Definitive, not uncertain | "This will delete the project" not "This might delete the project" |
| **Respectful** | Never condescending | "That email is already registered" not "You already signed up" |

### Tone by Context

| Context | Tone | Example |
|---|---|---|
| Success | Warm, brief | "Project created." |
| Error | Calm, actionable | "Could not connect. Check your internet and try again." |
| Warning | Direct, informative | "You have unsaved changes." |
| Destructive action | Serious, clear | "This will permanently delete all data. This cannot be undone." |
| Onboarding | Friendly, encouraging | "Welcome! Let's set up your workspace." |
| Empty state | Inviting, helpful | "No tasks yet. Create one to get started." |

### Writing Rules

| Rule | Bad | Good |
|---|---|---|
| Use sentence case for UI | "Create New Project" | "Create new project" |
| No periods on short labels | "Save." | "Save" |
| Periods on full sentences | "Changes saved" | "Your changes have been saved." (if full sentence) |
| Use contractions naturally | "You cannot access" | "You can't access" |
| Avoid "please" in buttons | "Please confirm" | "Confirm" |
| Use "we" sparingly | "We couldn't find..." | "No results found." (or "We couldn't load..." for errors) |
| Numbers: use digits | "three items" | "3 items" |
| Times: be specific | "recently" | "3 minutes ago" |
| Avoid "successfully" | "Successfully created" | "Created" or "Project created" |
| Avoid exclamation marks | "Welcome!" | "Welcome." (or no punctuation) |

### Button Labels

| Action Type | Pattern | Examples |
|---|---|---|
| Create | Verb + noun | "Create project", "Add member" |
| Confirm | Specific verb | "Delete project", "Send invite" |
| Navigate | Destination | "Go to settings", "View details" |
| Toggle | Current state | "Mark as complete", "Hide sidebar" |
| Cancel | "Cancel" | Never "No", "Back", or "Nevermind" |

### Placeholder Text

| Field | Placeholder | Notes |
|---|---|---|
| Search | "Search projects..." | Include what is being searched |
| Email | "you@example.com" | Show format |
| Name | "Jane Smith" | Show expected format |
| URL | "https://example.com" | Show full format |
| Description | "Describe your project..." | Action-oriented |
| Empty | (leave empty) | If format is obvious from label |

---

## Content Patterns Quick Reference

| Situation | Pattern |
|---|---|
| Action succeeded | "[Thing] [past-tense verb]." — "Project created." |
| Action failed | "Could not [verb]. [Reason or next step]." |
| Field validation | "[Field] must [requirement]." — "Name must be at least 2 characters." |
| Permission denied | "You don't have permission to [action]. [Who to contact]." |
| Loading | "[Verbing] your [things]..." — "Loading your projects..." |
| Empty | "No [things] yet. [Action to create one]." |
| Confirmation | "[Verb] [specific thing]? [Consequence]. [Confirm verb] / Cancel" |
| Tooltip | "[What it does]" or "[What it means]" (1 sentence max) |
| Onboarding | "[Benefit]. [What to do]." |

## Vendor-Neutral Help Text

**The rule.** When a product's connectors are stand-ins for a future provider
(a drive, a calendar, a mail system that will be swapped), write the guides,
the help drawer, and the model-facing prompts without any vendor's name:
"your drive", "your calendar", "a connected system", "Select Connect". The
switch then changes buttons and code, not words.

**Why.** Help text names the vendor in dozens of sentences, and every one is
a rewrite the day the connector changes. The product's own buttons are
generated from the connector and follow the switch on their own; prose does
not. A guide that says "your drive" is true before and after.

**Example.**

```
Before: Choose Upload from computer, or Google Drive when your organization has it connected.
After:  Choose Upload from computer, or your connected drive when your organization has one connected.

Before: a label like Google Meet when the event has a video link
After:  a video-call label when the event has a video link
```

When a step must quote a rendered label that names a vendor, either describe
the control generically ("Select Connect") or, if the exact label is needed,
fence it: a test lists each such phrase and fails the moment the phrase is
gone from the source, so the connector switch forces the guide update rather
than leaving a stale label behind.

**How to test for it.**

```ts
const vendors = /VendorA|VendorB|VendorC/;
expect(guidesSource).not.toMatch(vendors);
expect(renderedCorpus).not.toMatch(vendors);
// Or, while a fenced label remains:
for (const phrase of FENCED_LABELS) { expect(source).toContain(phrase); source = source.split(phrase).join(""); }
expect(source).not.toMatch(vendors);
```

## Recon Before a Copy Rewrite

**The rule.** Before proposing new wording for a page, a set of guides, or a
document set, run a read-only pass that prints every rendered string verbatim
with its file and line, in render order, and lists the tests that pin any of
them. Propose wording only from that output.

**Why.** Wording proposed from fragments (a grep hit, a memory of the page, a
summary) is wrong in ways that cost a commit each: a sentence that continues
across a JSX line break, a lead-in that is a separate bold element, a string
that is shared with another page, a test that pins the old text. The recon
also surfaces what a rewrite must keep current: a claims map, a seed, a
mock-up that shows the same words.

**How to run it.** One numbered read-only task per surface: the route file
and every component it renders; each section's strings in order; which
strings are shared with other pages; which mock-ups carry hard-coded labels
and which come from a shared source; which tests pin which substrings; which
claims the product no longer supports. Print prose in code fences so nothing
is reflowed, and never elide with "...". The rewrite that follows is then a
list of exact old-to-new edits, each with a line number, and every edit lands
with its test in one commit.

**How to test for it.** The rewrite commit adds pins for the new sentences
against the rendered markup (not the source), so the next rewrite starts from
a recon that lists them.
