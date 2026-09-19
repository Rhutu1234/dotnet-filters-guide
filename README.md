# Filters in ASP.NET Core MVC

*A deep-dive walkthrough of the ASP.NET Core MVC filter pipeline — covering all five filter types (Authorization, Resource, Action, Exception, Result) in depth with their execution order and short-circuiting mechanics, synchronous vs. asynchronous filter interfaces, applying filters at the global/controller/action scope and how ordering resolves across them, filter dependency injection via `ServiceFilter`/`TypeFilter`, and exactly how filters relate to — and differ from — the middleware pipeline covered elsewhere in this series.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [Why Filters Exist Alongside Middleware](#1-why-filters-exist-alongside-middleware)
3. [The Filter Pipeline: All Five Types, In Execution Order](#2-the-filter-pipeline-all-five-types-in-execution-order)
4. [Authorization Filters](#3-authorization-filters)
5. [Resource Filters](#4-resource-filters)
6. [Action Filters](#5-action-filters)
7. [Exception Filters](#6-exception-filters)
8. [Result Filters](#7-result-filters)
9. [Synchronous vs. Asynchronous Filter Interfaces](#8-synchronous-vs-asynchronous-filter-interfaces)
10. [Applying Filters: Attributes, Global Registration, and Scope](#9-applying-filters-attributes-global-registration-and-scope)
11. [Filter Ordering Across Scopes](#10-filter-ordering-across-scopes)
12. [Short-Circuiting: Setting context.Result](#11-short-circuiting-setting-contextresult)
13. [Dependency Injection in Filters](#12-dependency-injection-in-filters)
14. [IFilterFactory and Custom Filter Attributes with Parameters](#13-ifilterfactory-and-custom-filter-attributes-with-parameters)
15. [Common Pitfalls](#14-common-pitfalls)
16. [Quick Reference Table](#quick-reference-table)
17. [Conclusion](#conclusion)

---

## Introduction

Filters are ASP.NET Core MVC's own, purpose-built extensibility mechanism for cross-cutting concerns that specifically need to run around controller action execution — with direct access to MVC context (the matched action, its bound arguments, the action's return result) that this series' Middleware guide's Section 12 already identifies as the key thing raw middleware doesn't have visibility into. There isn't just one kind of filter: ASP.NET Core defines five distinct filter types, each running at a specific, different point relative to action execution, each solving a genuinely different problem — authorization filters decide access before anything else runs; action filters wrap the action itself; exception filters catch what the action throws; result filters wrap how the response gets written. This guide goes deep on each type, the concrete order they run in relative to one another, how to apply and scope them, and the dependency-injection considerations specific to filters.

```plaintext
Request reaches the matched MVC action →

  Authorization Filters → Resource Filters (before) → Model Binding →
    Action Filters (before) → THE ACTION ITSELF → Action Filters (after) →
  Resource Filters (after) → Result Filters (before) → THE RESULT (writes the response) →
    Result Filters (after)

  [Exception Filters wrap the ACTION and can catch anything it throws]
```

---

## 1. Why Filters Exist Alongside Middleware

### Middleware operates on `HttpContext` alone; filters operate with genuine MVC context

```plaintext
Middleware (this series' Middleware guide): sees ONLY the raw HttpContext
  — the request, the response, headers, the URL. It has NO idea what
  controller action will eventually handle this request, what arguments
  will be bound to it, or what that action's return value will be.
Filters: run SPECIFICALLY around MVC action execution, with direct access
  to the matched action's METADATA, its BOUND ARGUMENTS, and (for filters
  running after the action) its RETURN VALUE — genuinely richer context
  middleware structurally cannot provide.
```

This is the precise distinction this series' Middleware guide's Section 12 introduces — worth restating here as this guide's actual starting point: filters exist because certain cross-cutting concerns (validating a specific action's bound model, inspecting or transforming what a specific action returned, applying authorization rules scoped to a specific action or controller) genuinely need access to information that simply doesn't exist yet at the point raw middleware runs.

### Filters run INSIDE the MVC endpoint-invocation step of the middleware pipeline

```plaintext
Per this series' Middleware guide's Section 11: app.MapControllers()
  registers the terminal middleware that invokes MVC. The ENTIRE filter
  pipeline this guide describes runs INSIDE that single point in the
  broader middleware chain — from the outer pipeline's perspective,
  "run all the filters, then the action, then all the filters again" is
  just what happens when that one particular middleware step executes.
```

This is the genuine, structural relationship between the two systems: filters aren't a competing or parallel mechanism to middleware — they're a *nested* pipeline, specifically scoped to MVC action invocation, running entirely within the single terminal middleware step this series' Middleware guide's Section 11 describes. Understanding this hierarchy — middleware pipeline, containing an MVC-invocation step, containing the filter pipeline — is what makes sense of why filters can offer richer context: they're operating at a later, more specific point than middleware ever reaches.

---

## 2. The Filter Pipeline: All Five Types, In Execution Order

### The complete ordering, stated precisely

```plaintext
1. Authorization Filters   — runs FIRST, can short-circuit before ANYTHING else, including model binding
2. Resource Filters (before) — runs before model binding; can short-circuit the WHOLE rest of the pipeline
3. [Model Binding happens here]
4. Action Filters (before)  — runs immediately before the action method itself
5. [THE ACTION METHOD EXECUTES]
6. Action Filters (after)   — runs immediately after the action returns
7. Resource Filters (after) — the "after" half of step 2's resource filters
8. Result Filters (before)  — runs before the action's RESULT is executed (i.e., before the response is written)
9. [THE RESULT EXECUTES — this is what actually writes the HTTP response]
10. Result Filters (after)  — runs after the response has been written

Exception Filters: don't fit neatly into this linear sequence — they wrap
  steps 4 through 6 (the action filters and action execution), catching
  any UNHANDLED exception thrown from within that span.
```

This ordering is worth memorizing precisely, because it directly explains *why* each filter type exists as its own distinct thing rather than one generic "filter" concept — each type is defined by exactly *when* it runs relative to model binding, the action itself, and result execution, and that timing is what determines what each type can and cannot do.

### Why this layered structure mirrors the "before/after wrapping" shape this series' Middleware guide introduces

```plaintext
Every filter type follows the SAME wrapping pattern middleware does (per
  this series' Middleware guide's Section 1): code that runs BEFORE the
  next stage, a call (implicit or explicit, depending on the interface)
  that proceeds to that next stage, and (for most filter types) code that
  runs AFTER it returns — just applied at five specific, named points
  relative to MVC action execution, rather than as a single generic chain.
```

---

## 3. Authorization Filters

### The earliest filter type — runs before model binding, before resource filters, before anything else

```csharp
public class ApiKeyAuthorizationFilter : IAuthorizationFilter
{
    public void OnAuthorization(AuthorizationFilterContext context)
    {
        if (!context.HttpContext.Request.Headers.ContainsKey("X-Api-Key"))
        {
            context.Result = new UnauthorizedResult(); // short-circuits EVERYTHING after this point
        }
    }
}
```

Authorization filters exist specifically to answer "should this request even be allowed to proceed to this action at all" — and they run early enough (before model binding, before any resource filter's "before" logic) that a rejection here means genuinely nothing else in the filter pipeline, and certainly not the action itself, ever executes for this request.

### The built-in `[Authorize]` attribute is implemented as an authorization filter

```csharp
[Authorize(Roles = "Admin")]
public class AdminController : ControllerBase
{
    // every action here requires the caller to be authenticated AND in the "Admin" role
}
```

This is worth knowing explicitly: `[Authorize]`, one of the most commonly used attributes in ASP.NET Core, is itself implemented via this exact filter mechanism — it's an authorization filter that checks the current `HttpContext.User` (populated earlier by the authentication middleware this series' Middleware guide's Section 10 covers) against the attribute's configured requirements, short-circuiting with a 401/403 if they aren't met.

### Why authorization filters specifically should not depend on model-bound data

```plaintext
Because authorization filters run BEFORE model binding (Section 2's
  ordering), they genuinely cannot inspect the action's bound arguments —
  "is this user allowed to edit THIS SPECIFIC order" (where the order ID
  comes from a route or body parameter) is NOT something an authorization
  filter alone can express; that kind of resource-specific check typically
  belongs in the action itself, or in a resource filter (Section 4) running
  after binding, depending on exactly what data it needs.
```

This is a genuinely important scoping limitation worth understanding, not working around — authorization filters are the right tool for broad, identity-based checks ("is this caller authenticated," "does this caller have this role/policy"), not for checks requiring knowledge of the specific resource being acted upon.

---

## 4. Resource Filters

### The most powerful, most flexible filter type — wraps EVERYTHING from before model binding through after result execution

```csharp
public class CachingResourceFilter : IResourceFilter
{
    public void OnResourceExecuting(ResourceExecutionContext context)
    {
        // runs BEFORE model binding — can short-circuit the ENTIRE rest of the pipeline
        if (TryGetCachedResponse(context.HttpContext.Request.Path, out var cached))
        {
            context.Result = new ContentResult { Content = cached }; // skips model binding, the action, EVERYTHING
        }
    }

    public void OnResourceExecuted(ResourceExecutedContext context)
    {
        // runs AFTER result execution — the response has ALREADY been written by this point
        Console.WriteLine("Resource filter cleanup");
    }
}
```

Resource filters are genuinely the broadest-scoped filter type after authorization — they wrap not just the action, but model binding *and* result execution too, which is precisely why response caching (the example above) is a canonical use case: a resource filter can inspect the request, and if a cached response is available, skip the entire remaining pipeline (model binding, the action, everything) by setting `context.Result` directly, avoiding work that would otherwise be entirely wasted.

### Why resource filters are the right tool for concerns needing to wrap model binding specifically

```plaintext
Unlike action filters (Section 5), which only wrap the ACTION itself and
  run AFTER model binding has already happened, resource filters can
  short-circuit BEFORE binding occurs at all — genuinely useful for
  anything where even the COST of model binding (parsing a request body,
  validating it against a model) is worth avoiding when a short-circuit
  condition is already known to apply.
```

This is the precise, mechanical reason resource filters exist as their own distinct type rather than being redundant with action filters — the timing difference (before vs. after model binding) is a real, practical distinction for exactly this kind of "avoid even the binding cost" optimization.

---

## 5. Action Filters

### The most commonly used filter type — wraps the action method itself, with access to its bound arguments and return value

```csharp
public class LogActionFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        // runs immediately BEFORE the action — context.ActionArguments has the BOUND parameters
        foreach (var arg in context.ActionArguments)
            Console.WriteLine($"{arg.Key} = {arg.Value}");
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        // runs immediately AFTER the action — context.Result has whatever the action RETURNED
        Console.WriteLine($"Action returned: {context.Result}");
    }
}
```

Action filters are the workhorse filter type for most everyday cross-cutting concerns specific to a controller action — logging, simple validation, modifying bound arguments before the action sees them, or inspecting/modifying the action's result before it moves on to result execution (Section 7). `context.ActionArguments` is genuinely useful, direct access to exactly the parameter values the action is about to receive — something no earlier filter type or middleware could offer, since model binding hasn't happened yet at those earlier points.

### Modifying action arguments before the action runs

```csharp
public void OnActionExecuting(ActionExecutingContext context)
{
    if (context.ActionArguments.TryGetValue("request", out var value) && value is CreateOrderRequest req)
    {
        req.SubmittedAt = DateTimeOffset.UtcNow; // mutate the bound argument BEFORE the action sees it
    }
}
```

This is a genuinely powerful, if somewhat specialized capability — an action filter can directly modify `context.ActionArguments`, and the action method will receive the modified values, which is useful for cross-cutting concerns like automatically stamping a timestamp, normalizing input, or injecting a value the action itself shouldn't need to know how to compute.

### Short-circuiting the action itself, without preventing result execution

```csharp
public void OnActionExecuting(ActionExecutingContext context)
{
    if (!ModelStateIsValid(context))
    {
        context.Result = new BadRequestObjectResult(context.ModelState); // the ACTION never runs
        // but result filters (Section 7) STILL run, since context.Result now HAS a value to execute
    }
}
```

Setting `context.Result` inside `OnActionExecuting` skips the action method itself (this is precisely how the built-in automatic model-validation behavior works, when enabled) — but unlike a resource filter's short-circuit (Section 4), which can skip result execution entirely too, an action filter's short-circuit still allows the set `context.Result` to flow through result filters and be executed normally, producing the actual HTTP response.

---

## 6. Exception Filters

### The only filter type that doesn't fit the simple "before/after" pattern — it specifically catches unhandled exceptions

```csharp
public class ApiExceptionFilter : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        if (context.Exception is ValidationException validationEx)
        {
            context.Result = new BadRequestObjectResult(new { error = validationEx.Message });
            context.ExceptionHandled = true; // marks the exception as HANDLED — it won't propagate further
        }
        // if ExceptionHandled is left false, the exception continues propagating,
        // eventually reaching this series' Middleware guide's UseExceptionHandler, if configured
    }
}
```

Exception filters run specifically when an action filter or the action itself throws an unhandled exception — their job is to decide whether this specific filter can meaningfully handle it (setting `context.Result` and `context.ExceptionHandled = true`), or whether it should continue propagating outward, eventually reaching this series' Middleware guide's Section 9 exception-handling middleware if nothing at the filter level claims it.

### Why exception filters and middleware-level exception handling are complementary, not redundant

```plaintext
Exception FILTERS: scoped to a specific controller/action/global filter
  registration, with access to rich MVC context (which action threw,
  what its bound arguments were) — the right tool for handling exceptions
  in a way that's SPECIFIC to a particular controller or a particular
  category of MVC-level exception.
Exception-handling MIDDLEWARE (this series' Middleware guide's Section 9):
  a single, broad safety net for the ENTIRE application, including
  exceptions from NON-MVC middleware, and from any exception filters
  that chose NOT to handle what they saw.
```

Worth understanding as a genuinely complementary two-layer defense rather than choosing one or the other: exception filters handle specific, MVC-context-aware cases close to where they occur; middleware-level exception handling remains the final, universal safety net underneath everything, exactly the layered pattern this series' Middleware guide's own exception-handling section establishes.

### Exception filters do NOT catch exceptions thrown from resource filters, result filters, or authorization filters

```plaintext
Per Section 2's ordering: exception filters specifically wrap ACTION
  FILTERS and the ACTION ITSELF — an exception thrown from a RESOURCE
  filter's OnResourceExecuting, or from a RESULT filter, is OUTSIDE an
  exception filter's coverage entirely, and propagates directly to
  whatever's OUTSIDE the filter pipeline (ultimately, middleware-level
  exception handling).
```

This is a genuinely important, often-missed scoping limitation — worth stating explicitly since assuming exception filters are a universal MVC-level safety net (rather than one specifically scoped to action filters and the action) is a real, common source of "why didn't my exception filter catch this" confusion.

---

## 7. Result Filters

### The last filter type — wraps the execution of the action's RESULT (what actually writes the HTTP response)

```csharp
public class AddResponseHeaderFilter : IResultFilter
{
    public void OnResultExecuting(ResultExecutingContext context)
    {
        // runs BEFORE the result is executed — the response body has NOT been written yet
        context.HttpContext.Response.Headers.Append("X-Custom-Header", "value");
    }

    public void OnResultExecuted(ResultExecutedContext context)
    {
        // runs AFTER the response has ALREADY been written — you can no longer modify headers here
        Console.WriteLine("Response fully sent");
    }
}
```

Result filters wrap the step where whatever `IActionResult` the action (or an earlier filter's short-circuit) produced actually gets turned into bytes on the wire — an `OkObjectResult` gets serialized to JSON and written; a `ViewResult` gets a Razor view rendered. `OnResultExecuting` is genuinely the last reliable opportunity to modify response headers, since headers must be set before the response body begins streaming.

### Why this distinction (action filters vs. result filters) matters concretely

```plaintext
An ACTION filter's "after" hook (OnActionExecuted) sees the RESULT OBJECT
  the action produced (e.g., an OkObjectResult wrapping some data) — but
  the response HASN'T been written yet at that point.
A RESULT filter's "before" hook (OnResultExecuting) is the LAST chance to
  affect the response BEFORE it's actually serialized and sent — genuinely
  useful for concerns like adding response headers or wrapping/transforming
  the final response shape, which wouldn't make sense to do earlier,
  before the eventual result is even fully determined.
```

This is the practical reason to reach for a result filter specifically rather than an action filter for header-manipulation or response-shaping concerns — by the time an action filter's "after" hook runs, the result object exists, but result filters are the ones actually positioned immediately around its execution.

---

## 8. Synchronous vs. Asynchronous Filter Interfaces

### Every filter type has both a synchronous and an asynchronous interface variant

```csharp
// Synchronous — two separate methods, "before" and "after"
public class LogActionFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context) { }
    public void OnActionExecuted(ActionExecutedContext context) { }
}

// Asynchronous — ONE method, with an explicit `next` delegate, mirroring middleware's own shape
public class LogActionFilterAsync : IAsyncActionFilter
{
    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        Console.WriteLine("Before"); // the "before" logic
        var resultContext = await next(); // executes the ACTION (and everything after it in this filter's scope)
        Console.WriteLine("After");   // the "after" logic
    }
}
```

This mirrors, almost exactly, the two custom-middleware-writing approaches this series' Middleware guide covers in Sections 5 and 6 — the synchronous interface splits before/after into two separate methods; the asynchronous interface uses a single method with an explicit `next` delegate, structurally identical to middleware's own `RequestDelegate` pattern. For any genuinely `async` work (an `await`ed database call inside the filter, say), the asynchronous interface is required — the synchronous interfaces' methods cannot themselves be `async`.

### Why the asynchronous variant is generally preferred for new code

```plaintext
The async interfaces are STRICTLY more capable — they can express
  everything the synchronous interfaces can (simply don't await anything
  genuinely asynchronous, if there's nothing to await) PLUS genuinely
  asynchronous work, which the synchronous interfaces cannot support at
  all without resorting to this series' async/await guide's Section 7
  blocking-deadlock risk.
```

This is a direct, practical consequence of this series' async/await guide's own guidance — since the async filter interfaces can do everything the sync ones can and more, and since reaching for `.Result`/`.Wait()` inside a synchronous filter method to force async work to complete is exactly the deadlock risk that guide's Section 7 warns against, the async interfaces are the safer, more future-proof default choice for new filter code.

---

## 9. Applying Filters: Attributes, Global Registration, and Scope

### Attribute-based application: directly on an action or a controller

```csharp
public class OrdersController : ControllerBase
{
    [LogActionFilter] // applies to THIS ACTION only
    public IActionResult GetOrder(int id) { /* ... */ return Ok(); }
}

[LogActionFilter] // applies to EVERY ACTION in this controller
public class OrdersController : ControllerBase { /* ... */ }
```

Filters implemented as attributes (inheriting from `Attribute` and implementing a filter interface, or extending one of the convenience base classes like `ActionFilterAttribute`) can be applied directly, declaratively, at either the action or controller level — this is the most common, most visible way filters show up in real ASP.NET Core codebases.

### Global registration: applies to EVERY controller and action in the application

```csharp
builder.Services.AddControllers(options =>
{
    options.Filters.Add<LogActionFilter>(); // applies GLOBALLY, to every action, without any attribute needed
});
```

Registering a filter globally (via `MvcOptions.Filters`) applies it to every single MVC action in the application, without needing to decorate anything with an attribute — appropriate for concerns that genuinely apply universally (a standard logging filter, a global exception filter serving as the MVC-level counterpart to this series' Middleware guide's application-wide exception-handling middleware).

### The three scopes, and why the choice matters

```plaintext
Global: applies everywhere, automatically — the broadest, least targeted scope.
Controller: applies to every action within one controller — a natural
  fit for concerns specific to one resource/area of the API.
Action: applies to exactly one action — the narrowest, most targeted scope.
```

Choosing the right scope is a genuine design decision, not just a matter of convenience — over-applying a filter globally when it's only relevant to one controller adds unnecessary overhead (and potential unintended side effects) to every other action in the application; under-applying it (repeating an attribute on every individual action rather than once at the controller level) is unnecessary duplication when the concern genuinely applies to the whole controller.

---

## 10. Filter Ordering Across Scopes

### Multiple filters of the SAME type, applied at different scopes, run in a specific, defined order

```plaintext
For "before" hooks (OnActionExecuting, etc.): Global filters run FIRST,
  then Controller-level filters, then Action-level filters — OUTSIDE IN.
For "after" hooks (OnActionExecuted, etc.): the order REVERSES —
  Action-level filters run FIRST, then Controller-level, then Global —
  INSIDE OUT.
```

This is exactly the same "first in, last out" nesting behavior this series' Middleware guide's Section 2 establishes for the broader middleware pipeline, just applied here across filter *scopes* rather than registration order within a single list — the broadest-scoped filter (Global) wraps everything else, so its "before" logic runs first and its "after" logic runs last, with narrower scopes nested progressively inside it.

### The `Order` property: explicit control within the same scope

```csharp
[LogActionFilter(Order = 1)]
[ValidateModelFilter(Order = 2)]
public IActionResult CreateOrder(CreateOrderRequest request) { /* ... */ return Ok(); }
```

Within the same scope (two filters both applied at the action level, say), the `Order` property gives explicit, numeric control over execution sequence — lower values run their "before" logic earlier (and their "after" logic later, following the same nesting principle), letting you resolve ordering ambiguity between multiple filters that would otherwise have no clearly defined relative order.

---

## 11. Short-Circuiting: Setting context.Result

### The single, consistent mechanism every filter type uses to short-circuit

```csharp
public void OnActionExecuting(ActionExecutingContext context)
{
    context.Result = new BadRequestObjectResult("Invalid request"); // THIS is the short-circuit mechanism
}
```

This is worth stating as the one, consistent pattern spanning every filter type covered in this guide (with minor variations in which specific context object it's set on) — setting `.Result` on a "before" context is how a filter says "don't proceed any further down this pipeline; here's the response to use instead," directly analogous to this series' Middleware guide's Section 4 "don't call `next`" short-circuit pattern, just expressed as an assignment rather than an omitted method call.

### What short-circuiting skips, specifically, depends on WHICH filter type does it

```plaintext
Authorization filter sets Result: skips EVERYTHING — resource filters,
  binding, action filters, the action, result filters all still run for
  the RESULT that was set, but the intended business logic never executes.
Resource filter sets Result (in OnResourceExecuting): skips binding, action
  filters, and the action — but RESULT filters and result execution STILL
  run, to actually produce the response from the short-circuit Result.
Action filter sets Result (in OnActionExecuting): skips the action itself
  — result filters still run normally against the substituted Result.
```

This nuance is genuinely worth understanding precisely, since it's easy to assume "setting Result" always means "stop everything" — in every case, it specifically means "skip the remaining STEPS THIS FILTER TYPE WAS GOING TO WRAP," while the Result that was set still flows through and gets executed by whatever comes after (result filters, result execution) exactly as if it had come from the action itself.

---

## 12. Dependency Injection in Filters

### Filter attributes cannot receive constructor-injected services directly — attributes are constructed by the CLR, not by DI

```csharp
// ❌ This does NOT work as you might expect — attribute constructors are invoked by the
//    runtime's ATTRIBUTE mechanism, which has no knowledge of your DI container at all
public class LogActionFilterAttribute : ActionFilterAttribute
{
    private readonly ILogger _logger;
    public LogActionFilterAttribute(ILogger logger) => _logger = logger; // ❌ won't resolve from DI
}
```

This is a genuine, structural limitation worth understanding, not a bug — attributes in .NET are instantiated by the CLR's own attribute-application mechanism when the type/method they decorate is reflected over, entirely independent of ASP.NET Core's DI container; there's no path for the container to supply constructor arguments to an attribute the way it does for ordinary DI-resolved classes.

### `ServiceFilter`: resolving a filter's dependencies from DI, while still applying it via attribute

```csharp
public class LogActionFilter : IActionFilter // an ORDINARY class, NOT an attribute — DI-resolvable normally
{
    private readonly ILogger<LogActionFilter> _logger;
    public LogActionFilter(ILogger<LogActionFilter> logger) => _logger = logger; // genuine constructor injection
    public void OnActionExecuting(ActionExecutingContext context) => _logger.LogInformation("Action executing");
    public void OnActionExecuted(ActionExecutedContext context) { }
}

builder.Services.AddScoped<LogActionFilter>(); // register the FILTER CLASS itself in DI

[ServiceFilter(typeof(LogActionFilter))] // apply it via a DIFFERENT attribute that resolves it FROM DI
public IActionResult GetOrder(int id) { /* ... */ return Ok(); }
```

`ServiceFilter` is the bridge: `LogActionFilter` itself is an ordinary, DI-registered class (not an attribute), so it gets genuine constructor injection exactly like any other DI-resolved service (following whatever lifetime you registered it with — this series' ASP.NET Core Dependency Injection guide's lifetime rules apply directly here too); `[ServiceFilter(typeof(LogActionFilter))]` is a separate, built-in attribute whose entire job is telling MVC "resolve this filter type from the DI container, rather than trying to construct it as a plain attribute."

### `TypeFilter`: similar to `ServiceFilter`, but doesn't require pre-registering the filter class

```csharp
[TypeFilter(typeof(LogActionFilter))] // resolves LogActionFilter's dependencies from DI WITHOUT needing
                                        //  builder.Services.AddScoped<LogActionFilter>() beforehand
public IActionResult GetOrder(int id) { /* ... */ return Ok(); }
```

`TypeFilter` achieves a very similar result to `ServiceFilter`, but constructs the filter using `ActivatorUtilities` (which resolves constructor dependencies from DI on the fly) rather than requiring the filter class to be explicitly pre-registered — a convenient, slightly more self-contained alternative when you'd rather not add a separate registration line for every filter class used this way.

---

## 13. IFilterFactory and Custom Filter Attributes with Parameters

### The problem: an attribute needs a constructor parameter that ISN'T itself a DI-resolvable dependency

```csharp
[RequirePermission("orders.delete")] // "orders.delete" is a plain STRING, supplied directly in the attribute
public IActionResult DeleteOrder(int id) { /* ... */ return Ok(); }
```

This is a genuinely common, legitimate pattern — an attribute parameter like a required permission string, a cache duration, or a rate limit isn't something DI resolves; it's configuration data supplied directly at the point the attribute is applied — while the filter's *actual logic* might still need genuine DI-resolved services (an `IPermissionService`, say) to do its job.

### `IFilterFactory`: combining attribute parameters with DI-resolved dependencies

```csharp
public class RequirePermissionAttribute : Attribute, IFilterFactory
{
    private readonly string _permission;
    public RequirePermissionAttribute(string permission) => _permission = permission; // plain attribute constructor arg

    public bool IsReusable => false;

    public IFilterMetadata CreateInstance(IServiceProvider serviceProvider)
    {
        var permissionService = serviceProvider.GetRequiredService<IPermissionService>(); // resolved from DI
        return new RequirePermissionFilter(_permission, permissionService); // combines BOTH sources
    }
}

public class RequirePermissionFilter : IAuthorizationFilter
{
    private readonly string _permission;
    private readonly IPermissionService _permissionService;
    public RequirePermissionFilter(string permission, IPermissionService permissionService)
    {
        _permission = permission;
        _permissionService = permissionService;
    }
    public void OnAuthorization(AuthorizationFilterContext context)
    {
        if (!_permissionService.HasPermission(context.HttpContext.User, _permission))
            context.Result = new ForbidResult();
    }
}
```

`IFilterFactory` is the general mechanism underneath `ServiceFilter`/`TypeFilter` (Section 12) — its `CreateInstance` method receives the request's actual `IServiceProvider` directly, letting you combine attribute-supplied, compile-time-known values (the permission string) with genuinely DI-resolved runtime dependencies (`IPermissionService`) into a single, fully-constructed filter instance. This is the right tool specifically when a filter genuinely needs both kinds of input together, which is common enough in real applications to be worth knowing as its own distinct pattern rather than treating `ServiceFilter`/`TypeFilter` as the only options.

---

## 14. Common Pitfalls

| Pitfall | Why it hurts | Better approach |
|---|---|---|
| Assuming an exception filter catches exceptions from anywhere in the MVC pipeline | Exception filters specifically wrap only action filters and the action itself — exceptions from resource/result/authorization filters bypass them entirely | Understand the precise scope (Section 6); rely on middleware-level exception handling as the universal safety net for everything else |
| Putting resource-specific authorization logic (needing bound data) into an authorization filter | Authorization filters run BEFORE model binding — they cannot see the action's bound arguments | Use a resource filter (running after binding is possible within its scope) or check within the action itself for resource-specific authorization needs |
| Trying to inject DI services directly into a filter ATTRIBUTE's constructor | Attributes are constructed by the CLR's reflection mechanism, entirely outside the DI container's control | Use `ServiceFilter`, `TypeFilter`, or `IFilterFactory` to bridge attribute application with genuine DI resolution (Sections 12-13) |
| Using a synchronous filter interface for genuinely asynchronous work | Forces either fake-synchronous code or a blocking `.Result`/`.Wait()` call, risking this series' async/await guide's deadlock pattern | Implement the async filter interface (`IAsyncActionFilter`, etc.) whenever genuine `await`ed work is involved (Section 8) |
| Assuming a short-circuit (`context.Result` set) skips the ENTIRE remaining pipeline, regardless of filter type | Different filter types skip different amounts — an action filter's short-circuit still lets result filters run against the substituted result | Understand precisely what each filter type's short-circuit actually skips (Section 11) before relying on it to prevent specific downstream behavior |
| Registering a broadly-scoped concern (like logging) at the action level, repeated across many actions | Unnecessary duplication when the concern genuinely applies uniformly across a controller or the whole application | Use controller-level or global registration for concerns that genuinely apply that broadly (Section 9) |
| Confusing filters with middleware for a concern that needs to apply to non-MVC requests too | Filters only run for matched MVC controller actions — a minimal API endpoint or a static file request never triggers them at all | Use middleware (per this series' Middleware guide) for genuinely pipeline-wide concerns; reserve filters for MVC-action-specific needs |
| Overlooking that filter execution order across scopes is "outside-in, then inside-out," not a flat list | Assuming Global/Controller/Action filters all run in registration order alone leads to incorrect expectations about interaction between them | Understand the scope-based nesting (Section 10) — Global wraps Controller wraps Action, exactly like middleware wraps subsequent middleware |

---

## Quick Reference Table

| Filter Type | Interface (sync) | Runs Relative to Model Binding/Action | Typical Use |
|---|---|---|---|
| Authorization | `IAuthorizationFilter` | Before binding, before everything else | Identity/role/policy checks (`[Authorize]`) |
| Resource | `IResourceFilter` | Wraps binding, action, AND result execution | Caching, short-circuiting before binding cost is paid |
| Action | `IActionFilter` | Wraps only the action method itself | Logging, argument inspection/modification, model validation |
| Exception | `IExceptionFilter` | Catches exceptions from action filters + the action | MVC-context-aware error handling, before the middleware-level safety net |
| Result | `IResultFilter` | Wraps result execution (the actual response write) | Adding response headers, transforming the final response shape |
| — | `ServiceFilter(typeof(T))` | — | Applies a DI-registered filter class via attribute |
| — | `TypeFilter(typeof(T))` | — | Applies a filter class with DI-resolved constructor args, without pre-registration |
| — | `IFilterFactory` | — | Combines attribute-supplied parameters with genuine DI-resolved dependencies |

---

## Conclusion

Filters give ASP.NET Core MVC exactly the kind of rich, action-aware extensibility this series' Middleware guide's Section 12 identifies as structurally beyond raw middleware's reach — and the five distinct types exist precisely because "before/after the action" isn't one single moment; it's a whole sequence of increasingly specific points (before binding even happens, immediately around the action, around exceptions the action throws, around the eventual response being written), each offering different context and different short-circuiting reach. Understanding exactly what each type wraps, and exactly what its short-circuit mechanism skips, is what turns "add an `[Authorize]` attribute" from a memorized incantation into something you can reason about precisely — and extend confidently, whether that means writing a custom action filter for cross-cutting logging, a resource filter for response caching, or bridging attribute-supplied configuration with genuine dependency injection via `IFilterFactory`.

The relationship this guide keeps returning to — filters as a nested pipeline running entirely within a single step of the broader middleware pipeline this series' Middleware guide covers — is the key to keeping the two systems straight: middleware for concerns that genuinely apply to every request regardless of what's handling it; filters for concerns that specifically need MVC's own, richer context around action execution. Knowing when each is the right tool, and knowing the five filter types' precise ordering and scope, is what makes ASP.NET Core's cross-cutting-concern machinery something you actively design with, rather than a collection of attributes copied from examples without quite knowing why they're ordered the way they are.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the exception-filter-that-mysteriously-never-fired-because-the-throw-came-from-a-resource-filter debugging session that made the five filter types' precise scoping click far better than any table ever could.*
