Makes MarkDown documentation from reflection and XML, from reflection for gathering all types and their member data and XML for documentation (metyod/property comments)

To use the xml, enable XML documentation by adding to `.csproj`:
   ```xml
   <PropertyGroup>
     <GenerateDocumentationFile>true</GenerateDocumentationFile>
     <DocumentationFile>bin\$(Configuration)\$(TargetFramework)\$(AssemblyName).xml</DocumentationFile>
   </PropertyGroup>
   ```

to use
1. git clone https://github.com/hailstorm75/ModularDoc.git
2. dotnet build src/ModularDoc.sln --configuration Release
3. run bin/ModularDoc.App.exe

There is also ModularDoc.CLI.exe, which is only console-based application. You supply a path to the config created in ModularDoc.App

Supports C# <=9