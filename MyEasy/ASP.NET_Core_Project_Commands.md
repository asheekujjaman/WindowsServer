# ASP.NET Core Project Commands

Default projects folder: `C:\Development\Projects`

## 1. Create a new ASP.NET Core MVC project

```powershell
cd C:\Development\Projects

dotnet new mvc -n ITDashboard -f net10.0

cd ITDashboard

code .
```

Replace `ITDashboard` with your preferred project name.

## 2. Open an existing project in VS Code

```powershell
code -r "C:\Development\Projects\ITDashboard"
```

Replace the path with your existing project folder.

## 3. Run an existing project with automatic file watching

```powershell
cd C:\Development\Projects\ITDashboard

dotnet restore

dotnet watch run
```

Open the localhost URL printed in the terminal in Chrome or Edge. Keep the terminal open while the website is running.

## 4. Run a project without automatic file watching

```powershell
cd C:\Development\Projects\ITDashboard

dotnet run
```

## 5. Build the project and check for errors

```powershell
cd C:\Development\Projects\ITDashboard

dotnet build
```

## 6. Open the website in a browser

First run the project and note the URL printed in the terminal. Then open that exact URL, for example:

```powershell
Start-Process "https://localhost:7001"
```

The example port may be different on your computer.

## 7. Stop the running website

Press:

```text
Ctrl + C
```

in the terminal where the project is running.

## 8. List installed .NET SDK versions

```powershell
dotnet --list-sdks
```

---

**Tip:** Change `ITDashboard` and the folder path in these commands to match your actual project name and location.
