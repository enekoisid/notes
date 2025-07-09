# Fine Code Coverage Visual Studio Add-on
- Shows covered and uncovered code directly in the editor.
- Automatically analyzes coverage when running tests.
- Only available for Visual Studio.
- Apparently not working with MSTest.

# .NET Code Coverage with Coverlet
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

# C# Dev Kit VS Code Extension
- Use `F1 -> Test: Run All Tests with Coverage` to run tests with coverage in VS Code.

---

# Code Coverage Problem with Middleware
When testing middleware, you often need to execute it as a separate Task and assert the response from that Task. However, this approach does not reflect code coverage accurately. For example, even if you test all cases with unit tests (such as in transcoders), using this method will always show 0% coverage. This is because the middleware is tested as a standalone Task inside the unit test method, so the coverage tool does not recognize the code as being exercised.


