# Observations

A log of coding corrections and preferences, kept by the `code-agreements` skill. Not read before
coding, and not rules. The skill's `SKILL.md` describes the format and when an observation becomes a
proposed rule.

## 2026-09-21 · beerbox · parameter named for the caller's screen

Said: "These are poorly named. A service call doesn't know what's "shown", and the plain "offerId"
is ambiguous now." Context: `SetOfferQuantityAsync(credentials, shownOrderId, offerId, quantity)` on
the app's service connector — I had added `shownOrderId` (the order the app was displaying when the
customer tapped) in front of the existing `offerId`.
Promoted: Name a thing for what it is, not for the circumstances around it

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
Promoted: Name a thing for what it is, not for the circumstances around it

## 2026-09-21 · beerbox · an enum mixing a per-item result with a loop-control signal

Said: "Part of the problem is the enum is mixing things. I don't think EmailQuotaExhausted belongs
in it. It's only returned in one place and only read in one place" Context: a private enum returned
by a per-order action, whose third member told the generic loop to stop.
Rejected: too specific to the situation; the three corrections it was drawn from are different moves and do not generalise usefully

## 2026-09-21 · beerbox · a record type exposed for a flag only some callers want

Said: "I'm looking at ScheduleEntry and I think it's not needed. It's only needed by
AdministrationManager.ListProjectedClosingsAsync, but it is inadvertently exposed in
ClosingManager.ScheduleEntryAsync. The latter should dereference and returnClosing and not expose
ScheduleEntry. The former could then just return `IReadOnlyList<(Closing closing, bool stored)>`."
Changed: `public sealed record ScheduleEntry(Closing Closing, bool Stored)` returned by both methods
-> the single-week method returns `Closing`; the list method returns a tuple list. Context: a
manager's public record pairing a document with a "stored or projected" flag.
Rejected: too specific to the situation; the three corrections it was drawn from are different moves and do not generalise usefully

## 2026-09-22 · beerbox · doc comment clause justifying the code's shape

Said: "\"which is all the sweep needs of it\" is inappropriate and not needed in this comment. Are there similar comments that should be pruned?" then "prune them"
Changed: `A document's id and the account it belongs to, which is all the sweep needs of it.` -> `A document's id and the account it belongs to.`
Context: doc summaries and param docs on issue-348 whose trailing clause explained why the member's shape was sufficient, or described the caller; nine pruned.
Promoted: Start a doc summary with a verb that says what the member is for

## 2026-09-22 · beerbox · doc summary that describes the caller, again

Said: "And then you write another inappropriate comment, sigh. \"whose orders an admin asked for\""
Changed: `Reads the close and ship date of a closing whose orders an admin asked for.` -> `Reads the close and ship date of a closing.`; the exception tag's `, because no order is on a cancelled closing` was dropped too, and the method renamed from `InfoOfClosingWithOrders` to `GetClosingInfo`.
Context: a private helper on AdministrationManager written the same afternoon that nine such clauses were pruned from this branch.
Promoted: Start a doc summary with a verb that says what the member is for

## 2026-09-22 · beerbox · split a request instead of nullable fields with a mode flag

Said: "Why can't we just change AdjustClosingRequest to have the fields not-nullable? Or potentially split it into CancelClosingRequest with the bool only, and AdjustClosingRequest with the dates only."
Changed: `AdjustClosingRequest(ClosingID, bool Cancel, DateTime? Close = null, DateOnly? Ship = null)` -> `AdjustClosingRequest(ClosingID, DateTime Close, DateOnly Ship)` + `CancelClosingRequest(ClosingID)` on its own route
Context: the admin's closing adjustment message in Beerbox.Service.Contracts; one request had served both "set the dates" and "cancel" with a flag and nullable dates.
Rejected: too specific to the situation; the three corrections it was drawn from are different moves and do not generalise usefully

## 2026-09-23 · beerbox · a relative word needs its reference point

