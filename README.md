# Adding Preferences (via the iOS Settings App) to a FIAS project

How to create and utilize native iOS Preferences functionality (in the Apple-provided Settings App) and share those preferences with the FIAS solution database, using the `AppDelegate` in a FileMaker iOS App SDK (FIAS) project (v18).

- - -

### What You'll Learn
* How to add native App Settings functionality to a FIAS project in Xcode.
* How to share/pass the App Settings data with the FIAS solution database.

### What This Post Is Not
* A tutorial on FIAS
* A tutorial on Xcode
* A tutorial on Objective-C

### Requirements
* iOS App SDK 18+ (not tested on v17, but may work just fine)
* Xcode 10+

### What We're Going To Do
* Add an Objective-C App Delegate class
* Verify the App Delegate class
* Implement App Delegate Methods to communicate with FileMaker database
* Add a 'Settings bundle' file that defines the preferences to display
* Verify the Settings data is accessible by the App Delegate
* Verify the Settings data can be shared with a Filemaker script

- - -

### Your FIAS Project

Either open up and existing FIAS project or create a new one.

Make sure you can build it and run it, either in the iOS Simulator or on your own device.

- - -

### Xcode: Add an Objective-C App Delegate class

Bring up your existing App Delegate class in the Xcode editor or create a New File, using the file type 'Objective-C File' as the document type.

Be sure to save this new file into your existing 'Custom Application Resources' directory.

Note: If you are adding this App Delegate class to your project for the first time, be sure to follow the instructions in the iOS SDK Documentation for modifying the 'configFile.txt' file so that `applicationDelegateClass = MyAppDelegate`

Use the following code in your App Delegate file as a starting point.

```objective-c
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>
#import "FMX_Exports.h"

#define kUserNameKey            @"username_preference"
#define kPasswordKey            @"password_preference"

@interface MyAppDelegate : UIResponder <UIApplicationDelegate>
{
    NSString *settingsUsername;
    NSString *settingsPassword;
}
@property (strong, nonatomic) UIWindow *window;
@end

@implementation MyAppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    NSLog(@"MyAppDelegate: %s", __func__);

    return true;
}

@end
```

- - -

### Verify the App Delegate Class

Build the project (Command-B). You shouldn't have any errors.

Run the project (Command-R). When it runs the console should display the NSLog string from inside the function 'didFinishLaunchingWithOptions':

```objective-c
MyAppDelegate: -[MyAppDelegate application:didFinishLaunchingWithOptions:]
```

Note: to show the Console, type Command + Shift + c

- - -

### FileMaker: Create a Filemaker Script
Open your FIAS database locally in Filemaker Pro.
Create a new Script, using any naming convention you wish. For the following examples we will use the script name 'AppDelegate_didFinishLaunchingWithOptions'.


```objective-c
#========================================
#	Purpose:     FIAS AppDelegate - triggered by delegate method
#	Returns:     none
#	Parameters:  scriptParameter (optional)
#	             iOS variables (optional)
#	Called from: (FIAS) didFinishLaunchingWithOptions
#	Author:      Leland Long
#	Notes:       none
#	History:     2019-07-29 Leland Long - created
#========================================

Allow User Abort [ Off ]
Set Error Capture [ On ]

Set Variable [ $param; Value:Get ( ScriptParameter ) ]
Show Custom Dialog [ Title: "AppDelegate";
                     Message: "Script triggered in db: " & Get ( ScriptName ) & "¶"
                              & "Param: " & $param;
                     Default Button: "Excellent",
                     Commit: “Yes” ]

```

Close the database in FileMaker Pro and return to Xcode.

- - -

### Xcode: Trigger the FileMaker Script in your FIAS project

Modify your existing AppDelegate file to match the following, replacing <myFMdbName> with the actual name of your database filename, not including the extension '.fmp12':

```objective-c

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    NSLog(@"MyAppDelegate: %s", __func__);

    if (FMX_Queue_Script(@"<myFMdbName>", @"AppDelegate_didFinishLaunchingWithOptions",
                         kFMXT_Pause, @"A script param from Xcode", nil)) {
        NSLog(@"MyAppDelegate: FMX_Queue_Script AppDelegate_didFinishLaunchingWithOptions Succeeded");
    } else {
        NSLog(@"MyAppDelegate: FMX_Queue_Script AppDelegate_didFinishLaunchingWithOptions Failed");
    }

    return true;
}

```

