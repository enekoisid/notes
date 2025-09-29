## 1. Create the test project correctly

From the terminal, at the root of your solution:

```sh
dotnet new mstest -n Tests
```

This creates a test project already prepared for MSTest.

---

## 2. Test references: put them in the main project!

**Very important:**

- The references to MSTest and `Microsoft.NET.Test.Sdk` should go in the `.csproj` file of the main project (for example, `TranscoderCoreMS.csproj`).
- The `.csproj` file of the test project should be as clean as possible, only referencing the main project.

**Unit testing package references:**


```xml
<ItemGroup>
  <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.1.0" />
  <PackageReference Include="MSTest" Version="2.2.10" />
  <PackageReference Include="MSTest.TestAdapter" Version="2.2.10" />
  <PackageReference Include="MSTest.TestFramework" Version="2.2.10" />
  <PackageReference Include="coverlet.collector" Version="3.1.0" />
  <!-- ...other dependencies... -->
</ItemGroup>
```
---

### 3. Avoid multiple main aplication exception

In main .csproj's `<PropertyGroup>` add 
```XML
<StartupObject>project.Program</StartupObject>
```

---

## 4. Restore and test

Restore packages and run the tests:

```sh
dotnet restore
dotnet test
```

