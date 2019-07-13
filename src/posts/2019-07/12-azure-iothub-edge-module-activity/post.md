---
title: Azure IoT Hub devices and IoT Edge module activity
date: 2019-07-12
path: /2019-07/azure-iothub-edge-module-activity
author: Dennis
excerpt: ''
tags: [csharp, azure-iothub, azure-iot-edge]
coverImage: cover.png
---

For the purpose of detecting how many active devices are connected to an IoT Hub, we can use the Microsoft.Azure.Devices nuget package ([docs](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.devices?view=azure-dotnet)). It gets a little more difficult when the devices connected are Azure IoT Edge SDK devices.

## Using IoT Hub device statistics

The IoT Hub SDK provides generel methods to ask for the statistics of your IoT Hub. One of these is getting the number of *enabled* devices.

```csharp
var registry = Microsoft.Azure.Devices.RegistryManager.CreateFromConnectionString("IoT Hub connection string");
var stats = await registry.GetRegistryStatisticsAsync(cancellationToken);
return stats.EnabledDeviceCount;
```

This is equivalent to querying and counting the amount of devices that have the `State` property set to `enabled`, but doesn't tell us anything about the devices activity. I usually only use this operation for health-checking the status of an IoT Hub.

## Enumerating IoT Hub devices with queries

Using the IoT Hub query methods means taking pages of up to 100 devices at a time. The class we'll be using to implement our features start like this:

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using iothubapi.Models;
using Microsoft.Azure.Devices;
using Microsoft.Azure.Devices.Shared;
using Newtonsoft.Json;

namespace iothubapi.Iothub
{
    public class IothubManager
    {
        public IothubManager(IothubManagerConfigReader configReader)
        {
            var builder = IotHubConnectionStringBuilder.Create(configReader.IothubConnectionString);
            Registry = RegistryManager.CreateFromConnectionString(builder.ToString());
        }

        public RegistryManager Registry { get; }

        // ... methods implemented here
    }

    public class IothubManagerConfigReader {
        public string IothubConnectionString => "IoT Hub connection string";
    }
}
```

Lets have a method to get any number of device Twins from the IoT Hub.

We'll use the Json returning method `GetNextAsJsonAsync` and parse the result with the built-in `TwinJsonConverter` so that we limit the amount of bytes to transfer, as twins can get large if they have update history. It means that we won't get any properties in the resulting `Twin` other than what we specified in the query.

```csharp{21,30-32}
/// <summary>
///     Get twin result by query.
/// </summary>
/// <param name="query">The query</param>
/// <param name="continuationToken">The continuationToken or null</param>
/// <param name="numberOfResults">The max result or -1</param>
/// <param name="ct">Cancellation token</param>
/// <returns></returns>
private async Task<ResultWithContinuationToken<List<Twin>>> GetTwinsByQueryAsync(
    string query, string continuationToken, int numberOfResults, CancellationToken ct)
{
    var twins = new List<Twin>();
    var twinQ = Registry.CreateQuery(query);

    var options = new QueryOptions {ContinuationToken = continuationToken};

    while (twinQ.HasMoreResults && numberOfResults == -1 || twins.Count < numberOfResults)
    {
        ct.ThrowIfCancellationRequested();

        var response = await twinQ.GetNextAsJsonAsync(options);
        options.ContinuationToken = response.ContinuationToken;

        var convert = new TwinJsonConverter();
        var jsonSer = JsonSerializer.CreateDefault();
        twins.AddRange(
            response.Select(x => {
                    using (var reader = new JsonTextReader(new StringReader(x)))
                    {
                        return reader.Read()
                            ? convert.ReadJson(reader, typeof(Twin), null, jsonSer)
                            : null;
                    }
                })
                .Where(x => x != null)
                .Cast<Twin>()
        );
    }

    return new ResultWithContinuationToken<List<Twin>>(
        twins, twinQ.HasMoreResults ? options.ContinuationToken : null
    );
}

