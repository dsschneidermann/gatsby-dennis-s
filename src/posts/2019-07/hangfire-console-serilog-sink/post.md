---
title: Writing a Serilog Sink to dump logs into Hangfire Console
date: 2019-07-07
path: /2019-07/hangfire-console-serilog-sink
author: Dennis
excerpt: ""
tags: [csharp, aspnetcore, hangfire, serilog]
coverImage: cover.png
---

Working with a service using Hangfire ([hangfire.io](https://www.hangfire.io)) recently, I was using the fantastic Hangfire.Console package ([github](https://github.com/pieceofsummer/Hangfire.Console)) to get log output directly into the job's status page rather than getting it only in... well, the logs.

It looks like this:
![](1.png "Hangfire job execution with the Hangfire.Console package. Very nice color support.")

At first I was doing the most pragmatic "I'll just have two lines of code for every log statement" but that didn't last long as I wanted to get logs from other components rather than just from the job code itself, and it's clearly not sustainable.

I love using Serilog, so the answer for me is writing a small Serilog sink to put log entries into the console for whichever job is executing.

The general method for passing context in Serilog is with properties: the `ForContext` method can take a key and value and return a logger instance where the property is included:

```csharp
var logger = Log.ForContext("JobId", context.BackgroundJob.Id);

logger.Information("Now comes with the JobId property attached: {JobId}");
```

Writing a sink is trivial if there is a way to get a Hangfire Console for a given JobId, but there's not. Instead the Hangfire.Console package adds an extension method to write lines, used like so:

```csharp
private async Task RunAsync(PerformContext context)
{
    // Writing to the job console like this..
    context.WriteLine("My console message");
}
```

Serilog can't do this with only configuration, so we have to add a bit of code. I am going to show a couple of ways to do this.

# Flowing objects through Serilog context

One package I found related to this (and would make it very easy) is Serilog.Sinks.Map ([github](https://github.com/serilog/serilog-sinks-map)). It can take, say, a filename and write to it: ```lc.WriteTo.Map("Name", "Other", (name, wt) => wt.File($"./logs/log-{name}.txt"))```. However, it can only pull *strings* out of the Serilog properties.

That's not the fault of Serilog.Sinks.Map. Objects given as properties are converted to a string representation and we can't go back to the original reference after. This makes sense for Serilog as it has to be concerned about objects changing or getting disposed before log output happens.

The solution is to write an Enricher (`ILogEventEnricher`) and use a custom `LogEventPropertyValue` to keep track of our object reference.

## A bunch of code

Our new internal classes for Serilog to use:

```csharp
internal class PerformContextValue : LogEventPropertyValue
{
    // The context attached to this property value
    public PerformContext PerformContext { get; set; }

    /// <inheritdoc />
    public override void Render(TextWriter output, string format = null, IFormatProvider formatProvider = null)
    {
        // How the value will be rendered in Json output, etc.
        // Not important for the function of this code..
        output.Write(PerformContext.BackgroundJob.Id);
    }
}

internal class HangfireConsoleSerilogEnricher : ILogEventEnricher
{
    // The context used to enrich log events
    public PerformContext PerformContext { get; set; }

    /// <inheritdoc />
    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        // Create property value with PerformContext and put as "PerformContext"
        var prop = new LogEventProperty(
            "PerformContext", new PerformContextValue {PerformContext = PerformContext}
        );
        logEvent.AddOrUpdateProperty(prop);
    }
}
```

The Serilog sink has to be written to only handle our own log events, those that are enriched with the "PerformContext" property:

```csharp
public class HangfireConsoleSink : ILogEventSink
{
    /// <inheritdoc />
    public void Emit(LogEvent logEvent)
    {
        // Get property
        if (logEvent.Properties.TryGetValue("PerformContext", out var logEventPerformContext))
        {
            // Get the object reference from our custom property
            var performContext = (logEventPerformContext as PerformContextValue)?.PerformContext;

            // And write the line on it
            performContext?.WriteLine(GetColor(logEvent.Level), logEvent.RenderMessage());
        }

        // Some nice coloring for log levels
        ConsoleTextColor GetColor(LogEventLevel level)
        {
            switch (level)
            {
                case LogEventLevel.Fatal:
                case LogEventLevel.Error:
                    return ConsoleTextColor.Red;
                case LogEventLevel.Warning:
                    return ConsoleTextColor.Yellow;
                case LogEventLevel.Information:
                    return ConsoleTextColor.White;
                case LogEventLevel.Verbose:
                case LogEventLevel.Debug:
                    return ConsoleTextColor.Gray;
                default:
                    throw new ArgumentOutOfRangeException();
            }
        }
    }
}
```

That done, we can set up the logger configuration to write to the new sink:

```csharp
Log.Logger = new LoggerConfiguration()
    .WriteTo.Sink(new HangfireConsoleSink())
    .CreateLogger();
```

## Usage from job code

Using an extension method for converting a context to a logger:

```csharp
public static class HangfireConsoleSinkExtensions
{
    public static ILogger CreateLoggerForPerformContext<T>(this PerformContext context)
    {
        return Log.ForContext<T>()
            .ForContext(new HangfireConsoleSerilogEnricher {PerformContext = context});
    }
}
```

And use from the call site like so:

```csharp
private async Task RunAsync(PerformContext context)
{
    var logger = context.CreateLoggerForPerformContext<MyImplementation>();

    logger.Information("This goes to the job console automatically");

    // Pass the logger to components, so they can write too
    _myOtherImplementation.SomeMethodCall(log: logger);
}
```

## Result

The result's that the job will now pick up whatever is written to the logger and display it as formatted by Serilog:
![](2.png "My result.")

# Alternative Skycrane

The second way I want to show is a skycrane solution. I didn't end up using it, but it should be considered.

A 'skycrane' is the term for a solution that doesn't tie well into other pieces of code and therefore works very directly. However it usually comes with risk of not being able to support a growing project, not being easy to replace either.. And it's usually hard to isolate and test. I think this is a good example where it can fit well anyway.

The skycrane here is `AsyncLocal<T>` aka. what used to be the `LogicalCallContext` in .NET. `AsyncLocal<T>` is called so because it keeps the assigned value even when used across `async/await` code. It's the async and concurrency-safe way to have a `static` variable.

This is great for passing context to Serilog. We can make a sink that's aware of our `static AsyncLocal<T>` variable and have it get the object reference it needs from there.

```csharp
public static class AsyncPerformContext
{
    private static AsyncLocal<PerformContext> _capturedContext;
    public static PerformContext ExecutingContext
    {
        get
        {
            return _capturedContext?.Value;
        }
        set
        {
            _capturedContext = new AsyncLocal<PerformContext> {Value = value};
        }
   }
}

public class HangfireConsoleSink : ILogEventSink
{
    /// <inheritdoc />
    public void Emit(LogEvent logEvent)
    {
        // Magically get whatever is the current PerformContext and write on it
        AsyncPerformContext.ExecutingContext?
          .WriteLine(GetColor(logEvent.Level), logEvent.RenderMessage());
        
        // ... coloring omitted
    }
}
```

To use it, just set the `static AsyncLocal<T>` once when a job starts:

```csharp
private async Task RunAsync(PerformContext context)
{
    AsyncPerformContext.ExecutingContext = context;

    var logger = Log.ForContext<MyImplementation>();
    logger.Information("Every logger now goes to the current job console");
}
```

The most interesting effect is that we don't need to pass the logger instance around. Once we've set the `AsyncLocal<T>`, our sink will have the context automatically until the task ends.

## Making it a part of every job

Hangfire lets us have a way to hook into the job execution process, so we don't even need that first line of code in our job:

```csharp
public class PerformContextCaptureFilter : IJobFilter, IServerFilter
{
    private static AsyncLocal<PerformContext> _capturedContext;

    /// <inheritdoc />
    public bool AllowMultiple => false;

    /// <inheritdoc />
    public int Order { get; set; }

    public static PerformContext ExecutingContext => _capturedContext.Value;

    /// <inheritdoc />
    public void OnPerforming(PerformingContext filterContext)
    {
        _capturedContext = new AsyncLocal<PerformContext> {Value = filterContext};
    }

    /// <inheritdoc />
    public void OnPerformed(PerformedContext filterContext)
    {
        _capturedContext = null;
    }
}
```

In Hangfire config, activate the filter globally:
```csharp
services.AddHangfire(config =>
        // ...
        config.UseFilter(new PerformContextCaptureFilter());
```

## Caveat

Be aware that this method prevents us from adding stuff to our Serilog pipeline that can interfere with the immediate nature of writes, like using the package Serilog.Sinks.Async ([github](https://github.com/serilog/serilog-sinks-async)) or any of the packages that log to online services... In this case it's not a problem, but for other usages of `AsyncLocal<T>` and Serilog, keep it in mind.
