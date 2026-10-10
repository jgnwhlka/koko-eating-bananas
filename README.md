# Koko Eating Bananas: How Binary Search on the Answer Solves LeetCode 875 (and Every "Minimum Rate" Problem After It)

Most people land on this problem the same way. They read it, they write a loop that tries speed 1, 2, 3, 4…, it passes the sample tests, and then LeetCode hands them a pile array with values up to 10⁹ and the submission dies with a time limit exceeded. The fix isn't a clever trick. It's recognizing that you're not searching an array at all — you're searching the answer space, and that space is monotonic.

This is a walkthrough of LeetCode 875, the bugs that break it in interviews, and the handful of other problems that use the exact same shape. AlgoMonster's own free editorial for the problem lives in its per-problem library if you want a second pass at it after reading, and I'll get to when paying for a structured course actually saves you time versus grinding this out yourself.

## The problem, minus the monkey backstory

Koko has `n` piles of bananas. Pile `i` holds `piles[i]` bananas. The guards are gone for `h` hours. Koko picks a speed `k` bananas per hour and keeps it fixed. Each hour she attacks one pile: if it has at least `k` bananas left, she eats exactly `k`; if it has fewer, she eats the rest and then sits there for the remainder of the hour doing nothing. Return the smallest integer `k` that lets her clear everything within `h` hours.

| Example | piles | h | Answer |
| --- | --- | --- | --- |
| 1 | [3,6,7,11] | 8 | 4 |
| 2 | [30,11,23,4,20] | 5 | 30 |
| 3 | [30,11,23,4,20] | 6 | 23 |

Constraints, straight from the problem:

- `1 <= piles.length <= 10⁴`
- `piles.length <= h <= 10⁹`
- `1 <= piles[i] <= 10⁹`

That middle constraint is doing quiet work. `h` is never smaller than the number of piles, which guarantees an answer always exists: at worst Koko eats one pile per hour at a speed equal to the biggest pile.

The "wasted hour" rule is the part people trip over. It means piles are independent — whenever she finishes a pile early, that hour is gone and she can't pour the leftover time into the next pile. So the total time is just the sum of `ceil(pile / k)` across all piles. Order doesn't matter. You never need to simulate the eating.

## Why the obvious solution dies

Try every speed from 1 to `max(piles)` and return the first one that fits. That's `O(n × max(piles))`. With `n` at 10⁴ and pile values up to 10⁹, worst case is about 10¹³ operations. Dead.

The thing to notice is that the answer isn't hidden in an unsorted array. The candidate answers form a range, and the range has an order.

## The property that makes binary search legal

As eating speed goes up, hours needed goes down, or stays flat. Never up. That single observation gives you a clean dividing line:

- At speed `k`, if Koko finishes in time, she finishes at every speed above `k`.
- At speed `k`, if she doesn't finish in time, she fails at every speed below `k`.

One threshold, everything on one side fails, everything on the other side works, and you want the first speed on the working side. That's binary search. Not binary search over array indices — binary search over possible answers.

Search range: `[1, max(piles)]`. Why `max(piles)` as the ceiling? At that speed every single pile clears in one hour, so total time is exactly `n` hours, which the constraints guarantee is at most `h`. And going faster than the largest pile can't reduce the hour count, because each pile still eats up one hour minimum. It's a bound, not a guess.

## The feasibility check is 90% of the work

python
def feasible(piles, h, k):
    hours = 0
    for pile in piles:
        hours += (pile + k - 1) // k
    return hours <= h


`(pile + k - 1) // k` is integer ceiling division. If `pile = 7, k = 4`, you get `(7 + 3) // 4 = 10 // 4 = 2`. Correct: one hour eats 4, the next hour eats the remaining 3 and wastes the leftover capacity.

Write `7 // 4` instead and you get 1. That's the single most common failure on this problem, and it shows up as a wrong answer rather than a crash, so people stare at it for a while before spotting it.

After the feasibility function is right, the search is almost mechanical:

python
def minEatingSpeed(piles, h):
    left, right = 1, max(piles)
    while left < right:
        mid = (left + right) // 2
        if feasible(piles, h, mid):
            right = mid      # mid works, but something smaller might too
        else:
            left = mid + 1   # too slow, need faster
    return left


Time complexity is `O(n log m)` where `m = max(piles)`, and space is `O(1)`. The `log m` term is at most about 30 iterations, since pile values cap at 10⁹.

## Tracing [3,6,7,11] with h = 8

| k | Hours | ≤ 8? |
| --- | --- | --- |
| 3 | 1 + 2 + 3 + 4 = 10 | no |
| 4 | 1 + 2 + 2 + 3 = 8 | yes |
| 5 | 1 + 2 + 2 + 3 = 8 | yes |

Speed 3 fails, speed 4 is the first one that works, so the answer is 4. Speeds 5, 6, 7… all work too; you just don't want them.

Walking the actual binary search: start with `left = 1, right = 11`. `mid = 6` needs 6 hours — fits, so `right = 6`. `mid = 3` needs 10 — too slow, `left = 4`. `mid = 5` needs 8 — fits, `right = 5`. `mid = 4` needs 8 — fits, `right = 4`. Loop ends with `left = right = 4`.

Notice the loop shape: when `mid` works, you keep it as the right boundary rather than returning immediately, because you're hunting the *minimum* working speed, not any working speed.

## Five bugs that show up under interview pressure

