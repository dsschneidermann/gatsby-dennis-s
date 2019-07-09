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

This is equivalent to querying and counting the amount of devices that have the `State` property set to `enabled`, but doesn't tell us anything about the devices activity. I use this operation just for health-checking the status of an IoT Hub.

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

        // ... methods implemented here
    }

    public class IothubManagerConfigReader {
        public string IothubConnectionString => "IoT Hub connection string";
    }
}
```

Lets have a method to get any number of device Twins from the IoT Hub.

```csharp
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
    var twinQuery = Registry.CreateQuery(query);

    var options = new QueryOptions {ContinuationToken = continuationToken};

    while (twinQuery.HasMoreResults && numberOfResults == -1 || twins.Count < numberOfResults)
    {
        ct.ThrowIfCancellationRequested();

        var response = await twinQuery.GetNextAsJsonAsync(options);
        options.ContinuationToken = response.ContinuationToken;

        var convert = new TwinJsonConverter();
        twins.AddRange(
            response.Select(
                    x => {
                        using (var reader = new JsonTextReader(new StringReader(x)))
                        {
                            return reader.Read()
                                ? convert.ReadJson(reader, typeof(Twin), null,
                                    JsonSerializer.CreateDefault())
                                : null;
                        }
                    }
                )
                .Where(x => x != null)
                .Cast<Twin>()
        );
    }

    return new ResultWithContinuationToken<List<Twin>>(
        twins, twinQuery.HasMoreResults ? options.ContinuationToken : null
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

Using this method with a query and count:

```csharp
var connectedQuery = "SELECT deviceId FROM devices WHERE connectionState = 'Connected'";
var twins = await GetTwinsByQueryAsync(connectedQuery, null, -1, cancellationToken);
return twins.Result.Count;
```

That works to query devices by connectionState. However, the documentation [states](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-identity-registry#device-heartbeat):

> The IoT Hub identity registry contains a field called connectionState. **Only use the connectionState field during development and debugging**. IoT solutions should not query the field at run time. For example, do not query the connectionState field to check if a device is connected before you send a cloud-to-device message or an SMS. 

So the query above is not the suggested way. Instead, the suggestion is to implement heartbeat messages (empty data messages from the devices that are sent with known regularity), and keep a map of which devices have sent data by subscribing to the message stream.

We're going to use the device registry instead and check the `lastActivityTime` property to determine which devices have sent data. Let's write a query that does that.

```csharp
var activeSinceTime = TimeSpan.FromHours(1);
var activityDateTime = now.UtcDateTime.Subtract(TimeSpan.FromHours(activeSinceTime));

var activityQuery = 
    "SELECT deviceId, lastActivityTime FROM devices " +
    $"WHERE lastActivityTime >= '{activityDateTime:yyyy-MM-ddTHH:mm:ssZ}'";
var twins = await GetTwinsByQueryAsync(activityQuery, null, -1, cancellationToken);
return twins.Result.Count;
```

This works for devices that are implemented using the IoT Hub SDK to send data.

## Azure IoT Edge device activity

IoT Edge modules have their own identities with separate twins from the parent devices. The device twin is therefore not updated with the activity from any of the modules. To account for this, we have to do some more work.

We're going to write a general method in our `IothubManager` class so that devices along with their latest activity are returned whether they are Edge devices or not, and we're going to allow the user of the method to query by specific module name or restrict the query by passing `lastActivityTime >= ...` like we did before.

```csharp
// Queries to selects only the properties we need from twins
private const string QUERY_PREFIX = "SELECT deviceId, capabilities, status, lastActivityTime FROM devices";
private const string MODULE_QUERY_PREFIX = "SELECT deviceId, moduleId, lastActivityTime FROM devices.modules";

// Default conditions that are always used to limit the result
private const string EDGE_MODULE_ACTIVE_QUERY = "lastActivityTime > '0001-01-01T00:00:00'";
private const string DEVICE_ENABLED_QUERY = "status = 'enabled'";

// Get IoT Edge devices with their latest module active time
private async Task<Dictionary<string, Twin>> GetEdgeDevicesWithLatestActiveModule(
    string moduleQuery, CancellationToken ct)
{
    var queryActiveModulesOnly = CombinedQuery(moduleQuery, EDGE_MODULE_ACTIVE_QUERY);
    var twinQuery = $"{MODULE_QUERY_PREFIX} WHERE {queryActiveModulesOnly}";

    var twins = await GetTwinsByQueryAsync(twinQuery, null, -1, ct);

    return twins.Result.GroupBy(x => x.DeviceId)
        .Select(g => g.OrderByDescending(x => x.LastActivityTime).First())
        .ToDictionary(x => x.DeviceId, x => x);
}

// From the list of given device twins, get the ones that that identify having the IoT Edge capability
private async Task<Dictionary<string, (Twin Twin, Twin LastActiveModule)>> GetEdgeDevices(
    string moduleQuery, IEnumerable<Twin> deviceTwins, CancellationToken ct)
{
    var devicesWithModuleActivity = await GetEdgeDevicesWithLatestActiveModule(moduleQuery, ct);

    return deviceTwins.Where(twin => twin.Capabilities?.IotEdge ?? false)
        .Join(
            devicesWithModuleActivity, x => x.DeviceId, x => x.Key,
            (twin, pair) => (pair.Key, Twin: twin, ModuleTwin: pair.Value)
        )
        .ToDictionary(x => x.Key, x => (x.Twin, x.ModuleTwin));
}

// Our public method to get devices, both IoT Edge and normal devices, along with their LatestActivity
public async Task<List<TwinActivity>> GetDevicesAsync(
    string deviceQuery = null, string moduleQuery = null, CancellationToken cancellationToken = default)
{
    var queryEnabledOnly = CombinedQuery(deviceQuery, DEVICE_ENABLED_QUERY);
    var twinQuery = $"{QUERY_PREFIX} WHERE {queryEnabledOnly}";

    var twins = await GetTwinsByQueryAsync(twinQuery, null, -1, cancellationToken);

    var edgeDevices = await GetEdgeDevices(moduleQuery, twins.Result, cancellationToken);

    return twins.Result.Select(
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

The method can be used like before, but can take a device query and a module query and will restrict on both, for example:

```csharp
var activeSinceTime = TimeSpan.FromHours(1);
var activityDateTime = now.UtcDateTime.Subtract(TimeSpan.FromHours(activeSinceTime));

var moduleQuery = $"lastActivityTime >= '{activityDateTime:yyyy-MM-ddTHH:mm:ssZ}'";
var deviceQuery = $"tags.environment = 'production'";
var twins = await GetDevicesAsync(moduleQuery: moduleQuery);

return twins.Where(x => x.IsEdgeDevice).Count();
```

## Caveats

Note that for situations where the queries are being called as a reaction to something, it's important to limit the rate by using a short-lived [cache](https://github.com/App-vNext/Polly#cache), as the IoT Hub has throttling on the number of queries and reads that is quite low at the starting levels.

Today for Free and S1, it is 20 queries/min per unit and 100 Twin reads/second per unit (check it [here](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-quotas-throttling)).
