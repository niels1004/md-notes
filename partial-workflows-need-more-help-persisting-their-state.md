# Partial workflows need more help persisting their state

## Motivation
Partial workflows (PWs) have the responsibility to request persistance of their
state. It can be quite expensive to prepare the serialized state so Triart PWs
don't persist on every change performed by the user. Instead they write the
state only when it is deemed necessary. This includes when a step is deactivated,
and in order to make the transition between steps as smooth as possible and not
block the UI for noticable periods of time the state is prepared on a
background thread before executing the `WriteTransaction` call on the main
thread. This has an inherent race condition though, as 3DD may be in the process
of navigating to a page of a module that disallows output from other modules
while it is active. This is particularly problematic when navigating to the
Order Form. In Splint Design we currently have a bug, SP-835, related to this, as
subsequent changes in the Order Form can effectively destroy the state by resulting
in a call to `SetCaseDefinition` before the state has been persisted.

There are several ways to address this. I propose something like this:

Make it possible for a PW to ensure writability by requesting some token from
3DD. When any PW has obtained such a token, 3DD should ensure that any valid
call to `WriteTransaction` will succeed until the token has been disposed. E.g. by
postponing navigation to Order Form or other module that may be blocking case
changes. 3DD should also guarantee that `SetCaseDefinition` will not be called
for PWs which holds a token.

## API Proposal

```cs
class WriteGuaranteeToken : IDisposable
{
    // Details omitted
}

interface IPartialWorkflowContext
{
    // Most existing members omitted...

    // This new method gets the token. It may throw if CanWrite is false.
    WriteGuaranteeToken EnsureWritability();

    // Included for reference
    void WriteTransaction(Action<IPartialWorkflowContextTransaction> write);
}
```

## Example

```cs
// Represents serialized data that is ready to write in a transaction
class PreparedData { }

Task<PreparedData> PrepareDataAsync()
{
    // Produces serialized data ready to be used in the WriteTransaction call
}

void WriteData(IPartialWorkflowContextTransaction t, PrepareData d)
{
    // Writes the prepared data to the the provided transaction instance
}

// The actual example:
async Task DoWriteAsync()
{
    using (var token = context.EnsureWritability())
    {
        var data = await PrepareDataAsync();

        context.WriteTransaction(t => WriteData(t, data));
    }
}
```

## Alternatives
I discussed the idea with Vyacheslav Mokrov. He proposed an alternative: Make it
possible for modules to write at any time, which of course would require changes
in several modules, and additionally expose an event/callback (e.g.
`CaseDefinitionChanging`) that allows PWs to persist their state when they are
about to get a new case definition to handle. This is needed in order to avoid a
race where a module takes a long time to persist its state and the user manages
to make changes in the order form before the eventual call to `WriteTransaction`.