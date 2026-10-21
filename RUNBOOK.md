# Runbook: Deja Vu Tacos demo

This runbook is for presenting the demo. The [README](README.md) explains what it is and how it's built.

## What it shows

A customer orders tacos in a phone app. The order holds the payment, sends the order to the store's kitchen, waits for the kitchen to mark it ready, and only then takes the money. Partway through, the store loses its network connection. The same order runs two ways: as direct service calls (**Traditional**) and as a Temporal Workflow (**Temporal**).

The demo makes three problems visible:

- **The store goes offline after the payment hold is placed.** With direct calls, the order fails, the hold stays on the customer's card, and nobody finds out. With a Workflow, the store call retries until the cable is plugged back in, and the order goes through. Beats 1, 2 and 5 show it.
- **The store never comes back.** The Workflow stops retrying and releases the payment hold on its own. Beat 4 shows it.
- **The money should move only after the kitchen confirms the order.** The Workflow waits for the kitchen's "ready" Signal before it captures the payment. Beat 3 shows it.

On tmprl-demo.cloud you can also crash or stop the Workers in the middle of an order and watch it carry on when they're back (beat 6).

## Before you start

- Install the [Temporal CLI](https://docs.temporal.io/cli). With Docker installed, `scripts/start.sh` runs the API, Worker and app in containers. Without Docker, it needs [uv](https://docs.astral.sh/uv/) and [Node.js](https://nodejs.org/), plus the toolchain for the Worker language you pick (the README's [Running Locally](README.md#running-locally) section lists them).
- Clone the repo. The first start builds the containers, or installs the dependencies.
- Locally, the demo uses ports 7233 and 8233 (Temporal), 8000 (API) and 5173 (app), and the `default` Namespace on the dev server. `scripts/start.sh` always starts a local dev server; it has no Temporal Cloud option.

### On tmprl-demo.cloud

The catalog runs this demo at https://dejavu-tacos.tmprl-demo.cloud on a Temporal Cloud Namespace, with the Python Worker. The Temporal links in the app open that Namespace in Temporal Cloud.

The shared deployment keeps the **Demo Settings** in the API's memory, so changing them resets the store connection and orders for everyone using it at that moment. For a run of your own, use a crashable environment:

1. On the demo's page in the catalog, under **Crashable environment**, press **Create crashable environment** and wait for it to finish creating.
2. Press **Open environment**. It's a private copy of the demo with its own Task Queue and Workers, and it deletes itself after 1 hour unless you press **Extend +1h**.
3. The same panel has **Crash workers**, **Scale down workers** and, after scaling down, **Scale up workers**. Beat 6 uses them.

## Start it

From the repository root:

```bash
./scripts/start.sh            # Python Worker (default)
./scripts/start.sh go         # or: typescript, java, dotnet
```

The script prints a banner with the URLs when everything is up. Press `Ctrl-C` in that terminal to stop everything.

Open http://localhost:5173. A healthy start shows the **Customer App** tab with the phone on the **Déjà Vu Tacos** menu, an **⚡ Temporal** pill next to the gear icon, a **Behind the Scenes** panel, and **The Actual Code** panel on the right with the Workflow file for the running Worker's language. The Temporal UI is at http://localhost:8233.

The **⚡ Temporal** pill opens the Namespace in the Temporal UI. After you place an order, the **⚡ Temporal Workflow** badge at the top of **Behind the Scenes** opens that order's execution.

## The walkthrough

To place an order on the phone: press **+** next to one or more items, press **View Cart**, then **Proceed to Checkout**, then **Place Order**. The phone switches to **Order Status**. When an order is done, **Order Again** takes you back to the menu.

The demo starts in **Temporal** mode with the **Store Connectivity Failure** scenario, and each new page load starts with the store's cable unplugged. Leave **Presentation Mode** on **Detailed** (the default) to see attempt counts and error messages.

### 1. The store goes offline mid-order

1. Place an order.
2. **Validate Order**, **Validate Store**, **Authorize Payment** and **Clear Cart** turn green in **Behind the Scenes**. **Submit to Store** turns yellow with "Store connectivity lost — Bob accidentally unplugged the ethernet" and counts up **Attempt 2/10**, **3/10**, with a longer pause each time.
3. The phone says "Placing your order..." and "This may take a moment longer than usual."
4. Say: "The payment hold is already on the card, and now the store's system is unreachable. The customer sees a short delay. Behind it, Temporal retries the call to the store."

### 2. Plug the store back in

1. Switch to the **Kitchen Display** tab. It shows **Connection Lost!** and "Uh oh... Bob accidentally unplugged the ethernet again!"
2. Press **Plug It Back In**. The kitchen display comes online and, within a few seconds, the order appears as a card marked **PREPARING**.
3. Back on **Customer App**, **Submit to Store** is green and the phone says "The store has your order!".
4. Say: "Nobody restarted anything or replayed a message. The next retry found the store online and went through. The Workflow carried on from the step it was on."
5. For proof, click **⚡ Temporal Workflow**. While it was retrying, the pending Activity showed the attempt count and last error. Once it succeeds, the history records the Activity once, with the attempt number that succeeded.

The order card shows a total of $0.00 when the Worker runs as its own process, which is how `scripts/start.sh` and tmprl-demo.cloud run it. The phone and **Behind the Scenes** show the real total.

### 3. The kitchen marks it ready, and only then is the card charged

1. On **Kitchen Display**, press **Mark Ready** on the order card. It changes to **READY** and **✓ Done**.
2. On **Customer App**, **Await Order Ready** and then **Capture Payment** turn green, and the phone shows **Order Ready!**.
3. Say: "The Workflow was waiting for the kitchen. **Mark Ready** sent it a Signal, and only then did it capture the payment. Until that point the customer had a hold, not a charge."
4. For proof, find the `WorkflowExecutionSignaled` event for `order_ready` in the history, followed by the capture.

### 4. The store never comes back, so the hold is released

1. Press **Order Again**. On **Kitchen Display**, press **Unplug the Cable Again** (or reload the page, which starts unplugged).
2. Place an order and don't plug the cable back in.
3. **Submit to Store** retries up to **Attempt 10/10**, which took about a minute locally. Then a **Compensation (Saga)** branch appears with **Release Payment Hold**, the banner reads "↩ Payment released — customer notified", and the phone shows **Order Couldn't Be Processed**: "Your payment hold has been released. No charges were made."
4. Say: "The store retry has a limit. When it runs out, the Workflow undoes the step that touched the customer's money. That undo was registered before the payment hold was placed, so it runs even if the hold call itself had failed halfway."
5. For proof, open the execution. It ends **Completed**, not Failed: the Workflow catches the error, compensates, and returns `success: false`. The history shows `release_payment_hold_activity` after the last store attempt.

To cut the wait, start this beat while you talk through beat 3.

### 5. The same failure, with direct calls

1. Click the gear icon (**Demo Settings**) and pick **Traditional (Direct Calls)** under **Architecture Mode**. The pill changes to **🔗 Traditional**. Changing settings resets the store to unplugged.
2. Place an order.
3. **Submit to Store** fails on the first try and shows **No automatic recovery!**: "Payment hold was placed but the store never received the order." The phone shows **Something Went Wrong**: "Please contact support. Your payment may have been charged." Nothing appears on **Kitchen Display**.
4. Say: "Same services, same outage. The order code called each service directly. When the store call failed, the code stopped, the hold stayed on the card, and the only way to find it is a support ticket."
5. The right-hand panel is now **Evolution of "Simple" Architecture**. Press `→` (or `↓`) to step through its 8 stages, from direct HTTP calls through queues, retry logic, a dead letter queue, state management, monitoring and compensation, to "A Rube Goldberg machine." `←` and `↑` go back.
6. Say: "This is what teams build to get back what the Workflow gave us: retries, state, and the undo step. Every box is something to run and keep working."
7. Switch back to **Temporal** in **Demo Settings** before the next beat.

### 6. Crash the Workers mid-order (tmprl-demo.cloud)

This beat uses a crashable environment; see [On tmprl-demo.cloud](#on-tmprl-democloud).

1. In your environment, place an order and leave the cable unplugged so it's retrying.
2. In the catalog's **Crashable environment** panel, press **Crash workers**. Wait for it to report **Running** again.
3. Back in the app, plug the cable in on **Kitchen Display**. The order goes through, and **Mark Ready** finishes it.
4. Say: "We just killed the processes running this order. The Workflow's state lives in Temporal, not in the Worker, so the replacement Worker picked it up at the store step."
5. For a stronger version, press **Scale down workers** first. The panel shows **Workers scaled to zero**. Place an order: the phone says "Placing your order..." and nothing in **Behind the Scenes** moves. Press **Scale up workers**, and the order starts as soon as a Worker is polling.
6. Say: "The app accepted the order with no Workers running at all. Temporal held it until there was something to run it."

Locally, the same thing works by stopping and starting the Worker: `docker compose restart worker-python` with Docker, or `Ctrl-C` on the Worker process if you started the services one by one.

### Other scenarios and settings

- **Payment Service Error:** **Authorize Payment** fails three times and the order ends with **We Couldn't Complete Your Order**. No hold was placed, so there's nothing to release and no **Compensation (Saga)** branch appears.
- **Random Chaos:** any step fails 30% of the time. The outcome changes from run to run, so don't rely on it for a specific beat.
- **None (Happy Path):** the store is online from the start; the order goes straight to the kitchen.
- **Worker Language** only changes which file **The Actual Code** shows. It doesn't change the Worker that runs. To run another language locally, pass it to `scripts/start.sh`. tmprl-demo.cloud runs the Python Worker.

**The Actual Code** shows a shortened version of the Workflow, not the full file. Use [Show it in code](#show-it-in-code) for the real lines.

## Show it in code

All links point at commit [`a64d27b`](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/commit/a64d27b00be3b29f26dd587957de0ae21d755f09). Open a file locally with `code -g <path>:<line>`.

The entries use the Python Worker, which is what tmprl-demo.cloud runs. The other four languages implement the same Workflow with the same Activity names, Task Queue and Signal.

### Retry the store until it comes back

**When to use it:** after beat 2, or when someone asks what it took to survive the outage.

**Where it is:** [`workflows/python/src/dejavu_workflows/order_workflow.py` lines 103–118](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/workflows/python/src/dejavu_workflows/order_workflow.py#L103-L118) (`code -g workflows/python/src/dejavu_workflows/order_workflow.py:103`)

```python
# Step 5: Submit to Store
# This is the demo failure point — generous retry policy so the
# presenter can show retries happening in the UI, then "plug in"
# the store to let the next retry succeed.
self._status = "submitting_to_store"
await workflow.execute_activity(
    submit_to_store_activity,
    order_input,
    start_to_close_timeout=timedelta(seconds=10),
    retry_policy=RetryPolicy(
        initial_interval=timedelta(seconds=2),
        backoff_coefficient=1.5,
        maximum_interval=timedelta(seconds=8),
        maximum_attempts=10,
    ),
)
```

**What you need to understand:**

- The first retry waits 2 seconds, and each wait is 1.5 times the last, up to 8 seconds. Ten attempts in all, which is why beat 4 takes about a minute.
- The 10-second `start_to_close_timeout` limits one attempt. If a Worker dies during an attempt, Temporal retries the attempt after that timeout.
- Temporal schedules each retry on the server. The Worker holds nothing while it waits, and a Worker restart doesn't reset the count.
- The limit is a business choice. A real kitchen outage could last longer than a minute; a customer would set the limit to how long they're willing to hold an order, or use a `schedule_to_close_timeout` to cap the total time.
- Every other step uses `DEFAULT_RETRY = RetryPolicy(maximum_attempts=3)` ([line 21](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/workflows/python/src/dejavu_workflows/order_workflow.py#L21)), which is why **Payment Service Error** gives up after three tries.
- The "/10" in the UI is a label the Activity sends; it matches this policy but doesn't come from it.

**What the customer doesn't write:** a retry loop, a queue for orders the store couldn't take, a dead letter queue, or a job to resubmit them. Those are the boxes in beat 5's diagram.

**Docs:** [Retry Policies](https://docs.temporal.io/encyclopedia/retry-policies), [Python: Activity timeouts and retries](https://docs.temporal.io/develop/python/activities/timeouts)

### Register the undo step before placing the hold

**When to use it:** during beat 4, or when someone asks what happens if the payment call fails halfway.

**Where it is:** [`workflows/python/src/dejavu_workflows/order_workflow.py` lines 74–92](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/workflows/python/src/dejavu_workflows/order_workflow.py#L74-L92) (`code -g workflows/python/src/dejavu_workflows/order_workflow.py:74`)

```python
# Step 3: Authorize Payment (hold, not capture)
# Register compensation BEFORE executing — the activity may
# succeed but the ack could be lost, so we need the compensation
# on the stack before the side-effect occurs. The compensation
# must handle the case where the hold was never actually placed.
self._status = "authorizing_payment"
compensations.append(("release_payment", {**order_input}))
payment_result = await workflow.execute_activity(
    authorize_payment_activity,
    order_input,
    start_to_close_timeout=timedelta(seconds=15),
    retry_policy=DEFAULT_RETRY,
)
authorization_id = payment_result["authorization_id"]
# Update compensation input with the actual authorization_id
compensations[-1] = (
    "release_payment",
    {**order_input, "authorization_id": authorization_id},
)
```

**What you need to understand:**

- The Workflow keeps its own list of undo steps. It adds "release the payment" before it asks for the hold, then fills in the authorization ID once it has one.
- Registering first covers the case where the payment provider placed the hold but the response never came back. The release still runs.
- That means the release has to cope with a hold that doesn't exist. In this demo it returns early when there's no authorization ID ([`activities.py` lines 235–244](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/workflows/python/src/dejavu_workflows/activities.py#L235-L244)), which is why **Payment Service Error** shows no release step. A real release would look the hold up by order ID.
- The list is ordinary Workflow state. Temporal rebuilds it by replaying history, so it survives a Worker crash.

**Docs:** [Saga pattern](https://docs.temporal.io/design-patterns/saga-pattern), [Python: Implement the Saga pattern](https://docs.temporal.io/develop/python/best-practices/error-handling#implement-saga-pattern)

### Release the hold when the order can't finish

**When to use it:** right after the previous entry, at the end of beat 4.

**Where it is:** [`workflows/python/src/dejavu_workflows/order_workflow.py` lines 143–174](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/workflows/python/src/dejavu_workflows/order_workflow.py#L143-L174) (`code -g workflows/python/src/dejavu_workflows/order_workflow.py:143`)

```python
except Exception as e:
    workflow.logger.error(f"Order {order_id} failed: {e}")
    self._status = "compensating"

    # Run compensations in reverse order (saga pattern)
    # asyncio.shield ensures compensations run even if workflow is cancelled
    async def run_compensations():
        for comp_name, comp_input in reversed(compensations):
            if comp_name == "release_payment":
                try:
                    await workflow.execute_activity(
                        release_payment_hold_activity,
                        comp_input,
                        start_to_close_timeout=timedelta(seconds=10),
                        retry_policy=RetryPolicy(maximum_attempts=5),
                    )
                except Exception as comp_err:
                    workflow.logger.error(
                        f"Compensation failed for {comp_name}: {comp_err}"
                    )

    await asyncio.shield(asyncio.ensure_future(run_compensations()))

    # ...
    return {"success": False, "error": str(e), "order_id": order_id}
```

**What you need to understand:**

- When the store's tenth attempt fails, `execute_activity` raises, and every step after it is skipped. The `except` runs the undo steps newest first. Here there's one: release the hold.
- The Workflow then notifies the customer and returns `success: False` instead of raising. That's why the execution shows **Completed** in beat 4. Re-raising an `ApplicationError` instead would mark it **Failed**, which is easier to search for.
- The release retries up to 5 times. If it still fails, the Workflow logs it and carries on, and the hold stays on the card. A production version would retry the release without a limit, or alert someone.
- The demo doesn't cancel orders, so the cancellation case in the code comment isn't something you can show.

**What the customer doesn't write:** a table of half-finished orders, or a sweeper job that finds holds with no order behind them.

**Docs:** [Saga pattern](https://docs.temporal.io/design-patterns/saga-pattern)

### Capture the payment only after the kitchen says ready

**When to use it:** during beat 3, or when someone asks how the Workflow waits for a person.

**Where it is:** the Signal handler at [`workflows/python/src/dejavu_workflows/order_workflow.py` lines 36–39](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/workflows/python/src/dejavu_workflows/order_workflow.py#L36-L39) and the wait at [lines 120–131](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/workflows/python/src/dejavu_workflows/order_workflow.py#L120-L131) (`code -g workflows/python/src/dejavu_workflows/order_workflow.py:120`). **Mark Ready** sends the Signal from [`backend/src/dejavu_tacos/api/routes.py` lines 234–238](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/backend/src/dejavu_tacos/api/routes.py#L234-L238).

```python
@workflow.signal
def order_ready(self) -> None:
    """Signal sent from KDS when the store marks the order as ready."""
    self._order_ready = True
```

```python
# Step 6: Wait for order ready signal (human in the loop)
self._status = "preparing"
await workflow.wait_condition(lambda: self._order_ready)

# Step 7: Capture Payment (only after store confirms)
self._status = "capturing_payment"
await workflow.execute_activity(
    capture_payment_activity,
    {**order_input, "authorization_id": authorization_id},
    start_to_close_timeout=timedelta(seconds=15),
    retry_policy=DEFAULT_RETRY,
)
```

```python
if config.settings.mode == ArchitectureMode.TEMPORAL:
    # Send signal to the Temporal workflow (by string name)
    client = await get_temporal_client()
    handle = client.get_workflow_handle(f"order-{order_id}")
    await handle.signal("order_ready")
```

**What you need to understand:**

- `wait_condition` blocks until the Signal handler sets the flag. The Workflow uses no Worker resources while it waits, and the wait survives Worker restarts.
- The API sends the Signal by Workflow ID and Signal name, so it doesn't import the Workflow code. That's how one API works with Workers in five languages.
- There's no timeout on this wait. An order the kitchen never marks ready stays open. A production version would pass `timeout=` and release the hold when it expires.
- The `get_status` Query on the same class returns the current step, if someone asks how another service would read the order's state.

**Docs:** [Python: Signals and Queries](https://docs.temporal.io/develop/python/message-passing), [Approval pattern](https://docs.temporal.io/design-patterns/approval)

### The Traditional mode code

**When to use it:** during beat 5, or when someone asks whether the comparison is fair.

**Where it is:** [`backend/src/dejavu_tacos/traditional/handler.py` lines 89–113](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/backend/src/dejavu_tacos/traditional/handler.py#L89-L113) (`code -g backend/src/dejavu_tacos/traditional/handler.py:89`)

```python
# Step 5: Submit to Store — THIS IS WHERE IT BREAKS
# Note: payment hold is already placed. If this fails, the customer
# has been charged but the store never gets the order. No compensation.
await emit_event(order_id, "submit_to_store", StepStatus.RUNNING, mode=MODE)
try:
    store_result = await submit_to_store(order_id, items)
    # ...
except Exception as e:
    await emit_event(
        # ...
    )
    return {"success": False, "error": str(e), "failed_step": "submit_to_store"}
```

**What you need to understand:**

- Traditional mode calls the same service functions as the Activities, in the same order, from a background task in the API process. The only difference is that nothing retries, remembers progress, or undoes.
- Adding a retry loop here would help with short outages, but the loop's state lives in the API process. A restart in the middle loses the order. That's the point of beat 5's diagram.

### Same Workflow, other SDKs

**When to use it:** when someone asks what this looks like in their language.

**Where it is:** [`workflows/java/src/main/java/dejavu/OrderWorkflowImpl.java` lines 87–90](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/workflows/java/src/main/java/dejavu/OrderWorkflowImpl.java#L87-L90) and [lines 108–124](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/workflows/java/src/main/java/dejavu/OrderWorkflowImpl.java#L108-L124) (`code -g workflows/java/src/main/java/dejavu/OrderWorkflowImpl.java:87`)

```java
Saga saga = new Saga(new Saga.Options.Builder()
        .setParallelCompensation(false)
        .setContinueWithError(true)
        .build());
```

```java
saga.addCompensation(() -> {
    // ...
    compActs.releasePaymentHold(compInput);
});
```

**What you need to understand:**

- The Java SDK has a `Saga` class that keeps the undo list and runs it with `saga.compensate()`. Python, TypeScript and .NET keep a list by hand, as in the entries above. Go uses a small `Compensations` type in [`workflows/go/workflows/compensation.go`](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/workflows/go/workflows/compensation.go#L12-L49).
- All five register the release before the payment hold and use the same retry settings for the store call.
- To show another language's code live, start that Worker locally with `./scripts/start.sh <language>`.

**Docs:** [Saga pattern: implementation](https://docs.temporal.io/design-patterns/saga-pattern#implementation)

### What's demo-only

When someone asks what they'd change for their own system, these are the parts to leave out:

- **The failures are switches.** [`backend/src/dejavu_tacos/config.py` lines 94–107](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/backend/src/dejavu_tacos/config.py#L94-L107) decides which step fails from the scenario and the browser session's cable state, and [`backend/src/dejavu_tacos/services/store.py` lines 66–72](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/backend/src/dejavu_tacos/services/store.py#L66-L72) raises the error. A new session starts unplugged ([`config.py` lines 34–37](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/backend/src/dejavu_tacos/config.py#L34-L37)).
- **The services are mocks.** Payment, cart and store are functions that sleep and return made-up IDs. The Worker asks the API over HTTP whether to fail.
- **The UI events.** [`workflows/python/src/dejavu_workflows/activities.py` lines 23–71](https://github.com/temporal-sa/temporal-demo-dejavu-tacos/blob/a64d27b00be3b29f26dd587957de0ae21d755f09/workflows/python/src/dejavu_workflows/activities.py#L23-L71): each Activity posts its progress to the API so the screen can animate.
- **Traditional mode** exists only for the comparison.

## Reset and tear down

- **Between orders:** press **Order Again**. To unplug the store again, press **Unplug the Cable Again** on **Kitchen Display**, or reload the page.
- **Between beats:** changing anything in **Demo Settings** clears the API's orders and store state and unplugs the store. Running Workflows keep running.
- **Close what's still open:** an order the kitchen never marked ready stays **Running**, waiting for the Signal. Finish it with **Mark Ready**, or terminate it with `temporal workflow terminate --workflow-id order-<id>` or from the Temporal UI. A retrying order ends on its own when the retries run out.
- **On tmprl-demo.cloud:** press **Delete** in the **Crashable environment** panel when you're done, or let it delete itself after the hour.
- **To stop locally:** press `Ctrl-C` in the `scripts/start.sh` terminal. It stops the containers or local processes and the dev server.

## When something goes wrong

| What you see | Likely cause | Fix |
|---|---|---|
| The phone stays on "Placing your order..." and nothing moves in **Behind the Scenes** | No Worker is polling the Task Queue: it's down, scaled to zero, or on a different queue | Start or scale up the Worker. The API and Worker both read `TEMPORAL_TASK_QUEUE` and default to `dejavu-tacos` |
| The first order goes straight through with no retries | The store was already plugged in for this page, or the scenario isn't **Store Connectivity Failure** | Press **Unplug the Cable Again**, or reload the page, and check **Demo Settings** |
| **Plug It Back In** worked, but the order still retries | The cable state is per page load; the order came from another tab or an earlier load | Plug in from the same tab that placed the order |
| The order never appears on **Kitchen Display** | It's still retrying, it ran out of retries, or it's a Traditional mode order that failed | Check **Behind the Scenes** |
| The screen resets in the middle of a beat on the shared deployment | Someone else changed **Demo Settings** | Use a crashable environment |
| **The Actual Code** shows a different language from the running Worker | **Worker Language** is set by hand in **Demo Settings** | Pick the Worker's language; it's only a display setting |

## Keeping this runbook current

The code links point at a fixed commit. When a linked file changes, update its link, line numbers, and snippet together. Before an event, list the linked files that have changed:

```sh
grep -oE 'blob/[0-9a-f]{40}/[^#)]+' RUNBOOK.md | sort -u | while IFS=/ read -r _ sha file; do
  git diff --quiet "$sha" HEAD -- "$file" || echo "changed since ${sha:0:7}: $file"
done
```
