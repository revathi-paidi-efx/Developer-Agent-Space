---
name: story-naming-conventions
description: 'Compute and validate the feature branch name and the build/Docker image tag for a Jira story. Use whenever creating a feature branch, tagging a build image, or checking that an image tag stays within the 64-character limit. Enforces the feature/<TICKET>-<featureName> branch pattern and the <version>-feature-<TICKET>-<featureName>-SNAPSHOT-<build> image tag pattern.'
argument-hint: '<TICKET> <featureName> [version] [buildNumber]'
---

# Story Naming Conventions

Use this skill to derive **exact, valid** names for a feature branch and its build image tag,
and to guarantee the image tag never exceeds **64 characters**.

## When to use
- Before creating a feature branch for a Jira story.
- Before configuring the build/Docker image tag.
- Whenever you need to confirm an image tag fits inside the 64-char limit.

## Inputs
- `TICKET` — the Jira key, e.g. `I9P-48239`.
- `featureName` — a short **lowerCamelCase** verb+noun summarizing the work, e.g. `subscribeMsgs`,
  `addPubsubLogic`. No spaces, no underscores, ASCII letters/digits only.
- `version` — the Maven/app version, e.g. `1.0.0` (defaults to the module's current version).
- `buildNumber` — the CI build counter, e.g. `168`.

## Branch name

```
feature/<TICKET>-<featureName>
```

Examples:
- `feature/I9P-45567-addPubsubLogic`
- `feature/I9P-48239-subscribeMsgs`

Rules:
- Always under the `feature/` prefix (the "feature folder").
- Exactly one hyphen between `<TICKET>` and `<featureName>`.

## Image tag

```
<version>-feature-<TICKET>-<featureName>-SNAPSHOT-<buildNumber>
```

Example:
- `1.0.0-feature-I9P-48239-subscribeMsgs-SNAPSHOT-168`

### The 64-character rule

The full tag **must be ≤ 64 characters**. The fixed overhead around `featureName` is:

```
overhead = len(version) + len("-feature-")  # 9
         + len(TICKET)
         + len("-")                          # 1  (separator before featureName)
         + len("-SNAPSHOT-")                 # 10
         + len(buildNumber)

featureNameMax = 64 - overhead
```

Worked example (`version=1.0.0`, `TICKET=I9P-48239`, `buildNumber=168`):

```
overhead = 5 + 9 + 9 + 1 + 10 + 3 = 37
featureNameMax = 64 - 37 = 27  -> "subscribeMsgs" (13) fits easily
```

### Procedure
1. Choose a concise `featureName` (aim for ≤ 20 chars to stay safe as build numbers grow).
2. Compute `featureNameMax` with the formula above using the **largest** build number you expect.
3. If `len(featureName) > featureNameMax`, shorten it (abbreviate the noun, drop filler words)
   and re-check. Never truncate the `TICKET` or `version`.
4. Assemble both the branch name and the image tag; print them and the final tag length.
5. Report: branch name, image tag, tag length, and remaining headroom to 64.

### Quick check helper (optional)

Run this to validate a candidate tag length:

```powershell
$tag = "1.0.0-feature-I9P-48239-subscribeMsgs-SNAPSHOT-168"
"$tag  ->  $($tag.Length) chars  ->  $(if ($tag.Length -le 64) {'OK'} else {'TOO LONG'})"
```