private class ResultWithContinuationToken<T>
{
    public ResultWithContinuationToken(T queryResult, string continuationToken)
    {
        Result = queryResult;
        ContinuationToken = continuationToken;
    }

    public T Result { get; }
    public string ContinuationToken { get; }
}
```

And using this method with a query and count:

```csharp
var connectedQuery = "SELECT deviceId FROM devices WHERE connectionState = 'Connected'";
var twins = await GetTwinsByQueryAsync(connectedQuery, null, -1, cancellationToken);

// twins.Result is a list of Twin objects with only the 'DeviceId' set.
return twins.Result.Count;
```

To get the full `Twin` object, we can just replace `SELECT deviceId` with `SELECT *`.

So that works to query devices by the current connectionState. However, the [documentation states](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-identity-registry#device-heartbeat):

> The IoT Hub identity registry contains a field called connectionState. **Only use the connectionState field during development and debugging**. IoT solutions should not query the field at run time. For example, do not query the connectionState field to check if a device is connected before you send a cloud-to-device message or an SMS. 

So the query above is not the suggested way. Instead, the suggested way is to implement heartbeat messages (empty data messages from the devices that are sent with known regularity), and maintain a list of which devices are connected by subscribing to the message stream.

We're going to use the device registry instead, as it keeps track of the `lastActivityTime` property for us, and we can use it to determine which devices have sent data. Let's write the simplest query that does that.

```csharp
var activeSinceTime = TimeSpan.FromHours(1);
var activityAfter = now.UtcDateTime.Subtract(activeSinceTime);

var activityQuery = $@"
    SELECT deviceId, lastActivityTime FROM devices
    WHERE lastActivityTime >= '{activityAfter:yyyy-MM-ddTHH:mm:ssZ}'";

var twins = await GetTwinsByQueryAsync(activityQuery, null, -1, cancellationToken);
return twins.Result.Count;
```

This works for devices that are using the IoT Hub SDK to send messages.

## Azure IoT Edge device activity

IoT Edge modules have their own identities with separate twins from the parent devices. The device twin is therefore not updated with the activity from any of the modules. To account for this, we have to do some more work.

We're going to write a general method in our `IothubManager` class so that devices along with their latest activity are returned whether they are Edge devices or not.

We're also going to allow the user of the method to query by specific module name or restrict the query by passing `lastActivityTime >= ...` like we did above.

Let's get to it:

```csharp
// Queries to selects only the properties we need from twins
private const string QUERY_PREFIX =
    "SELECT deviceId, capabilities, status, lastActivityTime FROM devices";
private const string MODULE_QUERY_PREFIX =
    "SELECT deviceId, moduleId, lastActivityTime FROM devices.modules";

// Default conditions that are always used to limit the result
private const string DEVICE_ENABLED_QUERY = "status = 'enabled'";
private const string MODULE_ACTIVE_QUERY = "lastActivityTime > '0001-01-01T00:00:00Z'";