Said: "CloseIsBehind is not a clear term to me, I don't know what it is behind. What's a better name?"
Changed: `ServiceRejectionReason.CloseIsBehind` and "ahead/behind" doc wording -> `CloseIsInThePast`, "in the future / in the past"
Context: the rejection when an admin adjusts a closing to a close that has already happened.
Promoted: Use words the reader already knows

## 2026-09-23 · beerbox · tests of one class in one test class

Said: "Why is DataManagerLogicTest separate from DataManagerTest? Why not just have those tests inside DataManagerTest?"
Changed: `DataManagerLogicTest` beside `DataManagerTest` -> its tests moved into `DataManagerTest`'s regions, file deleted
Context: Beerbox.App.Core.Tests; the second class had been added during a coverage pass.
Promoted: Keep all tests for a production class in one test class

## 2026-09-23 · beerbox · a label that says what is counted

Said: "\"Weeks shown\" is not accurate, since it's only the projected weeks. Probably \"Future closings to show\" is better"
Context: the admin Closings page's projection-count picker.
Promoted: Use words the reader already knows

## 2026-09-25 · beerbox · internal term in user-facing text

Said: "I would rephrase to "This week has no orders to pack" because the packers likely won't know about the concept of "Closing" as a week."
Changed: "This closing has no orders to pack." -> "This week has no orders to pack."
Context: webapp Packing page messages for non-admin packers.
Promoted: Use words the reader already knows

## 2026-09-25 · beerbox · nested test classes inheriting the outer test class

Said: "Restore the grouping and change the inheritance to BaseTest"
Changed: `sealed class Adjust : ClosingManagerTest` -> `sealed class Adjust : BaseTest`
Context: ClosingManagerTest.cs; nested groups inheriting the outer class re-ran every outer test once per group.
Promoted: Keep all tests for a production class in one test class

## 2026-09-25 · beerbox · same list declared twice

Said: "We end up with 2 section lists, the adminSectionRoutes and adminSections. Is there some way we could combine these?"
Changed: exported `adminSections` + derived `adminSectionRoutes` -> one `adminSections` whose items carry `route`
Context: webapp `adminSections.tsx`.
Promoted: Prefer one entry per item over parallel lists keyed the same way

## 2026-09-25 · beerbox · helper file with one caller

Said: "Actually, move @src/webapp/src/components/admin/orderStatus.tsx into Orders, it's more confusing having it in a separate file"
Changed: `orderStatus.tsx` (status type, rule, chip) -> unexported members of `Orders.tsx`
Context: webapp Orders page Status column.
Promoted: Prefer inlining anything that is used only once

## 2026-09-25 · beerbox · parallel tables per key merged into one object

Said: "what if the object was `{ active: {color:'default', rank: 0 }}`?"
Changed: separate status list, colour record and label switch -> one `orderStatuses` object of `{ color, label, rank }`
Context: webapp Orders page Status column.
Promoted: Prefer one entry per item over parallel lists keyed the same way

## 2026-09-29 · beerbox · values from code restated in markdown docs

Said: "For #3, stop documenting specific values in those markdown files. Keeping those values in sync with the code is problematic." then, on the next doc fix, "fix it, but don't document raw values instead reference where they are defined"
Changed: "Every wait shares one budget (60s…)" in a skill doc and "has the same budget: 60s" in the MobileJourneys README -> "the fixture's `WaitBudget` in `BeerboxPlatforms.cs`" / "the fixture's `WaitBudget`, set on its config"
Context: journey wait-budget docs after the budget became per fixture.
Promoted: Link to the code that defines a value instead of repeating the value in a doc

## 2026-10-02 · beerbox · named the rating image after its artwork

