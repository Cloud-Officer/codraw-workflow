# Code Review: codraw/workflow

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
- **Finding 2** — `EventListener/AddTransitionNameToContextListener.php`: added a null guard on
  `getTransition()` before calling `getName()`, mirroring the null handling in
  `AddUserToContextListener`. A manually dispatched `workflow.transition` event without a
  transition no longer causes a null-method-call fatal.
- **Finding 3** — `composer.json`: added an explicit `"php": ">=8.5"` constraint to `require`
  (matching the repo convention), and added a `suggest` block for the optional integrations:
  `codraw/dependency-injection` (WorkflowIntegration) and `codraw/security`
  (AddUserToContextListener). Both stay in `require-dev` since the DI integration guards the
  Security dependency with `class_exists()` — same pattern as `codraw/log`. Also reordered
  `require-dev` alphabetically to match sibling packages. `composer validate --no-check-publish`
  passes.
- **Finding 5** — `README.md`: documented the reserved `_transitionName` / `_user` context keys,
  that the listeners run for every workflow/transition and overwrite caller-provided keys, and the
  `events_to_dispatch` caveat.

Validation (2026-07-20): `composer install` resolves cleanly, the full PHPUnit suite passes
(10 tests, 19 assertions — the 4 PHPUnit notices about mocks without expectations are
pre-existing PHPUnit 12 style notices, verified present without these changes), PHPStan
level 5 reports no errors, and markdownlint reports no violations. No test or baseline
updates were needed for the fixes above.

Not fixed (open items): Finding 1 (replacing the full `_user` object with an identifier would
change the context payload consumers may rely on — the behavior is now documented in the README);
Finding 4 (splitting `Tests/` out of the production autoload is a packaging change that could
affect consumers referencing test classes).

## Overall Assessment

`codraw/workflow` is a very small, focused companion package for `symfony/workflow`: two event
listeners that enrich the transition context (`_transitionName` and `_user`) plus a
`WorkflowIntegration` class that wires them into the shared Draw dependency-injection integration
system. The code is clean, strictly scoped, and fully unit tested, with an empty PHPStan baseline
at level 5. No security vulnerabilities were found. The findings below are edge-case robustness
issues, packaging/metadata gaps, and one design concern about placing the full user object into
the workflow context (which is frequently persisted or serialized downstream).

## Findings

### Medium

#### 1. Full `UserInterface` object is injected into the workflow context (`_user`)

- `EventListener/AddUserToContextListener.php:29-32`

The listener merges the entire user object into the transition context. In Symfony workflows the
context is passed to the marking store (`MethodMarkingStore::setMarking()`), commonly forwarded to
audit-trail listeners, loggers, and sometimes persisted alongside the marking or dispatched via
Messenger. Storing a full `UserInterface` implementation (typically a Doctrine entity that may
hold a password hash and other sensitive fields) instead of a stable identifier risks:

- serialization failures or lazy-proxy issues when the context is serialized/JSON-encoded,
- leaking sensitive user fields into logs or persisted audit data,
- stale object references if the context outlives the request.

Storing `$user->getUserIdentifier()` (optionally plus the class name or id) would be safer and
more portable. If passing the object is intentional for in-process consumers, this trade-off
should at least be documented in the README.

#### 2. **[FIXED]** `getTransition()` can be null; `getName()` would fatal on a manually dispatched event

- `EventListener/AddTransitionNameToContextListener.php:21`

`Symfony\Component\Workflow\Event\TransitionEvent` accepts a null transition (the package's own
test constructs one without a transition at
`Tests/EventListener/AddUserToContextListenerTest.php:48-51`), and the base
`Event::getTransition()` return type is `?Transition`. `addTransitionToContext()` calls
`$transitionEvent->getTransition()->getName()` unguarded. During a normal `Workflow::apply()` the
transition is always set, so this cannot trigger in the standard flow — but any code that
dispatches a `workflow.transition` event manually (tests, custom workflow implementations,
re-dispatch decorators) will hit a null-method-call fatal. A simple null guard (mirroring the null
check already done in `AddUserToContextListener`) removes the risk.

### Low

#### 3. **[FIXED]** `composer.json` has no explicit `php` requirement and no `suggest` for optional dependencies