/// <summary>
///     Query enabled device and modules for activity.
/// </summary>
/// <param name="deviceQuery">
///     The "Where" clause of official IoTHub query string, without "Where".
/// </param>
/// <param name="moduleQuery">
///     The "Where" clause of a module query string, devices without hits will not be returned as
///     Edge devices.
/// </param>
/// <param name="ct">Cancellation token</param>
/// <returns>List of devices with LatestActivity</returns>
public async Task<List<TwinActivity>> GetDeviceActivityAsync(
    string deviceQuery = null, string moduleQuery = null, CancellationToken ct = default)
{
    var queryEnabledOnly = CombinedQuery(deviceQuery, DEVICE_ENABLED_QUERY);
    var dQuery = $"{QUERY_PREFIX} WHERE {queryEnabledOnly}";

    var queryActiveModulesOnly = CombinedQuery(moduleQuery, MODULE_ACTIVE_QUERY);
    var mQuery = $"{MODULE_QUERY_PREFIX} WHERE {queryActiveModulesOnly}";

    var devicesTask = GetTwinsByQueryAsync(dQuery, null, -1, ct);
    var modulesTask = GetTwinsByQueryAsync(mQuery, null, -1, ct);

    // Parallel execution
    await Task.WhenAll(devicesTask, modulesTask);
    var (devices, modules) = (await devicesTask, await modulesTask);

    var edgeDevices = GetEdgeDevices(devices.Result, modules.Result);

    // Combine results from Edge devices and normal devices.
    return devices.Result.Select(
            twin => {
                var foundEdgeModule = edgeDevices.TryGetValue(twin.DeviceId, out var edgeDevice);
                return new TwinActivity
                {
                    DeviceId = twin.DeviceId,
                    IsEdgeDevice = foundEdgeModule,
                    LatestActivity = foundEdgeModule
                        ? edgeDevice.LastActiveModule.LastActivityTime
                        : twin.LastActivityTime
                };
            }
        )
        .ToList();
}

/// <summary>
/// From the lists of given device and module twins, get the ones that that identify having
/// IoT Edge capability and return the device and the last active module for that device.
/// </summary>
private static Dictionary<string, (Twin Twin, Twin LastActiveModule)>
    GetEdgeDevices(IEnumerable<Twin> deviceTwins, IEnumerable<Twin> moduleTwins)
{
    var devicesWithModuleActivity = moduleTwins.GroupBy(x => x.DeviceId)
        .Select(g => g.OrderByDescending(x => x.LastActivityTime).First())
        .ToDictionary(x => x.DeviceId, x => x);

    return deviceTwins.Where(twin => twin.Capabilities?.IotEdge ?? false)
        .Join(devicesWithModuleActivity, x => x.DeviceId, x => x.Key,
            (twin, pair) => (pair.Key, Twin: twin, ModuleTwin: pair.Value))
        .ToDictionary(x => x.Key, x => (x.Twin, x.ModuleTwin));
}

// Helper method to combine query constraints with "AND"
private static string CombinedQuery(params string[] queries) =>
    string.Join(" AND ", queries.Where(x => !string.IsNullOrEmpty(x)).ToArray());

// Our result type
public class TwinActivity
{
    public string DeviceId { get; set; }
    public DateTime? LatestActivity { get; set; }
    public bool IsEdgeDevice { get; set; }
}
```

The `GetDeviceActivityAsync` method can take a device query and a module query and will restrict on both to find IoT Edge devices:

```csharp
var activeSinceTime = TimeSpan.FromHours(1);
var activityAfter = now.UtcDateTime.Subtract(activeSinceTime);

// Find all active modules
var moduleQuery = $"lastActivityTime >= '{activityAfter:yyyy-MM-ddTHH:mm:ssZ}'";

// Restrict devices by tag
var deviceQuery = $"tags.environment = 'production'";

// We can count IoT Edge devices that fullfil the module query:
var twins = await GetDeviceActivityAsync(deviceQuery, moduleQuery);
return twins.Where(x => x.IsEdgeDevice).Count();
```

Or use LINQ on `LatestActivity` for a full count of active devices:

```csharp
var deviceQuery = $"tags.environment = 'production'";
var twins = await GetDeviceActivityAsync(deviceQuery);
return twins.Where(x => x.LatestActivity >= activityAfter).Count();
```

## Full source

Here is the code in a single file: [gist](https://gist.github.com/dsschneidermann/a9aaa560b8e77d49d17d6cc8f45564dd)

## Caveats

Note that for situations where the queries are being called as a reaction to something, it's important to limit the rate by using a short-lived [cache](https://github.com/App-vNext/Polly#cache), as the IoT Hub has throttling on the number of queries and reads that is quite low at the starting levels.

Today for Free and S1, it is 20 queries/min per unit and 100 Twin reads/second per unit (check it [here](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-quotas-throttling)).
