# Cómo crear un proyecto de Unit Testing en C#

## 1. Crear el proyecto de testing

Desde la terminal, en la raíz de tu solución:

```sh
dotnet new mstest -n Tests
```

Esto crea un proyecto de testing preparado para MSTest.

---

## 2. Referencias de testing: ponerlas en el proyecto de testing

**Muy importante:**

- Las referencias a MSTest y `Microsoft.NET.Test.Sdk` deben ir en el archivo `.csproj` del **proyecto de testing** (por ejemplo, `Tests.csproj`).
- El archivo `.csproj` del proyecto de testing debe referenciar el proyecto principal que quieres testear.

**Estructura del proyecto de testing `.csproj`:**

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <!-- Referencias de testing -->
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.1.0" />
    <PackageReference Include="MSTest" Version="2.2.10" />
    <PackageReference Include="MSTest.TestAdapter" Version="2.2.10" />
    <PackageReference Include="MSTest.TestFramework" Version="2.2.10" />
    <PackageReference Include="coverlet.collector" Version="3.1.0" />
  </ItemGroup>

  <ItemGroup>
    <!-- Referencia al proyecto principal que quieres testear -->
    <ProjectReference Include="..\TuProyectoPrincipal\TuProyectoPrincipal.csproj" />
  </ItemGroup>

</Project>
```

**Nota:** El proyecto principal NO debe tener referencias a las librerías de testing. Solo el proyecto de testing las necesita.

---

## 3. Evitar excepción de múltiples aplicaciones principales

En el `.csproj` del proyecto principal, dentro de `<PropertyGroup>`, añade:

```xml
<StartupObject>TuProyectoPrincipal.Program</StartupObject>
```

Esto evita conflictos cuando hay múltiples proyectos con métodos `Main`.

---

## 4. Restaurar paquetes y ejecutar tests

Restaura los paquetes y ejecuta los tests:

```sh
dotnet restore
dotnet test
```

Para ejecutar tests de un proyecto específico:

```sh
dotnet test Tests/Tests.csproj
```

---

## 5. Estructura de ejemplo

```
Solution/
├── TuProyectoPrincipal/
│   ├── TuProyectoPrincipal.csproj  (sin referencias de testing)
│   └── Program.cs
└── Tests/
    ├── Tests.csproj                 (con referencias de testing + referencia al proyecto principal)
    └── UnitTest1.cs
```

---

## 6. Code Coverage con Coverlet

Para generar code coverage en Visual Studio Code usando Coverlet:

1. **Instala el ReportGenerator global tool:**
   ```sh
   dotnet tool install -g dotnet-reportgenerator-globaltool
   ```

2. **Añade el paquete Coverlet MSBuild al proyecto de testing:**
   ```sh
   cd Tests
   dotnet add package coverlet.msbuild
   ```

3. **Ejecuta los tests con coverage habilitado:**
   ```sh
   dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=lcov /p:CoverletOutput=lcov.info
   ```

Esto generará un archivo `lcov.info` con los datos de coverage que puedes visualizar con extensiones de VS Code con `Coverga gutters` (ryanluker.vscode-coverage-gutters)