1. **Floor division instead of ceiling.** `pile // k` ignores leftover bananas. Use `(pile + k - 1) // k`.
2. **Returning `mid` after the loop.** `mid` is whatever the last probe was, not necessarily the boundary. Return `left`.
3. **Setting `right = mid - 1` on success.** That throws away a valid answer. It's `right = mid`, plus `left = mid + 1` on failure.
4. **Integer overflow in Java or C++.** The hours accumulator is a sum of up to 10⁴ terms, each potentially around 10⁹ when the speed is 1. That overflows a 32-bit `int`. Use `long` for the accumulator.
5. **Sorting the piles first.** Feels productive, changes nothing. The feasibility check sums independent terms, so order is irrelevant. Skip the `O(n log n)`.

There's also a boundary question people ask: should the condition be `hours < h` or `hours <= h`? It's `<=`. The guards come back *at* hour `h`, so finishing exactly on time counts as finishing.

## Recognize the shape on a problem you've never seen

LeetCode 875 isn't one problem. It's a template, and these all share it:

- **1011. Capacity To Ship Packages Within D Days** — same structure, capacity instead of speed, and the "at least one pile per hour" rule becomes "packages must ship in order."
- **410. Split Array Largest Sum** — minimize the largest subarray sum. The predicate is "can I split into at most `m` chunks each under this cap?"
- **1482. Minimum Number of Days to Make m Bouquets** — the search is over days, the predicate checks whether enough bouquets form by then.
- **1283. Find the Smallest Divisor Given a Threshold** — nearly identical arithmetic, different wording.

The tell in an interview is a phrase like "minimum rate," "minimum capacity," "smallest value such that…" combined with a feasibility check that's expensive to brute force but cheap to verify. If verifying a candidate answer takes `O(n)` and the candidates form a monotone yes/no line, that's binary search on the answer.

Two sentences worth saying out loud before you touch the keyboard, because interviewers grade the reasoning more than the code:

> "All speeds below `left` are known invalid, and the answer is inside `[left, right]`."

Say the invariant, name the predicate, explain the monotonicity, then code. Candidates who skip straight to typing usually lose the thread in the boundary conditions and spend five minutes debugging off-by-ones.

## Where a structured course helps (and where it doesn't)

The reason this problem feels hard isn't the binary search. It's that nothing in the problem statement says "binary search." There's no sorted array in sight. The leap from "find the minimum" to "the candidates are monotone, so bisect them" is a pattern-recognition skill, and it transfers to every problem in the list above.

AlgoMonster is built around exactly that gap. It's a text-and-diagrams course by ex-Google engineers organized by pattern rather than by problem, so binary search shows up as a concept with a template you reuse, not as a one-off solution you memorize. There's an inline editor and an AI assist embedded in the lessons, and its per-problem editorial library covers 875 directly with the feasibility function spelled out — some of that editorial content is free to read before you pay anything.

What it isn't: there are no video walkthroughs anywhere on the platform, and the problem count sits in the hundreds rather than LeetCode's thousands. If you learn by watching someone's screen, this will feel dry. The curriculum is designed to be finished in roughly four to eight weeks at an hour or two a day, which is the realistic window for someone with an interview scheduled.

You can 👉 [browse AlgoMonster's plans and free content](https://algo.monster?fpr=li69) to see whether the format suits how you read, or just keep going through the pattern list on your own — the template above is the whole idea, and it's in every interview prep book ever printed.

## AlgoMonster plans and what they currently cost

There's no monthly option. You pick annual or lifetime, and there's a free tier with limited sample content.

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample lessons and editorials | $0 | Forever | [Start on the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | Unlimited AI assistant and Speedrun, all patterns and lessons (500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials), company question banks, member discount on live bootcamps | $99/year (~$8.25/mo), and as low as $69.99/year (~$5.84/mo) during active promotions | Billed annually | [Get Pro Yearly](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates on all future content | $189 one-time, and as low as $89 during active promotions | One payment | [Get Pro Lifetime](https://algo.monster?fpr=li69) |

A caveat worth stating plainly: AlgoMonster runs frequent sales, and the price shown on the subscribe page moves with them. Third-party trackers have logged $119/year and $459 lifetime during other promotional cycles, which is a big spread for the same product. Check the page rather than trusting any number in a blog post, including this one.

Subscription alternatives for context: LeetCode Premium lists around $159/year with a monthly option near $35, and it gives you 3,000+ problems plus company frequency data but no explanation of *why* a pattern applies. If pattern selection is your weak spot, the smaller curated curriculum is the part worth paying for. If you already know what to study and just need reps, you don't need a second subscription.

## Which plan makes sense for Koko-level problems

If you're prepping for one loop in the next couple of months and your gap is pattern recognition, take the yearly plan. It costs less than two months of LeetCode Premium and it covers the full run-up.

Lifetime only pays off if you'll come back to this repeatedly — job-hopping every two or three years is the standard case. Two or three interview cycles and the one-time payment beats renewing annually at the sale price. If you're a student who just wants to get through 875 and a dozen siblings for a single internship season, the math doesn't justify the one-time buy.

And if you're here purely for this problem: the free editorial page plus the template above will get you through it. The paid course earns its money on the twenty problems after this one.

## Quick answers

**Why is `max(piles)` a valid upper bound?**
At that speed every pile finishes in one hour, so total time equals the number of piles, and the constraints guarantee `h >= n`. Any speed above it produces identical hour counts.

**Do I need to sort `piles`?**
No. Each pile's hours depend only on its own size and `k`.

**Can `h` be less than the number of piles?**
No. The constraint is `piles.length <= h`, which is what guarantees a solution always exists.

**Is there a faster approach than `O(n log m)`?**
Not for the general case. You need at least one `O(n)` pass to evaluate any candidate, and the search space is logarithmic in the maximum pile size.

**Does this appear in real interviews?**
It's a medium with roughly 50% acceptance on LeetCode, and the "binary search on the answer" family shows up regularly at companies that lean on rate, capacity, and scheduling questions. The individual problem varies; the pattern doesn't.
