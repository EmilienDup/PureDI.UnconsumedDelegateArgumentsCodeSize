# Pure.DI: the graph under nested `Func<T1, T>` delegates is duplicated for every nesting of arguments that nothing consumes

Minimal repro for Pure.DI 2.5.7 (and master at adaacfe): a chain of 11 view models, each opening the next through two delegates with different argument types, generates 20,800 lines and constructs the last view model 1,024 times. The same graph with argument-less `Func<T>` delegates generates about 400 lines and constructs it once. Nothing in the graph consumes the delegate arguments.

[`ISSUE.md`](ISSUE.md) is the text to paste into a Pure.DI GitHub issue; it includes the measurements from 8 to 14 levels.

```
dotnet build
```

The generated composition is written to `obj/Debug/net10.0/generated/Pure.DI/Pure.DI.SourceGenerator/Composition.g.cs`; `grep -c "global::Vm10(" Composition.g.cs` gives 1024.

To compare with argument-less delegates, replace `Func<ArgA, VmN>` and `Func<ArgB, VmN>` with `Func<VmN>` everywhere (including `Top`). Add `Vm11`, `Vm12`... to see the size double per level.

To try another package version: `dotnet build -p:PureDIVersion=<version>` (add `-p:RestoreAdditionalProjectSources=<folder>` for a locally built package).
