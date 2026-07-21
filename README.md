# Workflow

Additional features for the symfony/workflow component.

## Installation

```bash
composer require codraw/workflow
```

## Reserved context keys

When the listeners of this package are registered, they subscribe to the generic
`workflow.transition` event and therefore run for every workflow and every transition of the
application. On each transition they merge the following keys into the transition context,
overwriting any value passed with the same key to `Workflow::apply()`:

- `_transitionName`: the name of the transition being applied.
- `_user`: the currently authenticated user object, if any (only when `codraw/security` is
  installed).

Note that if a workflow restricts its `events_to_dispatch` to a list that excludes `transition`,
these listeners never run and the keys are not added.
