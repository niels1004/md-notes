# Producibility of Data Node values
## Where is the code?
https://git.3shape.local/NielsChristensen/dentaldesktop/merge_requests/2

## API additions
We are adding a new method to `IPartialWorkflowContextTransaction` that will enable writing of a *Data Node Container* during a transaction.

```csharp
public interface IPartialWorkflowContextTransaction
{
    //...existing members ommitted

    /// <summary>
    /// Writes a data node container.
    /// </summary>
    /// <param name="id">The id of the container.</param>
    /// <param name="parameters">The container state.</param>
    void WriteDataNodeContainer(DataNodeContainerId id, DataNodeContainerWriteParameters parameters);
}
```

A Data Node Container is identified by an instance of the class `DataNodeContainerId`

```csharp
public sealed class DataNodeContainerId
{
    /// <summary>
    /// Gets the name.
    /// </summary>
    public string Name { get; }

    /// <summary>
    /// Gets the public key.
    /// </summary>
    public string PublicKey { get; }

    /// <summary>
    /// Initializes a new instance of the <see cref="DataNodeContainerId"/> class.
    /// </summary>
    public DataNodeContainerId(string name, string publicKey)
    {
        Name = name;
        PublicKey = publicKey;
    }

    //...Some members omitted
}
```

The desired content of the container is described by an instance of the `DataNodeContainerWriteParameters` class.

```csharp
/// <summary>
/// Parameters for writing a container.
/// </summary>
public sealed class DataNodeContainerWriteParameters
{
    /// <summary>
    /// Initializes a new instance of the <see cref="DataNodeContainerWriteParameters"/> class.
    /// </summary>
    public DataNodeContainerWriteParameters(
        IEnumerable<DataNodeId> dataNodesIds,
        DataNodeContainerSignature nodesSignature,
        DataNodeContainerApprovalInfo approvalInfo,
        DataNodeContainerSignature approvalSignature = null) : this(
        dataNodesIds,
        nodesSignature,
        approvalInfo,
        approvalSignature,
        null,
        null)
    {
    }

    /// <summary>
    /// Initializes a new instance of the <see cref="DataNodeContainerWriteParameters"/> class.
    /// </summary>
    public DataNodeContainerWriteParameters(
        IEnumerable<DataNodeId> dataNodesIds,
        DataNodeContainerSignature nodesSignature,
        DataNodeContainerSealInfo sealInfo,
        DataNodeContainerSignature sealSignature) : this(
        dataNodesIds,
        nodesSignature,
        null,
        null,
        sealInfo,
        sealSignature)
    {
    }

    /// <summary>
    /// Initializes a new instance of the <see cref="DataNodeContainerWriteParameters"/> class.
    /// </summary>
    public DataNodeContainerWriteParameters(
        IEnumerable<DataNodeId> dataNodesIds,
        DataNodeContainerSignature nodesSignature,
        DataNodeContainerApprovalInfo approvalInfo,
        DataNodeContainerSignature approvalSignature,
        DataNodeContainerSealInfo sealInfo,
        DataNodeContainerSignature sealSignature)
    {
        DataNodes = dataNodesIds.ToList();
        NodesSignature = nodesSignature;
        ApprovalInfo = approvalInfo;
        ApprovalSignature = approvalSignature;
        SealInfo = sealInfo;
        SealSignature = sealSignature;
    }

    /// <summary>
    /// Gets the data nodes.
    /// </summary>
    public ICollection<DataNodeId> DataNodes { get; }

    /// <summary>
    /// Gets the nodes signature.
    /// </summary>
    public DataNodeContainerSignature NodesSignature { get; }

    /// <summary>
    /// Gets the approval information.
    /// </summary>
    public DataNodeContainerApprovalInfo ApprovalInfo { get; }

    /// <summary>
    /// Gets the approval signature.
    /// </summary>
    public DataNodeContainerSignature ApprovalSignature { get; }

    /// <summary>
    /// Gets the seal information.
    /// </summary>
    public DataNodeContainerSealInfo SealInfo { get; }

    /// <summary>
    /// Gets the seal signature.
    /// </summary>
    public DataNodeContainerSignature SealSignature { get; }
}

/// <summary>
/// Container seal information.
/// </summary>
public sealed class DataNodeContainerSealInfo
{
    // TODO
}

/// <summary>
/// Container approval information.
/// </summary>
public class DataNodeContainerApprovalInfo
{
    /// <summary>
    /// Initializes a new instance of the <see cref="DataNodeContainerApprovalInfo"/> class.
    /// </summary>
    public DataNodeContainerApprovalInfo(bool isApproved = false)
    {
        IsApproved = isApproved;
    }

    /// <summary>
    /// Gets a value that determines if the container is approved.
    /// </summary>
    public bool IsApproved { get; }

    //...TODO
}

/// <summary>
/// Data node container signature.
/// </summary>
public sealed class DataNodeContainerSignature
{
    private readonly byte[] _value;

    /// <summary>
    /// Initializes a new instance of the <see cref="DataNodeContainerSignature"/> class.
    /// </summary>
    /// <param name="rawSignature"></param>
    public DataNodeContainerSignature(byte[] rawSignature)
    {
        if (rawSignature.Length != 32)
        {
            throw new ArgumentException("Expected a SHA-256 digest", nameof(rawSignature))
        }

        _value = rawSignature;
    }

    /// <summary>Returns a string that represents the current object.</summary>
    /// <returns>A string that represents the current object.</returns>
    public override string ToString()
    {
        return BitConverter.ToString(_value);
    }

    /// <summary>
    /// Gets the bytes contained in the signature.
    /// </summary>
    public byte[] ToBytes()
    {
        return (byte[])_value.Clone();
    }

    /// <summary>
    /// Gets the Base64 representation of the signature.
    /// </summary>
    public string ToBase64String()
    {
        return Convert.ToBase64String(_value);
    }
}
```

