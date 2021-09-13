# Troubleshooting

## BenchmarkDotNet

You need to be aware of the fact that to ensure process-level isolation BenchmarkDotNet generates, builds and executes every benchmark in a dedicated process. For .NET and Mono we generate a C# file and compile it using Roslyn. For .NET Core and CoreRT we generate not only C# file but also a project file which later is restored and build with dotnet cli. If your project has some non-trivial build settings like a `.props` and `.target` files or native dependencies things might not work well out of the box.

How do you know that BenchmarkDotNet has failed to build the project? BDN is going to tell you about it. An example:

```log
// Validating benchmarks:
// ***** BenchmarkRunner: Start   *****
// ***** Found 1 benchmark(s) in total *****
// ***** Building 1 exe(s) in Parallel: Start   *****
// start dotnet restore  /p:UseSharedCompilation=false /p:BuildInParallel=false /m:1 /p:Deterministic=true /p:Optimize=true in C:\Projects\BenchmarkDotNet\samples\BenchmarkDotNet.Samples\bin\Release\netcoreapp2.1\c6045772-d3c7-4dbe-ab37-4aca6dcb6ec4
// command took 0.51s and exited with 1
// ***** Done, took 00:00:00 (0.66 sec)   *****
// Found 1 benchmarks:
//   IntroBasic.Sleep: DefaultJob

// Build Error: Standard output:

 Standard error:
 C:\Projects\BenchmarkDotNet\samples\BenchmarkDotNet.Samples\bin\Release\netcoreapp2.1\c6045772-d3c7-4dbe-ab37-4aca6dcb6ec4\BenchmarkDotNet.Autogenerated.csproj(36,1): error MSB4025: The project file could not be loaded. Unexpected end of file while parsing Comment has occurred. Line 36, position 1.

// BenchmarkDotNet has failed to build the auto-generated boilerplate code.
// It can be found in C:\Projects\BenchmarkDotNet\samples\BenchmarkDotNet.Samples\bin\Release\netcoreapp2.1\c6045772-d3c7-4dbe-ab37-4aca6dcb6ec4
// Please follow the troubleshooting guide: https://benchmarkdotnet.org/articles/guides/troubleshooting.html
```

If the error message is not clear enough, you need to investigate it further.

How to troubleshoot the build process:

1. Run the benchmarks.
2. Read the error message. If it does not contain the answer to your problem, please continue to the next step.
3. Go to the build artifacts folder (path printed by BDN).
4. The folder should contain: 
   * a file with source code (ends with `.notcs` to make sure IDE don't include it in other projects by default)
   * a project file (`.csproj`)
   * a script file (`.bat` on Windows, `.sh` for other OSes) which should be doing exactly the same thing as BDN does:
     * dotnet restore
     * dotnet build (with some parameters like `-c Release`)
5. Run the script, read the error message. From here you continue with the troubleshooting like it was a project in your solution.

The recommended order of solving build issues:

1. Change the right settings in your project file which defines benchmarks to get it working.
2. Customize the `Job` settings using available options like `job.WithCustomBuildConfiguration($name)`or `job.With(new Argument[] { new MsBuildArgument("/p:SomeProperty=Value")})`.
3. Implement your own `IToolchain` and generate and build all the right things in your way (you can use existing Builders and Generators and just override some methods to change specific behaviour).
4. Report a bug in BenchmarkDotNet repository.

## Debugging Benchmarks

## In the same process

If your benchmark builds but fails to run, you can simply debug it. The first thing you should try is to do it in a single process (host process === runner process).

1. Use `DebugInProcessConfig`

```cs
static void Main(string[] args) => BenchmarkSwitcher.FromAssembly(typeof(Program).Assembly).Run(args, new DebugInProcessConfig());
```

2. Set the breakpoints in your favorite IDE
3. Start debugging the project with benchmarks

## In a different process

Sometimes you won't be able to reproduce the problem in the same process. In this case, you have 3 options:

### Launch a debugger from the benchmark process using Debugger API

```cs
[GlobalSetup]
public void Setup()
{
    System.Diagnostics.Debugger.Launch();
}
```

### Attach a debugger from IDE

Modify your benchmark to sleep until the Debugger is not attached and use your favorite IDE to attach the debugger to benchmark process. **Do attach to the process which is running the benchmark** (the arguments of the process are going to be `--benchmarkId $someNumber --benchmarkName $theName`), not the host process.

```cs
[GlobalSetup]
public void Setup()
{
    while(!System.Diagnostics.Debugger.IsAttached)
        Thread.Sleep(TimeSpan.FromMilliseconds(100));
}
```

### One of the above, but with a Debug build

By default, BDN builds everything in Release. But debugging Release builds even with full symbols might be non-trivial. To enforce BDN to build the benchmark in Debug please use `DebugBuildConfig` and then attach the debugger.