Build and run the project. When it runs the console should display the 2 NSLog entries from inside the function 'didFinishLaunchingWithOptions':

```objective-c
MyAppDelegate: -[MyAppDelegate application:didFinishLaunchingWithOptions:]
MyAppDelegate: FMX_Queue_Script appDelegate_didFinishLaunchingWithOptions Failed
```

`Notice:` the script failed to be triggered. How? What? Why?

- - -

### Create a NSTimer Class Object To Fix Our Script Issue

I won't pretend to know the answer as to why this failed, but I can provide an educated guess.
The FileMaker framework handles all of the UIWindow, UIViewController, and UIView pieces and parts.
This all takes time.
My assumption is that when this function is triggered in our AppDelegate, the FileMaker database is not yet quite ready to receive and run our script request, so it quietly fails.
Going on this assumption, I decided to play around with adding a slight delay in our code to see if that might solve this dilemma.
It does solve it, but is not the best solution.
The reason I mention this is because you will notice slight differences with the length of delay required to successfully fire off this script request, between trying this solution on the iOS Simulator and actual iOS devices (iPhones and iPads).

But let's give it a try to see what your experience is in your environment.

Modify your existing AppDelegate file to match the following:

```objective-c

- (void)triggerScript_didFinishLaunchingWithOptions:(NSTimer *)timer {
    NSLog(@"MyAppDelegate: %s", __func__);

    if (FMX_Queue_Script(@"<myFMdbName>", @"AppDelegate_didFinishLaunchingWithOptions",
                         kFMXT_Pause, @"A script param from Xcode", nil)) {
        NSLog(@"MyAppDelegate: FMX_Queue_Script AppDelegate_didFinishLaunchingWithOptions Succeeded");
    } else {
        NSLog(@"MyAppDelegate: FMX_Queue_Script AppDelegate_didFinishLaunchingWithOptions Failed");
    }
}

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    NSLog(@"MyAppDelegate: %s", __func__);
    NSLog(@"didFinishLaunchingWithOptions: creating a timer to launch script 'AppDelegate_didFinishLaunchingWithOptions'");

    // FIAS needs a little time to begin responding to script requests, so add a slight delay
    // simulator is fairly quick (2 seconds works fine) but iOS device seems to take longer, at least with debugger, needing 10 seconds
    [NSTimer scheduledTimerWithTimeInterval: 2.0
                                     target: self
                                   selector: @selector(triggerScript_didFinishLaunchingWithOptions:)
                                   userInfo: nil
                                    repeats: NO];

    return true;
}

```

Build and run the project. When it runs the console should display the 2 NSLog entries from inside the function 'didFinishLaunchingWithOptions':

If it runs successfully you will see the success message in the Console and you will also see the Custom Dialog appear in your App, that you added previously in your Script.
You should also see that your Script Parameter was successfully passed to the Script.

![didFinishLaunching image](/images/didFinishLaunching.png)*didFinishLaunching script*

If it still did not trigger successfully, try increasing the TimeInterval value from 2.0 to 4.0, or 6.0, or whatever you want to try , in order to determine the sweet spot of a small enough value to work every time.
My own experience has resulted in a 2.0 second delay for the Simulator and a 10.0 second delay in my iPhone XSMax running on iOS v13 beta.

If it still does not trigger, or if the Console shows that it triggered successfully, but your App does not present the expected Custom Dialog, be sure to refer back to the iOS SDK installation instructions regarding copying the solution file (database) into the iOS project.
This is a setting you can adjust in the `configFile.txt` file.
During development I would recommend selecting `always` so that it copies the database every time you run the project.
- - -

### Xcode: Optional Additional AppDelegate Script Trigger Methods

All of the following code is optional. You can pick and choose which ones to add to your project.
Most, if not all of these, will not need a NSTimer delay in order to successfully trigger a FileMaker Script to be run.
I would highly recommend having each and every additional FileMaker Script that you add for any of these methods to contain a `ShowCustomDialog` script step so that you can get a feel for how each of these methods actually perform while the User is using your App.
For instance, the method `didChangeStatusBarOrientation` seems to be triggered twice in succession for every single time it is triggered by the OS.
This is the type of scenario you need to be aware of ahead of time while you develop your project, so that you can plan your process flow accordingly.

