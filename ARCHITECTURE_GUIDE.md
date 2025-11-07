# Ocelot Architecture Guide - Based on Source Code Analysis

## Table of Contents
1. [Overview](#overview)
2. [Core Architecture](#core-architecture)
3. [Startup and Initialization](#startup-and-initialization)
4. [Middleware Pipeline](#middleware-pipeline)
5. [Configuration System](#configuration-system)
6. [Request Processing Flow](#request-processing-flow)
7. [Key Components](#key-components)
8. [Advanced Features](#advanced-features)

---

## Overview

**Ocelot is a .NET API Gateway** built on top of ASP.NET Core middleware infrastructure. It acts as a reverse proxy that sits between clients and downstream microservices, providing a unified entry point to a distributed system.

### What is an API Gateway?

An API gateway is a server that acts as an intermediary between clients and backend services. It receives client requests, routes them to appropriate backend services, aggregates responses if needed, and returns the final response to the client.

### Core Purpose

From analyzing the source code structure and core components:
- Provides a unified entry point for microservices architectures
- Runs on any platform that ASP.NET Core supports
- Handles HTTP(S) requests through a configurable middleware pipeline
- Transforms requests and responses as they flow through the gateway

---

## Core Architecture

### Foundational Design Pattern

Ocelot is built as **a series of ASP.NET Core middlewares arranged in a specific order**. This is evident from the main pipeline builder in `src/Ocelot/Middleware/OcelotPipelineExtensions.cs`:

```csharp
public static RequestDelegate BuildOcelotPipeline(this IApplicationBuilder app, 
    OcelotPipelineConfiguration configuration)
{
    // Sets up the downstream context and gets the config
    app.UseMiddleware<ConfigurationMiddleware>();
    
    // Registers to catch any global exceptions
    app.UseMiddleware<ExceptionHandlerMiddleware>();
    
    // Then we get the downstream route information
    app.UseMiddleware<DownstreamRouteFinderMiddleware>();
    
    // ... more middleware
    
    // Fires off the request and sets the response
    app.UseMiddleware<HttpRequesterMiddleware>();
    
    return app.Build();
}
```

### Key Architectural Principles

1. **Middleware Chain Pattern**: Each middleware has a specific responsibility and processes the request in order
2. **HttpContext Extension**: Ocelot uses `HttpContext.Items` dictionary to pass data between middleware
3. **Error Handling**: Each middleware can set errors that short-circuit the pipeline
4. **Extensibility**: Custom middleware can be injected at specific points in the pipeline

### Base Middleware Class

All Ocelot middleware inherit from `OcelotMiddleware` (located in `src/Ocelot/Middleware/OcelotMiddleware.cs`):

```csharp
public abstract class OcelotMiddleware
{
    protected OcelotMiddleware(IOcelotLogger logger)
    {
        Logger = logger;
        MiddlewareName = GetType().Name;
    }

    public IOcelotLogger Logger { get; }
    public string MiddlewareName { get; }
}
```

This provides:
- Consistent logging across all middleware
- Standard naming convention
- Common error handling patterns

---

## Startup and Initialization

### Adding Ocelot to the Application

From `src/Ocelot/DependencyInjection/ServiceCollectionExtensions.cs`, Ocelot is added to the service collection:

```csharp
public static IOcelotBuilder AddOcelot(this IServiceCollection services, 
    IConfiguration configuration)
{
    return new OcelotBuilder(services, configuration);
}
```

### OcelotBuilder Registration

The `OcelotBuilder` class (`src/Ocelot/DependencyInjection/OcelotBuilder.cs`) registers all necessary services:

**Configuration Services**:
```csharp
Services.TryAddSingleton<IInternalConfigurationCreator, FileInternalConfigurationCreator>();
Services.TryAddSingleton<IInternalConfigurationRepository, InMemoryInternalConfigurationRepository>();
Services.TryAddSingleton<IConfigurationCreator, ConfigurationCreator>();
```

**Route Management**:
```csharp
Services.TryAddSingleton<IRoutesCreator, StaticRoutesCreator>();
Services.TryAddSingleton<IDynamicsCreator, DynamicRoutesCreator>();
Services.TryAddSingleton<IAggregatesCreator, AggregatesCreator>();
```

**Request Processing**:
```csharp
Services.TryAddSingleton<IDownstreamPathPlaceholderReplacer, DownstreamPathPlaceholderReplacer>();
Services.TryAddSingleton<IHttpRequester, HttpRequester>();
Services.TryAddSingleton<IHttpResponder, HttpContextResponder>();
```

**Load Balancing**:
```csharp
Services.AddSingleton<ILoadBalancerCreator, NoLoadBalancerCreator>();
Services.AddSingleton<ILoadBalancerCreator, RoundRobinCreator>();
Services.AddSingleton<ILoadBalancerCreator, CookieStickySessionsCreator>();
Services.AddSingleton<ILoadBalancerCreator, LeastConnectionCreator>();
Services.TryAddSingleton<ILoadBalancerFactory, LoadBalancerFactory>();
```

### Activating Ocelot Middleware

From `src/Ocelot/Middleware/OcelotMiddlewareExtensions.cs`, the `UseOcelot` method:

```csharp
public static async Task<IApplicationBuilder> UseOcelot(
    this IApplicationBuilder builder, 
    OcelotPipelineConfiguration pipelineConfiguration)
{
    // 1. Create configuration from file system
    _ = await CreateConfiguration(builder);
    
    // 2. Configure diagnostic listener for monitoring
    ConfigureDiagnosticListener(builder);
    
    // 3. Build the Ocelot pipeline
    return CreateOcelotPipeline(builder, pipelineConfiguration);
}
```

**Configuration Loading Process**:
1. Reads `FileConfiguration` from options monitor
2. Uses `IInternalConfigurationCreator` to transform file config into internal config
3. Stores internal config in `IInternalConfigurationRepository`
4. Sets up file watching to reload configuration on changes

---

## Middleware Pipeline

### Complete Pipeline Order

From `src/Ocelot/Middleware/OcelotPipelineExtensions.cs`, the middleware pipeline executes in this order (note that it includes conditional branching for WebSockets and optional user-defined middleware):

```
1. ConfigurationMiddleware          - Sets up downstream context and configuration
2. ExceptionHandlerMiddleware       - Catches global exceptions and sets Request ID
3. WebSocket Fork                   - Branches to WebSocket-specific pipeline if needed
4. PreErrorResponderMiddleware      - User-defined error handling (optional)
5. ResponderMiddleware              - Responds to requests with errors
6. DownstreamRouteFinderMiddleware  - Finds the matching downstream route
7. MultiplexingMiddleware           - Handles request multiplexing/aggregation
8. SecurityMiddleware               - IP whitelist/blacklist security
9. MapWhenOcelotPipeline            - Custom pipeline branching (optional)
10. HttpHeadersTransformationMiddleware - Transforms headers
11. DownstreamRequestInitialiserMiddleware - Initializes downstream request
12. RateLimitingMiddleware          - Rate limiting checks
13. RequestIdMiddleware             - Adds/updates request ID
14. PreAuthenticationMiddleware     - Pre-authentication logic (optional)
15. AuthenticationMiddleware        - Authenticates the request
16. ClaimsToClaimsMiddleware        - Claims transformations
17. PreAuthorizationMiddleware      - Pre-authorization logic (optional)
18. AuthorizationMiddleware         - Authorizes the request
19. ClaimsToHeadersMiddleware       - Adds claims as headers
20. PreQueryStringBuilderMiddleware - Pre-query string logic (optional)
21. ClaimsToQueryStringMiddleware   - Adds claims to query string
22. ClaimsToDownstreamPathMiddleware - Modifies path based on claims
23. LoadBalancingMiddleware         - Selects downstream host
24. DownstreamUrlCreatorMiddleware  - Creates the final downstream URL
25. OutputCacheMiddleware           - Caching logic
26. HttpRequesterMiddleware         - Makes the actual HTTP request to downstream services (final middleware in Ocelot's processing chain)
```

Note: `HttpRequesterMiddleware` is the last middleware registered in the Ocelot pipeline. It does call `await _next.Invoke()` to continue the ASP.NET Core pipeline, but there are no more Ocelot-specific middleware after it.

### WebSocket Pipeline

For WebSocket requests, a separate pipeline is used:

```csharp
app.MapWhen(httpContext => httpContext.WebSockets.IsWebSocketRequest,
    ws =>
    {
        ws.UseMiddleware<DownstreamRouteFinderMiddleware>();
        ws.UseMiddleware<MultiplexingMiddleware>();
        ws.UseMiddleware<DownstreamRequestInitialiserMiddleware>();
        ws.UseMiddleware<LoadBalancingMiddleware>();
        ws.UseMiddleware<DownstreamUrlCreatorMiddleware>();
        ws.UseMiddleware<WebSocketsProxyMiddleware>();
    });
```

### Middleware Details

#### 1. ConfigurationMiddleware

Located in `src/Ocelot/Middleware/ConfigurationMiddleware.cs`, this middleware:
- Retrieves internal configuration from the repository
- Stores it in `HttpContext.Items` for downstream middleware access
- Short-circuits if configuration is unavailable

#### 2. DownstreamRouteFinderMiddleware

From `src/Ocelot/DownstreamRouteFinder/Middleware/DownstreamRouteFinderMiddleware.cs`:

```csharp
public async Task Invoke(HttpContext httpContext)
{
    var upstreamUrlPath = httpContext.Request.Path.ToString();
    var upstreamQueryString = httpContext.Request.QueryString.ToString();
    var internalConfiguration = httpContext.Items.IInternalConfiguration();
    var upstreamHost = hostHeader.Split(':')[0];
    var upstreamHeaders = httpContext.Request.Headers
        .ToDictionary(h => h.Key, h => string.Join(';', (IList<string>)h.Value));
    
    var provider = _factory.Get(internalConfiguration);
    var response = provider.Get(upstreamUrlPath, upstreamQueryString, 
        httpContext.Request.Method, internalConfiguration, upstreamHost, upstreamHeaders);
    
    if (response.IsError)
    {
        httpContext.Items.UpsertErrors(response.Errors);
        return;
    }
    
    httpContext.Items.UpsertTemplatePlaceholderNameAndValues(
        response.Data.TemplatePlaceholderNameAndValues);
    httpContext.Items.UpsertDownstreamRoute(response.Data);
    
    await _next.Invoke(httpContext);
}
```

**Responsibilities**:
- Extracts upstream request information (path, query, method, headers, host)
- Matches request to configured routes
- Extracts placeholder values from URL (e.g., `/api/users/{id}` → `{id: "123"}`)
- Stores matched route and placeholders in `HttpContext.Items`

#### 3. HttpRequesterMiddleware

From `src/Ocelot/Requester/Middleware/HttpRequesterMiddleware.cs`:

```csharp
public async Task Invoke(HttpContext httpContext)
{
    var response = await _requester.GetResponse(httpContext);
    CreateLogBasedOnResponse(response);
    
    if (response.IsError)
    {
        Logger.LogDebug("IHttpRequester returned an error, setting pipeline error");
        httpContext.Items.UpsertErrors(response.Errors);
        return;
    }
    
    Logger.LogDebug("Setting HTTP response message...");
    httpContext.Items.UpsertDownstreamResponse(new DownstreamResponse(response.Data));
    await _next.Invoke(httpContext);
}
```

**Responsibilities**:
- Makes the actual HTTP request to the downstream service
- Uses `IHttpRequester` to send the request
- Stores the response in `HttpContext.Items`
- This is the last Ocelot middleware in the pipeline (though it does call `_next.Invoke()` to continue to ASP.NET Core's default handlers)
- Response flows back up through the middleware chain

---

## Configuration System

### File Configuration Model

The configuration is defined in JSON format and mapped to C# models in `src/Ocelot/Configuration/File/`.

#### FileConfiguration

From `src/Ocelot/Configuration/File/FileConfiguration.cs`:

```csharp
public class FileConfiguration
{
    public FileConfiguration()
    {
        Routes = new();
        DynamicRoutes = new();
        Aggregates = new();
        GlobalConfiguration = new();
    }

    public List<FileRoute> Routes { get; set; }
    public List<FileDynamicRoute> DynamicRoutes { get; set; }
    public List<FileAggregateRoute> Aggregates { get; set; }
    public FileGlobalConfiguration GlobalConfiguration { get; set; }
}
```

#### FileRoute

From `src/Ocelot/Configuration/File/FileRoute.cs`, a route contains:

```csharp
public class FileRoute : IRouteUpstream, IRouteGrouping, IRouteRateLimiting, ICloneable
{
    // Request matching
    public string UpstreamPathTemplate { get; set; }
    public List<string> UpstreamHttpMethod { get; set; }
    public string UpstreamHost { get; set; }
    
    // Downstream configuration
    public string DownstreamPathTemplate { get; set; }
    public string DownstreamScheme { get; set; }
    public List<FileHostAndPort> DownstreamHostAndPorts { get; set; }
    public string DownstreamHttpMethod { get; set; }
    
    // Authentication & Authorization
    public FileAuthenticationOptions AuthenticationOptions { get; set; }
    public Dictionary<string, string> RouteClaimsRequirement { get; set; }
    
    // Transformations
    public Dictionary<string, string> AddHeadersToRequest { get; set; }
    public Dictionary<string, string> AddClaimsToRequest { get; set; }
    public Dictionary<string, string> AddQueriesToRequest { get; set; }
    public Dictionary<string, string> ChangeDownstreamPathTemplate { get; set; }
    public IDictionary<string, string> DownstreamHeaderTransform { get; set; }
    public IDictionary<string, string> UpstreamHeaderTransform { get; set; }
    
    // Features
    public FileQoSOptions QoSOptions { get; set; }
    public FileRateLimiting RateLimiting { get; set; }
    public FileCacheOptions CacheOptions { get; set; }
    public FileLoadBalancerOptions LoadBalancerOptions { get; set; }
    public FileSecurityOptions SecurityOptions { get; set; }
    
    // Service Discovery
    public string ServiceName { get; set; }
    public string ServiceNamespace { get; set; }
    
    // Other
    public List<string> DelegatingHandlers { get; set; }
    public int Priority { get; set; }
    public int? Timeout { get; set; }
    public Dictionary<string, string> Metadata { get; set; }
}
```

### Internal Configuration

From `src/Ocelot/Configuration/InternalConfiguration.cs`, the file configuration is transformed into an internal representation:

```csharp
public class InternalConfiguration : IInternalConfiguration
{
    public InternalConfiguration() => Routes = [];
    public InternalConfiguration(Route[] routes) => Routes = routes ?? [];

    public string AdministrationPath { get; init; }
    public CacheOptions CacheOptions { get; set; }
    public Version DownstreamHttpVersion { get; init; }
    public HttpVersionPolicy DownstreamHttpVersionPolicy { get; init; }
    public string DownstreamScheme { get; init; }
    public HttpHandlerOptions HttpHandlerOptions { get; init; }
    public LoadBalancerOptions LoadBalancerOptions { get; init; }
    public MetadataOptions MetadataOptions { get; init; }
    public QoSOptions QoSOptions { get; init; }
    public RateLimitOptions RateLimitOptions { get; init; }
    public string RequestId { get; init; }
    public Route[] Routes { get; init; }
    public ServiceProviderConfiguration ServiceProviderConfiguration { get; init; }
    public int? Timeout { get; init; }
}
```

### Configuration Transformation Pipeline

The transformation from `FileConfiguration` to `InternalConfiguration` happens through several creators registered in `OcelotBuilder`:

1. **IRoutesCreator** - Creates Route objects from FileRoute
2. **IDynamicsCreator** - Creates dynamic routes
3. **IAggregatesCreator** - Creates aggregate routes
4. **IQoSOptionsCreator** - Creates QoS options
5. **IRateLimitOptionsCreator** - Creates rate limiting options
6. **ILoadBalancerOptionsCreator** - Creates load balancer options
7. **IConfigurationCreator** - Orchestrates all creators

---

## Request Processing Flow

### End-to-End Request Flow

Here's how a request flows through Ocelot:

```
Client Request
    ↓
[1] ASP.NET Core Pipeline
    ↓
[2] ConfigurationMiddleware
    - Loads InternalConfiguration from repository
    - Stores in HttpContext.Items
    ↓
[3] ExceptionHandlerMiddleware
    - Sets up global exception handling
    - Sets initial Request ID
    ↓
[4] DownstreamRouteFinderMiddleware
    - Extracts: path, query, method, headers, host
    - Matches against configured routes
    - Example: /api/users/123 matches /api/users/{id}
    - Stores: DownstreamRoute, TemplatePlaceholderNameAndValues
    ↓
[5] SecurityMiddleware
    - Checks IP whitelist/blacklist
    - Short-circuits if IP not allowed
    ↓
[6] HttpHeadersTransformationMiddleware
    - Applies header transformations
    - Find and replace header values
    ↓
[7] DownstreamRequestInitialiserMiddleware
    - Creates DownstreamRequest object
    - Maps HttpRequest to DownstreamRequest
    - Applies initial transformations
    ↓
[8] RateLimitingMiddleware
    - Checks rate limit rules
    - Increments counters
    - Short-circuits if limit exceeded
    ↓
[9] RequestIdMiddleware
    - Adds or updates request ID header
    - Overrides global setting if route-specific ID exists
    ↓
[10] AuthenticationMiddleware
    - Authenticates using configured schemes
    - Short-circuits if authentication fails
    ↓
[11] ClaimsToClaimsMiddleware
    - Transforms claims
    ↓
[12] AuthorizationMiddleware
    - Checks authorization policies
    - Validates required claims
    - Short-circuits if authorization fails
    ↓
[13] ClaimsToHeadersMiddleware
    - Adds claims as headers to downstream request
    ↓
[14] ClaimsToQueryStringMiddleware
    - Adds claims to query string
    ↓
[15] ClaimsToDownstreamPathMiddleware
    - Modifies downstream path based on claims
    ↓
[16] LoadBalancingMiddleware
    - Selects specific downstream host
    - Uses configured load balancer (RoundRobin, LeastConnection, etc.)
    ↓
[17] DownstreamUrlCreatorMiddleware
    - Replaces placeholders in path template
    - Example: /api/users/{id} → /api/users/123
    - Creates final downstream URL
    ↓
[18] OutputCacheMiddleware
    - Checks cache for response
    - Returns cached response if exists
    - Otherwise continues to request
    ↓
[19] HttpRequesterMiddleware
    - Creates HttpRequestMessage
    - Sends request to downstream service
    - Receives HttpResponseMessage
    - Stores in DownstreamResponse
    - Last Ocelot middleware in the pipeline
    ↓
Response flows back up through middleware
    ↓
[20] ResponderMiddleware
    - Maps DownstreamResponse to HttpResponse
    - Sets status code, headers, body
    ↓
Client Response
```

### Data Flow Through HttpContext.Items

Middleware communicates through `HttpContext.Items` dictionary. Extension methods in `src/Ocelot/Middleware/HttpItemsExtensions.cs` provide type-safe access:

```csharp
// Storing data
httpContext.Items.UpsertDownstreamRoute(downstreamRoute);
httpContext.Items.UpsertTemplatePlaceholderNameAndValues(placeholders);
httpContext.Items.UpsertDownstreamRequest(downstreamRequest);
httpContext.Items.UpsertDownstreamResponse(downstreamResponse);
httpContext.Items.UpsertErrors(errors);

// Retrieving data
var route = httpContext.Items.DownstreamRoute();
var placeholders = httpContext.Items.TemplatePlaceholderNameAndValues();
var request = httpContext.Items.DownstreamRequest();
var response = httpContext.Items.DownstreamResponse();
var errors = httpContext.Items.Errors();
```

### Error Handling

Each middleware can add errors to the pipeline:

```csharp
if (someConditionFails)
{
    httpContext.Items.UpsertErrors(new List<Error> 
    { 
        new UnauthorizedError("Authentication failed") 
    });
    return; // Short-circuit pipeline
}
```

The `ResponderMiddleware` checks for errors and returns appropriate responses.

---

## Key Components

### 1. Route Finding

#### DownstreamRouteProvider

Routes are found using providers in `src/Ocelot/DownstreamRouteFinder/Finder/`. The system supports two types:

**Static Routes**: `DownstreamRouteFinder`
- Matches URL patterns using regex
- Extracts placeholder values
- Example: `/api/products/{id}` matches `/api/products/123`

**Service Discovery Routes**: `DiscoveryDownstreamRouteFinder`
- Integrates with service discovery (Consul, Eureka, Kubernetes)
- Resolves service names to actual endpoints

#### URL Matching

From `src/Ocelot/DownstreamRouteFinder/UrlMatcher/RegExUrlMatcher.cs`:

```csharp
public interface IUrlPathToUrlTemplateMatcher
{
    Response<UrlMatch> Match(string upstreamUrlPath, string upstreamQueryString, 
        UpstreamPathTemplate pathTemplate);
}
```

The matcher:
1. Converts path template to regex pattern
2. Matches incoming URL against pattern
3. Extracts placeholder values
4. Returns `UrlMatch` with captured values

Example:
- Template: `/api/products/{category}/{id}`
- URL: `/api/products/electronics/123`
- Result: `{category: "electronics", id: "123"}`

### 2. Load Balancing

#### Load Balancer Factory

From `src/Ocelot/LoadBalancer/Interfaces/ILoadBalancerFactory.cs`, load balancers are created based on configuration:

```csharp
public interface ILoadBalancerFactory
{
    Response<ILoadBalancer> Get(DownstreamRoute route, ServiceProviderConfiguration config);
}
```

#### Built-in Load Balancers

Located in `src/Ocelot/LoadBalancer/`:

**NoLoadBalancer**: Single host, no balancing
**RoundRobin**: Distributes requests evenly across hosts
**LeastConnection**: Routes to host with fewest active connections
**CookieStickySessions**: Uses cookies to maintain session affinity

#### Load Balancer Creators

From `src/Ocelot/LoadBalancer/Creators/`:

```csharp
// RoundRobinCreator.cs
public class RoundRobinCreator : ILoadBalancerCreator
{
    public Response<ILoadBalancer> Create(DownstreamRoute route, 
        IServiceDiscoveryProvider serviceProvider)
    {
        return new OkResponse<ILoadBalancer>(
            new RoundRobin(async () => await serviceProvider.GetAsync()));
    }
    
    public string Type => nameof(RoundRobin);
}
```

#### Lease System

Load balancers return a `Lease` object:

```csharp
public class Lease
{
    public Lease(ServiceHostAndPort hostAndPort)
    {
        HostAndPort = hostAndPort;
    }
    
    public ServiceHostAndPort HostAndPort { get; }
}
```

The `LoadBalancingMiddleware` uses this lease to select the downstream host.

### 3. Caching

#### Output Cache Middleware

From `src/Ocelot/Cache/OutputCacheMiddleware.cs`:

```csharp
public async Task Invoke(HttpContext httpContext)
{
    var downstreamRoute = httpContext.Items.DownstreamRoute();
    
    if (!downstreamRoute.IsCached)
    {
        await _next.Invoke(httpContext);
        return;
    }
    
    var cacheKey = _cacheKeyGenerator.GenerateRequestCacheKey(httpContext);
    var cached = _cache.Get<CachedResponse>(cacheKey, downstreamRoute.CacheOptions.Region);
    
    if (cached != null)
    {
        // Return cached response
        httpContext.Items.UpsertDownstreamResponse(new DownstreamResponse(cached));
        return;
    }
    
    // Continue to make request
    await _next.Invoke(httpContext);
    
    // Cache the response
    var response = httpContext.Items.DownstreamResponse();
    _cache.Add(cacheKey, response, downstreamRoute.CacheOptions.TtlSeconds);
}
```

#### Cache Key Generation

From `src/Ocelot/Cache/DefaultCacheKeyGenerator.cs`:

```csharp
public string GenerateRequestCacheKey(HttpContext httpContext)
{
    var downstreamUrlKey = $"{httpContext.Request.Method}-{downstreamUrl}";
    var hashedUrl = MD5Helper.GenerateMd5(downstreamUrlKey);
    return hashedUrl;
}
```

Cache keys are based on:
- HTTP method
- Complete downstream URL (including query parameters)

#### Cache Implementations

**DefaultMemoryCache**: In-memory caching using `MemoryCache`
**CacheManager Integration**: Through `Ocelot.Cache.CacheManager` package for distributed caching

### 4. Rate Limiting

#### Rate Limiting Middleware

From `src/Ocelot/RateLimiting/RateLimitingMiddleware.cs`:

```csharp
public async Task Invoke(HttpContext httpContext)
{
    var downstreamRoute = httpContext.Items.DownstreamRoute();
    
    if (!downstreamRoute.EnableEndpointEndpointRateLimiting)
    {
        await _next.Invoke(httpContext);
        return;
    }
    
    var identity = _clientRequestIdentity.SetIdentity(httpContext);
    var rateLimitCounter = await _rateLimitStorage.GetAsync(identity.Key);
    
    if (rateLimitCounter.HasExceededLimit)
    {
        httpContext.Items.UpsertErrors(new List<Error> 
        { 
            new QuotaExceededError() 
        });
        return;
    }
    
    await _rateLimitStorage.IncrementAsync(identity.Key);
    await _next.Invoke(httpContext);
}
```

#### Client Identity

From `src/Ocelot/RateLimiting/ClientRequestIdentity.cs`:

```csharp
public class ClientRequestIdentity
{
    public ClientRequestIdentity(string key, string clientId = null, 
        string requestId = null)
    {
        Key = key;
        ClientId = clientId;
        RequestId = requestId;
    }
    
    public string Key { get; }
    public string ClientId { get; }
    public string RequestId { get; }
}
```

Clients can be identified by:
- IP address
- Client ID from authentication
- Custom headers

#### Rate Limit Counter

From `src/Ocelot/RateLimiting/RateLimitCounter.cs`:

```csharp
public class RateLimitCounter
{
    public long Count { get; set; }
    public DateTime Timestamp { get; set; }
    
    public bool HasExceededLimit(long limit)
    {
        return Count > limit;
    }
}
```

#### Storage Options

**MemoryCacheRateLimitStorage**: In-memory rate limiting
**DistributedCacheRateLimitStorage**: Distributed rate limiting using `IDistributedCache`

### 5. Service Discovery

#### Service Discovery Provider Factory

From `src/Ocelot/ServiceDiscovery/ServiceDiscoveryProviderFactory.cs`:

```csharp
public interface IServiceDiscoveryProviderFactory
{
    Response<IServiceDiscoveryProvider> Get(ServiceProviderConfiguration config, 
        DownstreamRoute route);
}
```

#### Providers

Located in `src/Ocelot/ServiceDiscovery/Providers/`:

**ConfigurationServiceProvider**: Uses hosts from configuration
**ConsulServiceProvider**: Queries Consul for service instances (via separate package)
**EurekaServiceProvider**: Queries Eureka for service instances (via separate package)
**KubernetesServiceProvider**: Queries Kubernetes API (via separate package)

#### Service Discovery Flow

```
LoadBalancingMiddleware
    ↓
IServiceDiscoveryProvider.GetAsync()
    ↓
Returns List<ServiceHostAndPort>
    ↓
ILoadBalancer.Lease()
    ↓
Selects one ServiceHostAndPort
    ↓
Stores selected host in HttpContext
```

### 6. Authentication

#### Authentication Middleware

From `src/Ocelot/Authentication/Middleware/AuthenticationMiddleware.cs`:

```csharp
public async Task Invoke(HttpContext httpContext)
{
    var downstreamRoute = httpContext.Items.DownstreamRoute();
    
    if (!downstreamRoute.IsAuthenticated)
    {
        await _next.Invoke(httpContext);
        return;
    }
    
    var scheme = downstreamRoute.AuthenticationOptions.AuthenticationProviderKey;
    var result = await httpContext.AuthenticateAsync(scheme);
    
    if (!result.Succeeded)
    {
        httpContext.Items.UpsertErrors(new List<Error> 
        { 
            new UnauthenticatedError() 
        });
        return;
    }
    
    httpContext.User = result.Principal;
    await _next.Invoke(httpContext);
}
```

Ocelot delegates to ASP.NET Core authentication:
- Uses standard authentication schemes (JWT, Cookie, OAuth, etc.)
- Integrates with IdentityServer for token validation
- Supports multiple authentication schemes per route

### 7. Authorization

#### Authorization Middleware

From `src/Ocelot/Authorization/Middleware/AuthorizationMiddleware.cs`:

```csharp
public async Task Invoke(HttpContext httpContext)
{
    var downstreamRoute = httpContext.Items.DownstreamRoute();
    
    if (!downstreamRoute.IsAuthorized)
    {
        await _next.Invoke(httpContext);
        return;
    }
    
    var authorized = await _claimsAuthorizer.Authorize(
        httpContext.User, 
        downstreamRoute.RouteClaimsRequirement);
    
    if (authorized.IsError)
    {
        httpContext.Items.UpsertErrors(authorized.Errors);
        return;
    }
    
    await _next.Invoke(httpContext);
}
```

#### Claims Authorization

From `src/Ocelot/Authorization/ClaimsAuthorizer.cs`:

```csharp
public Response<bool> Authorize(ClaimsPrincipal claimsPrincipal, 
    Dictionary<string, string> routeClaimsRequirement)
{
    foreach (var required in routeClaimsRequirement)
    {
        var claim = claimsPrincipal.Claims
            .FirstOrDefault(c => c.Type == required.Key);
        
        if (claim == null || claim.Value != required.Value)
        {
            return new ErrorResponse<bool>(
                new ClaimValueNotAuthorizedError());
        }
    }
    
    return new OkResponse<bool>(true);
}
```

Authorization checks:
- Required claims exist on the user
- Claim values match configured requirements
- Scope-based authorization (OAuth scopes)

### 8. Request Aggregation (Multiplexing)

#### Multiplexing Middleware

From `src/Ocelot/Multiplexer/MultiplexingMiddleware.cs`:

```csharp
public async Task Invoke(HttpContext httpContext)
{
    var downstreamRoute = httpContext.Items.DownstreamRoute();
    
    if (!downstreamRoute.IsMultiplex)
    {
        await _next.Invoke(httpContext);
        return;
    }
    
    // Make multiple downstream requests
    var responses = new List<DownstreamResponse>();
    foreach (var route in downstreamRoute.Routes)
    {
        var response = await MakeRequest(httpContext, route);
        responses.Add(response);
    }
    
    // Aggregate responses
    var aggregator = _aggregatorFactory.Get(downstreamRoute);
    var aggregated = await aggregator.Aggregate(responses);
    
    httpContext.Items.UpsertDownstreamResponse(aggregated);
}
```

#### Response Aggregators

From `src/Ocelot/Multiplexer/`:

**SimpleJsonResponseAggregator**: Merges JSON responses into single object
**UserDefinedResponseAggregator**: Custom aggregation logic

Example aggregated response:
```json
{
  "user": { "id": 1, "name": "John" },
  "orders": [ { "id": 101, "total": 50.00 } ],
  "preferences": { "theme": "dark" }
}
```

Each key corresponds to a route key in the configuration.

### 9. Quality of Service (QoS)

#### QoS Options

From `src/Ocelot/Configuration/QoSOptions.cs`:

```csharp
public class QoSOptions
{
    public int ExceptionsAllowedBeforeBreaking { get; set; }
    public int DurationOfBreak { get; set; }
    public int TimeoutValue { get; set; }
}
```

#### Circuit Breaker Pattern

Ocelot integrates with Polly for circuit breaker functionality:

```
Normal State
    ↓
Exception occurs
    ↓
Count exceptions
    ↓
Threshold reached → Open Circuit
    ↓
Wait DurationOfBreak
    ↓
Half-Open → Test one request
    ↓
Success → Close Circuit
Failure → Open Circuit again
```

#### Timeout Policy

Requests timeout after `TimeoutValue` milliseconds:
- Cancels long-running requests
- Returns timeout error to client
- Prevents resource exhaustion

### 10. WebSocket Support

#### WebSocket Proxy Middleware

From `src/Ocelot/WebSockets/WebSocketsProxyMiddleware.cs`:

The WebSocket pipeline:
1. Detects WebSocket upgrade request
2. Routes to WebSocket-specific pipeline
3. Establishes downstream WebSocket connection
4. Proxies messages bidirectionally
5. Handles connection lifecycle

WebSocket proxy features:
- Full-duplex communication
- Message framing preservation
- Connection keep-alive
- Graceful shutdown

---

## Advanced Features

### 1. Claims Transformation

#### Claims to Headers

From `src/Ocelot/Headers/Middleware/ClaimsToHeadersMiddleware.cs`:

```csharp
public async Task Invoke(HttpContext httpContext)
{
    var downstreamRoute = httpContext.Items.DownstreamRoute();
    
    foreach (var claimToHeader in downstreamRoute.ClaimsToHeaders)
    {
        var claim = httpContext.User.Claims
            .FirstOrDefault(c => c.Type == claimToHeader.Key);
        
        if (claim != null)
        {
            httpContext.Items.DownstreamRequest()
                .Headers.Add(claimToHeader.Value, claim.Value);
        }
    }
    
    await _next.Invoke(httpContext);
}
```

Example: Add user ID from claims to `X-User-Id` header

#### Claims to Query String

From `src/Ocelot/QueryStrings/Middleware/ClaimsToQueryStringMiddleware.cs`:

Similar to headers, but adds claims to query parameters:
- Extract claim value from user
- Add to downstream request query string

#### Claims to Path

From `src/Ocelot/DownstreamPathManipulation/Middleware/ClaimsToDownstreamPathMiddleware.cs`:

Modifies the downstream path based on claim values:
- Replace placeholders with claim values
- Example: `/api/{userId}/profile` → `/api/123/profile`

### 2. Header Transformation

#### Find and Replace

From `src/Ocelot/Headers/HttpContextRequestHeaderReplacer.cs`:

```csharp
public void Replace(HttpContext context, List<HeaderFindAndReplace> fAndRs)
{
    foreach (var f in fAndRs)
    {
        if (context.Request.Headers.ContainsKey(f.Key))
        {
            var value = context.Request.Headers[f.Key].ToString();
            var newValue = value.Replace(f.Find, f.Replace);
            context.Request.Headers[f.Key] = newValue;
        }
    }
}
```

Supports:
- Regex-based find and replace
- Static value replacement
- Applies to both request and response headers

### 3. Security

#### IP Security

From `src/Ocelot/Security/Middleware/SecurityMiddleware.cs`:

```csharp
public async Task Invoke(HttpContext httpContext)
{
    var downstreamRoute = httpContext.Items.DownstreamRoute();
    
    if (!downstreamRoute.IsSecured)
    {
        await _next.Invoke(httpContext);
        return;
    }
    
    var ip = httpContext.Connection.RemoteIpAddress;
    var allowed = _ipSecurity.IsAllowed(ip, 
        downstreamRoute.SecurityOptions);
    
    if (!allowed)
    {
        httpContext.Items.UpsertErrors(new List<Error> 
        { 
            new UnauthorizedError("IP not allowed") 
        });
        return;
    }
    
    await _next.Invoke(httpContext);
}
```

Security options:
- IP whitelist
- IP blacklist
- CIDR notation support

### 4. Request ID Correlation

#### Request ID Middleware

From `src/Ocelot/RequestId/Middleware/RequestIdMiddleware.cs`:

```csharp
public async Task Invoke(HttpContext httpContext)
{
    var downstreamRoute = httpContext.Items.DownstreamRoute();
    
    if (string.IsNullOrEmpty(downstreamRoute.RequestIdKey))
    {
        await _next.Invoke(httpContext);
        return;
    }
    
    var requestId = httpContext.Request.Headers[downstreamRoute.RequestIdKey]
        .FirstOrDefault() ?? Guid.NewGuid().ToString();
    
    httpContext.Items.DownstreamRequest()
        .Headers.Add(downstreamRoute.RequestIdKey, requestId);
    
    await _next.Invoke(httpContext);
}
```

Request ID features:
- Correlation across microservices
- Distributed tracing support
- Configurable header name
- Auto-generation if not provided

### 5. Delegating Handlers

#### Custom HTTP Message Handlers

From `src/Ocelot/Requester/DelegatingHandlerHandlerFactory.cs`:

Ocelot supports custom `DelegatingHandler` implementations:

```csharp
public class CustomHandler : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, 
        CancellationToken cancellationToken)
    {
        // Pre-request logic
        var response = await base.SendAsync(request, cancellationToken);
        // Post-request logic
        return response;
    }
}
```

Use cases:
- Custom logging
- Request/response modification
- Retry policies
- Custom authentication

### 6. Metadata

Routes can have custom metadata:

```csharp
public Dictionary<string, string> Metadata { get; set; }
```

Metadata use cases:
- Custom business logic
- Feature flags
- A/B testing
- Tagging and filtering

Access metadata in custom middleware:
```csharp
var metadata = httpContext.Items.DownstreamRoute().Metadata;
```

### 7. Dynamic Routes

From `src/Ocelot/Configuration/File/FileDynamicRoute.cs`:

Dynamic routes use service discovery without static configuration:

```csharp
public class FileDynamicRoute
{
    public string ServiceName { get; set; }
    public FileRateLimiting RateLimiting { get; set; }
}
```

Dynamic routing flow:
1. Request arrives with service name in path or header
2. Query service discovery for available instances
3. Apply load balancing
4. Forward request

### 8. Administration API

Ocelot provides an administration API for runtime configuration:

From `src/Ocelot/DependencyInjection/IAdministrationPath.cs`:

```csharp
public interface IAdministrationPath
{
    string Path { get; }
}
```

Administration endpoints:
- `GET /administration/configuration` - Get current configuration
- `POST /administration/configuration` - Update configuration
- `POST /administration/outputcache/{region}` - Clear cache

### 9. Extensibility Points

Ocelot provides multiple extension points:

#### Custom Middleware

Inject custom middleware at specific points:

```csharp
var configuration = new OcelotPipelineConfiguration
{
    PreAuthenticationMiddleware = async (ctx, next) =>
    {
        // Custom logic before authentication
        await next();
    },
    PreAuthorizationMiddleware = async (ctx, next) =>
    {
        // Custom logic before authorization
        await next();
    },
    PreQueryStringBuilderMiddleware = async (ctx, next) =>
    {
        // Custom query string logic
        await next();
    },
    PreErrorResponderMiddleware = async (ctx, next) =>
    {
        // Custom error handling
        await next();
    }
};

await app.UseOcelot(configuration);
```

#### Custom Load Balancer

Implement `ILoadBalancerCreator`:

```csharp
public class CustomLoadBalancerCreator : ILoadBalancerCreator
{
    public Response<ILoadBalancer> Create(DownstreamRoute route, 
        IServiceDiscoveryProvider serviceProvider)
    {
        return new OkResponse<ILoadBalancer>(
            new CustomLoadBalancer(serviceProvider));
    }
    
    public string Type => "CustomLoadBalancer";
}
```

Register in DI:
```csharp
services.AddSingleton<ILoadBalancerCreator, CustomLoadBalancerCreator>();
```

#### Custom Response Aggregator

Implement `IDefinedAggregator`:

```csharp
public class CustomAggregator : IDefinedAggregator
{
    public async Task<DownstreamResponse> Aggregate(
        List<HttpContext> contexts)
    {
        // Custom aggregation logic
    }
}
```

---

## Summary

Ocelot is a comprehensive API Gateway built on ASP.NET Core middleware:

1. **Architecture**: Sequential chain of middleware components with conditional branching, each with a specific responsibility
2. **Configuration**: JSON-based configuration transformed into internal representation
3. **Request Flow**: Sequential pipeline with conditional branches for route finding, processing, and request execution
4. **Features**: Load balancing, caching, rate limiting, authentication, authorization
5. **Extensibility**: Multiple extension points for custom behavior
6. **Service Discovery**: Integration with Consul, Eureka, Kubernetes
7. **Advanced**: Request aggregation, QoS, WebSockets, claims transformation

The key to understanding Ocelot is recognizing it as a sophisticated middleware orchestrator that:
- Receives upstream requests
- Routes them to downstream services
- Applies transformations along the way
- Returns aggregated/transformed responses

All of this is configured declaratively through JSON and executed through a well-designed, extensible middleware pipeline.
