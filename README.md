# Osysharp.Llm

Call a model from your app — and know, for every call, which model served it, why that one, and what it cost.

`Llm.Complete`, `Llm.Extract<T>` and `Llm.Decide<T>` are the calls you write. Behind each of them is one road, written
in Osy#: your router picks the model, room is reserved against every budget the call falls under before anything is
sent, the provider is called, the next model is tried when one fails, and a meter row is written for every attempt —
priced from your own price book, in millionths of a dollar.

```osy
use Osysharp.Llm@2;
use Osysharp.Models.Anthropic@0 { egress "api.anthropic.com"; }

app.Secrets = [ new Secret("Anthropic") ];

app.Llm = new LlmSetup {
  Models = [
    new LlmModel { Name = "Fast",   Client = new AnthropicClient { ApiKey = Secret.Anthropic }, Model = "claude-haiku-5-5",
                   Price = new LlmPrice { Input = 0.10m, Output = 0.50m } },
    new LlmModel { Name = "Strong", Client = new AnthropicClient { ApiKey = Secret.Anthropic }, Model = "claude-opus-5-5",
                   Price = new LlmPrice { Input = 4, Output = 20, CacheRead = 0.20m } },
  ],
};

var line    = await Llm.Complete("Summarise this in one line: " + note);            // text in, text out
var invoice = await Llm.Extract<Invoice>(scannedText);                               // a typed value, by the type's own schema
var urgency = await Llm.Decide<Urgency>(message, "How urgent is this for them?");    // a typed decision, with a confidence
```

`Llm.Complete`, `Llm.Extract<T>` and `Llm.Decide<T>` each answer a task. `await` it where you make it for the plain
call; keep the tasks and several calls are in flight at once, each routed, budgeted and metered on its own.

## Use case

An app that puts a model to work on its own data: reading invoices, sorting an inbox, drafting replies, answering
questions over its records. The first call is easy. What is hard arrives a month later — a bill nobody can explain, a
provider outage that took a feature down with it, one user who spent the whole month's allowance in an afternoon, a
test suite that needs a paid key to run.

This kit is the part of that you would otherwise write yourself, and get subtly wrong:

- **One place decides which model serves a call.** No call site names a provider. You list the models you have keys
  for; a router ([`Osysharp.LlmRouter`](../llm-router) or a class of your own) orders them for each call.
- **A provider that fails is not an outage.** The call moves to the router's next model — and only to a model on the
  router's list.
- **A limit is a limit.** A budget's room is reserved before the call leaves, atomically, so twelve calls arriving at
  once cannot all take the last dollar.
- **Every attempt is on the record.** Served, failed or refused: the model, the rule that chose it, the facts the rule
  decided from, the tokens the provider reported, the cost at the price in force.
- **Tests need no model.** A scripted model stands where a provider would, and everything around it is real.

It has no screens. It is the layer other packages and your own pages build on.

## What it does