```objective-c
- (void)applicationDidBecomeActive:(UIApplication *)application {
    NSLog(@"MyAppDelegate: %s", __func__);

    if (FMX_Queue_Script(@"<myFMdbName>", @"appDelegate_applicationDidBecomeActive", kFMXT_Pause, @"A script param", nil)) {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_applicationDidBecomeActive Succeeded");
    } else {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_applicationDidBecomeActive Failed");
    }
}


- (void)completedReturnToForegroundActive {
    NSLog(@"MyAppDelegate: %s", __func__);

    if (FMX_Queue_Script(@"<myFMdbName>", @"appDelegate_completedReturnToForegroundActive", kFMXT_Pause, @"A script param", nil)) {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_completedReturnToForegroundActive Succeeded");
    } else {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_completedReturnToForegroundActive Failed");
    }
}

- (void)applicationWillTerminate:(UIApplication *)application {
    NSLog(@"MyAppDelegate: %s", __func__);

    if (FMX_Queue_Script(@"<myFMdbName>", @"appDelegate_applicationWillTerminate", kFMXT_Pause, @"A script param", nil)) {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_applicationWillTerminate Succeeded");
    } else {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_applicationWillTerminate Failed");
    }
}

- (void)applicationWillResignActive:(UIApplication *)application {
    NSLog(@"MyAppDelegate: %s", __func__);

    if (FMX_Queue_Script(@"<myFMdbName>", @"appDelegate_applicationWillResignActive", kFMXT_Pause, @"A script param", nil)) {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_applicationWillResignActive Succeeded");
    } else {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_applicationWillResignActive Failed");
    }
}

- (void)applicationDidEnterBackground:(UIApplication *)application {
    NSLog(@"MyAppDelegate: %s", __func__);

    if (FMX_Queue_Script(@"<myFMdbName>", @"appDelegate_applicationDidEnterBackground", kFMXT_Pause, @"A script param", nil)) {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_applicationDidEnterBackground Succeeded");
    } else {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_applicationDidEnterBackground Failed");
    }
}

- (void)applicationWillEnterForeground:(UIApplication *)application {
    NSLog(@"MyAppDelegate: %s", __func__);

    if (FMX_Queue_Script(@"<myFMdbName>", @"appDelegate_applicationWillEnterForeground", kFMXT_Pause, @"A script param", nil)) {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_applicationWillEnterForeground Succeeded");
    } else {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_applicationWillEnterForeground Failed");
    }
}

- (void)applicationSignificantTimeChange:(UIApplication *)application {
    NSLog(@"MyAppDelegate: %s", __func__);

    if (FMX_Queue_Script(@"<myFMdbName>", @"appDelegate_applicationSignificantTimeChange", kFMXT_Pause, @"A script param", nil)) {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_applicationSignificantTimeChange Succeeded");
    } else {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_applicationSignificantTimeChange Failed");
    }
}

- (void)applicationDidReceiveMemoryWarning:(UIApplication *)application {
    NSLog(@"MyAppDelegate: %s", __func__);

    if (FMX_Queue_Script(@"<myFMdbName>", @"appDelegate_applicationDidReceiveMemoryWarning", kFMXT_Pause, @"A script param", nil)) {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_applicationDidReceiveMemoryWarning Succeeded");
    } else {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_applicationDidReceiveMemoryWarning Failed");
    }
}


- (void)application:(UIApplication *)application didChangeStatusBarOrientation: (UIInterfaceOrientation)oldStatusBarOrientation {
    // seems to trigger twice every time
    NSLog(@"MyAppDelegate: %s  Calling FMX_Queue_Script appDelegate_didChangeStatusBarOrientation", __func__);

    if (FMX_Queue_Script(@"<myFMdbName>", @"appDelegate_didChangeStatusBarOrientation", kFMXT_Pause, @"A script param", nil)) {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_didChangeStatusBarOrientation Succeeded");
    } else {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_didChangeStatusBarOrientation Failed");
    }
}
```

- - -

### Passing Multiple Script Parameters

Sometimes you want to pass multiple script parameters to a script, and the SDK has a very nice and easy process for doing so.

Modify your existing `triggerScript_didFinishLaunchingWithOptions` method to match the following:

