# Getting started with .NetCore/.NetStandard Analyzers

1. Read through the [.NET Compiler Platform SDK](https://learn.microsoft.com/dotnet/csharp/roslyn-sdk/) for understanding the different Roslyn elements `(Syntax Nodes, Tokens, Trivia)`. The factory methods and APIs are super useful.  
2. Learning this [tutorial](https://learn.microsoft.com/dotnet/csharp/roslyn-sdk/tutorials/how-to-write-csharp-analyzer-code-fix) for custom analyzer and trying it some level is very useful to get started. It is pretty easy step by step tutorial, it is time saving as it has a template generated for us (with analyzer, fixer and unit test), has good explanation, would give you pretty good understanding on how Roslyn analyzers work.
3. Clone the `dotnet/roslyn-analyzers` repo, install all required dependencies and build the repo by the [instructions](https://github.com/dotnet/roslyn-analyzers#getting-started).
4. Follow the coding style of the `dotnet/roslyn-analyzers` repo. [Guidelines about new rule id and doc](https://github.com/dotnet/roslyn-analyzers/blob/main/GuidelinesForNewRules.md).
5. Open `RoslynAnalyzers.sln` and open the package where you are creating your analyzer. In our case, it is mostly `Microsoft.CodeAnalysis.NetAnalyzers`->`Microsoft.NetCore.Analyzers`. Create your analyzer and/or fixer class in the corresponding folder.  
6. Add a message, title and description for your analyzer into `MicrosoftNetCoreAnalyzersResources.resx` and build the repo before using them, the language specific resources will be generated.
7. Make sure you have done everything from the [Definition of done list](#definition-of-done) below.

## Branch Definitions

|Branch| SDK | Description|
|--------|--------|--------|
|[2.9.x](https://github.com/dotnet/roslyn-analyzers/tree/2.9.x)| Does not ship in the .NET SDK | A special branch compatible with Visual Studio 2017 where security analyzers are shipped from.
|[main](https://github.com/dotnet/roslyn-analyzers/tree/main)| .NET SDK 7.0.0xx  | Currently active branch. All work should target this branch unless it is a bugfix for a previous release
|[release/5.0.3xx](https://github.com/dotnet/roslyn-analyzers/tree/release/5.0.3xx)| .NET SDK 5.0.3xx | Servicing branch for the .NET 5 SDK.
|[release/6.0.1xx](https://github.com/dotnet/roslyn-analyzers/tree/release/6.0.1xx)| .NET SDK 6.0.0xx | Servicing branch for the .NET 6 SDK. Currently accepting targeting fixes until the .NET 6 SDK ships

## Definition of done

- Analyzer implemented to work for C# and VB.
  - Unit tests for C#:
    - All scenarios covered.
      - Prefer markup syntax for the majority of tests.
      - If your analyzer has placeholders in the diagnostic message and you want to test the arguments, write a smaller number of tests using the `VerifyCS.Diagnostic` syntax to construct specific diagnostic forms.
    - Unit tests for VB:
      - Obvious positive and negative scenarios covered.
      - If the implementation uses any syntax-specific code, then all scenarios must be covered.
- Fixer implemented for C#, using the language-agnostic APIs if possible.
  - If the fixer can be entirely implemented with language-agnostic APIs `(IOperation)`, then VB support is essentially free.
  - With a language-agnostic fixer, apply the attribute to indicate the fixer also applies to VB and add mainline VB tests.
  - If language-specific APIs are needed to implement the fixer, the VB fixer is not required.
  - Do not separate analyzer tests from code fix tests. If the analyzer has a code fix, then write all your tests as code fix tests.
    - Calling `VerifyCodeFixAsync(source, source)` verifies that the analyzer either does not produce diagnostics, or produces diagnostics where no code fix is offered.
    - Calling `VerifyCodeFixAsync(source, fixedSource)` verifies the diagnostics (analyzer testing) and verifies that the code fix on source produces the expected output.
- Run the analyzer locally against `dotnet/runtime` and `dotnet/roslyn-analyzers` [instructions](#Testing-against-the-Runtime-and-Roslyn-analyzers-repo).
  - Review each of the failures in those repositories and determine the course of action for each.
  - Use the failures to discover nuance and guide the implementation details.
  - Run the analyzer against `dotnet/roslyn` [instruction](#Testing-against-the-Roslyn-repo), and if feasable with `dotnet/aspnetcore` repos.
  - Document for review: matching and non-matching scenarios, including any discovered nuance.
  - All warnings and errors in these repos are addressed (to prevent build failures)
    - `Info` level diagnostics do not need to be fully resolved or suppressed as they do not cause build failures
- Document for review: severity, default, categorization, numbering, titles, messages, and descriptions.
- Create the appropriate documentation for [docs.microsoft.com](https://github.com/dotnet/docs/tree/main/docs/fundamentals/code-analysis/quality-rules) within **ONE WEEK**, instructions available on [Contribute docs for .NET code analysis rules to the .NET docs repository](https://learn.microsoft.com/contribute/dotnet/dotnet-contribute-code-analysis).
- PR merged into `dotnet/roslyn-analyzers`.
- Validate the analyzer's behavior with end-to-end testing using the command-line and Visual Studio:
  - Use `dotnet new console` and `dotnet build` from the command-line, updating the code to introduce diagnostics and ensuring warnings/errors are reported at the command-line
  - Use Visual Studio to create a new project, introduce diagnostics, and observe the warnings/errors/info messages without invoking a build

## Testing against the Runtime and Roslyn-analyzers repo

1. Navigate to the root of the Roslyn-analyzers repo and run these commands:
    - `cd roslyn-analyzers`
    - Set RUNTIMEPACKAGEVERSION variable with a version value which major part is equal to the major part of the version of  [runtime](https://github.com/dotnet/runtime/blob/main/eng/Versions.props#L53)/[roslyn-analyzers](https://github.com/dotnet/roslyn-analyzers/blob/main/eng/Versions.props#L50) repo is using, example: `set RUNTIMEPACKAGEVERSION=7.0.0`
    - `build.cmd -ci /p:AssemblyVersion=%RUNTIMEPACKAGEVERSION% /p:AutoGenerateAssemblyVersion=false /p:OfficialBuild=true -c Release`
    - `cd artifacts\bin\Microsoft.CodeAnalysis.CSharp.NetAnalyzers\Release\netstandard2.0`
2. Copy the two DLLs and replace the NuGet cache entries used by `dotnet/runtime` and `dotnet/roslyn-analyzers`. They might be in `"runtime/.packages/..."` or `"%USERPROFILE%/.nuget/packages/... "`. You can check the exact path by building something in runtime with /bl and checking the binlog file.
    - Example: `copy /y *.dll %USERPROFILE%\.nuget\packages\Microsoft.CodeAnalysis.NetAnalyzers\%RUNTIMEPACKAGEVERSION%\analyzers\dotnet\cs`
    - Note that `RUNTIMEPACKAGEVERSION` value is different for runtime and roslyn-analyzers repo
3. Build the roslyn-analyzers with `build.cmd`, now new analyzers will be used from updated nuget packages and you would see the warnings if diagnostics found.
4. If failures found, review each of the failures and determine the course of action for each.
    - Improve analyzer to reduce false positives, fix valid warnings, in a very rare edge cases suppress them.
5. Make sure all failures addressed and corresponding PR(s) merged.
6. Switch to the runtime repo.
7. Add a row for your new analyzer ID with a value of `warning` to make sure it would warn for findings in [CodeAnalysis.src.globalconfig](https://github.com/dotnet/runtime/blob/main/eng/CodeAnalysis.src.globalconfig) file. For example if you are authored a new analyzer with id `CA1234` add a row: `dotnet_diagnostic.CA1234.severity = warning`
8. Build the runtime repo, either do a complete build or build each repo separately (coreclr, libraries, mono).
9. In case no any failure introduce an error somewhere to prove that the rule ran.
    - Be careful about in which project you are producing an error, choose an API not having reference from other APIs, else all dependent API's will fail.
10. If failures found, repeat step 4-5 to evaluate and address all warnings.
    - In case you want to [debug some failures](#debugging-analyzer-with-runtime-repo-projects).

## Testing against the Roslyn repo

1. Clone `dotnet/roslyn` and build it with this command:
    - `Build.cmd -restore -Configuration Release`
2. Build `dotnet/roslyn-analyzers` in debug mode:
    - `Build.cmd -Configuration Debug`
3. Run AnalyzerRunner from the Roslyn root directory to get the diagnostics.
    - `.\artifacts\bin\AnalyzerRunner\Release\netcoreapp3.1\AnalyzerRunner.exe ..\roslyn-analyzers\artifacts\bin\Microsoft.NetCore.Analyzers.Package\Debug\netstandard2.0 .\Roslyn.sln /stats /concurrent /a AnalyzerNameToTest /log Output.txt`
    - Do not forget change value after `/a` option with your testing analyzer name.
The diagnostics reported by the analyzer will be listed in Output.txt.

## Debugging analyzer with runtime repo projects

1. Copy over debug build of analyzer assemblies on top of NetAnalyzers nuget package in your packages folder. (Instructions are same as the step 1 and 2 of [Testing against the Runtime repo step](#testing-against-the-runtime-and-roslyn-analyzers-repo))
2. Start VS and open a project you want to debug
3. Note the process ID for `ServiceHub.RoslynCodeAnalysisService.exe` corresponding to that VS instance
    - If you are using `Visual Studio` older than version `16.8 Preview2` then analyzers run in `devenv.exe`, you will need to attach that process instead
    - Code fixes and analyzers run in different processes. If you want to debug the CodeFixProvider corresponding to the analyzer, attach `devenv.exe` instead.
4. Open another VS instance for `RoslynAnalyzers.sln` and set breakpoints in the analyzer solution where you want to debug
5. Attach to above process ID with the RoslynAnalyzers debugger: `Debug -> Attach to Process...`
6. Start typing in the other project, the breakpoints should hit
    - If breakpoints are not hitting then the RoslynAnalyzers.sln build might not the same as the build you copied to the step 1 repeat the step again or check if you copied into the correct path