Said: "In the code you called the star rating a "paw", that should be renamed to not use "paw" since that's the current image theme but could change. it should be just "RatingImage" or something like that."
Changed: `rating_paw.svg`, `AppImages.RatingPaw`, `pawSize`, `RenderPaws()` -> `rating_image.svg`, `AppImages.RatingImage`, `imageSize`, `RenderImages()`
Context: ProductCard draws five copies of the rating image and clips the top row to the rating; the name described the current artwork, not the role.
Promoted: Name a thing for what it is, not for the circumstances around it

## 2026-10-02 · beerbox · a mechanism that would exist for one journey

Said: "I generally prefer to avoid special or one-off cases. Maybe it's better to leave it separate."
Context: the CardSetupRoundTrip journey repeats five of Home's screens. Moving it under Home needed a test-control command (or a per-journey scenario tweak) that only this one journey would use; he kept the duplicated journey instead.

## 2026-10-02 · beerbox · a limit and a format list restated in a deployment doc

Changed: "Every upload is scaled down to no taller than 2184 pixels, keeping its shape, and stored as whichever of WebP, JPEG or PNG comes out smallest. AVIF is never stored: Android decodes it with its AV1 video decoder, which on many phones refuses an image over 2048 pixels on a side." -> "Every upload is scaled down, keeping its shape, to no taller than `MaxProductImageHeight` in [`AdministrationManager`](…), and stored in whichever of the formats listed in `OutputTypes` in [`TinifyImageCompressionService`](…) comes out smallest. Both constants carry the reasons for their values, including why AVIF is never stored."
Context: the Tinify section of docs/deployment.md. In the same change I had added a method name to docs/development.md's list of migration passes; that section now reads "How to add, order and retire a migration is documented on the class."
Promoted: Link to the code that defines a value instead of repeating the value in a doc

## 2026-10-03 · beerbox · a limit's doc comment citing the data it was sized from

Changed: `/// The longest the style may be, which leaves room above the longest style in the product data: 101 characters.` -> `/// <summary>The longest the style may be.</summary>`
Context: `ProductDetails.MaximumStyleLength = 110`, a new limit he chose after five stored styles broke the old 50. Made the same day the rule "Link to the code that defines a value instead of repeating the value in a doc" was approved, whose second sentence says to keep the reason for a value beside its definition.

## 2026-10-05 · beerbox · a page's states built from nested stacks

Said: "The page structure is needlessly nested, I think you just need the single VStack layer with elements conditionally included in that layer."
Context: `WaitForEmailVerification`; the page's VStack held one conditional child that chose between two inner VStacks, one for the waiting content and one for the resend prompt.

## 2026-10-05 · beerbox · repeated `waiting ? … : null` children

Said: "Is it possible to reduce the number of conditionals like `waiting ? ... : null`? Could the array spread operator allow us to do a single conditional on `waiting`?"
Changed: five children written `waiting ? x : null` / `waiting ? null : x` -> one `.. cond ? (VisualNode[])[…] : […]` spread inside the VStack's collection expression
Context: the same page, after it was flattened.

## 2026-10-05 · beerbox · a resource key renamed

Said: "Also rename AppResources.WaitVerificationDidntGetEmail to something like WaitVerificationLinkNotClicked"
Context: the string shown when 90 s of polling pass without the login link being followed; its text had just changed to "Didn't get an email at **EMAIL**? Check the address and your spam folder, then try again."

## 2026-10-05 · beerbox · secrets found by matching field names

Said: "I don't like this strategy for detecting secrets. It's error prone and could miss secrets or match non-secrets. Is there a better way to do this than string matching?"
Context: `BaseHandler.GetLoggableBodyAsync` replaced logged JSON values whose names contained token, credential, secret or password.

## 2026-10-05 · beerbox · a form panel pinned to the top

Changed: `.VStart()` -> `.VCenter()`
Context: the login page's translucent panel inside its ScrollView; I had pinned it to the top so the form would stay above the keyboard.

## 2026-10-05 · beerbox · similar screens each laying out their own panel