```objective-c
- (void)triggerScript_didFinishLaunchingWithOptions:(NSTimer *)timer {
    NSLog(@"MyAppDelegate: %s", __func__);

    NSDictionary<NSString *, NSString *> *variables = @ {
        @"$a": @"Value of $a",
        @"$z": @"Value of $z"
        };

    if (FMX_Queue_Script(@"<myFMdbName>", @"appDelegate_didFinishLaunchingWithOptions", kFMXT_Pause, @"ima script param", variables)) {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_didFinishLaunchingWithOptions Succeeded");
    } else {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_didFinishLaunchingWithOptions Failed");
    }
}

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    NSLog(@"MyAppDelegate: %s  Calling FMX_Queue_Script appDelegate_didFinishLaunchingWithOptions", __func__);

    NSDictionary<NSString *, NSString *> *variables = @ {
        @"$a": @"Value of $a",
        @"$z": @"Value of $z"
        };

    if (FMX_Queue_Script(@"<myFMdbName>", @"appDelegate_didFinishLaunchingWithOptions", kFMXT_Pause, @"A script param", variables)) {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_didFinishLaunchingWithOptions Succeeded");
    } else {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_didFinishLaunchingWithOptions Failed");
    }

    return true;
}
```

Now modify your existing FileMaker Script `AppDelegate_didFinishLaunchingWithOptions` to match the following:

```objective-c
#========================================
#	Purpose:     FIAS AppDelegate - triggered by delegate method
#	Returns:     none
#	Parameters:  scriptParameter (optional)
#	             iOS variables (optional)
#	Called from: (FIAS) didFinishLaunchingWithOptions
#	Author:      Leland Long
#	Notes:       none
#	History:     2019-07-29 Leland Long - created
#========================================

Allow User Abort [ Off ]
Set Error Capture [ On ]

Set Variable [ $param; Value:Get ( ScriptParameter ) ]
Show Custom Dialog [ Title: "AppDelegate";
                     Message: "Script triggered in db: " &
                     Get ( ScriptName ) & "¶" &
                     "Param: " & $param & "¶" &
                     "sdk_$a: " & $a & "¶" &
                     "sdk_$z: " & $z
                     Default Button: "Excellent",
                     Commit: “Yes” ]

```

Build and run the project. When it runs you should see a Custom Dialog confirming that these 2 variables were successfully passed into your Script.

I have not tested this, but this code snippet leads me to believe that many multiple parameters can be passed into a FileMaker Script by simply following the code structure and syntax of the NSDictionary object we just added.
I have no idea how many or what limits there are, but I would suppose that you would be able to pass in just about whatever you desire to pass in and it would just work.
As an example, here would be the adjustment to that NSDictionary code in order to pass in 4 additional parameters, instead of the 2 we passed earlier:

```objective-c
    NSDictionary<NSString *, NSString *> *variables = @ {
        @"$a": @"Value of $a",
        @"$z": @"Value of $z",
        @"$anotherCoolScriptParameter": @"$1,234,567.99",
        @"$dontForgetAboutMeParameter": @"Lorem ipsum dolor sit amet, consectetur adipiscing elit.
        Nunc tincidunt hendrerit tellus, id vestibulum odio venenatis et.
        Integer arcu est, efficitur sed cursus vitae, efficitur quis nisl."
        };
```

- - -

### The iOS Settings App

First off, here is a quote from Apples Human Interface Guidelines, regarding the Settings App and it's intended purpose:

> Expose infrequently changed configuration options in Settings.
The Settings app is a central location for making configuration changes throughout the system, but people must leave your app to get there.
It’s far more convenient to adjust settings directly within your app.
If you must provide settings that rarely require change, see Implementing an iOS Settings Bundle in Preferences and Settings Programming Guide for developer guidance.

So, with that in mind, let us create a file (called `Settings.bundle`) that will be automatically installed by Xcode, that will generate our own custom settings into the Settings.app.
Then later we will add code to interact with the data our Users will enter into the Settings.app.

Create a `New File` from the File menu, select `Settings Bundle` from the iOS Resource section, leave the default filename as `Settings.bundle`, and be sure to select `Custom Application Resources` in the Group drop-down menu of your project.

Look for your newly created file in the left pane of Xcode (Project navigator) and click the disclosure arrow on the left side of the filename to expand it's contents.

You should see 2 items: a folder titled `en.lproj` and a file named `Root.plist`.

Right-click on the file `Root.plist` and select `OpenAs->SourceCode`.

