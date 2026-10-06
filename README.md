# 🚀 CI/CD на C#/.NET CLI с публикацией бинарников в GitHub Releases

📦 Сборка **.NET CLI** в один `self-contained` исполняемый файл.

- ⚡ **.NET** умеет cross-compilation — один runner собирает под все платформы (как Go/Rust).
- 🏷️ **GitHub Releases** — раздел репозитория для публикации версий (теги + файлы).

🎯 **Цель:** автоматически собирать бинарники .NET CLI под 5 платформ (Linux, macOS, Windows × 2 архитектуры) и публиковать их в GitHub Releases при push тега `v*`.

🔍 **Ключевое отличие от Python:**
- 🐍 Python — для каждой ОС свой runner (нет cross-compilation)
- 🟣 .NET — один runner собирает под все платформы через `-r <RID>`

---

## 1. 🗂️ Структура проекта

```text
hello-dotnet/
├── .github/workflows/ci.yml
├── src/
│   ├── HelloDotnet.csproj
│   ├── Greeting.cs
│   └── Program.cs
├── tests/
│   ├── HelloDotnet.Tests.csproj
│   └── GreetingTests.cs
└── .gitignore
```

```shell
mkdir -p hello-dotnet/{.github/workflows,src,tests} && \
cd hello-dotnet && \

cat > src/HelloDotnet.csproj << 'EOF'
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <AssemblyName>hello-dotnet</AssemblyName>
    <RootNamespace>HelloDotnet</RootNamespace>
    <InvariantGlobalization>true</InvariantGlobalization>
    <Version>0.1.0</Version>
  </PropertyGroup>

</Project>
EOF

cat > src/Greeting.cs << 'EOF'
namespace HelloDotnet;

public static class Greeting
{
    public static string Greet(string name)
    {
        return $"Hello, {name}!";
    }

    public static int SumRange(int from, int to)
    {
        int sum = 0;
        for (int i = from; i <= to; i++)
        {
            sum += i;
        }
        return sum;
    }
}
EOF

cat > src/Program.cs << 'EOF'
using System.Reflection;
using System.Runtime.InteropServices;
using HelloDotnet;

var version = Assembly.GetExecutingAssembly()
    .GetCustomAttribute<AssemblyInformationalVersionAttribute>()?
    .InformationalVersion
    .Split('+')[0] ?? "unknown";

Console.WriteLine($"hello-dotnet version {version}");
Console.WriteLine("Hello from C# in GitHub Actions! 🚀📦");
Console.WriteLine($"OS: {RuntimeInformation.OSDescription}");
Console.WriteLine($"Arch: {RuntimeInformation.OSArchitecture}");
Console.WriteLine(Greeting.Greet("GitHub"));
Console.WriteLine($"Sum 1..10 = {Greeting.SumRange(1, 10)}");

if (args.Length > 0)
{
    Console.WriteLine("Аргументы:");
    for (int i = 0; i < args.Length; i++)
    {
        Console.WriteLine($"  {i + 1}: {args[i]}");
    }
}

return 0;
EOF

cat > tests/HelloDotnet.Tests.csproj << 'EOF'
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.11.1" />
    <PackageReference Include="xunit" Version="2.9.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.2" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\src\HelloDotnet.csproj" />
  </ItemGroup>

</Project>
EOF

cat > tests/GreetingTests.cs << 'EOF'
using HelloDotnet;
using Xunit;

namespace HelloDotnet.Tests;

public class GreetingTests
{
    [Fact]
    public void Greet_ReturnsExpectedMessage()
    {
        Assert.Equal("Hello, .NET!", Greeting.Greet(".NET"));
        Assert.Equal("Hello, CI!", Greeting.Greet("CI"));
    }

    [Fact]
    public void SumRange_ReturnsCorrectSum()
    {
        Assert.Equal(55, Greeting.SumRange(1, 10));
        Assert.Equal(5050, Greeting.SumRange(1, 100));
    }
}
EOF

cat > .gitignore << 'EOF'
bin/
obj/
*.user
.vs/
.idea/
.vscode/
*.suo
.env
dist/
EOF

echo "✅ Структура создана:"
find . -type f | sort
```

---

## 2. ⚙️ `.github/workflows/ci.yml`