| | |
|---|---|
| **Providers** | A provider is a class implementing `ILlmClient` (text and tools) or `IDecisionModel` (typed decisions). [`Osysharp.Models.Anthropic`](../anthropic), [`Osysharp.Models.OpenAi`](../openai) (and any endpoint that speaks its wire) and [`Osysharp.Models.Gemini`](../gemini) are three; an endpoint nobody ships support for is a class you write. |
| **Routing** | `LlmSetup.Router` is an `ILlmRouter`: a pure function from the facts of a call (its kind, its step, its size, what it needs, what it may cost) to an ordered list of your models, with the reason each other one was dropped. A call site may ask for a model by name; that can only narrow what the router admits. With no router, a call goes to the model it asked for, or to `Default`. |
| **Fallback** | The router's order is the fallback order. A provider that fails (an outage, a rate limit, an overload) is recorded and the next model is asked. A stream falls back until its first event; after that the answer is that model's. |
| **Budgets** | `SetBudget("requester", "", LlmBudgetWindow.Day, 5m)` — five dollars a day for each person. A budget is for a scope: `app`, `requester`, `model`, or any scope a caller names on its calls (`task:8f1c…`, `agent:Screener`). Per day, per month or in total. A call that does not fit is refused before anything is sent (`LlmBudgetExceeded`) — or, when the budget or the call says `StepDown`, the router is asked again with what is left as the ceiling and a cheaper model serves. |
| **The meter** | One `ModelCall` row per attempt: the model, the provider's id for it, the router's rule, the facts and the rejections as JSON, who asked, the scopes charged, the provider's token counts (cached reads and writes apart), the cost in millionths of a dollar and the price-book row it was priced from. Written once, never changed. `SpentOn("task:" + task.Id)` totals a scope. |
| **The price book** | `SetModelPrice("Fast", 0.10m, 0.50m)` — dollars per million tokens, from a moment on. A call keeps the price it was made at: re-pricing never rewrites what was spent. Until the book has a row, the price on the model's own declaration applies. |
| **Typed values** | `Llm.Extract<T>` sends `T`'s own schema (`Json.SchemaOf<T>()`, with the words you wrote on its members) and reads the answer as a `T`. An answer that is not the shape is sent back to the model once. |
| **Typed decisions** | `Llm.Decide<T>` answers a `Decision<T>`: the value, a confidence, and whether that confidence is calibrated. A model built to decide (`LlmModel.Decider`) serves it when you have one; any chat model can stand in, and says its confidence is its own estimate. |
| **Concurrency** | `LlmModel.MaxConcurrent` caps the calls on their way to one model; a call past the cap moves to the next candidate. |
| **The operator's switch** | When the operator has switched generative AI off for the app, every call is refused before anything is sent (`LlmSwitchedOff`). |
| **Events** | `LlmBudgetNearlySpent` once per period when a budget passes its warning share, `LlmBudgetSpent` when a call is refused. Hear them with `[On]`. |

## Who may do what

Three policies, each with a default you override by declaring it:

```osy
policy UsesModels        => IsStaff;      // who may call a model at all        (default: everyone)
policy ReadsModelUsage   => IsFinance;    // who reads the meter                (default: nobody)
policy ManagesModelSpend => IsAdmin;      // who sets prices and budgets        (default: nobody)
```

## Calls

```osy
// Text.
string answer = await Llm.Complete(prompt);
string answer = await Llm.Complete(prompt, "Strong");                  // one of your models, by its name

// Everything said: the whole request, and what the call is.
LlmResult result = await Llm.Complete(request, new LlmCall {
  Kind = LlmTaskKind.Reason, Step = LlmStep.Plan,                // what the router's floors are declared per
  Scopes = ["task:" + task.Id], Purpose = "weekly digest",       // what it is charged to, and what for
  OnLimit = LlmOnLimit.StepDown, MaxCostMicros = 20000,          // what to do when money is short
});
result.Text();  result.Model;  result.CostMicros;  result.Attempts;

// As it is written, for an answer somebody watches arrive.
stream<LlmStreamEvent> Draft(string prompt) {
  foreach (var e in Llm.Stream(prompt)) { yield return e; }
}

// A typed value.
class Invoice { public string Supplier = ""; public decimal Total; public DateOnly? Due; }
Invoice invoice = await Llm.Extract<Invoice>(scannedText, "The supplier is the company that sent it.");

// A typed decision.
enum Urgency { Low, Normal, High }
Decision<Urgency> d = await Llm.Decide<Urgency>(message.Body, "How urgent is this for the customer?");
if (d.Value == Urgency.High && d.IsAtLeast(0.8m)) { Escalate(message); }
```

A call is refused, before anything is sent, with an exception you can tell apart: `LlmSwitchedOff`, `LlmRouteDenied`,
`LlmBudgetExceeded` (it carries the scope, the subject, the window and what is left), `LlmBusy` — all of them
`LlmRefused`. A provider's own failure is an `ExternalServiceException`.

