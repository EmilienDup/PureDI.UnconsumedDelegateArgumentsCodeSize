# The graph under nested `Func<T1, T>` delegates is duplicated for every nesting of arguments that nothing consumes: generated code doubles per level

**Pure.DI 2.5.7** and **master (adaacfe)**, .NET SDK 10.0.403, macOS. Repro project attached (`dotnet build`, about 4 s, 20,800 generated lines).

When delegates with arguments are nested, the graph under each delegate is generated again for every combination of delegate arguments above it, even when nothing in that graph consumes those arguments. The generated code and the generation time double with every level.

## Minimal code

Every view model opens the next one through two delegates with different argument types. A chain of singletons is shared by all view models. No class consumes `ArgA` or `ArgB`.

```csharp
using Pure.DI;

var composition = new Composition();
Console.WriteLine(composition.Root.GetType().Name);

public sealed record ArgA(string Value);
public sealed record ArgB(string Value);

public sealed class S0;
public sealed class S1(S0 previous);
// ... same pattern ...
public sealed class S9(S8 previous);

public sealed class Vm0(S9 services, Func<ArgA, Vm1> openWithA, Func<ArgB, Vm1> openWithB);
public sealed class Vm1(S9 services, Func<ArgA, Vm2> openWithA, Func<ArgB, Vm2> openWithB);
// ... same pattern ...
public sealed class Vm9(S9 services, Func<ArgA, Vm10> openWithA, Func<ArgB, Vm10> openWithB);
public sealed class Vm10(S9 services);
public sealed class Top(Func<ArgA, Vm0> open);

partial class Composition
{
    private static void Setup() => DI.Setup(nameof(Composition))
        .Bind<S0>().As(Lifetime.Singleton).To<S0>()
        // ... every Sk as Singleton ...
        .Bind<S9>().As(Lifetime.Singleton).To<S9>()
        .Root<Top>("Root");
}
```

## Expected

The same generated code as with argument-less delegates (replace every `Func<ArgA, VmN>` / `Func<ArgB, VmN>` with `Func<VmN>`): about 400 lines, one construction of each view model, generated in 2 s for every size below. The overrides of `ArgA` and `ArgB` reach the whole graph under the delegate, but nothing there consumes them, so the rewritten graph is the same whichever delegates lead to it.

## Actual

`levels` = number of view models with two delegates (`Vm0` to `Vm(levels-1)`); the attached project has 10.

| levels | 2.5.7 | master (adaacfe) | generated lines (both) | constructions of the last view model |
|---|---|---|---|---|
| 8 | 2 s | 2 s | 5,440 | 256 |
| 10 (attached) | 3 s | 7 s | 20,800 | 1,024 |
| 12 | 7 s | 78 s | 82,237 | 4,096 |
| 13 | 13 s | 327 s | 164,157 | 8,192 |
| 14 | 25 s | | 327,997 | 16,384 |
| 12, argument-less `Func<VmN>` | | 2 s | 408 | 1 |

The singletons are created once in one `Ensure...Exists()` function. What doubles is the delegate lambdas: each one inlines the whole graph of the next view model, including its own two delegates, so `VmN` is constructed 2^N times.

(The steeper time on master is a separate regression, reported in its own issue.)

## Real-world impact

On a real application (about 2,500 bindings, view models created through `Func<...>` factories carrying contexts, options and navigation services, over a shared service layer), the generator never finishes on 2.5.7 or master. We added logging to `GraphOverrider.Override` on master. After 47 s it had made 1.5 million calls and stored 1.4 million rewritten nodes. Singleton services that consume no delegate argument (formatters, a barcode service, a team service) had each been rewritten 15,000 to 21,000 times, once per distinct chain of delegates above them. A typical path at that point went through 14 nested delegates with arguments.
