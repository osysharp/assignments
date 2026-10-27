# Osysharp.Assignments.Agents

An agent's work, attached to the assignment it is for.

```osy
var outcome = await AssignedWork.Do(a, Agents.Of<Resolver>(), new InvestigateDelivery(), order, key: "investigate");
```

The same call as `agent.Do(skill, input)`, with the assignment in charge of it: the work is attached under its key
before the agent starts, what it spends is the assignment's cost, and a step that is run again does not do finished
work twice.

## Use case

An agent's run is short and can die. The thing it was working on — a delivery problem, a repair, a hire — lasts
days. When the run is the only record, a crash loses the thread: nobody knows what was being waited on, the retry
starts from nothing, and the refund is made a second time.

- **The assignment waits on the work, and says so.** The piece of work is attached in a unit of work of its own
  before the agent starts. A run that dies leaves the assignment saying "waiting on Resolver for its result".
- **Finished work is not done again.** Asked for under the same key, work that had finished is answered from its
  receipt: the typed result, and no model called.
- **Work that never reported back is run again** on the same attachment; work that failed or was stopped is a new
  attempt beside the old one.
- **What it cost is the assignment's cost** — the receipt's own figure, with the work it handed on.
- **A spent assignment starts no more.** An assignment whose budget is used up refuses new agent work before a
  receipt is opened.
- **The agent has to be on the assignment**, and so does whoever asks: somebody who only watches a case gives the
  agent no work on it.

## What it does

| | |
|---|---|
| `AssignedWork.Do(a, agent, skill, input, key, about)` | Attach, run, settle. Answers the same `AgentOutcome<TOut>` as `agent.Do`. |
| `AssignedWork.Attach(a, task, key)` | Attach a piece of agent work that was run elsewhere, by its receipt. It is settled when it ends, whoever is looking. |
| `AssignedWork.Of(a)` · `AssignedWork.ReceiptOf(work)` | The agent work attached to an assignment, and each one's receipt. |
| Work the agent hands on | Attached to the same assignment, a piece each — "Resolver: investigate delivery — handed on by CaseLead" — for every specialist that is on it; settled when it ends. The piece at the top counts the whole tree; a handed-on piece counts nothing itself. A specialist that is not on the assignment is not attached, and the piece above it says so. |

Pair it with `DoneFor` / `RecordDone` in the tools that must not act twice — a refund, a booking — and a run that is
repeated finds what was done:

```osy
[Action("Refund the order")]
public string RefundOrder(ShopOrder order, string reason) {
  var theCase = DeliveryCase.Single(c => c.Order == order).Assignment;
  var done = DoneFor(theCase, "refund:" + order.Number);
  if (done != null) { return "Already refunded: " + done.Result; }
  order.Status = OrderStatus.Refunded;
  RecordDone(theCase, "refund:" + order.Number, "Refund order " + order.Number, $"{order.Total} EUR refunded: {reason}");
  return $"Refunded {order.Total} EUR.";
}
```

The assignment must be stored before agent work is attached to it: open it in an earlier step, or
`UnitOfWork.Commit()` first.

## Its tests

`kits/assignments-agents/tests` is a small tea shop whose customer says "my order has not arrived, and I need it
before Friday". One test carries the case from her report to a verified outcome and reopens it; the others take it
apart:

| | |
|---|---|
| The whole case, and opening it again | `a_delivery_case_is_carried_from_the_customers_report_to_a_verified_outcome_and_opened_again` |
| A crash while the agent works | `an_assignment_resumes_after_a_crash_during_agent_work` |
| A crash after a tool acted | `an_assignment_resumes_after_a_crash_during_a_tool_call_without_repeating_its_effect` |
| Finished work, asked for again | `agent_work_that_had_finished_is_answered_from_its_receipt_with_no_model_called` |
| Failed work, tried again | `work_that_failed_is_a_new_attempt_beside_the_old_one` |
| The money runs out | `budget_exhaustion_completes_partially_with_the_findings_kept` |
| A spent assignment | `an_assignment_whose_budget_is_spent_starts_no_more_agent_work` |
| An agent that is not on it | `an_agent_that_is_not_on_the_assignment_does_no_work_for_it` |
| Somebody who is not on it | `somebody_who_is_not_on_a_case_gives_the_agent_no_work_on_it` |
| Work that is handed on | `work_an_agent_hands_on_is_attached_to_the_same_assignment_and_counted_once`, `a_specialist_that_is_not_on_the_assignment_is_not_attached_and_the_case_says_so` |

No model is called: the shop's one model is scripted.

```
osy test kits/assignments-agents/tests
```

## Source and package

- Package: `Osysharp.Assignments.Agents` — it arrives by itself in an app that uses
  [`Osysharp.Assignments`](../assignments) and [`Osysharp.Agents`](../agents); or `use Osysharp.Assignments.Agents@1;`
- Source: [`kits/assignments-agents`](.) — `model/AgentWork.osy`

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.Llm;
using Osysharp.Agents;
using Osysharp.Agents.Testing;
using Osysharp.Assignments;
using Osysharp.Assignments.Agents;
using Osysharp.WorkRepositories;
using Teashop;