Data Nodes get two new attributes, Producibility and Frozenness, which can be queried using new methods on `IPartialWorkfowContext`. Also, a Partial Workflow can subscribe to the new event `OutputDataNodeChanged` in order to receive notifications about changes to its outputs. The primary usecase is to observe if outputs are frozen which happens if they are placed in a Data Node Container with a seal or if they are dependencies of a Data Node that is placed in such a container.

```csharp
public interface IPartialWorkflowContext
{
    //...existing members ommitted

    /// <summary>
    /// Fires when the outputs change.
    /// </summary>
    event OutputDataNodeChangedEventHandler OutputDataNodeChanged;

    /// <summary>
    /// Gets the producibility value of a data node.
    /// </summary>
    /// <returns><c>true</c> if data node value is producible.</returns>
    bool GetIsProducible(DataNodeId dataNodeId);

    /// <summary>
    /// Gets the producibility value of a data node.
    /// </summary>
    /// <returns><c>true</c> if data node value is producible.</returns>
    bool GetIsFrozen(DataNodeId dataNodeId);
}
```

```csharp
[Flags]
public enum ChangeScope
{
    None = 0,
    [Obsolete("This parameter became useless and instead of it you must use Staleness and Value enum values")]
    Content = 1 << 0,
    Progress = 1 << 1,
    Staleness = 1 << 2,
    Value = 1 << 3,

    // New members below
    Producibility = 1 << 4,
    Frozenness = 1 << 5
}
```

For backwards compatibility modules will be required to declare if they honor the producibility flags of a Data Node. Modules that do not will not be able to read values for Data Nodes that does not contain a producible value. Modules that do, can read the values for any Data Nodes in its input, but are required to only export values that are producible.

```csharp
public abstract class DentalDesktopModuleInfo : IDentalDesktopModuleInfo
{
    //...Other member ommitted
    public virtual bool HonorsProducibility => false;
}
```

# Example
The following example uses helpers from Dental Desktop's integration test framework. Invocations of methods on the `Provider` property of module instance return by the `SeamlessSwitchingModulerBuilder` actually forward to the `IPartialWorkflowContext` for the module. `WriteOutputAndState` forwards to `WriteTransaction` and `GetIsProducible` forwards to the method of the same name.