Calls run from a task too: `Parallel.ForEach(notes, note => { Summarise(note); })` meters and budgets each one. The
kit's own writes are committed by themselves, so a call in the middle of your work commits none of your work, and a
call whose surrounding work then fails is still on the meter.

## Budgets

```osy
SetBudget("app", "", LlmBudgetWindow.Month, 500m);                          // the whole app, a month
SetBudget("requester", "", LlmBudgetWindow.Day, 5m);                        // each person, a day — one row for everybody
SetBudget("requester", vip.Id.ToString(), LlmBudgetWindow.Day, 50m);        // one person, instead of the row above
SetBudget("task", task.Id.ToString(), LlmBudgetWindow.Total, 2m,            // one piece of work, in all —
          LlmOnLimit.StepDown);                                             //   and step down rather than stop

decimal spent = SpentIn("requester", person.Id.ToString(), LlmBudgetWindow.Day);
```

Every call is charged to `app`, to `requester` (the signed-in person, or the actor the code runs as), to `model`, and
to whatever scopes its caller names. A reservation holds the most the call could cost — its prompt and the longest
answer it may give, at the model's price — and is settled at what the provider says it used. A call whose process
died gives its room back after `LlmSetup.HoldFor` (ten minutes).

Code can also hold its own calls to a limit, with no budget set by anybody — "the calls for this job, half a dollar
in all":

```osy
await Llm.Complete(request, new LlmCall {
  Limits = [ new LlmLimit { Scope = "job:" + job.Id, LimitMicros = 500000 } ],
});
decimal used = SpentAgainst("job:" + job.Id);
```

Every call that carries the limit is counted against it and reserves its room before it leaves, like a budget. It
binds only the calls that carry it — code limits its own work, never anybody else's — and budgets apply on top.

Days and months are UTC.

### A workflow's run is a scope, with nothing said

A call made in a workflow's run is charged to the workflow and to that run — `workflow:Onboarding` and
`run.Onboarding:8f1c…` — so what a process costs is counted as it happens:

```osy
SetBudget("workflow", "Onboarding", LlmBudgetWindow.Month, 200);       // every run of it together, a month
SetBudget(RunKind("Onboarding"), "", LlmBudgetWindow.Total, 2);        // each run of it, two dollars in all
var cost = SpentOn(RunScope("Onboarding", run));                       // what one run has cost
```

`LlmCall.ChargedToItsRun = false` is for code that names the run itself.

## Testing

Put a `ScriptedModel` where a provider would go, and answer for it from the test:

```osy
// the test app's setup
app.Llm = new LlmSetup { Models = [
  new LlmModel { Name = "Fast", Client = new ScriptedModel { Name = "Fast" }, Model = "scripted-fast",
                 Price = new LlmPrice { Input = 1, Output = 5 } },
] };

[Test]
void a_note_is_summarised_and_the_call_is_on_the_meter() {
  ScriptedAnswer.Stub((model, request) => Scripted.Text("Early beans, saved in 2024.", 1000, 200));
  Assert.Equal("Early beans, saved in 2024.", Summarise(note));
  Assert.Equal(2000, ModelCall.Single().CostMicros);          // 1000 tokens at $1 a million, 200 at $5
}
```

`Scripted.Text`, `Scripted.ToolUse` and `Scripted.Decision` build answers; throw an `ExternalServiceException` from the
script to play a provider that is down; `GenerationIsOn.Stub(() => false)` plays the operator's switch. Routing,
budgets and the meter are the real ones, so a test asserts what a call cost the same way your app reads it.

## Writing a provider

```osy
public class MyClient : ILlmClient {
  public string ApiKey = "";
  internal LlmResponse Complete(LlmRequest request) { … }           // request.Model is already decided: serve it or throw
  internal stream<LlmStreamEvent> Stream(LlmRequest request) { … }  // end with MessageComplete, carrying the counts
}
```