Note: once you have created this file using the XML Source Code editor, you can go back and look at this same code in the Property List editor.
It can be a bit tricky adding these various fields, dividers, sections, etc. in the Property List editor, so in this guide I will just have you copy and paste the complete code block in the easier Source Code editor.

Replace the code with the following:

```objective-c
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>PreferenceSpecifiers</key>
	<array>
		<dict>
			<key>Title</key>
			<string>Access Credientials</string>
			<key>Type</key>
			<string>PSGroupSpecifier</string>
		</dict>
		<dict>
			<key>Type</key>
			<string>PSTextFieldSpecifier</string>
			<key>Title</key>
			<string>UserName</string>
			<key>Key</key>
			<string>username_preference</string>
			<key>KeyboardType</key>
			<string>Alphabet</string>
			<key>IsSecure</key>
			<false/>
			<key>AutocorrectionType</key>
			<string>No</string>
			<key>AutocapitalizationType</key>
			<string>None</string>
		</dict>
		<dict>
			<key>Type</key>
			<string>PSTextFieldSpecifier</string>
			<key>Title</key>
			<string>Password</string>
			<key>Key</key>
			<string>password_preference</string>
			<key>IsSecure</key>
			<false/>
			<key>AutocorrectionType</key>
			<string>No</string>
			<key>AutocapitalizationType</key>
			<string>None</string>
		</dict>
		<dict>
			<key>Type</key>
			<string>PSGroupSpecifier</string>
			<key>Title</key>
			<string>Datasource</string>
		</dict>
		<dict>
			<key>Type</key>
			<string>PSTextFieldSpecifier</string>
			<key>Title</key>
			<string>URL</string>
			<key>Key</key>
			<string>datasource_preference</string>
			<key>KeyboardType</key>
			<string>Alphabet</string>
			<key>IsSecure</key>
			<false/>
			<key>AutocorrectionType</key>
			<string>No</string>
			<key>AutocapitalizationType</key>
			<string>None</string>
		</dict>
	</array>
	<key>StringsTable</key>
	<string>Root</string>
</dict>
</plist>

```

Run the Project.

Nothing will look different at this stage.

Now Quit the App, either in the Simulator or on the device.

Find the Settings.app and Open it.

Scroll down to the bottom of the list of Settings, and you should now see your App listed.

Tap on your App.

![custom settings image](/images/Settings.png)*Custom App Settings*

And look at that, several fields for you to enter Settings, or Preferences, or whatever you wish to have the User enter in this Settings.app.

Now if you go back to Xcode and look at the `Root.plist` file using the Property List editor, and expand all the disclosure arrows, you should see the required format for creating whatever fields you desire to use.
The field option that is key to Xcode being able to communicate with these fields is labeled `Identifier`.
Whatever you enter into this `Identifier` field is the name of the variable you will use to pass the user-entered data into your code, and eventually into FileMaker as a parameter.

![bundle plist image](/images/bundle_plist.png)*Settings.bundle Root.plist*

- - -

### Extracting the Settings.app User Data Into Xcode AppDelegate

Modify your existing AppDelegate code (the interface section near the top of the file) to match the following:

```objective-c
@interface MyAppDelegate : UIResponder <UIApplicationDelegate>
{
    NSString *settingsUsername;
    NSString *settingsPassword;
    NSString *settingsDatasource;
}
@property (strong, nonatomic) UIWindow *window;
@end
```

Then modify your existing `triggerScript_didFinishLaunchingWithOptions` method to match the following:

```objective-c
- (void)triggerScript_didFinishLaunchingWithOptions:(NSTimer *)timer {
    NSLog(@"MyAppDelegate: %s", __func__);

    if (settingsUsername == nil) {
        settingsUsername = @"";
    }
    if (settingsPassword == nil) {
        settingsPassword = @"";
    }
    if (settingsDatasource == nil) {
        settingsDatasource = @"<invalid>";
    }

    NSDictionary<NSString *, NSString *> *variables = @ {
        @"$username": settingsUsername,
        @"$password": settingsPassword
        };

    if (FMX_Queue_Script(@"<myFMdbName>", @"appDelegate_didFinishLaunchingWithOptions", kFMXT_Pause, settingsDatasource, variables)) {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_didFinishLaunchingWithOptions Succeeded");
    } else {
        NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_didFinishLaunchingWithOptions Failed");
    }
}
```

Now modify your `` method to match the following:

```objective-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    NSLog(@"MyAppDelegate: %s", __func__);

    // FIAS needs a little time to begin responding to script requests, so add a slight delay
    [NSTimer scheduledTimerWithTimeInterval: 10.0
                                     target: self
                                   selector: @selector(triggerScript_didFinishLaunchingWithOptions:)
                                   userInfo: nil
                                    repeats: NO];

    // Pulling data from Settings
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    settingsUsername = [defaults objectForKey:kUserNameKey];
    settingsPassword = [defaults objectForKey:kPasswordKey];
    settingsDatasource = [defaults objectForKey:kDatasourceKey];
    NSLog(@"MyAppDelegate - standardUserDefaults - username: %@", settingsUsername);
    NSLog(@"MyAppDelegate - standardUserDefaults - password: %@", settingsPassword);
    NSLog(@"MyAppDelegate - standardUserDefaults - datasource: %@", settingsDatasource);

    return true;
}
```

Then modify your FileMaker Script `` to match the following:

```objective-c
#========================================
#	Purpose:     FIAS AppDelegate - triggered by delegate method
#	Returns:     none
#	Parameters:  scriptParameter (optional)
#	             iOS variables (optional)
#	Called from: (FIAS) didFinishLaunchingWithOptions
#	Author:      Leland Long
#	Notes:       none
#	History:     2019-07-29 Leland Long - created
#========================================

Allow User Abort [ Off ]
Set Error Capture [ On ]

Set Variable [ $param; Value:Get ( ScriptParameter ) ]
Show Custom Dialog [ Title: "AppDelegate";
                     Message: "Script triggered in db: " &
                     Get ( ScriptName ) & "¶" &
                     "Param: " & $param & "¶" &
                     "sdk_$username: " & $username & "¶" &
                     "sdk_$password: " & $password
                     Default Button: "Excellent",
                     Commit: “Yes” ]

```

Before running your app, go back into the Settings.app and enter in some random values into those 3 fields so that they are not blank.

Then close the Settings.app and return to Xcode.

Now Run your project.

If all went well you should have seen the user-entered Settings values show up in the Console and inside the Custom Dialog in the App itself.

- - -

### Conclusion

You now should have completed what we set out to do.
There are several ways you can customize these examples to match what you would like to do in your own app.

Hopefully this has been a helpful and in-depth guide on putting the AppDelegate to work for you in a useful and meaningful way.

Happy FileMaking.

- - -

### Further Reading

You can read more about the various AppDelegate Methods here:

https://developer.apple.com/documentation/uikit/uiapplicationdelegate

- - -

### A Future Feature Request

It would be great to have a way to communicate between the database solution file and the AppDelegate Methods.
Maybe with a pre-determined standard global variable; something like `$$SDK.SHARED.DATA`.
I'm not sure what everyone would use it for, but here is an idea I have come up with for helping to solve to delay issue we dealt with earlier in this project.
Instead of statically creating a delay of 2.0 - 10.0 seconds, it would be a much better solution to create a loop in the Delegate code, maybe with a 0.5 - 1.0 second delay between iterations, to check for the existence of a certain value in this proposed global variable.
For example:

```objective-c
@interface MyAppDelegate : UIResponder <UIApplicationDelegate>
{
    NSTimer *myTimer;
    NSString *sdk_shared_data;
}
@property (strong, nonatomic) UIWindow *window;
@end


@implementation MyAppDelegate

- (void)triggerScript_didFinishLaunchingWithOptions:(NSTimer *)timer {
    NSString *fmpVariable = sdk_shared_data;

    if ([fmpVariable isEqual: @"helloFromFileMakerGlobalVariable"]) {

        [myTimer invalidate];
        myTimer = nil;

        if (FMX_Queue_Script(@"<myFMdbName>", @"appDelegate_didFinishLaunchingWithOptions", kFMXT_Pause, @"none", nil)) {
                NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_didFinishLaunchingWithOptions Succeeded");
            } else {
                NSLog(@"MyAppDelegate: FMX_Queue_Script appDelegate_didFinishLaunchingWithOptions Failed");
            }
    }
}

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    NSLog(@"MyAppDelegate: %s", __func__);

    myTimer = [NSTimer scheduledTimerWithTimeInterval: 1.0
                                            target: self
                                            selector: @selector(triggerScript_didFinishLaunchingWithOptions:)
                                            userInfo: nil
                                            repeats: YES];

    return true;
}
```