```yaml
name: .NET CI/CD

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Cache NuGet
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Restore dependencies
        run: |
          dotnet restore src/HelloDotnet.csproj
          dotnet restore tests/HelloDotnet.Tests.csproj

      - name: Format check (src)
        run: dotnet format src/HelloDotnet.csproj --verify-no-changes --verbosity diagnostic

      - name: Format check (tests)
        run: dotnet format tests/HelloDotnet.Tests.csproj --verify-no-changes --verbosity diagnostic

      - name: Build
        run: dotnet build tests/HelloDotnet.Tests.csproj --configuration Release --no-restore

      - name: Run tests
        run: dotnet test tests/HelloDotnet.Tests.csproj --configuration Release --no-build --verbosity normal

  release:
    needs: test
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest

    permissions:
      contents: write

    strategy:
      matrix:
        include:
          - rid: linux-x64
            artifact: hello-dotnet-linux-x64
          - rid: linux-arm64
            artifact: hello-dotnet-linux-arm64
          - rid: win-x64
            artifact: hello-dotnet-windows-x64.exe
          - rid: osx-x64
            artifact: hello-dotnet-macos-x64
          - rid: osx-arm64
            artifact: hello-dotnet-macos-arm64

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Cache NuGet
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Publish for ${{ matrix.rid }}
        run: |
          dotnet publish src/HelloDotnet.csproj \
            --configuration Release \
            --runtime ${{ matrix.rid }} \
            --self-contained true \
            -p:PublishSingleFile=true \
            -p:IncludeNativeLibrariesForSelfExtract=true \
            --output dist/${{ matrix.rid }}

      - name: Rename binary
        run: |
          if [[ "${{ matrix.rid }}" == win-* ]]; then
            mv dist/${{ matrix.rid }}/hello-dotnet.exe dist/${{ matrix.artifact }}
          else
            mv dist/${{ matrix.rid }}/hello-dotnet dist/${{ matrix.artifact }}
          fi

      - name: Upload to Release
        uses: softprops/action-gh-release@v2
        with:
          files: dist/${{ matrix.artifact }}
          generate_release_notes: true
```

---

## 3. 🧪 Тесты в Docker

**Git Bash / Linux / WSL / macOS:**
```shell
cd ~/hello-dotnet
docker run --rm \
  -u "$(id -u):$(id -g)" \
  -e HOME=/tmp \
  -e DOTNET_CLI_HOME=/tmp \
  -e NUGET_PACKAGES=/tmp/nuget \
  -e DOTNET_CLI_TELEMETRY_OPTOUT=1 \
  -e DOTNET_NOLOGO=1 \
  -v "$(pwd)":/app \
  -w /app \
  mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet test tests/HelloDotnet.Tests.csproj
```

**PowerShell:**
```powershell
cd ~/hello-dotnet
docker run --rm `
  -e DOTNET_CLI_TELEMETRY_OPTOUT=1 `
  -e DOTNET_NOLOGO=1 `
  -v "${PWD}:/app" `
  -w /app `
  mcr.microsoft.com/dotnet/sdk:8.0 `
  dotnet test tests/HelloDotnet.Tests.csproj
```

✅ Ожидаемый вывод:
```
Passed!  - Failed:     0, Passed:     2, Skipped:     0, Total:     2
```

---

## 4. 🔨 Локальная сборка бинарника в Docker

**Git Bash / Linux / WSL / macOS:**
```shell
cd ~/hello-dotnet
docker run --rm \
  -u "$(id -u):$(id -g)" \
  -e HOME=/tmp \
  -e DOTNET_CLI_HOME=/tmp \
  -e NUGET_PACKAGES=/tmp/nuget \
  -e DOTNET_CLI_TELEMETRY_OPTOUT=1 \
  -e DOTNET_NOLOGO=1 \
  -v "$(pwd)":/app \
  -w /app \
  mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet publish src/HelloDotnet.csproj \
    -c Release \
    -r linux-x64 \
    --self-contained true \
    -p:PublishSingleFile=true \
    -p:IncludeNativeLibrariesForSelfExtract=true \
    -o dist/linux-x64
```

**PowerShell:**
```powershell
cd ~/hello-dotnet
docker run --rm `
  -e DOTNET_CLI_TELEMETRY_OPTOUT=1 `
  -e DOTNET_NOLOGO=1 `
  -v "${PWD}:/app" `
  -w /app `
  mcr.microsoft.com/dotnet/sdk:8.0 `
  dotnet publish src/HelloDotnet.csproj -c Release -r linux-x64 --self-contained true -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true -o dist/linux-x64
```

🐧 Запуск в чистом Debian:
```shell
docker run --rm \
  -v "$(pwd)/dist/linux-x64":/dist \
  debian:stable-slim \
  /dist/hello-dotnet
```

