# StateleSSE.Abstractions

Core abstractions for the StateleSSE ecosystem. Defines the `ISseBackplane` interface for building custom backplane implementations.

## Installation

```bash
dotnet add package StateleSSE.Abstractions
```

## What is this?

This package provides the interface contract for SSE backplanes, enabling:

- **Dependency inversion** - Your application code depends on `ISseBackplane`, not specific implementations
- **Pluggable backends** - Swap Redis, in-memory, RabbitMQ, NATS, or custom implementations
- **Testability** - Mock `ISseBackplane` for unit tests
- **Extensibility** - Build your own backplane by implementing the interface

## ISseBackplane Interface

```csharp
public interface ISseBackplane
{
    // Subscription management
    (ChannelReader<object> reader, Guid subscriberId) Subscribe(string groupId);
    void Unsubscribe(string groupId, Guid subscriberId);

    // Publishing
    Task PublishToGroup(string groupId, object message);
    Task PublishToGroups(IEnumerable<string> groupIds, object message);
    Task PublishToAll(object message);

    // Diagnostics
    int GetLocalSubscriberCount(string groupId);
    IEnumerable<string> GetLocalGroups();
    BackplaneDiagnostics GetDiagnostics();
}
```

## Available Implementations

| Package | Backend | Use Case |
|---------|---------|----------|
| `StateleSSE.Backplane.Redis` | Redis pub/sub | Production horizontal scaling |
| `StateleSSE.Backplane.InMemory` | In-process | Single-server, development, testing |

## Creating a Custom Backplane

```csharp
using System.Threading.Channels;
using StateleSSE.Abstractions;

public class MyCustomBackplane : ISseBackplane
{
    private readonly ConcurrentDictionary<string, ConcurrentDictionary<Guid, Channel<object>>> _subscribers = new();

    public (ChannelReader<object> reader, Guid subscriberId) Subscribe(string groupId)
    {
        var channel = Channel.CreateUnbounded<object>();
        var subscriberId = Guid.NewGuid();
        var channels = _subscribers.GetOrAdd(groupId, _ => new());
        channels.TryAdd(subscriberId, channel);
        return (channel.Reader, subscriberId);
    }

    public void Unsubscribe(string groupId, Guid subscriberId)
    {
        if (_subscribers.TryGetValue(groupId, out var channels))
        {
            if (channels.TryRemove(subscriberId, out var channel))
            {
                channel.Writer.Complete();
            }
        }
    }

    public async Task PublishToGroup(string groupId, object message)
    {
        if (_subscribers.TryGetValue(groupId, out var channels))
        {
            var tasks = channels.Values.Select(ch => ch.Writer.WriteAsync(message).AsTask());
            await Task.WhenAll(tasks);
        }
    }

    // Implement remaining methods...
}
```

## Usage with StateleSSE.AspNetCore

```csharp
// Register your backplane
builder.Services.AddSingleton<ISseBackplane, MyCustomBackplane>();

// Use with extension methods
[HttpGet("events")]
public async Task StreamEvents([FromQuery] string gameId)
{
    var channel = ChannelNamingExtensions.Channel("game", gameId);
    await HttpContext.StreamSseAsync(backplane, channel);
}
```

## Diagnostics API

```csharp
public record BackplaneDiagnostics
{
    public required int TotalGroups { get; init; }
    public required int TotalLocalSubscribers { get; init; }
    public required GroupInfo[] Groups { get; init; }
}

public record GroupInfo
{
    public required string GroupId { get; init; }
    public required int LocalSubscribers { get; init; }
}
```

Example usage:

```csharp
var diagnostics = backplane.GetDiagnostics();
Console.WriteLine($"Active groups: {diagnostics.TotalGroups}");
Console.WriteLine($"Total subscribers: {diagnostics.TotalLocalSubscribers}");

foreach (var group in diagnostics.Groups)
{
    Console.WriteLine($"  {group.GroupId}: {group.LocalSubscribers} subscribers");
}
```

## Channel Naming Conventions

While not enforced by the interface, recommended channel patterns:

```csharp
// Domain-specific: "{domain}:{identifier}:{EventType}"
"game:abc123:PlayerJoinedEvent"

// Domain-scoped: "{domain}:{identifier}"
"game:abc123"

// Broadcast: "{domain}:all"
"system:all"
```

## License

MIT
