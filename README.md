# Redundant Connection: Union-Find Walkthrough for LeetCode 684, Why You Return the Last Edge, and How to Spot the Pattern in Interviews

Most explanations of Redundant Connection get to "just use union-find" in about two sentences and then paste code. That's technically correct and almost useless if the part tripping you up is the sentence at the end of the problem: *if there are multiple answers, return the answer that occurs last in the input*.

So here's the full version. What the problem is really testing, why the union-find loop is short enough to write in an interview without hesitation, and where people quietly lose points.

## What the problem is actually asking

A tree with `n` nodes has exactly `n - 1` edges and no cycles. You're handed a graph that started as such a tree, then had one extra edge added between two nodes that weren't already connected. So the graph has `n` edges and exactly one cycle.

The job: find an edge you can delete to get the tree back.

The constraints matter more than they look:

- `n == edges.length`, with `3 <= n <= 1000`
- `edges[i] = [ai, bi]` with `1 <= ai < bi <= edges.length`
- No duplicate edges, no self-loops, and the graph is guaranteed connected

Two consequences. First, `n` is small enough that a sloppy `O(n²)` solution passes, which is exactly why people submit sloppy solutions. Second, because there's only one cycle, the "multiple answers" warning in the prompt is about the edges *inside that cycle*: removing any single edge of a cycle breaks it and leaves a tree. Every edge on the cycle is a valid answer. The problem wants the one that appears last in the input array.

Example 1: `edges = [[1,2],[1,3],[2,3]]` returns `[2,3]`. You could also delete `[1,3]` and still have a tree on three nodes. `[2,3]` comes later, so that's the answer.

Example 2: `edges = [[1,2],[2,3],[3,4],[1,4],[1,5]]` returns `[1,4]`.

## The brute-force route, and why it's the wrong instinct

The obvious approach: iterate from the end of the list, remove that edge, check whether what's left is still a connected tree, and return the first removal that works. Each connectivity check is a BFS or DFS over `O(V + E)`, done `n` times, so you land around `O(E × (V + E))`. At `n = 1000` it survives the time limit.

It's still the wrong answer to give in an interview, for a reason that has nothing to do with speed. It doesn't teach you anything transferable. Interviewers ask this problem because it's the smallest possible instance of *incremental connectivity*: edges arrive one at a time, and you need to know whether a new edge joins two things that are already connected. Re-running BFS treats every question as a fresh graph. A union-find treats it as a running one.

## The union-find insight in one line

Union-find (disjoint set union) keeps every node pointing at a representative, the root of its component. Two operations:

- `find(x)` returns the root of `x`'s component, with path compression so that repeated lookups are cheap.
- `union(x, y)` merges the two components.

Now walk the edges in input order and ask a single question before each merge: **are these two endpoints already in the same component?** If yes, this edge closes a cycle, and since the graph has exactly one cycle, this edge is the extra one. If no, merge them and move on.

That's the whole solution. AlgoMonster's own write-up frames it as Kruskal's minimum spanning tree algorithm run on a graph where every edge has equal weight, so the sorting step disappears entirely: instead of silently discarding an edge that would create a cycle, you return it.

Worth being precise about why exactly one edge triggers the check, because "return the last edge" feels like it should need extra logic:

1. Everything except the added edge forms a spanning tree, so when you process those edges in any order, each one connects two different components at the moment you reach it. No trigger.
2. The added edge is the only one whose endpoints are already connected through the rest of the tree.
3. Since it's the highest-index edge of the cycle, returning the first edge that triggers the check automatically returns the last valid answer.

No bookkeeping, no second pass.

## Tracing Example 1

`edges = [[1,2],[1,3],[2,3]]`

| Step | Edge | Roots before the check | Action |
| --- | --- | --- | --- |
| 1 | `[1,2]` | `find(1)=1`, `find(2)=2` | Different, union them |
| 2 | `[1,3]` | `find(1)=1`, `find(3)=3` | Different, union them |
| 3 | `[2,3]` | `find(2)=1`, `find(3)=1` | Same root, return `[2,3]` |

For Example 2, the same logic plays out over more steps: `[1,2]`, `[2,3]`, and `[3,4]` each merge two separate components, `[1,4]` then finds `1` and `4` already sitting in the same component, and the function returns before it ever looks at `[1,5]`.

## Python

python
class Solution:
    def findRedundantConnection(self, edges: List[List[int]]) -> List[int]:
        parent = list(range(len(edges) + 1))  # node labels are 1-indexed

        def find(x):
            if parent[x] != x:
                parent[x] = find(parent[x])  # path compression
            return parent[x]

        for u, v in edges:
            ru, rv = find(u), find(v)
            if ru == rv:
                return [u, v]      # same component: this edge closes the cycle
            parent[ru] = rv

        return []  # unreachable given the problem's guarantee


Note `len(edges) + 1` for the array size. Node labels run from `1` to `n`, so index `0` is a dummy you never touch. Allocating `n` slots and then indexing with node `n` is the single most common runtime error on this problem.

## Java

java
class Solution {
    private int[] parent;

    public int[] findRedundantConnection(int[][] edges) {
        int n = edges.length;
        parent = new int[n + 1];
        for (int i = 1; i <= n; i++) parent[i] = i;

        for (int[] e : edges) {
            int ra = find(e[0]), rb = find(e[1]);
            if (ra == rb) return e;
            parent[ra] = rb;
        }
        return new int[0];
    }

    private int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }
}


C++, JavaScript, and Go versions are the same shape. If you're learning the pattern rather than the language, AlgoMonster keeps a free editorial for [👉 this problem with Python and Java implementations](https://algo.monster/liteproblems/684?fpr=li69) you can read without a subscription.

