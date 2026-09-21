# Observations

A log of coding corrections and preferences, kept by the `code-agreements` skill. Not read before
coding, and not rules. The skill's `SKILL.md` describes the format and when an observation becomes a
proposed rule.

## 2026-09-21 · beerbox · using line instead of a fully qualified name

Said: "For the usings, they should never try to mirror the other usings around them and should
generally always use a new using line when needed. The exceptions are if that wolud cause a name
conflict." Changed: `using (new AwesomeAssertions.Execution.AssertionScope())` ->
`using AwesomeAssertions.Execution;` plus `using (new AssertionScope())` Context: a new test in a
file whose four existing uses wrote the type fully qualified; I had matched them rather than add the
`using`, because adding it made those four lines trip IDE0001. Promoted: Add a `using` directive
instead of writing a fully qualified type name

## 2026-09-21 · beerbox · parameter named for the caller's screen

Said: "These are poorly named. A service call doesn't know what's "shown", and the plain "offerId"
is ambiguous now." Context: `SetOfferQuantityAsync(credentials, shownOrderId, offerId, quantity)` on
the app's service connector — I had added `shownOrderId` (the order the app was displaying when the
customer tapped) in front of the existing `offerId`.

## 2026-09-21 · beerbox · names that don't say what the method does

Said: "FindShownOrderAsync shouldn't use the word "shown" and doesn't do what it says, what it does
is return the active order or throws an exception." "That exception is a lie, it says it is
ConcurrentModification, but there was no modification, it just checked if the "shownOrder" is the
same as the active order." ""Find" is the wrong word too. Find implies searching through multiple
items to find one that matches. I believe the old name was "GetActiveOrderAsync", which is a better
verb." "SettleAndFindActiveOrderAsync is poorly named. What does "settle" mean? The method doc is
also unclear and seems to use the term "open order" even though the method is "active order"."
Context: `OrderManager` customer paths — I had added `SettleAsync`, `SettleAndFindActiveOrderAsync`
and `FindShownOrderAsync`, coined "settle" for "ship or ride on after the close", and used "open
order" and "active order" for the same thing. Promoted: Choose a method's verb to say what the
method actually does

## 2026-09-21 · beerbox · doc comment that never says what the method does

Said: "the doc for FindShownOrderAsync is very poorly written. It starts with "The account's open
order" but never says what about that. It also talks about quantity, but there is no quantity in the
method name or the args. It's completely incomprehensible." Context: a private method's `<summary>`
I wrote as a noun phrase plus a condition ("The account's open order, which has to be the one the
app was showing…"), followed by a sentence giving a caller's reason ("a quantity is absolute, so…")
for a method that has no quantity in its signature. He then approved five rules for doc comments:
verb-first summary saying what the member does; every noun in the signature or the glossary; no
caller's reasons, history or mechanism; tags carry the specifics; a summary that cannot be written
plainly means the method does too much. Promoted: Start a doc summary with a verb that says what the
member is for

## 2026-09-21 · beerbox · doc summary verb names the purpose, not the mechanics

Said: "instead of "Throws unless", I would same something like "Checks if the given order ID is the
same as the active order and throws if they are not the same". And instead of "Returns the account's
active order" I would say something like "Fetches the account's active order. If that order...". The
verbs should be about its purpose, "checks" or "fetches", more so than what the code does; "throw"
or "return"." Changed:
`/// <summary>Throws unless <paramref name="order"/> is the order the request named.</summary>` ->
"Checks if the given order ID is the same as the active order and throws if they are not the same";
`/// Returns the account's active order. …` -> "Fetches the account's active order. If that order…"
Context: two example doc summaries I had offered as the corrected form for a guard method and a
lookup. Promoted: Start a doc summary with a verb that says what the member is for

## 2026-09-21 · beerbox · "check" on a method that makes changes

Said: "`CheckClosingsAsync` is not a good name, it doesn't say what it actually does. "Check" is
passive, but this then actually makes changes as well." Context: my proposed name for the timer
job's entry method, which closes or rolls forward orders and sends emails. Promoted: Choose a
method's verb to say what the method actually does

## 2026-09-21 · beerbox · a method's name includes what it calls

Said: "RunClosingCheckJobAsync is still not a good name. Your justification that it does nothing
itself and just calls other methods is meaningless. If you used that criteria then many methods
would do nothing. The name should be inclusive of what it calls." Changed: `RunClosingCheckJobAsync`
-> `CloseOrRollForwardOrdersAndSendClosingEmailsAsync` Context: an umbrella method that calls three
step methods in order. Promoted: Choose a method's verb to say what the method actually does

## 2026-09-21 · beerbox · name bottom-up, helpers and result types first

Said: "It may be helpful to work from the bottom up. Currently `ProcessOrdersAsync` doesn't say what
"process" means, so let's clarify that. Then `CheckClosedOrdersAsync` and
`CheckClosingSoonOrdersAsync` don't say what "check" means, so let's clarify those too." Then:
"OrderProgress is poorly named and so is OrderOutcome. Neither say what they mean clearly." Context:
a rename list for a manager class; I had named the top-level methods before the private loop helper
and its result types. Promoted: Choose a method's verb to say what the method actually does

## 2026-09-21 · beerbox · a term says what the thing does, not when it runs

Said: "For "hourly run", the term is not helpful. Anything could run hourly, and the timing could be
changed so it is not hourly. What's a better term that says what it does?" Changed: glossary term
`hourly run` -> `closing check job` Context: the glossary's name for a timer-triggered job.

## 2026-09-21 · beerbox · an enum mixing a per-item result with a loop-control signal

Said: "Part of the problem is the enum is mixing things. I don't think EmailQuotaExhausted belongs
in it. It's only returned in one place and only read in one place" Context: a private enum returned
by a per-order action, whose third member told the generic loop to stop.

## 2026-09-21 · beerbox · the caller, not the generic loop, knows why the loop stops

Said: "I don't think that a great solution. It requires that ProcessOrdersAsync knows that an email
quota error means that the loop should be aborted. That knowledge should really belong in the
caller. That is, only the caller knows that the action for one order (sending an email) affects all
orders after it (quota exceeded). If the caller passed an `IEnumerable<Order>` instead of a list
then it could dynamically abort the loop by having that ienumerable check a condition." Changed: a
private exception caught by the loop -> the caller passes `orders.TakeWhile(_ => !quotaExhausted)`
and the loop knows nothing about quota; the result enum collapsed to a `bool`. Context: a generic
for-each-order helper with failure isolation, used by three steps of which two send email.

## 2026-09-21 · beerbox · a record type exposed for a flag only some callers want

Said: "I'm looking at ScheduleEntry and I think it's not needed. It's only needed by
AdministrationManager.ListProjectedClosingsAsync, but it is inadvertently exposed in
ClosingManager.ScheduleEntryAsync. The latter should dereference and returnClosing and not expose
ScheduleEntry. The former could then just return `IReadOnlyList<(Closing closing, bool stored)>`."
Changed: `public sealed record ScheduleEntry(Closing Closing, bool Stored)` returned by both methods
-> the single-week method returns `Closing`; the list method returns a tuple list. Context: a
manager's public record pairing a document with a "stored or projected" flag.

## 2026-09-21 · beerbox · a counting loop that builds a list

Said: "also couldn't this section be done with some simple LINQ expression?" Changed:
`for (var week = from; weeks.Count < count; week = week.GetNext()) { weeks.Add(week); }` ->
`Enumerable.Range(0, count).Select(from.AddWeeks).ToList()` Context: building `count` consecutive
weeks starting at `from`.
