# Osysharp.Assignments.Allocations

What several assignments compete for — and who holds how much of it.

```osy
use Osysharp.Assignments@1;
use Osysharp.Assignments.Allocations@1;

policy KeepsSharedPools      => IsOperations;                  // says what there is; decides who gives way
policy ClaimsFromSharedPools => IsOperations || IsRegionLead;  // sees the pools; claims for a job they act on

var stock = OpenPool(new NewPool { Key = "malmo-housings", Title = "Housings in the Malmö warehouse", Unit = "housings", Available = 120 });

Claim(stock, nordics, 80, "Nordic orders due this month");     // held: nobody else can take these 80
Claim(stock, dach, 60, "DACH orders due this month");          // refused — 40 are left, and it says what the rest is held for
AskFor(stock, uk, 30, "UK orders due this month");             // nothing is left: it waits its turn, and its job says so

CorrectPool(stock, 90, "The count found 30 housings damaged by damp.");   // 120 are held: the pool is overdrawn
CutAllocation(nordicsClaim, 50, "Nordic orders 14 to 19 can ship in June.");   // the keeper decides who gives way
```

![A shared stock, overdrawn: 90 housings, 120 held, both holders' shares in question, and a claim that was withdrawn with why](docs/stock-overdrawn-wide-light.png)

## Use case

A pump maker's supplier of housings shuts down. Three sales regions have orders that need the 120 housings left in
one warehouse. Each region's recovery is its own job, planned by somebody who sees their own part — and all three
draw on the same stock. Without one place that says who holds what, two plans are both approved for the same
housings, and nobody finds out until a lorry is loaded.

- **Two assignments never hold the same unit.** A claim is held, or refused saying how much is left and what the
  rest is held for. Claimed again under the same key — a step run again after a crash — it is the claim already
  made.
- **A claim that does not fit can wait its turn.** The assignment then carries a piece of blocked work that says
  what it is waiting for; waiting claims are served in the order they asked, when room appears.
- **Less than was thought is somebody's decision.** A count that finds less than is held overdraws the pool. Nothing
  is taken from anybody: every holder is told its share is in question, nothing of it can be used, and whoever keeps
  the pool decides who gives way. `PoolOverdrawn` and `AllocationCut` are events an app hears to plan again.
- **What is used is gone.** `UseAllocation` takes it out of the pool for good.
- **An assignment that closes holds nothing.** Completed, called off or failed, what it held goes back and whoever
  waited is served.
- **Whoever sees a pool need not be on the assignments that draw on it.** A claim says what it is for in its
  claimant's own words, and that is what the others read; an assignment's title is shown only to who may read it.

![A region's job, as its lead reads it: what it holds of what is shared, and that it is in question](docs/job-share-in-question-wide-light.png)

## What it adds

| | |
|---|---|
| `SharedPool`, `Allocation` | the pool and one assignment's claim on it |
| `OpenPool` · `PoolBy` · `LeftIn` | open a pool (or find the one open under the key); how much nobody holds |
| `Claim` · `AskFor` | hold now or be refused (`NotEnoughLeft`) · hold now or wait in turn |
| `ReleaseAllocation` · `UseAllocation` | give back · used up |
| `CorrectPool` · `CutAllocation` | what there is now · who gives way — for whoever keeps the pool |
| `PoolOf` · `AllocationsOf` | a pool, and an assignment's claims, as whoever is looking may read them |
| `PoolCard(pool)` | the pool on a page |
| "Shared resources" on every assignment's page | a contribution to `AssignmentPanel` |
| `AllocationHeld` · `AllocationWaiting` · `AllocationReleased` · `PoolOverdrawn` · `AllocationCut` · `PoolChanged` | events |
| `policy KeepsSharedPools` · `policy ClaimsFromSharedPools` | nobody, until the app says |

## Its tests

`kits/assignments-allocations/tests` is Halden Pumps, whose supplier of housings shuts down. The response is walked
from the first report to every commitment fulfilled or resolved, over two weeks: agents assess and plan; each
region's lead approves what is taken from the shared stock — one proposal is refused by a person, one by the stock,
one region waits; a foundry outside answers through a link; a region's job is handed over and then called off; a
count finds less than is held, the director decides who gives way, and that region plans again. The one model is
scripted. No model is ever called.

| | |
|---|---|
| A claim is held, and one that does not fit is refused | `a_claim_is_held_and_one_that_does_not_fit_is_refused_saying_who_holds_the_rest` |
| The same key is the same claim | `a_claim_made_again_under_the_same_key_is_the_claim_already_made` |
| Waiting, and being served in turn | `a_claim_that_does_not_fit_can_wait_and_is_served_in_the_order_it_asked` |
| A closed job holds nothing | `a_job_that_is_closed_holds_nothing_and_whoever_waited_is_served` |
| Overdrawn, and who gives way | `less_than_was_thought_overdraws_the_pool_and_its_keeper_decides_who_gives_way` |
| Who may do what | `each_person_does_what_is_theirs_with_a_pool_and_nothing_else` |
| The whole response, day by day | `a_disruption_is_answered_region_by_region_from_one_stock_and_planned_again_when_the_stock_falls_short` |
| The run tells it | `the_run_of_a_response_tells_the_plan_the_shortfall_and_the_plan_made_again` |
| Each reader sees what is theirs | `each_reader_of_a_response_sees_what_is_theirs` |
| On screen: the stock overdrawn, a region's job, the revised plan in an inbox, the run — wide and on a phone, light and dark | four tests in `response-ui.test.osy` |

```
osy test kits/assignments-allocations/tests            # 13 tests
osy test kits/assignments-allocations/tests --pixels   # the same in a real browser, photographed
```

Two claims made at the same moment cannot both pass because the pool's held amount is checked on every write
(`[ConcurrencyCheck]`). That is the platform's check; one test body is one writer, so no test here stages the race.

## Source and package

- Package: `Osysharp.Assignments.Allocations` — add `use Osysharp.Assignments.Allocations@1;`
- Source: [`kits/assignments-allocations`](.) — `model/Pools.osy`, `model/ui/Pools.osy`
- Uses [`Osysharp.Assignments`](../assignments) and `Osysharp.Ui`.
