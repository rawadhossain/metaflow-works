# Metaflow Decorator Lifecycle Hooks Proposal

### Issue Link: [Issue](https://github.com/saikonen/metaflow/issues/3)

### Linked PR for this issue: [PR Link](https://github.com/rawadhossain/metaflow/pull/10)

## Exploring the current decorator lifecycle

While exploring the decorator system in **Metaflow**, I looked into how decorators are initialized and how lifecycle hooks are executed during a flow run.

Metaflow decorators expose several lifecycle hooks that allow decorators to influence execution at different stages. Some of the commonly used ones include:

- `step_init`
- `runtime_step_cli`
- `task_pre_step`
- `task_finished`

These hooks are called at different points during step execution, mainly from the runtime code paths in `task.py` and `runtime.py`.

At a high level the lifecycle looks roughly like this:

```
decorators attached to step
        ↓
_init_step_decorators()
        ↓
step_init hooks
        ↓
runtime_step_cli
        ↓
task_pre_step
        ↓
task_finished
```

Decorators are stored on each step as a list (`step.decorators`), and lifecycle hooks are executed by iterating through that list.

One important observation is that **decorator execution order is determined by the order of that list**. But that order is only implicit: it follows Python decorator application order and any late attachment, and **there was no supported way to declare that one decorator must run before another** when both run in the same stage.

Decorators can also be attached through:

- `--with` CLI options
- decorator mutators
- extensions

These can change the order in which decorators appear internally.

Because of this, **decorators cannot reliably assume that another decorator has already run within the same stage** unless they defensively scan the full list or avoid ordering-sensitive work.

Hence the issue was raised.

# Observations from existing decorators

While reviewing the existing decorator implementations, I noticed two patterns.

1. Some decorators implicitly depend on behavior from other decorators.

    For example, the `parallel` decorator has a comment explaining why it avoids relying on environment variables that may not yet be set.

2. Several decorators repeat similar logic in the same lifecycle hooks.

Two examples appeared frequently:

### i. Metadata registration

Multiple decorators construct `MetaDatum` entries and call `metadata.register_metadata(...)`

This pattern appeared in multiple places with small variations.

### ii. Compute decorator setup

Both **Batch** and **Kubernetes** decorators share very similar logic in:

- `runtime_step_cli`
- `task_pre_step`
- `task_finished`

For example:

- appending package metadata to CLI arguments
- syncing metadata from the datastore
- starting logging or monitoring sidecars
- terminating sidecars when execution finishes

Although the environments are different, the structure of the code was very similar.

### iii. State carried on `self` across hooks

Some decorators set `self.metadata`, `self.task_datastore`, or similar fields in `task_pre_step` and read them again in `task_finished`. If `task_pre_step` does not finish cleanly, `task_finished` can fail or see stale values.

# Changes I made

I experimented with changes to improve the situation while keeping **backwards compatibility**. They are listed below.

## 1. Declared ordering with `DEPENDS_ON`

To make ordering **explicit** when it matters, each `StepDecorator` defines:

```python
DEPENDS_ON = []
```

`DEPENDS_ON` is a list of **other decorators’ `name` strings** that must run **before** this decorator in every lifecycle pass that walks `step.decorators`. For a single dependency you can also use a string; it is normalized to a one-element list inside `_sort_step_decorators`.

The runtime builds a **directed graph**: an edge from provider `P` to dependent `D` means `D` listed `P` in `DEPENDS_ON`. It then runs a **topological sort** (Kahn’s algorithm). Among nodes with indegree zero, the sort picks the **smallest original index first**, so when there are no dependency edges, **order stays the same as today’s implicit source order**.

Sorting runs when decorator lists are finalized, in the same places as before:

- `_init_step_decorators`
- `_process_late_attached_decorator`

So decorators attached dynamically (for example via `--with`) go through the same ordering logic.

If the graph has a **cycle**, Metaflow raises **`MetaflowException`** with a message naming the decorators involved, instead of leaving order undefined.

**Default `DEPENDS_ON = []`:** existing decorators do not declare edges, so the topological sort produces the same order as before. Opt-in only.

### Validating ordering behavior manually

To sanity-check that dependencies affect hook order, you can use a tiny flow with two custom decorators: one provides a `name`, and the other lists that name in `DEPENDS_ON`.

Example sketch:

- Code

    ```python
    from metaflow import FlowSpec, step
    from metaflow.decorators import StepDecorator

    def step_deco(deco_cls):
        def decorator(func):
            if not hasattr(func, "is_step"):
                from metaflow.decorators import BadStepDecoratorException
                raise BadStepDecoratorException(deco_cls.name, func)

            if deco_cls.name in [deco.name for deco in func.decorators]:
                from metaflow.decorators import DuplicateStepDecoratorException
                raise DuplicateStepDecoratorException(deco_cls.name, func)
            func.decorators.append(deco_cls(attributes={}, statically_defined=True))
            return func
        return decorator

    class SetupDecorator(StepDecorator):
        name = "setup"

        def task_pre_step(
            self,
            step_name,
            task_datastore,
            metadata,
            run_id,
            task_id,
            flow,
            graph,
            retry_count,
            max_user_code_retries,
            ubf_context,
            inputs,
        ):
            print("SETUP runs first")

    class AfterSetupDecorator(StepDecorator):
        name = "after_setup"
        DEPENDS_ON = ["setup"]

        def task_pre_step(
            self,
            step_name,
            task_datastore,
            metadata,
            run_id,
            task_id,
            flow,
            graph,
            retry_count,
            max_user_code_retries,
            ubf_context,
            inputs,
        ):
            print("AFTER_SETUP runs second")

    class DependsOnExampleFlow(FlowSpec):

        @step_deco(AfterSetupDecorator)
        @step_deco(SetupDecorator)
        @step
        def start(self):
            print("STEP BODY")
            self.next(self.end)

        @step
        def end(self):
            print("END STEP")

    if __name__ == "__main__":
        DependsOnExampleFlow()
    ```

Even if the decorators are attached bottom-to-top so the raw list order would run `after_setup` before `setup`, after `_sort_step_decorators` the **`setup` hook runs before `after_setup`**, which you should see in the printed order.

**Note:** Python applies decorators from bottom to top, which is why the internal list order does not always match visual order in the source. That is unchanged; what changes is that **explicit edges override ambiguous cases** when you need a guarantee.

### What `DEPENDS_ON` is meant for

It is for **relationships between decorators on the same step** (“run after decorator X by name”), not for arbitrary global priorities. Cycles are rejected at sort time.

### Summary

This gives a **lightweight, explicit dependency model** on top of the existing list-based execution, with **no mandatory ordering** for decorators that keep the default empty `DEPENDS_ON`.

## 2. Explicit arguments to `task_finished`

The task runtime already knows `metadata`, the task datastore (`output`), `run_id`, and `task_id` when it calls the final decorator hook. I extended `StepDecorator.task_finished` so these can be passed as **keyword arguments** from `task.py`, and updated overrides (batch, kubernetes, argo, airflow, step functions, cards, etc.) to accept the same optional parameters.

The goal is to **avoid relying on `self` fields** set in `task_pre_step` for data that the runtime can supply directly at the end of the task.

## 3. Metadata abstraction

The repeated metadata registration pattern was centralized by introducing a helper on `StepDecorator`:

`_register_metadata(...)`

This helper builds `MetaDatum` entries (with `attempt_id` tags) and calls `metadata.register_metadata(...)` when there is something to register. It supports `skip_none=True` where the old code skipped `None` values.

It is now used by several decorators that previously duplicated that logic.

The goal here was mainly to **reduce duplicated boilerplate**.

## 4. Shared helpers for compute decorators

In **Batch** and **Kubernetes** decorators, many lifecycle operations were structurally identical.

To reduce duplication, helper methods were added on `StepDecorator`, including:

- `_append_package_metadata_to_cli`
- `_sync_local_metadata_from_datastore`
- `_start_log_and_spot_sidecars`
- `_terminate_sidecars`

Both decorators reuse these helpers where the behavior matches.

These helpers mainly affect logic inside:

- `runtime_step_cli`
- `task_pre_step`
- `task_finished`

The idea was to remove duplicated code without introducing a new base class.

# Testing

I added a small unit test suite `test/unit/test_decorators_helpers.py`

The tests cover:

- **`DEPENDS_ON` / `_sort_step_decorators`:** no dependencies preserves source order; simple dependency (A after B); diamond-shaped dependencies; circular dependencies raise `MetaflowException` with a clear message
- **`_register_metadata`:** registers entries with `attempt_id` tags; default behavior keeps `None` values; `skip_none=True` drops `None` keys from registered metadata

I also ran existing tests related to decorators and compute configuration.

```
python -m pytest \
test/unit/test_decorators_helpers.py \
test/unit/test_conda_decorator.py \
test/unit/test_pypi_decorator.py \
test/unit/test_compute_resource_attributes.py \
test/unit/test_aws_util.py \
test/unit/test_argo_workflows_cli.py \
test/unit/test_secrets_decorator.py \
-q
```

You can attach a screenshot of the terminal output for the PR if helpful; **35** tests passed in the last targeted run.

# My current view on the issue

From exploring the decorator system and experimenting with these changes, my current view is that:

- Decorator lifecycle hooks themselves are already well defined.
- The main gaps were **implicit ordering** when two decorators need a relative order in the same stage, and **fragile cross-hook state** when `task_finished` assumed `task_pre_step` had fully populated `self`.
- **`DEPENDS_ON`** addresses the ordering problem in a **name-based, opt-in** way, with **cycle detection** instead of silent ambiguity.
- **Passing `metadata`, `task_datastore`, `run_id`, and `task_id` into `task_finished`** makes teardown and bookkeeping hooks more robust.
- **Helpers** for metadata and for batch/kubernetes-style setup reduce duplication without changing the overall hook model.

The changes described above are intended to **implement that direction** while preserving default behavior for decorators that do not opt into dependencies.
