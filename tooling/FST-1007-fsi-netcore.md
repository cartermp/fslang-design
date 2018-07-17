# F# RFC FST-1027 - FSI in a .NET Core world

* [x] Approved in principle
* [ ] [FSLang Suggestion](https://github.com/fsharp/fslang-suggestions/issues/542)
* [ ] Details: [under discussion](https://github.com/fsharp/fslang-design/issues/167)
* [x] Implementation: [In progress](https://github.com/Microsoft/visualfsharp/pull/2483)

# Summary
[summary]: #summary

This RFC proposes the following:

### Enabling FSI from the .NET CLI

F# Interactive is enabled on the .NET CLI as `dotnet fsi` with appropriate options detailed further in this document.

### Add a `/compilertools` switch to enable FSI extensibility

This switch for the compiler and FSI allows you to supply a compiler extension, such as a package manager, or a type provider, at the command line.

### Make `#r` pluggable in the F# compiler and provide some extensions

A pluggable `#r` allows you to specify a string corresponding to a tool or action that the compiler can invoke.

The following extensions to `#r` are proposed as a part of this:

* `#r "[dependency manager]:<dependency manager command>"`
   * `#r "nuget:<package-name>,<package-version>"` is built-in.
* `#r "project:<project-name>"`

That is, "referencing a package" and "referencing a project" will be default supported mechanisms for `#r` in F# Interactive.

This extends the "language" of `#r`, based on the `/compilertools:<toolpath>` mechanism that specifies, at build time, an extension `.dll` that implements the `#r` functionality.

### Ensure the desktop `fsi.exe` supports it all

In addition to enabling .NET SDK F# interace, these mechanisms (and additional command-line options) will work for the desktop `fsi.exe`.

### Enable sending a `.dll` to fsi in Visual Studio for .NET SDK projects

Bring .NET SDK projects up to parity with legacy F# projects and offer the right-click menu item to send a referenced `.dll` to F# Interactive.

### Enable sending a project to fsi in Visual Studio

Implement a right-click menu item on F# projects (of any flavor) that will send a reference to that project to F# interactive.

### Fully-defined matrix of capabilities for F# scripting in Visual Studio

F# scripting is cross-cutting across project flavors, runtimes, and target frameworks. There are certain combinations of things that work, and certain combinations of things that simply cannot work.

### Template for building NuGet packages with the .NET CLI

??? TODO define better

### Enable the .NET CLI to be activated from within Visual Studio

To run F# scripts in the context of .NET Core for Visual Studio, the .NET CLI will be capable of being activated do that `dotnet fsi` can be used to execute those scripts.

# Motivation
[motivation]: #motivation

Motivation for this change is twofold:

1. There is a [strong desire](https://github.com/fsharp/fslang-suggestions/issues/542) to reference packages via `#r` instead of assemblies.  There has also been [experimental work](https://github.com/Microsoft/visualfsharp/pull/2483) to support Paket in this way.
2. .NET Core uses both Reference Assemblies and Implementation Assemblies, which FSI must understand.  It does not understand this split today.

# Detailed design
[design]: #detailed-design

## Enabling FSI from the .NET CLI

The following options are available, given as if invoked from the command line:
```
$ dotnet fsi -h
Usage: dotnet fsi [options]

-h, --help                       Show help information
-f, --framework <FRAMEWORK>      Run fsi against a supplied target framework
--no-restore                     Do not restore the target framework before building
--force                          Force all dependencies to be resolved even if the last restore was successful
--use:<file>                     Use the given file on startup as initial input
--load:<file>                    #load the given file on startup
--reference:<file>               Reference an assembly (Short form: -r)
--optimize[+|-]                  Enable optimizations (Short form: -O)
--deterministic[+|-]             Produce a deterministic assembly (including module version GUID and timestamp)
--warnaserror[+|-]               Report all warnings as errors
--warnaserror[+|-]:<warn;...>    Report specific warnings as errors
--warn:<n>                       Set a warning level (0-5)
--nowarn:<warn;...>              Disable specific warning messages
--warnon:<warn;...>              Enable specific warnings that may be off by default
--consolecolors[+|-]             Output warning and error messages in color
--checked[+|-]                   Generate overflow checks
--define:<string>                Define conditional compilation symbols (Short form: -d)
--mlcompatibility                Ignore ML compatibility warnings
--nologo                         Suppress compiler copyright message
--codepage:<n>                   Specify the codepage used to read source files
--utf8output                     Output messages in UTF-8 encoding
--preferreduilang:<string>       Specify the preferred output language culture name (e.g. es-ES, ja-JP)
--fullpaths                      Output messages with fully qualified paths
--lib:<dir;...>                  Specify a directory for the include path which is used to resolve source files and assemblies (Short form: -I)
--simpleresolution               Resolve assembly references using directory-based rules rather than MSBuild resolution
--exec                           Exit fsi after loading the files or running the .fsx script given on the command line
--quiet                          Suppress fsi writing to stdout
--readline[+|-]                  Support TAB completion in console (on by default)
```

Supporting this design requires significant changes in how FSI references assemblies.  As stated above, .NET Core (and .NET Standard) introduces a new model by which assemblies are used and laid out on disk.  FSI cannot assume everything will be in the same place as it is in a .NET Framework world.  Furthermore, when interactive with .NET Standard-based things, code is compiled against *Reference Assemblies* and ran against *Implementation Assemblies*.

To begin, existing behavior with `#r` will remain unchanged for FSI.  That is, if you reference an assembly today via `#r`, it will always continue to work in the same way that it always has.

On .NET Core, however, `#r "ref: <path-to-reference-assembly>"` will be used for referencing *Reference Assemblies*.  To handle *Implementation Assemblies*, we use the notion of `#r impl: <path-to-implementation-assembly>`.  This is needed to directly reference implementation assemblies pulled down in packages or in the .NET Core Shared Framework.

This split for .NET Core necessitates the introduction of `#r "nuget: name, version"` and `#r "paket: paket-command"` to simplify referencing dependencies in F# script files for .NET Core.  The intention is that FSI Scripts in the future will always specify dependencies via `#r "nuget` or `#r "paket` (or a future extended command, like `#r "project"`), allowing only FSI to be concerned with referencing the correct assemblies.

There are two primary considerations here: design-time behavior and runtime behavior.

## Design-Time

A few things should happen at design time when using F# Scripting in an editor (VS, VS for Mac, Ionide) which supports IntelliSense.

In the case of referencing a package via a specific tool (e.g., `#r "nuget:..."` or `#r "paket:..."`), F#

1. Look for the tool on the PATH (or some other blessed location).  If it can't find it, error out with a nice error message.
2. Call out to the tool to fetch packages and resolve dependencies.
3. Reference the resolved `.dll`s via `#r` in an ephemeral script.
4. Implicitly `#load` that ephemeral script to the current script.

In the case of referencing a project, the editor will fall back on MSBuild.

### Example - Paket:

```fsharp
#r "paket: nuget Newtonsoft.Json ~> 9.0.1" // Example. All paket features are allowed
```

After the above is typed, the editor will:

1. Check if it can find a registered dependency manager which reacts to the prefix "paket:"
2. If the paket tool is found it will call the `resolve` method on it
    a. Paket will internally check if it needs to restore the dependency or if everything is already in place. Any error from Paket will be shown.
    b. Paket will create an ephemeral script that `#r` all direct and indirect libraries for the specified dependency.
4. Implicitly `#load` the generated script in the current script.
   
After everything is fetched, resolved, and loaded, types from `Newtonsoft.Jon` will be available after `open`ing the namespace.

### Example - NuGet:

```fsharp
#r "nuget: Newtonsoft.Json, 9.0.1" // Example version, could be anything
```

After the above is typed, the editor will:

1. Check to see if the specified dependency is already in the `packages` folder on the given machine.

    a. If the exist, create an ephemeral script which loads the `.dll` with `#r`, and then implicitly `#load` that script into the current one.

2. If there is no such folder or dependency, will look for NuGet on the PATH (or some other blessed location).
3. Call out to NuGet to fetch and resolve the dependency.
4. Create an ephemeral script and `#r` the assemblies that NuGet just resolved.  Any error from NuGet will be shown.
5. Implicitly `#load` the generated script into the current script.

### Example - Projects:

```fsharp
#r "project: MyProject"
```

After the above is typed, the editor will:

1. Check to see if the containing project already has a reference to `MyProject.dll`.

    a. If it does not find it, it will call out MSBuild (from the PATH or other blessed location - if it can't find it, error) for MSBuild to build that project.
    b. Once it has `MyProject.dll`, it will reference it via `#r` in an ephemeral script.

2. Implicitly `#load` the ephemeral script into the current script in the editor.

## Runtime

Behavior should not be so different at runtime.  When executing code interactive from a design-time scripting session, things are already resolved, so there is no extra consideration here - FSI has all the information it needs to execute code.

### Starting FSI with the script

When launching FSI with a script using one of the new `#r` references, FSI will do the following:

1. Check which tools is registered for a given #r prefix.  Error out if no tool can be found.
2. Let the tool check if each dependency exists in the known place for the specified tool or otherwise the tool resolve all dependencies.
3. `#r` the resolved dependencies in an ephemeral script (or the script generated by Paket if using Paket).
4. `#load` that script into the FSI session, thus giving it the references it needs to execute.

### Running a script in an active FSI session

This scenario is as follows: I have launched FSI at some point, and now I wish to execute a script which may or may not have references that I already have loaded into FSI.  The behavior should actually be the same as if FSI were launched with the script:

1. Check which tools is registered for a given #r prefix.  Error out if no tool can be found.
2. Call out to the tool to resolve dependencies.
3. `#r` the resolved dependencies in an ephemeral script (or the script generated by Paket if using Paket).  Check version numbers against what we already have loaded in the FSI session.  If any `.dll` names and versions match, don't `#r` them into the script.
4. `#load` that script into the FSI session, thus giving it the references it needs to execute.

This is because:

1. A reference may already be loaded for the current FSI session, but a different version was specified in that script.  We need to allow that to error out, because the script code may depend on a higher or lower version than what is already loaded.  That could cause a runtime error if we just allowed it to run on whatever we had already loaded.
2. It's too complicated to attempt to only partially load stuff if something is already referenced.

### Referencing a new dependency in an active FSI session

This is the interactive scenario.  Interactively in an active FSI session, someone types, for example, `#r "paket: nuget Newtonsoft.Json"`.  Behavior should be identical to when running a script in an active FSI session:

1. Check if each dependency to be loaded exists in the known place for the specified tool.  If it does, `#r` that dependency when executing FSI.
2. If a dependency does not exist, search for the tool in a known/blessed area.  Error out if it can't be found.
3. Call out to the tool to resolve dependencies.
4. `#r` the resolved dependencies in an ephemeral script (or the script generated by Paket if using Paket).  Check version numbers against what we already have loaded in the FSI session.  If any `.dll` names and versions match, don't `#r` them into the script.
5. `#load` that script into the FSI session, thus giving it the references it needs to execute.

As above, if they somehow specify something which is already loaded, we should just let FSI attempt to load it and generate the error.

## Errors

FSI will have to display errors from the underlying assembly resolver.  For example, if someone specifies `#r "paket: <paket-command>"` and Paket generates an error, FSI must display that.

There is also another scenario to consider, which we will explicitly not handle.  For example, given the following:

```fsharp
#r "nuget: Foo.Bar" // Depends on System.Whatever version 4.1.0
#r "nuget: Baz.Qux" // Depends on System.Whatever version 4.2.0

// some scripting code
```

This will generate an error because `System.Whatever` version `4.1.0` will already be loaded.  It's not possible to load two versions of the same assembly in the same session without doing so within an `AssemblyLoadContext`, which is not something we will do at this time.  Thus, a meaningful error must be generated stating that the `nuget: Baz.Qux` requires `System.Whatever` version `4.2.0`, but version `4.1.0` was already loaded.

We may consider using `AssemblyLoadContext` in some clever way in the future, but it's very difficult to get right and well outside the scope of what we need to support for a .NET Core 2.0 timeframe.

## Handler resolution

FSI/Design time support will look at the following places in order:

* if the script is a physical file, check current folder and browse all parent folders looking for .fsharp/fsx-extensions folder in each one
* look into ~/.fsharp/fsx-extensions
* look into .fsharp/fsx-extensions folder next to fsi.exe

gather all the distinct dll names, order of precedence favorising those in the same order shown above, and load them in the process if their assembly contains an arbitrary attribute (resolved by name rather than dependency on external library) and types marked with same attributes.

On .NET Framework, the dll are loaded through `Assembly.LoadFrom`.

On .NET Core the loading mechanism is yet to be determined (**TBD**), although the extensions will need to be targetting .NET Standard 1.6 or higher.

The fact that those dll will be loadable for both .NET Core and .NET Framework compilers  is yet to be determined (**TBD**).

In context of tooling, the location are scanned initially once per interactive session, but as evaluation of additional script occurs, it might be necessary to scan additional locations (same rules apply, but it will only scan the additional places from first point in the list above, and won't scan again those folders already scanned).

If a handler key (such as `nuget` or `paket`) is found several times, report a warning showing location of assemblies and showing which one was picked (we apply same order of precedence as for finding the assemblies).

# Drawbacks
[drawbacks]: #drawbacks

A potential for increased complexity in Scripting.  This also has downstream effects in documentation and the general mindshare on how to reference assemblies for F# scripting.

Also, using `#r "impl:"` (or `#r-impl`) in general should be discouraged since it's more of a way to implement a mechanism required for .NET Core.  Having a new directive or capability that people *could* use, but *shouldn't* use, always opens things up for abuse.

# Alternatives
[alternatives]: #alternatives

Another alternative is to add more directives, such as the following:

* `#r-impl "<path-to-impl-assembly>`
* `#r-nuget "<package-name>, <package-version>`
* `#r-paket "<paket-command>: <package-or-dependency>`
* `#r-project "<projectname>"`

A drawback here is that this is not something that C# would likely do in C# scripting.  Although the goal isn't to align with whatever C# is doing, differentiating on directives doesn't seem worth it unless a new directive is added for each *type* of operation we wish to perform.  Having a new directive for one type of operation (e.g., `#r-impl`), but no new directive for the rest would be strange and inconsistent.

# Unresolved questions
[unresolved]: #unresolved-questions


* One technical concern about this mechanism is that the F# scripting model implementation currently has a set of default references to a bunch of DLLs. For facade assemblies these can resolve to, say, System.IO 4.1.1.0.  But later packages may need later versions of facade DLLs such as System.IO.  It seems that the package manage should be given the opportunity to decide what to do with these default references  so that a more up-to-date set of references can be determined at scripting engine startup

* The discussion here is relevant: https://github.com/Microsoft/visualfsharp/pull/3307#issuecomment-313856347

* Should we be adding a ``#r-typeprovider`` facility for package managers to decide to reference type providers independent of any specific runtime DLL

* Comment here:

> At the high level I'm just wondering if we can use .targets/.props in the nuget package to compute the relevant type provider references, and incorporate this into the #r "nuget: Foo" and/or #r "paket: Foo" mechanism for incrementally added references during scripting.