Report the token counts the provider reported — the meter prices from them. Throw `ExternalServiceException` when the
provider fails; that is what moves the call to the next model. When `request.OutputSchemaJson` is set, answer with one
text block whose text is the JSON document.

The two methods are `internal`, as they are on `ILlmClient`: any package implements them, and only this package calls
them — from the call path, which routes, reserves and meters around each call. Code that holds a client and writes
`client.Complete(request)` does not compile; it calls `Llm.Complete`. So there is no model call that is not on the
meter.

## What only the kit writes

The meter, the price book and the budgets are rows in your app's database that only this package's functions write.
Each of these is a test in the kit's test app:

| | |
|---|---|
| A meter row is made only by the call path, and never changed | `a_meter_row_is_written_only_by_the_call_path_and_never_changed` |
| What a budget has counted is changed only by the call path | `what_a_budget_has_counted_is_changed_only_by_the_call_path` |
| Budgets are set only by who manages the spend, through `SetBudget` | `budgets_are_set_only_by_who_manages_the_spend_and_only_through_the_kit` |
| Prices are set only by who manages the spend, through `SetModelPrice` | `prices_are_set_only_by_who_manages_the_spend_and_only_through_the_kit` |
| A call is refused before anything is sent when its budget has no room | `a_call_that_does_not_fit_its_budget_is_refused_before_anything_is_sent` |
| Twelve calls at once take exactly what fits | `a_budget_reserves_atomically_under_parallel_calls` |
| A limit a call carries is held the same way, and binds only the calls that carry it | `a_limit_a_call_carries_is_reserved_against_before_the_call_leaves`, `a_limit_binds_only_the_calls_that_carry_it` |
| What a limit has counted is changed only by the call path | `what_a_limit_has_counted_is_changed_only_by_the_call_path` |
| A provider is called only by the kit's call path — an app or another package calling one directly does not compile | `AProviderIsCalledOnlyByTheLlmKitTests` (the platform's own tests) |
| A call is on the meter though the work around it failed | `a_call_is_on_the_meter_though_the_work_it_was_made_for_failed` |
| The meter is read only by who the app says | `a_call_is_recorded_for_who_it_ran_for_and_read_only_by_who_the_app_says` |
| A person is taken off the meter only by who manages the spend, through `ForgetModelCallRequester`, and what their calls cost stays counted | `a_person_is_taken_off_the_meter_and_what_their_calls_cost_stays_counted`, `only_whoever_manages_spend_takes_a_person_off_the_meter_and_only_through_the_kit` |

Anyone who may call a model can read the budgets and what each has counted; the per-call detail is the meter's, and
`ReadsModelUsage` decides who reads that — with whoever manages the spend, who deletes old rows and takes a person
off them.

**The meter keeps no text of a call, and it does say who the call was for.** `ForgetModelCallRequester(person)`
takes a person off every row and deletes what was counted against a budget of their own; the rows and their cost
stay. With [`Osysharp.Privacy`](../privacy) in the app, [`Osysharp.Privacy.Llm`](../privacy-llm) does it when the
person asks and when the period ends.

## Its tests

`kits/llm/tests` is a seed bank that reads the notes growers send in with their seed: 51 tests over three scripted
models and a small router of its own — the call path, the fallback walk, the price book, budgets under twelve
simultaneous calls, typed extraction with its repair, and decisions from a decision model and from a chat model
standing in.

```
osy test kits/llm/tests
```

## Source and package

- Package: `Osysharp.Llm` — `use Osysharp.Llm@2;`
- Source: [`kits/llm`](.) — `model/Contract.osy` (providers), `Routing.osy`, `Setup.osy`, `Call.osy` (the call path),
  `Streaming.osy`, `Structured.osy` (extract, decide), `Budgets.osy`, `Meter.osy` (the meter and the price book),
  `Llm.osy` (the calls an app writes), `Scripted.osy` (the scripted model)
- Uses nothing but the language: no host, no secret. The provider packages ask for theirs.