[Test(Shop)]
void agent_work_that_had_finished_is_answered_from_its_receipt_with_no_model_called() {
  var id = NoraReports();
  Guid first = Guid.Empty;
  runas (Sana) {
    var script = InvestigationScript(Case(id));
    ScriptedAnswer.Stub((model, request) => script.Answer(model, request));
    first = Investigate(Case(id)).Task;
  }
  Background.Settle();
  runas (Sana) {
    // The step is run again — a process that was restarted after the work had finished. No model answers this time.
    ScriptedAnswer.Stub((model, request) => { throw new InvalidOperationException("No model should be asked."); });
    var again = Investigate(Case(id));
    Assert.True(again.Done);
    Assert.Equal(first, again.Task);
    Assert.True(again.Result.Lost);
    Assert.Equal("Held as damaged at the Malmö depot since 6 March.", again.Result.What);
    var again1 = Case(id).Assignment;
    Assert.Equal(1, again1.Work.Count());
    Assert.Equal(800, CostOf(Case(id).Assignment).WorkMicros);
  }
  runas (Lars) { Assert.Equal(1, AgentTask.Count()); }
}
```

From [`tests/delivery.test.osy`](tests/delivery.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Llm;
using Osysharp.Agents;
using Osysharp.Agents.Testing;
using Osysharp.Assignments;
using Osysharp.Assignments.Agents;
using Osysharp.WorkRepositories;
using Teashop;

[Test(Shop)]
void a_specialist_that_is_not_on_the_assignment_is_not_attached_and_the_case_says_so() {
  TestClock.Set(Tuesday());
  var id = NoraReports();
  Guid leading = Guid.Empty;
  runas (Sana) {
    var c = Case(id);
    // Only what the lead hands on, with nothing written to the case by the resolver: it is not on it.
    var cast = AgentCast.Of()
      .Plays<CaseLead>(AgentScript.Of(
        Says.HandTo<Resolver>(new InvestigateDelivery(), Says.Row(c.Order.Id)),
        Says.HandBack(new DeliveryFinding { Lost = true, What = "Held at the depot.", Suggests = "refund" })))
      .Plays<Resolver>(AgentScript.Of(
        Says.Tool(Actions.Of(TrackParcel), OrderArg(c)),
        Says.HandBack(new DeliveryFinding { Lost = true, What = "Held at the depot.", Suggests = "refund" })));
    ScriptedAnswer.Stub((model, request) => cast.Answer(model, request));
    var outcome = Lead(c, withTheResolver: false);
    Assert.Equal("", outcome.Problem);
    leading = outcome.Task;
  }
  Background.Settle();
  runas (Sana) {
    var a = Case(id).Assignment;
    // One piece on the case — the lead's — which still counts the whole tree, and a note of what was handed to whom.
    Assert.Equal(["Lead the case of order T-1042"], AssignedWork.Of(a).Select(w => w.Title).ToList());
    Assert.Equal(800, AssignedWork.Of(a)[0].CostMicros);
    Assert.Equal(1, AssignmentActivity.Count(n => n.Assignment == a
      && n.Said == "Lead the case of order T-1042: Case lead handed part of this to Resolver, which is not on the assignment."));
  }
}
```

From [`tests/handed-on.test.osy`](tests/handed-on.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Llm;
using Osysharp.Agents;
using Osysharp.Agents.Testing;
using Osysharp.Assignments;
using Osysharp.Assignments.Agents;
using Osysharp.WorkRepositories;
using Teashop;

[Test(Shop)]
void work_that_failed_is_a_new_attempt_beside_the_old_one() {
  var id = NoraReports();
  runas (Sana) {
    // The model never manages a finding that fits: the work fails.
    var bad = AgentScript.Of(Says.HandBack(new DeliveryFinding { Suggests = "shrug" }), Says.HandBack(new DeliveryFinding { Suggests = "dunno" }));
    ScriptedAnswer.Stub((model, request) => bad.Answer(model, request));
    var failed = Investigate(Case(id));
    Assert.Equal(AgentTaskStatus.Failed, failed.Status);
    var a = Case(id).Assignment;
    Assert.Equal(WorkState.Failed, WorkOf(a, "investigate")!.State);
    Assert.StartsWith("Resolver could not finish:", WorkOf(a, "investigate")!.Result);
    // What it spent failing is still the case's cost.
    Assert.Equal(400, CostOf(a).WorkMicros);

    var script = InvestigationScript(Case(id));
    ScriptedAnswer.Stub((model, request) => script.Answer(model, request));
    Assert.True(Investigate(Case(id)).Done);
    a = Case(id).Assignment;
    Assert.Equal(WorkState.Failed, WorkOf(a, "investigate")!.State);
    Assert.Equal(WorkState.Done, WorkOf(a, "investigate#2")!.State);
    Assert.Equal("Find out where order T-1042 is (attempt 2)", WorkOf(a, "investigate#2")!.Title);
    Assert.Equal(1200, CostOf(a).WorkMicros);
  }
}
```

From [`tests/delivery.test.osy`](tests/delivery.test.osy).

## Learn more

- [The Osysharp.Assignments.Agents reference](https://osysharp.com/reference/agent/kit-assignments-agents/)
- [Osysharp.Assignments.Agents in the package catalogue](https://osysharp.com/packages/#osysharp-assignments-agents)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
