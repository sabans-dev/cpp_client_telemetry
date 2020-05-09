
This tutorial guides you through the process of integrating the 1DS SDK into your Android app.

## 1. Clone the repository

Run `git clone https://github.com/microsoft/cpp_client_telemetry.git` to clone the repo. If your project requires UTC to send telemetry, you need to add `--recurse-submodules` when cloning to tell git to add `lib/modules` repo. You will be prompted to enter your credentials to clone. Use your MSFT GitHub username and GitHub token.

## 2. Build all

You will ideally build the SDK using the same versions of the Android SDK, NDK, and build tooling as your existing app. The repo includes a directory lib/android_build that can be used as an Android Studio project or to build using Gradle. The GitHub CI loop uses Gradle in this directory to build and test the SDK. As with any C++ code that exposes std:: container types in its API, you need to ensure that templates compile compatibly across compilation units, and you need to ensure that all compilation units are linking against the same C++ shared library. The ```build.gradle``` configuration in both modules in ```android_build``` selects the Android llvm shared C++ library. The SDK will not compile and link against the older Android C++ libraries; you will need to use this same library option in your application in order to use the SDK.

The Gradle wrapper in ```android_build``` builds two modules, ```app``` and ```maesdk```. The ```maesdk``` module is the SDK packaged as an AAR, with both the Java and C++ components included. The AAR includes C++ shared libraries for four ABIs (two ARM ABIs for devices and two Intel ABIs for the emulator). Android Gradle (as usual) supports debug and release builds, and the Gradle task ```maesdk:assemble``` should build both flavors of AAR.

## 3. Integrate the SDK into your C++ project

If you use the lib/android_build Gradle files, they build the SDK into maesdk.aar in the output folders of the maesdk module in lib/android. You can package or consume this AAR in your applications modules, just as you would any other AAR.

For the curious: the app module is an Android application. The GitHub CI loop uses it as a platform to run the unit test suite against the SDK.

## 4. Instrument your code to send a telemetry event

1. Include the main 1DS SDK header file in your main.cpp by adding the following statement to the top of your app's implementation file.

	```
    #include "LogManager.hpp"
	```
    
2. Use the namespace, either via using namespace or a namespace alias.

    1. Use namespace by adding the following statement after your include statements at the top of your app's implementation file.

    ```
    using namespace Microsoft::Applications::Events; 
    ```

    2. Or if you prefer to avoid namespace collision, you can create a namespace alias instead:

    ```
    namespace MAE = Microsoft::Applications::Events;
    ```

    3. Or you can simply prefix all the SDK names with ```Microsoft::Application::Events```, as you prefer.

3. Create the default LogManager instance for your project using the following macro in your main C++ file:

	  ```
    LOGMANAGER_INSTANCE
    ```

4. Load the shared library from the maesdk AAR (or .so) file. In Java, you call ```System.load_library``` to load the shared object. Loading the dynamic library registers its JNI entry points and permits the Java component of the SDK to call into the C++ portion of the SDK. This is typically done in static initialization in the application:

    ```
    static {
      System.loadLibrary("maesdk");
    }
    ```

5. Initialize the Java layer. Some part of the application must create a singleton instance of the Java class
```com.microsoft.applications.events.HttpClient```. The constructor for this class takes one parameter, the application ```Context``` object. In most cases, it will be easiest to do this from Java:

    ```
    HttpClient client = new HttpClient(getApplicationContext());
    ```

  One could create this instance from C++ code if that code has the appropriate JNIEnv* or JavaVM* pointers (one can obtain JavaVM* from JNIEnv* or vice-versa) and a jobject reference to the application ```Context``` instance. This should occur before initialization of the C++ side of the SDK.

  The lifetime of the reference created here is unimportant; the C++ side of the SDK will take a global (static) reference on this singleton and keep it alive until it destructs. 

6. Initialize the SDK, create and send a telemetry event, and then flush the event queue and shut down the telemetry
logging system by adding the following statements to your main() function.

    ```
    // preface the references to the SDK symbols with MAE:: if you use that namespace alias declaration
    ILogger* logger = LogManager::Initialize("0123456789abcdef0123456789abcdef-01234567-0123-0123-0123-0123456789ab-0123");
    logger->LogEvent("My Telemetry Event");
    ...
    LogManager::FlushAndTeardown();
    ```

You're done! You can now compile and run your app, and it will send a telemetry event using your ingestion key to your tenant.

Please refer to [EventSender](https://github.com/microsoft/cpp_client_telemetry/tree/master/examples/cpp/EventSender) sample for more details. Other sample apps can be found [here](https://github.com/microsoft/cpp_client_telemetry/tree/master/examples/cpp/). The lib/android_build gradle wrappers will use the Android gradle plugin, and that in turn will use CMake/nmake to build C++ object files.