## Complexity, and the "nearly O(1)" claim people repeat badly

You process `n` edges. Each edge costs one or two `find` calls plus one pointer assignment, and the array is `n + 1` slots long.

- Time: `O(n)` amortized. With path compression alone, an operation is `O(log n)` amortized; add union by rank or size and it drops to `O(α(n))`, the inverse Ackermann function, which is under 5 for any input that fits in memory. Saying "effectively constant" is fine. Saying "O(1)" without the caveat is the kind of thing a picky interviewer will poke at, especially since this solution usually skips union by rank.
- Space: `O(n)` for the parent array, plus recursion depth in `find` if you write it recursively. With no union by rank, a long chain can push that recursion deeper than you'd like at `n = 1000`. It passes, but an iterative `find` is the safer habit.

## Four places people actually lose points

**Comparing nodes instead of roots.** `if u == v` is not the check. You need `find(u) == find(v)`, because two nodes can be in the same component while pointing at different parents.

**Returning `None` or an empty list as a fallback.** The problem guarantees a redundant edge exists, so the loop always returns. Adding a fallback that never executes tells the interviewer you didn't read the guarantee.

**Skipping path compression "because it's just 1000 nodes."** True, and it still matters: the same template shows up in Number of Islands II and other problems where the input is large and the union-find tree degrades without compression.

**Assuming your 684 solution also solves Redundant Connection II.** It doesn't. LeetCode 685 switches to a *directed* graph where every node except the root should have exactly one parent. That introduces a second failure mode: a node with two incoming edges. You then have to check whether removing the earlier or the later of the two candidates restores a valid tree, and you can't early-return while checking. Same data structure, different decision logic on top of it. If you paste the 684 loop into 685 and hope, you'll pass the sample cases and fail the rest.

## Where this exact pattern reappears

The trigger phrase in a problem statement is usually some version of "are these two connected?", "merge these groups", or "connections arrive over time." When you see it, union-find should be the first thing you reach for, and BFS or DFS the fallback.

- **Number of Provinces** — union every connected pair, then count distinct roots.
- **Accounts Merge** — union emails that share an account, group by root afterwards.
- **Number of Islands II** — grid cells are added one at a time, so re-running BFS per addition is wasteful.
- **Kruskal's MST / Connecting Cities With Minimum Cost** — union-find is the cycle check that makes the greedy step work.
- **Satisfiability of Equality Equations** — union the `==` constraints, then contradict any `!=` pair sharing a root.

That's the actual value of solving 684 carefully. The problem is small; the pattern is not.

## Practicing the pattern instead of memorizing one problem

Reading a solution and being able to reproduce it a week later under time pressure are different skills, and 684 is deceptive because the code is six lines long. What tends to break in a real interview isn't the code, it's the recognition step: realizing that "one extra edge" plus "undirected graph" plus "return an edge" means cycle detection during incremental merges.

This is the gap AlgoMonster is built around. The platform organizes prep by pattern rather than by problem number, with text and diagram explanations rather than video, plus a free tier you can use to test whether that format works for you before paying anything.

## Plans and current pricing

AlgoMonster doesn't do monthly billing. There are three tiers: a free tier with limited access, an annual Pro plan, and a one-time lifetime plan.

| Plan | What you get | Price | Billing cycle | Get started |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample patterns, lessons, and problems | $0 | Free forever | [ Browse the free lessons](https://algo.monster/subscribe?fpr=li69) |
| Pro — Yearly | All patterns, lessons, and problems; unlimited AI assistant and Speedrun; company questions updated monthly; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; bonus Foundation and Build-an-LLM courses; member discount on live bootcamps | $99/year (about $8.25/month billed annually) | 12 months, renewable, cancel or upgrade anytime | [ Start Pro yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates, one payment | $189 | One-time | [ Get lifetime access](https://algo.monster/subscribe?fpr=li69) |

Prices move. AlgoMonster runs frequent promotions, and list prices of $198/year and $378 lifetime have been shown struck through during sales, so what you see at checkout can differ from what a review site quoted last month. Specialty tracks developed with outside partners, such as Quant Interview Prep, are sold separately and aren't part of either Pro plan.

## Is paying for this worth it for one graph problem?

No, and it's worth saying plainly. The free editorial for 684 covers the approach, and you can get the union-find template from a dozen places. Paying for a subscription to solve a single Medium problem is a bad trade.

The math changes if you're preparing broadly. LeetCode Premium runs around $159/year, and AlgoMonster's annual plan undercuts that; the lifetime plan costs about $189 once, which is less than a year of most alternatives and pays for itself the second time you job-search. Reviewers who count the curriculum tend to land on roughly 48 patterns and a few hundred problems, which is a small fraction of LeetCode's problem bank. If your plan is to grind 500 problems, this isn't the tool. If your plan is to stop grinding randomly and learn maybe fifty reusable templates well enough to place a new problem into one of them, the structure is the product and the problem count is beside the point.

Two things to check on the site rather than take from a blog post, including this one: the current price at checkout, and the terms of the job guarantee AlgoMonster advertises, which depends on completing the curriculum and has conditions attached.

## Quick answers to the questions this problem generates

**Do I have to use union-find here?** No. A DFS that detects the cycle also runs in `O(n)` and is perfectly acceptable. Union-find is the version that generalizes, so it's the one worth owning.

**Why does the answer depend on input order?** Because several edges on the cycle are valid deletions, and the problem tells you to pick the last. Change the order of `edges` and the returned edge can change with it.

**What's the actual difference between 684 and 685?** 684 is an undirected graph with one extra edge. 685 is directed, a node can end up with two parents, and you have to reason about in-degrees before union-find even helps. Different problem, same toolkit.
