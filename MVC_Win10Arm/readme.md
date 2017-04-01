# H1 dotnet core 2.0 MVC app on Raspberry Pi 3
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

If running on a Dev machine with multiple dotnetcore SDK's installed, you need to create a global.json file so the dotnet.exe muxer will use the preview version 2.X SDK and not default to the latest official release version the SDK (probably 1.1)

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
To check it is using the 2.X preview SDK run
```sh
dotnet.exe --version
```
You should see 
```sh
	2.0.0-preview1-005685
```
You can check that the SDK parameter in your global.json file matches the installed SDK version on your Dev machine
```sh
eg open folder C:\Program Files\dotnet\sdk and look at the list of installed SDK folders
```
Create a new dotnetcore MVC application using dotnet.exe 
In the Command Line type
```sh
dotnet new mvc
```
You should now have a dotnet.exe scaffolded basic MVC template.
Open the .csproj file
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

Before doing a dotnet restore, to restore packages create a new NuGet config file.
```sh
notepad NuGet.config
```
Yes, to create new file
Add this to the blank document and save
```sh
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="dotnet-core" value="https://dotnet.myget.org/F/dotnet-core/api/v3/index.json" />
    <add key="aspnetcore-dev" value="https://dotnet.myget.org/F/aspnetcore-ci-dev/api/v3/index.json" />
  </packageSources>
</configuration>
```

Now restore the packages using nuget (this will take a while as it brings down the latest packages & dependencies from nuget)
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

now lets build the app with
```sh
dotnet.exe build
```
This will build and create the ...\bin\Debug\netcoreapp2.0\ folder and you should see the MVC_Win10Arm.dll (which you could run with "dotnet run MVC_Win10Arm.dll" (or corehost.exe MVC_win10Arm.dll)
But now lets do one last step and turn it into a .EXE so we dont have to do above, as this step didn't work for me at first.

Open the .csproj file and add the <OutputType> line
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

For some reason, it looks like the <OutputType> gets ignored in .csproj? (same if use win8-arm runtime identifier) but if you use
```sh
dotnet.exe build -r win10-arm
```
You get a ..\Debug\netcoreapp2.0\win10-Arm folder with and .exe in it

OK, if this is working, now we need to package it all up with all these dependencies for deployment to the Win10IoT device
Because of the same issue above, use below to publish
```sh
dotnet.exe publish -r win10-arm
```
you should get a ..\Debug\netcoreapp2.0\win10-arm\publish folder
Copy this folder to the Win10IoT device
The easiest way is to use the IOTDashboard and get a network share to the file system.
Open the dashboard and right click the device and "open network share"
Browse to the folder where you wish to run
```sh
\\10.1.1.200\c$\Data\Users\administrator\Downloads
```
Copy the publish folder and all its content here.

PowerShell to the device using the Dashboard, or remote PowerShell using "Enter-PSSession -ComputerName <machine-name or IP Address> -Credential <machine-name or IP Address or localhost>\Administrator", but the easiest way if you have the IoTDashboard installed.

cd to the publish folder
```sh
cd \Data\Users\administrator\Downloads\publish
```
run the .exe we just built and copied over
```sh
.\MVC_Win10Arm.exe
```
run it and should see
```sh
[10.1.1.200]: PS C:\Data\Users\administrator\Downloads\publish> .\MVC_Win10Arm.exe
Hosting environment: Production
Content root path: C:\Data\Users\administrator\Downloads\publish
Now listening on: http://localhost:5000
Application started. Press Ctrl+C to shut down.
```
But because Kestrel is default bound to the loopback network adapter (Localhost), we make a chnage so it listens on all adapters.
Modify Program.cs
```sh
public static void Main(string[] args)
{
var host = new WebHostBuilder()
.UseKestrel()
.UseUrls("http://*:5000")
<<-- Other lines omitted -->>
```
Run another publish and copy the files to device.

Almost there, but because the Win10IoT firewall will block port 5000
either disable it totally (!!) or open a port
From PowerShell prompt
```sh
netsh advfirewall set allprofiles state off
```
or better still
```sh
netsh advfirewall firewall add rule name="MVCWin10 Port 5000" dir=in action=allow protocol=TCP localport=5000
```
OK, now lets run the new .exe from PowerShell
```sh
.\MVC_Win10Arm.exe
```
Open a browser and browse to the IP of your WIN10I0T device, for example
```sh
http://10.1.1.200:5000/
```
You should see a MVC web site (takes a little while first time, but surprisingly quick)

Before converting the project into a .EXE I used too use the dotnet runtime and load the MVC_Win10Arm.dll
This took a bit more work, but you can do it by, downloading the latest Windows(arm32) runtime from
https://github.com/dotnet/core-setup#daily-builds
Extract them
Create a new folder called runtime, eg md c:\Data\Users\administrator\Downloads\runtime in PowerShell
Copy the extracted contents to this folder on the PI WIN10IoT device

Should contain folders
```sh
	host
	shared
```
To run the .dll, execute (remember to use the full paths)
```sh
C:\Data\Users\administrator\Downloads\runtime\.\dotnet C:\Data\Users\administrator\Downloads\publish\MVC_Win10Arm.dll
```
If you don't want to use those huge paths, perhaps create the following Environment Variables on the Pi Win10IOT
```sh
$env:Path=$(Get-ChildItem env:path).Value + ";C:\Data\Users\administrator\Downloads\runtime"
```
(If you make a mistake and clobber the Path, I have copied my original default Path below)

Now you can run it by CD'ing into the publish folder and run
```sh
dotnet C:\Data\Users\administrator\Downloads\publish\MVC_Win10Arm.dll
```

# H5 Thats it!

***
---

# H5 Other things in PowerShell

To list Environment Variables
```sh
Get-ChildItem Env:
```

If you would like to get some output out of dotnet while it spins up set environment variable CoreHost_Trace = 1
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

Any questions or suggestions welcome


