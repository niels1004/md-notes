# Partial workflows need help with persisting their state

## Motivation
Partial workflows (PWs) decide when to persist their state. They have a few
options:
1. Write transaction every time internal state changes
2. Persist on certain events. E.g. step deactivated

It can be quite expensive to prepare the serialized state. In this case it is
impossible to do 1 without affecting the user experience. (Modern) Triart PWs
write the state only when it is deemed necessary. This includes when a step is
deactivating. In order to make the transition between steps as smooth as
possible and not block the UI for noticable periods of time, the state is
prepared on a background thread before executing the `WriteTransaction` call on
the main thread. This has an inherent race condition though as 3DD may be in the
process of navigating to a page of a module that disallows output from other
modules while it is active, e.g. the Order Form. In Splint Design we currently
have a bug, SP-835, related to this as subsequent changes in the Order Form can
effectively destroy the state before it has been persisted.

Unless modules that block output are going to disappear in the near future I
think Module API should be extended with methods that allow for better
cooperation between 3DD and PWs during transactions in order to address this
problem. I think what is needed is a way for the PW to express its intent to
write a transaction in the near future. In this case 3DD could hold back actions
that prevent output - such as navigation to modules that disables output from
other modules - until the intent is fullfilled.

## Proposal

Below I present two alternative API shapes That seems implementable to me.

### First form

```cs
class TransactionToken : IDisposable
{
    // Details omitted
}

interface IPartialWorkflowContext
{
    // Existing members omitted...

    TransactionToken PrepareTransaction();

    // WriteTransaction may throw a TimeoutException if the PW has spent too much time to prepare the state
    void WriteTransaction(Action<IPartialWorkflowContextTransaction> write, TransactionToken token);
}
```

### Second form

```cs
interface IPartialWorkflowContext
{
    // Existing members omitted...

    // 3DD may stop the transaction and request cancellation if the PW takes too long to prepare and write the transaction
    Task WriteTransactionAsync(Func<IPartialWorkflowContextTransaction, CancellationToken, Task> write);
}
```

## Examples

The following module-defined type and methods will be used below:

```cs
// Represent serialized data that is ready to write in a transaction
class PreparedData { }

Task<PreparedData> PrepareDataAsync()
{
    // Produces serialized data ready to be used in the WriteTransaction call
}

void WriteData(IPartialWorkflowContextTransaction t, PrepareData d)
{
    // Writes the prepared data to the the provided transaction instance
}
```

### Example using first form

```cs
using (var token = context.PrepareTransaction())
{
    var data = await PrepareDataAsync();

    try
    {
        context.WriteTransaction(t => WriteData(t, data), token);
    }
    catch (TimeoutException)
    {
        // Module did not start the transaction in time
    }
}
```

### Example using second form

```cs
// May throw an OperationCancelledException (or TimeoutException?)
await context.WriteTransactionAsync(async (t, ct) =>
{
    var data = await PrepareDataAsync(ct);

    WriteData(t, data);
});
```
