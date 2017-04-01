# dotnet core 2.0 MVC app on Rasberry Pi 3
[![N|Solid](https://www.primetss.com.au/Content/images/PTSSLogoSml.gif)](https://www.primetss.com.au)

Install Win10IoT preview v.10.0.15063.0 or greater on your device
Useful tool to manage the Pi is the Windows 10 IoT Core Dashboard
https://developer.microsoft.com/en-us/windows/iot/docs/iotdashboard

- Download dotnetcore v2 SDK (currently 2.0.0-preview1-005685) and install on your Dev machine
https://github.com/dotnet/cli

Open command line
```sh
md MVC_Win10Arm
cd MVC_Win10Arm
```

If running on a Dev machine with multiple dotnetcore SDK's installed you need to create a global.json file so the dotnet.exe muxer will use the preview version 2.X SDK and not default to the latest offical release version the SDK (probably 1.1)

Create a new file global.json
```sh
Add
{
    "sdk": {
        "version": "2.0.0-preview1-005685"
    }
}
```
Save
Test you it is not using the 2.X preview of SDK
type dotnet.exe --version
You should see 
	2.0.0-preview1-005685

If you dont, check that your global.json file matches what SDK your Dev machine has installed
eg open folder C:\Program Files\dotnet\sdk and look at the list of installed SDK folders

Create a new dotnetcore MVC application using dotnet.exe 

On Command Line type
```sh
dotnet new mvc
```
You should now have a dotnet.exe schaffolded basic MVC template

open the .csproj file
```sh
notepad MVC_Win10Arm.csproj
```
Modify the following lines to modify the project into netcoreapp2.0 to match the new v2 SDK we just installed

```sh
<Project Sdk="Microsoft.NET.Sdk.Web">
<PropertyGroup>
	<TargetFramework>netcoreapp2.0</TargetFramework>
	<RuntimeFrameworkVersion>2.0.0-beta-001834-00</RuntimeFrameworkVersion>	
	<RuntimeIdentifiers>win10-arm</RuntimeIdentifiers>
</PropertyGroup>
<ItemGroup>
	<PackageReference Include="Microsoft.AspNetCore" Version="1.2.0-preview1-*" />
	<PackageReference Include="Microsoft.AspNetCore.Mvc" Version="1.2.0-preview1-*" />
	<PackageReference Include="Microsoft.AspNetCore.StaticFiles" Version="1.2.0-preview1-*" />
	<PackageReference Include="Microsoft.Extensions.Logging.Debug" Version="1.2.0-preview1-*" />
	<PackageReference Include="Microsoft.VisualStudio.Web.BrowserLink" Version="1.2.0-preview1-*" />
</ItemGroup>
</Project>
```

Before doing a dotnet restore to restore packages create a new NuGet.
```sh
notepad NuGet.config
```
Yes to create new file
Add this the the blank document and save
```sh
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="dotnet-core" value="https://dotnet.myget.org/F/dotnet-core/api/v3/index.json" />
    <add key="aspnetcore-dev" value="https://dotnet.myget.org/F/aspnetcore-ci-dev/api/v3/index.json" />
  </packageSources>
</configuration>
```

Now restore the packages using nuget (this will take awhile as it brings down the latest packages from nuget)
```sh
dotnet restore
```
Should complete with no errors or warnings
```sh
  NuGet Config files used:
      D:\DATA\My GitHub\DotNetCore\MVC_Win10Arm\NuGet.Config
      C:\Users\your name\AppData\Roaming\NuGet\NuGet.Config
      C:\Program Files (x86)\NuGet\Config\Microsoft.VisualStudio.Offline.config

  Feeds used:
      https://dotnet.myget.org/F/dotnet-core/api/v3/index.json
      https://dotnet.myget.org/F/aspnetcore-ci-dev/api/v3/index.json
      https://api.nuget.org/v3/index.json
      C:\Program Files (x86)\Microsoft SDKs\NuGetPackages\

  Installed:
      74 package(s) to D:\DATA\My GitHub\DotNetCore\MVC_Win10Arm\MVC_Win10Arm.csproj
```

now lest build the app with
```sh
dotnet.exe build
```
This will build and create the ...\bin\Debug\netcoreapp2.0\ folder and you should see the MVC_Win10Arm.dll (which ou could run with dotnet run MVC_Win10Arm.dll (or corehost.exe MVC_win10Arm.dll)Now lest turn it into a .EXE, as this didnt work for me at first

open .csproj
```sh
notepad MVC_Win10Arm.csproj
```

```sh
Add <OutputType>Exe</OutputType> 
<PropertyGroup>
	<OutputType>Exe</OutputType>
	<TargetFramework>netcoreapp2.0</TargetFramework>
	<RuntimeIdentifiers>win10-arm</RuntimeIdentifiers>
</PropertyGroup>
```

now rebuild
```sh
dotnet.exe build
```

For some reason, it looks like the <OutputType> gets ignored (same if use win8-arm runtime identifier) but if you use
```sh
dotnet.exe build -r win10-arm
```
You get a ..\Debug\netcoreapp2.0\win10-Arm folder with and .exe in it

OK, if this is working, now we need to package it all up with all these depenancies for deplyment to the Win10IoT device
Because of above, maybe issue? use below to publish
```sh
dotnet.exe publish -r win10-arm
```
you should get a ..\Debug\netcoreapp2.0\win10-arm\publish folder
copy that to the Win10IoT device
Easiest way is to use the IOTDashboard and get a network share to the file system
Open the dashboard and right click the device and "open network share"
Browse to the folder where you wish to run
```sh
\\10.1.1.200\c$\Data\Users\administrator\Downloads
```
copy the publish folder and all its content here

Powershell to the device using the Dashboard
or


cd to the publish folder
```sh
cd \Data\Users\administrator\Downloads\publish
```
run the .exe 
```sh
.\MVC_Win10Arm.exe
```
run it and should get
```sh
[10.1.1.200]: PS C:\Data\Users\administrator\Downloads\publish> .\MVC_Win10Arm.exe
Hosting environment: Production
Content root path: C:\Data\Users\administrator\Downloads\publish
Now listening on: http://localhost:5000
Application started. Press Ctrl+C to shut down.
```
But because Kestral is default bound to the loopback network adapter, we need to change that so it listens on all adapters
Modify Program.cs
```sh
public static void Main(string[] args)
{
var host = new WebHostBuilder()
.UseKestrel()
.UseUrls("http://*:5000")
<<-- Other lines obmitted -->>
```
Run another publish and copy files to device

Becuase the Win10IoT firewall will block port 5000
either disable it totally (!!) or open a port
From PowerShell prompt
```sh
netSh advfirewall set allprofiles state off
or 
netsh advfirewall firewall add rule name="MVCWin10 Port 5000" dir=in action=allow protocol=TCP localport=5000
```
once copied, run the new .exe from PowerShell
```sh
.\MVC_Win10Arm.exe
```
Open a browser and browse to the IP of your device
```sh
http://10.1.1.200:5000/
```
You should see a MVC web site

Before converting the project into a .EXE I used to use the dotnet runtime and load the MVC_Win10Arm.dll
This took a bit more work, but you can do it
Download the lastest Windows(arm32) runtime from
https://github.com/dotnet/core-setup#daily-builds
Extract them
Create a new folder called runtime, eg md c:\Data\Users\administrator\Downloads\runtime in powershell
Copy the extracted contents to this folder on the PI WIN10IoT device

Should contain folders
	host
	shared

to run the .dll, execute (remember to use the full paths)
```sh
C:\Data\Users\administrator\Downloads\runtime\.\dotnet C:\Data\Users\administrator\Downloads\publish\MVC_Win10Arm.dll
```
If you dont want to use the huge paths, perhaps create the following Enviroment Variables on the Pi Win10IOT
```sh
$env:Path=$(Get-ChildItem env:path).Value + ";C:\Data\Users\administrator\Downloads\runtime"
```
(If you make a mistake, I have copied my original default Path below)

now you can run it by cd'ing into the publish folder and run
```sh
dotnet C:\Data\Users\administrator\Downloads\publish\MVC_Win10Arm.dll
```


Other things in PowerShell

To list Enviroment Variables
```sh
Get-ChildItem Env:
```

If you would like to get some output out of dotnet while it spins up set Set enviroment variable CoreHost_Trace = 1
```sh
$env:COREHOST_TRACE=1
```
To set Paths to dotnetcore runtime

CORE_ROOT - The directory where to find the runtime DLLs itself (e.g. CoreCLR.dll).
Defaults to be next to the corerun.exe host itself.
CORE_LIBRARIES - A Semicolon separated list of directories to look for DLLS to resolve any assembly references. It defaults CORE_ROOT if it is not specified.
https://github.com/dotnet/coreclr/blob/master/Documentation/workflow/UsingCoreRun.md

DefaultPath String:
C:\windows\system32;C:\windows;C:\windows\System32\Wbem;C:\windows\System32\WindowsPowerShell\v1.0\;C:\Data\Users\administrator\AppData\Local\Microsoft\WindowsApps

Any questions or suggestions

