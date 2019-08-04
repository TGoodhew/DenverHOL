# Lab 3 - Customizing your image

One of the benefits of the Windows IoT Core image creation process is that you can easily create a number of different products based on the same core image and board support package. In this case we're going to create a new product based on the same setup as our original product.

## Supported Application Types

### Universal Windows Platform (UWP) Apps
IoT Core is a UWP centric OS and UWP apps are its primary app type.

Universal Windows Platform (UWP) is a common app platform across all version of Windows 10, including Windows 10 IoT Core. UWP is an evolution of Windows Runtime (WinRT). You can find more information and an overview to UWP [here](https://docs.microsoft.com/windows/uwp/get-started/universal-application-platform-guide).

### Traditional UWP Apps
UWP apps just work on IoT Core, just as they do on other Windows 10 editions. A simple, blank Xaml app in Visual Studio will properly deploy to your IoT Core device just as it would on a phone or Windows 10 PC. All of the standard UWP languages and project templates are fully supported on IoT Core.

There are a few additions to the traditional UWP app-model to support IoT scenarios and any UWP app that takes advantage of them will need the corresponding information added to their manifest. In particular the "iot" namespace needs to be added to the manifest of these standard UWP apps.

Inside the attribute of the manifest, you need to define the iot xmlns and add it to the IgnorableNamespaces list. The final xml should look like this:

```xml
<Package
  xmlns="http://schemas.microsoft.com/appx/manifest/foundation/windows10"
  xmlns:mp="http://schemas.microsoft.com/appx/2014/phone/manifest"
  xmlns:uap="http://schemas.microsoft.com/appx/manifest/uap/windows10"
  xmlns:iot="http://schemas.microsoft.com/appx/manifest/iot/windows10"
  IgnorableNamespaces="uap mp iot">
```

### Background Apps

In addition to the traditional UI apps, IoT Core has added a new UWP app type called "Background Applications". These applications do not have a UI component, but instead have a class that implements the "IBackgroundTask" interface. They then register that class as a "StartupTask" to run at system boot. Since they are still UWP apps, they have access to the same set of APIs and are supported from the same language. The only difference is that there is no UI entry point.

Each type of IBackgroundTask gets its own resource policy. This is usually restrictive to improve battery life and machine resources on devices where these background apps are secondary components of foreground UI apps. On IoT devices, Background Apps are often the primary function of the device and so these StartupTasks get a resource policy that mirrors foreground UI apps on other devices.

You can find in-depth information on Background apps on [MSDN](https://docs.microsoft.com/windows/iot-core/develop-your-app/backgroundapplications).

### Non-UWP (Win32) Apps
IoT Core supports certain traditional Win32 app types such as Win32 Console Apps and NT Services. These apps are built and run the same way as on Windows 10 Desktop. Additionally, there is an IoT Core C++ Console project template to make it easy to build such apps using Visual Studio.

There are two main limitations on these non-UWP applications:

1. No legacy Win32 UI support: IoT Core does not contain APIs to create classic (HWND) Windows. Legacy methods such as CreateWindow() and CreateWindowEx() or any other methods that deal with Windows handles (HWNDs) are not available. Subsequently, frameworks that depend on such APIs including MFC, Windows Forms and WPF, are not supported on IoT Core.

2. C++ Apps Only: Currently, only C++ is supported for developing Win32 apps on IoT Core.

### App Service
App services are UWP apps that provide services to other UWP apps. They are analogous to web services, on a device. An app service runs as a background task in the host app and can provide its service to other apps. For example, an app service might provide a bar code scanner service that other apps could use. App services let you create UI-less services that apps can call on the same device, and starting with Windows 10, version 1607, on remote devices. Starting in Windows 10, version 1607, you can create app services that run in the same process as the host app.

Additional information regarding creating a background app service as well as consuming the service from a uwp apps (as well as background tasks/services) can be found [here](https://docs.microsoft.com/en-us/windows/uwp/launch-resume/how-to-create-and-consume-an-app-service).

## Our appx package

Universal Windows Platform (UWP) applications can run on both Windows, Windows IoT Enterprise and Windows IoT COre. Building a UWP application is outside of the scope of this lab so we'll be using a pre-built UWP App. This app is similar to the default application that runs on the device but has been annotated so you can see the difference.

We'll start the process of including it in our OS Image by using the AppX package that is created by Visual Studio.

## Repackage the Appx

The first step is to repackage the Appx file, which will allow you to customize it and build it using the Windows ADK (when you build the FFU image). It also adds entries into the feature manifest for the image assiting in the customization process.

1. Open `IoTCorePShell.cmd`. It should prompt you to run as an administrator.

2. Create the package for your Appx by using `New-IoTAppxPackage`. Replace the file path location and package name with your Appx package. For this lab, the command is as follows:

```powershell
Add-IoTAppxPackage    "C:\DefaultApp\IoTCoreDefaultApp_1.2.0.0_ARM_Debug_Test\IoTCoreDefaultApp_1.2.0.0_ARM_Debug_Test.appx" fga Appx.MyUWPApp
```

>The fga parameter indicates the Appx file is a foreground application. If you specify your package as a background application (with the bga parameter) and have no other foreground applications in the image, the system will get stuck when booting up (displays a spinner indefinitely).

This creates a new folder at `C:\MyWorkspace\Source-<arch>\Packages\Appx.MyUWPApp`, copies the appx files and its dependencies and generates a `customizations.xml` file as well as a package xml file that is used to build the package.

Be aware that if your Appx has dependencies you will need the Dependencies subdirectory to be present in the same location as your Appx when you run this command. Failure to include this will result in errors when you build your FFU image.

This also adds a FeatureID APPX_MYUWPAPP to the `C:\MyWorkspace\Source-<arch>\Packages\OEMFM.xml` file.

3. From the IoT Core Shell Environment, you can now build the package into a .CAB fileusing `New-IoTCabPackage`.

```powershell
New-IoTCabPackage Appx.MyUWPApp
```

This will build the package into a .CAB file under `C:\MyWorkspace\Build\<arch>\pkgs\<oemname>.Appx.MyUWPApp.cab`.

## Create a new product

Create a new product using Add-IoTProduct:

```powershell
Add-IoTProduct ProductB HummingBoardEdge_iMX6Q_2GB
```

You will be prompted to enter the **SMBIOS** information such as Manufacturer name (OEM name), Family, SKU, BaseboardManufacturer and BaseboardProduct. Here are some example values:

- **System OEM Name**: HOLLab
- **System Family Name**: HOLLabHub
- **System SKU Number**: AI-001
- **Baseboard Manufacturer**: NXP
- **Baseboard Product**: HummingBoardEdge_iMX6Q_2GB
    
This creates the folder: `C:\MyWorkspace\Source-arm\Products\ProductB`.

## Update the project's configuration files

You can now update your project configuration files to include your app in the FFU image biuld.

1. Add the FeatureID for our app package using `Add-IoTProductFeature`:

   ```powershell
   Add-IoTProductFeature ProductB Test APPX_MYUWPAPP -OEM
   ```

2. Remove the sample test apps `IOT_BERTHA` using `Remove-IoTProductFeature`

   ```powershell
   Remove-IoTProductFeature ProductB Test IOT_BERTHA
   ```

## Build the image
From the IoT Core PowerShell Environment, get your environment ready to create products by building all of the packages in the working folders (using `New-IoTCabPackage`):

```powershell
New-IoTCabPackage All
```

Build the FFU image again, as specified in Lab 1: Create a basic image. You can use this command:

```powershell
New-IoTFFUImage ProductB Test -Verbose
```

Now that the image has been rebuilt without errors we're ready to bring everything together and deploy the image to the device using the same steps as in lab 2.

### Direct lab links
- [Lab 1 - Create a basic image](https://github.com/TGoodhew/DenverHOL/blob/master/Labs/Lab1.md)
- [Lab 2 - Deploying an image to the device](https://github.com/TGoodhew/DenverHOL/blob/master/Labs/Lab2.md)
- [Lab 3 - Customizing your image](https://github.com/TGoodhew/DenverHOL/blob/master/Labs/Lab3.md)