✅ Ожидаемый вывод:
```
hello-dotnet version 0.1.0
Hello from C# in GitHub Actions! 🚀📦
OS: Debian GNU/Linux 12 (bookworm)
Arch: X64
Hello, GitHub!
Sum 1..10 = 55
```

💡 Отличие от Python: один `dotnet publish -r <RID>` собирает под любую платформу.

---

## 5. 🌐 Создание пустого репозитория на GitHub

Создайте пустой репозиторий `hello-dotnet`. ⚠️ Не добавляйте `README.md`, `.gitignore`, лицензию — иначе push будет отклонён.

---

## 6. 📤 Push проекта

```shell
cd ~/hello-dotnet
git init
git add .
git commit -m "Initial commit: .NET CLI with CI/CD to Releases"
git branch -M main
```

**Git Bash / Linux / WSL / macOS:**
```shell
read -p "Введите ваш GitHub username: " GITHUB_USER
git remote add origin "https://github.com/${GITHUB_USER}/hello-dotnet.git"
git remote -v
git push -u origin main
```

**PowerShell:**
```powershell
$GITHUB_USER = Read-Host "Введите ваш GitHub username"
git remote add origin "https://github.com/$GITHUB_USER/hello-dotnet.git"
git remote -v
git push -u origin main
```

---

## 7. 🟢 Первый запуск CI

После push в `main` откройте **Actions**. Workflow ~2–3 минуты.

- ✅ **Job test** — `dotnet format`, `dotnet build`, `dotnet test`
- ⏭️ **Job release** — пропущен (push в ветку, не тег)

---

## 8. 🏁 Создание релиза

```shell
git tag v0.1.0
git push origin v0.1.0
```

Workflow ~3–4 минуты (5 сборок параллельно).

- ✅ `test` — тесты
- ✅ `release` — 5 бинарников
- 🎉 Создастся Release **v0.1.0** с файлами:
  - 🐧 `hello-dotnet-linux-x64`
  - 🐧 `hello-dotnet-linux-arm64`
  - 🪟 `hello-dotnet-windows-x64.exe`
  - 🍎 `hello-dotnet-macos-x64`
  - 🍎 `hello-dotnet-macos-arm64`

Проверка: `https://github.com/<USERNAME>/hello-dotnet/releases`

---

## 9. ⬇️ Скачивание и запуск бинарника

**🐧 Linux (x64):**
```shell
read -p "Введите ваш GitHub username: " GITHUB_USER
wget "https://github.com/${GITHUB_USER}/hello-dotnet/releases/download/v0.1.0/hello-dotnet-linux-x64" -O hello-dotnet
chmod +x hello-dotnet
./hello-dotnet
```

**🍎 macOS (Apple Silicon):**
```shell
read -p "Введите ваш GitHub username: " GITHUB_USER
curl -L "https://github.com/${GITHUB_USER}/hello-dotnet/releases/download/v0.1.0/hello-dotnet-macos-arm64" -o hello-dotnet
chmod +x hello-dotnet
./hello-dotnet
```

**🪟 Windows (PowerShell):**
```powershell
$USERNAME = Read-Host "Введите ваш GitHub username"
Invoke-WebRequest -Uri "https://github.com/$USERNAME/hello-dotnet/releases/download/v0.1.0/hello-dotnet-windows-x64.exe" -OutFile "hello-dotnet.exe"
.\hello-dotnet.exe
```

✅ Ожидаемый вывод:
```
hello-dotnet version 0.1.0
Hello from C# in GitHub Actions! 🚀📦
OS: Linux 6.5.0-27-generic
Arch: X64
Hello, GitHub!
Sum 1..10 = 55
```

🍎 На macOS при первом запуске: Системные настройки → Приватность и безопасность → Всё равно открыть.

---

## 10. 🔄 Обновление релиза

⚠️ Релизы в GitHub неизменяемы. Для нового кода — новая версия.

### 10.1. 📊 Тип изменений (SemVer)

| Изменение | Версия | Пример |
|---|---|---|
| 🐛 Баг | patch | `0.1.0` → `0.1.1` |
| ✨ Функция | minor | `0.1.0` → `0.2.0` |
| 💥 Ломающее | major | `0.1.0` → `1.0.0` |

Выбираем **0.2.0**.

### 10.2. ✏️ Обновление версии

Только в `src/HelloDotnet.csproj`:
```xml
<Version>0.2.0</Version>
```

