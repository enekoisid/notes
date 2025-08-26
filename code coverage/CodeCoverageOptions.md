# Options available

## Fine Code Coverage Visual Studio Add-on
- Shows covered and uncovered code directly in the editor.
- Automatically analyzes coverage when running tests.
- Only available for Visual Studio.
- Apparently not working with MSTest.

## .NET Code Coverage with Coverlet
1. Install the .NET ReportGenerator global tool:
   ```powershell
   dotnet tool install -g dotnet-reportgenerator-globaltool
   ```
2. Add the Coverlet MSBuild package to your test project:
   ```powershell
   dotnet add package coverlet.msbuild
   ```
3. Run your tests with coverage enabled:
   ```powershell
   dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=lcov /p:CoverletOutput=lcov.info
   ```

## C# Dev Kit VS Code Extension
- Use `F1 -> Test: Run All Tests with Coverage` to run tests with coverage in VS Code.



# **Info**  
> The third approach is the best option, as it provides the most comprehensive coverage without introducing potential issues or side effects.


