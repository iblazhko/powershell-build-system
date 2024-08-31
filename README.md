# Build System with PowerShell

This is a simple 1-file build system template for C#/F# solutions implemented in
PowerShell.

This build system expects following solution layout:

```
solution_root/
  build.ps1                  -- PowerShell build CLI
  docker-compose.yaml        -- Docker Compose definition of the complete environment, including all required infrastructure
  src/
    Project1/
      Project1.csproj        -- project base filename matches directory name
    Project1.Tests/
      Project1.Tests.csproj  -- tests projects are xUnit-based; project name must have suffix '.Tests'
    Project2/
      Project2.fsproj
      imagename.Dockerfile   -- if the `<imagename>.Dockerfile` is present, we'll build Docker image `<imagename>`
    Project2.Tests/
      Project2.Tests.fsproj
    SolutionName.sln         -- only one '.sln' file in 'src'
  version.yaml               -- core part of solution SemVer
```

### Prerequisites

-   .NET SDK: <https://dotnet.microsoft.com/download>
-   PowerShell: <https://github.com/powershell/powershell>
-   Docker with Docker Compose: <https://www.docker.com/>,
    or Podman <https://podman.io/> with
    [`docker` wrapper](https://podman-desktop.io/docs/migrating-from-docker/emulating-docker-cli-with-podman)

### Installation

-   Copy [`build.ps1`](./build.ps1) file to the root of your solution
-   Make sure that the file is executable. On macOS/Linux:
    ```bash
    chmod +x build.ps1
    ```
    On Windows it can be invoked as-is locally, but if you will be using it e.g.
    in GitHub Actions, make it executable before committing:
    ```bash
    git add build.ps1
    git update-index --chmod=+x build.ps1
    ```
-   For cross-platform compatibility (e.g. if your local dev environment is
    Windows but CI/CD uses Linux-based build agents), it is recommended to
    use `LF` line endings. Add following lines to your `.gitattributes`:
    ```
    * text=auto
    *.ps1 text eol=lf
    ```
-   Edit the `build.ps1` file to add / remove build targets (see "Build Targets"
    section below), and possibly change the approach to versioning,
    change testing framework, etc., as needed

### Usage

```bash
./build.ps1 <Target> [-Configuration Debug|Release] [-VersionSuffix <suffix>]
```

Examples:

```bash
./build.ps1 Dotnet.Test
./build.ps1 Dotnet.Build -Configuration Release -VersionSuffix 240831
./build.ps1 DockerCompose.StartDetached
```

Sample output:

```
$ ./build.ps1 DotNet.Build

*** BUILD: DotNet.Build (Release) in /Users/iblazhko/Projects/DDD-CQRS-ES/cqrs-fsharp
--- PRELUDE: .NET CLI

--- TARGET: Dotnet.Restore
--- STEP: dotnet restore /Users/iblazhko/Projects/DDD-CQRS-ES/cqrs-fsharp/src/CQRS.sln --verbosity quiet -nologo

--- TARGET: Dotnet.Build
--- STEP: dotnet build /Users/iblazhko/Projects/DDD-CQRS-ES/cqrs-fsharp/src/CQRS.sln --no-restore --configuration Release --verbosity quiet -nologo /p:Version=0.1.0

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:00.93

DONE
*** Completed in: 00:00:01.7404229
```

### Build Targets

Convention used in this build system is that we have a set of build targets
and a build target may depend on another target.

E.g. to only compile the code, we need to do

-   `dotnet restore`
-   `dotnet build`

and to run tests, we need to do

-   `dotnet restore`
-   `dotnet build`
-   `dotnet test`

We can implement this with a set of following targets:

-   `Dotnet.Restore`
-   `Dotnet.Build` (depends on `Dotnet.Restore`, uses `--no-restore`)
-   `Dotnet.Test` (depends on `Dotnet.Build`, uses `--no-build`)

Naming conventions used in this build system:

-   targets are implemented as a function with internal prefix `Target_`
    followed by target name (`.` replaced with `_`), e.g.
    function `Target_Dotnet_Build` implements `Dotnet.Build` target
-   each target may have one ore more steps implemented as separate functions
    with `Step_` prefix
-   targets dependencies are declared with `DependsOn "<Target.Name>"` as a
    first statement in the target definition, e.g.
    ```
    function Target_Dotnet_Test {
        DependsOn "Dotnet.Build"
        ...
    ```
-   you will see two internal targets `Prelude.DotnetCli` and
    `Prelude.DockerCli` that check if `dotnet` or `docker` CLI is available,
    and will fail the build process immediately if the respective CLI is
    not available.

Following targets are available in this template:

-   `Dotnet.Restore`
-   `Dotnet.Build`
-   `Dotnet.Test`
-   `Dotnet.Publish`
-   `FullBuild` (default target,
    `Dotnet.Build` + `Dotnet.Test` + `Dotnet.Publish`)
-   `Docker.Build`
-   `DockerCompose.Start`
-   `DockerCompose.StartDetached`
-   `DockerCompose.Stop`
-   `Prune.Build`
-   `Prune.Docker`
-   `Prune` (`Prune.Build` + `Prune.Docker`)
