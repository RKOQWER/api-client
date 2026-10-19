1. Implementation plan format — when I ask for a phase plan, use only these headers. Include nothing outside the phase scope. Do not list what we are not doing.

## Goal
<one sentence: what this phase achieves>

## Build
- <concrete item> — <why we build it>
- <concrete item> — <why we build it>

## Acceptance Criteria
- <test that proves it works>

## Gate
<single bar to clear before the next phase>

2. Coding implementation — minimal, accurate, and exact to the phase. Do not code before architecture review.

## Architecture before code
Present class/module details using lavish-axi HTML so class relationships and responsibilities are visually reviewable. Include:
- Class/module name and responsibility
- Key methods and what they own
- Dependencies between classes

I will share the architecture and comments first. You review and give go-ahead. Only then I write code.

3. Comments — explain the why, not the what. Keep short. Place above the block they explain. Do not state the obvious (code already says it).

Good:
// read secret from OS keychain, never from project files, to avoid leaking tokens in VCS
const token = await keychain.get('api-token')

Bad:
// increment i by 1
i = i + 1

4. Git commit message format — feat/bugfix/chore: <what changed from a functional point of view>. No implementation details. No file or function names. For bugs, state the root cause and the fix in functional terms only.

5. Always the answer that I ask you should be short and crisp and crystal clear without any abstraction