- `composer.json:17-26`

The `require` section declares only `symfony/event-dispatcher` and `symfony/workflow`. There is no
`"php"` constraint (the code uses constructor property promotion, PHP >= 8.0; the effective floor
comes only transitively via Symfony 6.4). More importantly, `AddUserToContextListener` hard-depends
on `Draw\Component\Security\Core\Security` (`codraw/security`) and `WorkflowIntegration` on
`codraw/dependency-injection`, yet both are only `require-dev` and are not listed under `suggest`.
The DI integration guards with `class_exists(Security::class)`
(`DependencyInjection/WorkflowIntegration.php:34-36`), but a consumer wiring
`AddUserToContextListener` manually without `codraw/security` installed gets a fatal
"class not found" with no hint from the package metadata. Adding a `suggest` block (and an explicit
`php` constraint) would make the optional coupling discoverable.

#### 4. Tests ship inside the production autoload namespace

- `composer.json:29-33`

The PSR-4 mapping `"Draw\\Component\\Workflow\\": ""` at the package root means
`Tests/` is autoloadable in production installs as `Draw\Component\Workflow\Tests\...` (there is no
`autoload-dev` split and no `.gitattributes` with `export-ignore`). This bloats production installs
and exposes test classes (which reference dev-only dependencies such as `symfony/security-core`)
to production autoloading. The DI integration correctly excludes `Tests/` from service registration
(via `IntegrationTrait::getDefaultExcludedDirectories()`), so this is a packaging hygiene issue,
not a runtime bug.

#### 5. **[FIXED]** Listeners apply globally to every workflow, and silently overwrite context keys

- `EventListener/AddTransitionNameToContextListener.php:13,19-22`
- `EventListener/AddUserToContextListener.php:14,29-32`

Both listeners subscribe to the generic `workflow.transition` event, so once the integration is
enabled they fire for every workflow and every transition in the application, with no per-workflow
opt-out. Additionally, `array_merge()` will silently clobber any `_transitionName` or `_user` key a
caller passed in `Workflow::apply($subject, $name, $context)`. Both behaviors are probably
intentional for this framework, but they are invisible side effects; a short note in the README
about the reserved `_transitionName` / `_user` context keys would help consumers. Note also that
if an application restricts a workflow's `events_to_dispatch` to a list that excludes `transition`,
these listeners silently never run — worth documenting since other Draw components may rely on
`_transitionName` being present.

## Strengths

- Small, single-purpose classes with clear names; no dead code, no over-engineering.
- Null user is handled correctly in `AddUserToContextListener` (`EventListener/AddUserToContextListener.php:24-27`).
- Conditional service registration: the Security-dependent listener is removed from the container
  when `codraw/security` is absent (`DependencyInjection/WorkflowIntegration.php:34-36`), keeping
  the package usable with only `symfony/workflow` installed.
- Consistent with the framework's integration conventions (`IntegrationTrait`, `draw.workflow.`
  service-id prefix), which keeps the multi-package ecosystem uniform.
- PHPStan level 5 with an empty baseline (`phpstan-baseline.neon`) — no suppressed static-analysis
  debt.
- Tests use mocks and `uniqid()`-based data to avoid accidental coupling to fixed values.

## Test Coverage

Coverage is effectively complete for a package this size:

- `Tests/EventListener/AddTransitionNameToContextListenerTest.php` — verifies subscriber contract,
  subscribed event map, and that `_transitionName` lands in the context.
- `Tests/EventListener/AddUserToContextListenerTest.php` — verifies subscriber contract, the
  no-user early return (context untouched), and the merged `_user` entry, including that existing
  context keys are preserved.
- `Tests/DependencyInjection/WorkflowIntegrationTest.php` — exercises the integration through the
  shared `IntegrationTestCase`, asserting both renamed service definitions
  (`draw.workflow.event_listener.*`) are produced from an empty configuration.

Gaps (all minor): no test for the null-transition edge case in
`AddTransitionNameToContextListener` (finding 2); no test of the integration branch where
`Security` is absent (the `removeDefinition` path in `WorkflowIntegration.php:34-36` is untested);
no functional test with a real `Workflow` dispatch confirming the context actually reaches the
marking store.