Said: "We now have a number of pages with very similar layout. … I think we should combine some of this to ensure consistency. I think maybe a reusable control that is the semi-transparent VStack with the logo on the top and a collection of nodes under it could be extracted. Maybe a reusable full-page component."
Context: Loading, OldVersion, WaitForEmailVerification, Login and the Offers empty states each built the translucent band, logo and spacing themselves; I had just added a shared style for the band only.

## 2026-10-05 · beerbox · a ScrollView on only one of the similar screens

Said: "I noticed that Login wraps it in a ScrollView, but the other pages don't. Generally that shouldn't matter, but for small devices with large fonts it could, so probably better to include that everywhere."
Context: the same screens; only Login's content could scroll.

## 2026-10-06 · beerbox · a trace ID passed to some tracking methods

Said: "The traceID still feels a beer confusing and error prone. For example, how does the TrackOrderClosingAction know it should receive a traceID, but TrackPageAppearing does not?"
Changed: `Track…(…, ActivityTraceId? traceID)` -> `Track…(…, ServiceResult result)`, reading the trace ID from the result
Context: `BeerboxTelemetryService`'s methods that report how a service request went.

## 2026-10-06 · beerbox · request outcomes tracked by the screens

Said: "should those Track* calls to Telemetry happn in DataManage directly, instead of just after the calls to DataManager?"
Context: pages called the outcome-tracking methods after each `DataManager` call; the method that sends the request now tracks it.

## 2026-10-06 · beerbox · a method named for where it runs

Said: "`UnderTrace` is not a good name. It doesn't describe what this does"
Context: a telemetry helper that ran a send inside a trace.

## 2026-10-06 · beerbox · a generic Track left reachable

Said: "There Track methods are better, but don't fully solve the problem. They still expose the generic Track method. I suggest creating a nested class that wraps TelemetryClient and has the 2 Track methods that we want."
Context: `AppInsightsTelemetryService`; the SDK client was a field, so its generic `Track` stayed callable.

## 2026-10-06 · beerbox · a field set on first use

Said: "Should we set _sender at construction so it is never null?" then "Could we use Lazy<>?"
Changed: a nullable `_sender` created in `ProcessItem` -> `Lazy<Sender>`
Context: `AppInsightsTelemetryService`; the client must be created on the background thread.

## 2026-10-06 · beerbox · a result type nested in its producer

Said: "I'm not sure it should be nested, or should OfferQuantityResult be moved out of DataManager. If it remains nested then the reference becomes DataManager.OfferQuantityResult.OfferQuantityOutcome, which is quite verbose."
Context: a result returned by `DataManager`, read by a component, and taken by the telemetry service.

## 2026-10-07 · beerbox · a local function used once

Said: "convertEntries is only called once, I don't think it needs to be a nested method"
Context: `DataManager.GetOrderFromDocumentAsync`; the local function was the converter callback passed to `Order.FromDocAsync`.
Promoted: Prefer inlining anything that is used only once

## 2026-10-07 · beerbox · a factory with one caller

Said: "Should FromDocAsync be a method at all? It is only called from DataManager, couldn't its logic also be inlined?"
Context: `Order.FromDocAsync` copied document fields into the constructor.
Promoted: Prefer inlining anything that is used only once

## 2026-10-07 · beerbox · a rename to "Create" rejected

Said: "No, don't rename. 'Create' implies creating to records/documents."
Context: I proposed renaming the private `GetOffersAsync`, which builds app offers from offer documents, to `CreateOffersAsync`.

## 2026-10-07 · beerbox · an outcome named "Result"

Said: "I think AccountResult is not a good name. It isn't clear what it is, also "result" conflicts with ServiceResult."
Changed: `AccountResult` -> `AccountChangeOutcome`
Context: the enum saying how an account change went, carried inside a `ServiceResult`.

## 2026-10-07 · beerbox · one sentence mapping shared by pages that need different ones

