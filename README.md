# Fruit Into Baskets: A Sliding Window Walkthrough of LeetCode 904, With Working Code and the Bugs That Break It

The first time you read LeetCode 904, it sounds like a story about a farmer. Two baskets, one fruit type per basket, walk right and stop when you hit a third kind of fruit. Twenty seconds later you realize the story is decoration, and the actual question is much shorter:

**Find the longest contiguous subarray that contains at most two distinct values.**

That's it. Everything else is flavor text. Once you see it that way, the problem stops being a reading comprehension test and becomes a standard sliding window, which is exactly why interviewers like it — the gap between candidates who can restate the problem and candidates who can code it is enormous.

## Why brute force dies, and why the window survives

The naive approach is to lock a starting index and walk right until you've seen three fruit types. That's correct, and it's O(n²). The constraints say `fruits.length` can reach 10⁵. At n = 100,000, an O(n²) scan is on the order of 10¹⁰ operations, which is not a solution, it's a timeout.

The sliding window fixes this because the two pointers never move backward. `right` scans the array once. `left` only ever advances. Each element enters the window once and leaves once, so the total work is linear even though the code has a nested loop in it.

python
from collections import defaultdict

def totalFruit(fruits):
    counts = defaultdict(int)   # fruit type -> how many are in the window
    left = 0
    best = 0

    for right, f in enumerate(fruits):
        counts[f] += 1

        # too many types in the baskets: shrink from the left
        while len(counts) > 2:
            counts[fruits[left]] -= 1
            if counts[fruits[left]] == 0:
                del counts[fruits[left]]
            left += 1

        best = max(best, right - left + 1)

    return best


Two things carry all the weight here. The dictionary keyed by fruit type tells you how many *distinct* types are in the window, and the `while` loop restores the "at most two types" invariant before you measure the window.

## Tracing `[1, 2, 3, 2, 2]` by hand

The expected output for this input is 4, from the subarray `[2, 3, 2, 2]`. Here's the state after each step:

| right | fruit | counts after adding | window after shrinking | best |
| --- | --- | --- | --- | --- |
| 0 | 1 | `{1:1}` | `[1]` | 1 |
| 1 | 2 | `{1:1, 2:1}` | `[1, 2]` | 2 |
| 2 | 3 | `{1:1, 2:1, 3:1}` → shrink | `[2, 3]` | 2 |
| 3 | 2 | `{2:2, 3:1}` | `[2, 3, 2]` | 3 |
| 4 | 2 | `{2:3, 3:1}` | `[2, 3, 2, 2]` | 4 |

Notice what happened at `right = 2`. The window never became invalid for long — one shrink was enough, because dropping the single `1` left exactly two types. The window didn't restart at the `3`; it slid forward and kept the `2` that was already in the basket. That's the whole trick, and it's also where most wrong answers come from.

## The bugs that actually break this solution

### Using `if` instead of `while` to shrink

python
if len(counts) > 2:
    # remove one element


This removes one element per iteration of the outer loop. If the window ever holds three types after a single removal, you compute `right - left + 1` on a window that violates the constraint and return a number that's too large. Sometimes one removal is enough, which is what makes this bug so annoying — it passes the small examples and fails on inputs like `[1, 1, 2, 1, 3, 3, 3]`.

### Forgetting to delete keys that hit zero

If you decrement `counts[fruits[left]]` to 0 and leave the key in the dictionary, `len(counts)` never drops below 3. The `while` loop keeps shrinking past the point where the window was valid, `left` blows past `right`, and you get nonsense. The `del` line is not decoration.

### Resetting the window instead of shrinking it

A common instinct: when a third fruit type shows up, clear the map and restart from the current tree. This throws away legal overlap. On `[1, 2, 2, 1, 3]`, the correct answer is 4 (the subarray `[1, 2, 2, 1]`), and a reset-based approach reports something smaller because it discards the run that didn't need discarding.

### Shrinking to the wrong boundary, or by a fixed amount

`left` must advance one step at a time until the window is legal again. Jumping `left` to a remembered index, or unconditionally advancing it, produces windows that are either too small (missing the maximum) or too large (violating the constraint).

If you want a check that catches all four, run `[1, 1, 2, 1, 3, 3, 3]` and confirm you get 4, then run `[1, 2, 1]` and confirm you get 3, and then run an array where every value is identical — you should get the full length back.

## The complexity answer interviewers actually want

For a 10⁵ array:

- **Time:** O(n). `right` moves n times; `left` moves at most n times total across the entire run, so the inner loop adds a constant amount of work amortized.
- **Space:** O(1). This one trips people up. You're using a hash map, so it feels like extra memory, but the map can hold at most 3 keys — the two valid fruit types plus the third one that triggered the shrink, before the `del` fires. A constant-bounded map is constant space, not O(n).

You can also solve 904 with no map at all: track the two basket types plus the start index of the trailing contiguous block of identical fruit. When a third type appears, you know the previous fruit's block is worth keeping, so you jump `left` back to the start of that block, drop the older basket, and continue. Same O(n) time, same O(1) space, fewer allocations. It's a nice thing to mention if the interviewer asks whether you can avoid the hash map, but the map version is the one you should write first — it's harder to get wrong under pressure.

> The pattern generalization is the real prize here. Replace "2 baskets" with "k baskets" and you're solving LeetCode 340. The window mechanics don't change one line.

## Where the same pattern shows up next

Once you've internalized 904, these are variations on the same skeleton:

- **340. Longest Substring with At Most K Distinct Characters** — same code with `2` replaced by a variable. If you can't solve this after 904, you memorized the problem instead of the pattern.
- **159. Longest Substring with At Most Two Distinct Characters** — the string version of 904, locked behind LeetCode Premium.
- **3. Longest Substring Without Repeating Characters** — same shrink-until-valid structure, different validity condition.
- **1004. Max Consecutive Ones III** — the window tracks a budget (number of flips) instead of a type count.
- **992. Subarrays with K Different Integers** — "exactly K" is computed as `atMost(K) - atMost(K-1)`, which is worth knowing as a technique on its own.

The reason pattern-based prep beats grinding is visible right here. Five problems, one code skeleton, and the only thing that changes is what "valid" means.

## How AlgoMonster handles this problem specifically

AlgoMonster is a pattern-first interview prep platform built by engineers who came out of Google, and sliding window is one of the modules its curriculum is organized around. There's a dedicated walkthrough for 904 that shows the window state after every index rather than just dropping a finished solution on you — counters, shrink events, the whole trace — plus Python and Java implementations. You can see it here: 👉 [see AlgoMonster's Fruit Into Baskets walkthrough](https://algo.monster?fpr=li69).

That step-by-step visualization matters more than it sounds. The bugs listed above are all *state* bugs, not syntax bugs. Watching `counts`, `left`, and `right` move together across an example is what makes the `while` loop feel obvious instead of arbitrary.

The platform's structure is worth understanding before you pay anything. It doesn't sell a problem bank — it sells reusable patterns, each with a template, a set of decision flowcharts for figuring out which pattern a problem needs, and a handful of practice problems. Reviews consistently put the catalog at 48 patterns, and the product page advertises access to all patterns, lessons and problems, unlimited usage of its AI assistant for hints and explanations, unlimited Speedrun (timed practice runs), monthly-updated questions from large tech companies, and 3,000+ editorials. Subscribers also get bonus courses bundled in, including a fundamentals track for CS basics and an LLM-building course.

The tradeoffs are equally well documented across reviews, and they're real:

- **No video.** Every explanation is text plus interactive diagrams. If you learn by watching someone code, this format will frustrate you.
- **A smaller problem set** than LeetCode's thousands of questions. It's designed as a curriculum, not a warehouse.
- **Light system design coverage.** For senior-level loops, you'll need a second resource.

For this specific problem, none of those tradeoffs bite. Sliding window is a pattern, and 904 is a pattern problem. That's the case where this platform is at its strongest.

## AlgoMonster plans and pricing

There are three tiers, no monthly billing option, and prices swing with promotions, so confirm the number on the checkout page before you commit. The figures below are what the plans page currently shows.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a portion of the pattern library and lessons | $0 | Free | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; monthly-updated company questions; 3,000+ editorials; bonus courses; member discount on live bootcamps | $99/year ($8.25/mo billed annually) | Annual, 12 months of access | [Get Pro Yearly](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in Yearly, plus lifetime updates, all future content at no extra cost, and an internal referral to a top tech company for qualified candidates | $189 one-time | One payment | [Get Pro Lifetime](https://algo.monster?fpr=li69) |

A note on those numbers: the promotional page presents them as discounted, listing the yearly plan at $198 and lifetime at $378. Pricing has also moved around over time — current captures of the plans page show both a $99/year, $189 lifetime configuration and a cheaper promotional configuration — so treat the checkout page as the authoritative source rather than any blog post, including this one. The free tier is genuinely free and doesn't expire, which is enough to judge whether you can stand learning from text and diagrams.

## Which plan to pick if you're stuck on this one problem

If sliding window is your only weak spot and you have an interview in three weeks, neither plan is obviously right for you. Read the free tier, work the 904 walkthrough, solve 340 and 3 on your own, and see whether the pattern sticks. If it does, you don't need to buy anything for this topic.

Where the paid tiers start to make sense is when the problem isn't one pattern but the *selection* between them — you can write code fine, you just spend fifteen minutes of a forty-five-minute interview trying approaches that don't fit. That's the failure mode a pattern catalog is built to fix.

Choosing between the two: the yearly plan costs less than a single interview-prep bootcamp and covers a realistic 4-to-8 week prep window. Lifetime only pays off if you expect to come back — engineers who switch jobs every two or three years, or who want the material sitting there the next time they prep. If this is a one-time push before a single loop, the yearly plan is the cheaper honest answer.

## Quick answers

**Is 904 a sliding window problem?** Yes, and it's the canonical "at most K distinct" window where K = 2. Brute force is O(n²); the window is O(n) time and O(1) space.

**What's the one-line restatement?** Longest contiguous subarray with no more than two distinct values.

**What's the most common wrong answer?** Using `if` where the shrink needs `while`, or leaving zero-count keys in the map so the distinct-type count never drops.

**Do I need a hash map?** No. You can track the two basket types and the start of the trailing identical block instead, which uses no extra allocations. The map version is easier to write correctly under interview pressure.

**Where do I practice more of this?** 340 for the general K version, 3 for the repeating-character variant, 1004 for the budget variant, and 992 for "exactly K". 👉 [Work through the sliding window pattern end to end on AlgoMonster](https://algo.monster?fpr=li69) if you'd rather follow a structured sequence than pick problems yourself.

The problem is small. The pattern behind it isn't — and that's the part worth spending an afternoon on.