`Program.cs` читает версию автоматически из атрибута сборки.

### 10.3. 🎨 Форматирование

```shell
cd ~/hello-dotnet
docker run --rm \
  -u "$(id -u):$(id -g)" \
  -e HOME=/tmp \
  -e DOTNET_CLI_HOME=/tmp \
  -e NUGET_PACKAGES=/tmp/nuget \
  -e DOTNET_CLI_TELEMETRY_OPTOUT=1 \
  -e DOTNET_NOLOGO=1 \
  -v "$(pwd)":/app \
  -w /app \
  mcr.microsoft.com/dotnet/sdk:8.0 \
  sh -c "dotnet format src/HelloDotnet.csproj && dotnet format tests/HelloDotnet.Tests.csproj"
```

После — закоммитьте изменения.

### 10.4. 🧪 Тесты

```shell
docker run --rm \
  -u "$(id -u):$(id -g)" \
  -e HOME=/tmp \
  -e DOTNET_CLI_HOME=/tmp \
  -e NUGET_PACKAGES=/tmp/nuget \
  -e DOTNET_CLI_TELEMETRY_OPTOUT=1 \
  -e DOTNET_NOLOGO=1 \
  -v "$(pwd)":/app \
  -w /app \
  mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet test tests/HelloDotnet.Tests.csproj
```

### 10.5. 📤 Коммит и push

```shell
git add .
git commit -m "feat: bump version to 0.2.0"
git push origin main
```

Job `release` пропускается (push в ветку).

### 10.6. 🏷️ Новый тег

```shell
git tag v0.2.0
git push origin v0.2.0
```

🎉 Создастся Release `v0.2.0` с 5 бинарниками.

### 10.7. ✅ Проверка

```shell
cd ~
read -p "Введите ваш GitHub username: " GITHUB_USER
URL="https://github.com/${GITHUB_USER}/hello-dotnet/releases/download/v0.2.0/hello-dotnet-linux-x64"
wget "$URL" -O hello-dotnet
chmod +x hello-dotnet
./hello-dotnet
```

### 10.8. 🚑 Откат тега

```shell
git tag -d v0.2.0
git push origin :refs/tags/v0.2.0
```

Release удалить на GitHub → **Delete**.

```shell
code src/HelloDotnet.csproj
git add .
git commit -m "fix: correct version in csproj"
git push origin main

git tag v0.2.0
git push origin v0.2.0
```

### 10.9. 📋 Шпаргалка

```shell
# 1. Изменить код
#    - src/Program.cs

# 2. Обновить версию
#    - src/HelloDotnet.csproj

# 3. Формат + тесты
docker run --rm -u "$(id -u):$(id -g)" -e HOME=/tmp \
  -e DOTNET_CLI_HOME=/tmp -e NUGET_PACKAGES=/tmp/nuget \
  -e DOTNET_CLI_TELEMETRY_OPTOUT=1 -e DOTNET_NOLOGO=1 \
  -v "$(pwd)":/app -w /app mcr.microsoft.com/dotnet/sdk:8.0 \
  sh -c "dotnet format src/HelloDotnet.csproj && dotnet format tests/HelloDotnet.Tests.csproj"

docker run --rm -u "$(id -u):$(id -g)" -e HOME=/tmp \
  -e DOTNET_CLI_HOME=/tmp -e NUGET_PACKAGES=/tmp/nuget \
  -e DOTNET_CLI_TELEMETRY_OPTOUT=1 -e DOTNET_NOLOGO=1 \
  -v "$(pwd)":/app -w /app mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet test tests/HelloDotnet.Tests.csproj

# 4. Коммит и push
git add .
git commit -m "feat: bump version to 0.2.0"
git push origin main

# 5. Новый тег
git tag v0.2.0
git push origin v0.2.0
```

---

## 🎓 Итог

- 🟣 .NET CLI — `dotnet build`, `test`, `publish`, `format`
- 🧪 xUnit, 🎨 dotnet format
- ⚡ Cross-compilation — один runner, 5 платформ через `-r <RID>`
- 🆔 Runtime Identifiers — `linux-x64`, `win-x64`, `osx-arm64`
- 📦 Self-contained + PublishSingleFile
- 🔀 Матричные сборки в GitHub Actions, кэш NuGet
- 🚀 Публикация через `softprops/action-gh-release`
- 🏷️ SemVer: `v0.1.0`, `v0.2.0`
- 🔄 Полный цикл обновления релиза
- 🎯 Один файл версии — без рассинхронов