Said: "I think it is conflating different things. The outcome should be purely the outcome. I think the problem is because we're trying to convert an outcome into a string in a reusable way when it isn't actually reusable. For example, `CardNotRegistered` is also only valid in a subset of cases. Maybe we should get rid of AccountChangeOutcomeExtensions and the places that use it should have their own switch on the Outcome for the values they actually care about."
Context: `GetProblemText(outcome, emailAlreadyExistsText)`, which took a parameter for the one sentence that differed between pages.

## 2026-10-07 · beerbox · a property redeclared instead of inherited

Said: "I would prefer not to duplicate FieldErrors."
Changed: `AccountChangeResult<TErrors> : ServiceResult` with its own `FieldErrors` -> `: ServiceResult<NoDocument, TErrors>`
Context: the result of an account change, which has no document.

## 2026-10-07 · beerbox · an anonymous pair type

Said: "Instead of anonymous (string, string), should this be KeyValuePair or a nested record?"
Changed: `(string, string)` with `Item1`/`Item2` -> `(string Name, string Value)`
Context: telemetry event properties.

## 2026-10-07 · beerbox · an escape hatch in the class that normalizes options

Said: "I don't think calling SerializeWith here makes sense. The purpose of @src/Beerbox.Service.Contracts/Json.cs is to standardize and normalize serialization options, but this method specifically specifices its own options."
Changed: `Json.SerializeWith(value, options)` -> a named `SerializerOptions.Redacted` set with `Json.SerializeRedacted`
Context: the log formatter's own options built on the defaults.

## 2026-10-07 · beerbox · two attributes treated the same

Said: "Do we need SecretAttribute and Personal? They both end up getting erased, right?"
Context: `[Secret]` and `[PersonalData]`, which the only reader treated identically.

## 2026-10-07 · beerbox · an attribute named for one use

Said: "But "NotLogged" name the use case (logging). Maybe [Redact] or [Redacted] is better?"
Context: the single attribute replacing `[Secret]`/`[PersonalData]`.

## 2026-10-07 · beerbox · an ID encoded for a URL

Said: "I believe we don't assume a DB ID is a GUID anywhere else in code, so I'm not sure we should do it here (and Cosmos doesn't enforce IDs are GUIDs). So maybe it should just be a non-encoded string?"
Context: login and email-change links carried base64 of a pending document's ID; I had proposed a `LinkToken(Guid)` type.

## 2026-10-07 · beerbox · ID strings passed where the objects were at hand

Said: "`OfAccount` and `OfOrder` should take Account and Order objects, not strings."
Changed: `OfOrder(order.ID, order.AccountID)` -> `OfOrder(order)`
Context: `IEmailService.Addressee` factories.

## 2026-10-07 · beerbox · internals opened for a test

Said: "Why was this added? I believe we've had difficulties doing this in the past."
Changed: `InternalsVisibleTo` added to Service.Core for one `internal` method -> removed; the method private and tested through the emails it produces
Context: `AccountManager.CreateLink`.

## 2026-10-07 · beerbox · a doc that knows how its input arrived

Said: "This comment is bad. It shouldn't know about the mechanism (a link)."
Context: `IAccountManager.VerifyEmailChangeAsync`'s doc spoke of the link the ID came from.

## 2026-10-07 · beerbox · a query whose shape needs explaining

Said: "I think this should be extracted into a helper function with a comment explaining why the query is its shape"
Context: two inline `ListItemsAsync(d => d.ID == id).SingleOrDefault()` lookups used instead of a read by ID.

## 2026-10-07 · beerbox · a class named for its caller

Said: "I think RequestLogFormatter is poorly named. Its code doesn't seem to have anything to do with a "request". And "body" should probably be "jsonString" or similar."
Context: a static class with a wrapper over `Json.SerializeRedacted` and a method describing the shape of any text.

## 2026-10-07 · beerbox · names that suggest something else