```csharp
[TestMethod]
public void Should_NotBeProducible_When_NodeIsDependentOnNodeInContainer()
{
    // Arrange
    const string Module1Id = "Module1";
    const string Module2Id = "Module2";
    const string Node1Id = "Node1";
    const string Node2Id = "Node2";
    const string ContainerId = "Container";

    var moduleForOutput = new SeamlessSwitchingModuleBuilder(TestHelper.Services)
        .WithAdditionalModuleId(Module1Id)
        .WithPartialWorkflowProviderBuilder(
            new PartialWorkflowProviderBuilder(Guid.NewGuid().ToString("N"))
                .WithOutputNodes(
                    new OutputDataNodeDescription(
                        new DataNodeId(Node1Id),
                        new DataNodeFormatVersion(1, 0))))
        .Build();

    var moduleForInput = new SeamlessSwitchingModuleBuilder(TestHelper.Services)
        .WithAdditionalModuleId(Module2Id)
        .WithPartialWorkflowProviderBuilder(
            new PartialWorkflowProviderBuilder(Guid.NewGuid().ToString("N"))
                .WithInputNodes(new InputDataNodeDescription(new DataNodeId(Node1Id), new[] { new DataNodeFormatVersion(1, 0) }))
                .WithOutputNodes(new OutputDataNodeDescription(new DataNodeId(Node2Id), new DataNodeFormatVersion(1, 0))))
        .Build();

    _modules.AddModule(moduleForOutput);
    _modules.AddModule(moduleForInput);

    _workflow.SetActiveWorkItem(_case.Case, true);
    _workflow.SetActiveModule(moduleForOutput);

    var digest = CalculateNodeIdDigest(
        new[] { new DataNodeId(Node1Id) });

    moduleForOutput.Provider.WriteOutputAndState(
        transaction =>
        {
            transaction.Write(
                new DataNodeId(Node1Id),
                new DataNodeWriteParameters(
                    new Dictionary<string, IBinaryContent> { ["Key"] = ContentHelper.StringToBinaryContent("Value") }));

            transaction.WriteDataNodeContainer(
                new DataNodeContainerId(ContainerId, ContainerPublicKey),
                new DataNodeContainerWriteParameters(
                    new[] { new DataNodeId(Node1Id) },
                    GetSignature(digest),
                    new DataNodeContainerApprovalInfo()));
        });

    _workflow.SetActiveModule(moduleForInput);

    // Act
    moduleForInput.Provider.WriteOutputAndState(
        transaction =>
        {
            transaction.Write(
                new DataNodeId(Node2Id),
                new DataNodeWriteParameters(
                    new Dictionary<string, IBinaryContent> { ["Key"] = ContentHelper.StringToBinaryContent("Value") }));
        });


    // Assert
    Assert.IsFalse(moduleForInput.Provider.GetIsProducible(Node2Id));
}
```
# Known problems
## Seal signature
Ideally the seal signature is based on the hash of node values in the container and all their dependencies. This would require that a partial workflow can read values or hashes for all these nodes. Currently, a partial workflow is restricted to read its inputs. Enabling partial workflows to read arbitrary nodes has both safety and performance implications.
## Behavior of WriteTransaction
Allowing approvals without seals leads to some questions whose answers may not be obvious.

Scenario:
1. Partial workflow writes approval container in transaction.
2. On next transaction it does not write a container.

Questions:
* Is this allowed?
* If allowed, will the container cease to exist?
* If container does not cease to exist, what will the producibility state of nodes initially included in the container be?
    - Specifically, what will the producibility state of the nodes that are output of the PW be?
    - What will the producibility state of the nodes included in the container, that are not outputs of the PW be?

Scenario:
1. PW1 writes approval container which includes output O of PW2.
2. PW2 updates value of O

Questions:
* What is the producibility of O? I guess false is the only sensible option.
* What is the producibility of the values of the other Data Nodes in the container?

The behavior of seals seems easier, but

Scenario:
Some output Data Nodes of a PW are frozen as dependencies of a Data Node that is written to a sealed container.

Question: Can the PW perform a partial update of its outputs? (Today it is required to write all its outputs in a transaction) I guess yes is the answers, but how does such an update look like in code?

## Safety
The containers proposal relies on cryptographic signatures in order to ensure data integrity. I believe the proposed implementation can protect against most accidental changes to data from outside of the module, but it is not secure in the sense that it protects the module from malicious code in 3DD or other modules.

Regarding producibility two questions come to my mind:

1. The logic for producibility is implemented in 3DD. It could fail. Does this affect the safety classification of 3DD? Can risks related to producibility be mitigated by modules?
2. 3DD can by mistake drop container info from the case which would make any nodes not frozen and producible. Could modules have a way of detecting this?