Said: "LinkAttempt is a weird name." / "I would expect state to be the actual state, like "logged in", or "pending"" / "I don't think "link" is clear either, because it implies a clickable link, not a conection." / ""connect" … implies an active connection."
Changed: `LinkAttempt` -> `Nonce`; Untappd "link" -> "connect" -> "access"
Context: the Untappd OAuth nonce and the name for an account's Untappd relationship.

## 2026-10-07 · beerbox · a comment that named neither its subject nor its link to the code

Said: "What does this comment mean? 3.1.2 of which SDK? what is fixed upstream? Where? How is that comment related to the if check under it?"
Changed: `// SDK 3.1.2 throws from every Track call on a client that has no connection string. Fixed upstream, not yet released.` -> a comment naming the package, and saying that `CreateClient` disables telemetry on such a client, which is what the `if` checks
Context: a guard `if (_client.IsEnabled())` around sending telemetry.

## 2026-10-07 · beerbox · a redesign for a one-case workaround

Said: "You made connection string nullable and many other changes just for this test/mock situation. There should be a better way to accomplish this."
Changed: a nullable connection string through the configuration interface, three test doubles and a visibility change -> one guard in the one method that calls the SDK
Context: an SDK crash that only happens with no connection string, which only the development environments have.

## 2026-10-07 · beerbox · a comment that cited a library version

Said: "It seems to say that 3.1.2 has a problem, but it isn't clear if that problem affects us or is something in the future we need to worry about, or even is an old version that we've moved past."
Changed: `… Microsoft.ApplicationInsights 3.1.2 throws a NullReferenceException from Track.` -> `Don't send through such a client: the Application Insights SDK will throw a NullReferenceException.`
Context: the same guard comment; he wrote the final wording himself.

## 2026-10-08 · beerbox · a rule that checked one of two fields that arrive together

Said: "You said enable save only if the record has a brand or last four digits. Shouldn't both brand AND 4 digits be required?"
Context: the payment page's rule for enabling Save; the brand and the digits of a card record are always written together.

## 2026-10-08 · beerbox · a message for a case the change beside it removes

Said: "When would this be shown? If we do fix #1, then how could an attempt to save a card when one is not registered ever happen?"
Context: I proposed a new error text for a refused Save in the same breath as a rule that stops Save being pressed in that case.

## 2026-10-08 · beerbox · an error text that contradicts the screen it appears on

Said: "The error message seems weird. The fact that it shows card info implies it already is registered, or where did that info come from? Maybe the word "register" is not clear."
Context: "Your card isn't registered. Please register a card." shown beside a displayed card and a "replace card" button.

## 2026-10-08 · beerbox · a fallback that used a card nobody confirmed

Said: "We should only charge the card they registered, not some other card that happens to be there."
Changed: charge the customer's default card, else their most recently added card -> the default card only
Context: the Stripe charge and the card lookup the payment page uses.

## 2026-10-08 · beerbox · a flag and a value that had to agree

Said: "The check for \_cardDescription being null went away, what happens if it is null?"
Changed: `HasRegisteredCard(bool)` + `CardDescription(string?)` -> one `RegisteredCard(string? description)`
Context: the payment input component; I proposed the merge after he asked the question.

## 2026-10-08 · beerbox · a method left with one caller by a refactor

Said: "CompleteCardSetupAsync is the only caller of SetPaymentMethodAsync (plus a test). Should they be combined?"
Changed: a public static `SetPaymentMethodAsync` -> deleted; its one caller uses the existing private `WriteAccountAsync`
Context: the method had two callers until this change moved the other one to a shared refresh.
Promoted: Prefer inlining anything that is used only once

## 2026-10-08 · beerbox · an activity indicator aligned to the start

Changed: `ActivityIndicator()….HStart()` -> `.HCenter()`
Context: the indicator shown in the payment input while the card is being confirmed; he edited it between turns.
