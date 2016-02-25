# XVim - Xcode plugin for Vim keybindings
XVim is a plugin for Xcode4+ which enables certain vim-like features.

## Goal
The goal of this project is to provide confortable environment for Vim users to
use Xcode4+ to develop their software. This means the goal is NOT to provide
full compatiblity to Vim operations. Some keybinds may behave differently than
Vim's due to the differences between Vim's traditional terminal environment and
the "*feature-rich*"(TM) IDE environment of Xcode.

### Short term
This project started in January 2012.
Major short-term goals have been completed and included:
* implement major operations:
    * navigation with "h,j,k,l"
    * search with "/,?"
    * replace with ":s"

See [the project page][project] for a list of features and their implementation
status.

## Internal Design

It may be difficult to implement new features or fix bugs without understanding
the overall design of the plugin. The following sections are an attempt to
concisely explain the plugin's architecture.

### Big picture
This plugin consists of 2 main parts: ***Hooks*** and ***Operations***.

For further details, please see the [source][hooks-src] [code][op-src].

#### Hooks
XVim operates by 'hooking' some methods of Xcode's own classes, the most
important of which is the `keyDown:` method in the `DVTSourceTextView` class.

XVim essentially intercepts the `keyDown:` message to the instance of
`DVTSourceTextView` and sends the information it gathers to the
**[Operations](#operations)** section.

`DVTSourceTextView` is the class of the source-code editing window in Xcode.

#### Operations
Once the `keyDown:` message has been intercepted, XVim needs to handle the
keyboard and mouse input, perform the associated operation, and finally
reflect the results of that operation in the current instance of
`DVTSourceTextView` (or other views, if they apply).

### Loading
Unfortunately, Xcode's plugin system is largely undocumented and the community
does not fully understand the loading sequence of plugins. The only thing we (or
maybe *I*) know is that bundles in `~/Library/Application
Support/Developer/Shared/Xcode/Plug-ins/` are loaded automatically at runtime.

Once the plugin has loaded, the plugin's principal class (in our case `XVim`) is
sent the [`+load`][xvim+load] message. This is the main entry point of the plugin.

In the [`+load`][xvim+load] method, the [`XVimHookManager`][xvhm-src] class is
sent the [`+hookWhenPluginLoaded`][xvhm-hook] message, which in turn sends
messages to each of the relevant Xcode classes. See the
[source code][xvim-main-src] [for details][xvhm-src].

### Hooks
XVim's hooks are essentially [method-swizzling][swizzle] that replace the
implementation of certain methods in Xcode classes with XVim's implementation.

For example:
> [`DVTSourceTextView`][dvt-src]'s [`-keyDown:`][dvt-keyDown] method
> is swizzled with the [`-xvim_keyDown`][xvim-keyDown] implementation, so when an
> instance of [`DVTSourceTextView`][dvt-src] is sent the `-keyDown:` message, it
> performs the routines defined in [`-xvim_keyDown`][xvim-keyDown].

[NSHipster][hipster] has an [excellent write-up about method swizzling][hip-sw]
(and [Objective-C runtime-hackery in general][hip-rt-hack]) that I highly
recommend reading.

> ### An Example
> This is all a bit complicated, so the following is an example of what happens
> when XVim swizzles [`DVTSourceTextView`][dvt-src]'s [`-keyDown:`][dvt-keyDown]
> method.
>
> 1. **Loading & Initialization (both class and instance)**
>    The class is loaded, and the method is swizzled. If you don't know what I
>    am talking about, go back and read the [NSHipster article][hip-sw]!  I'll
>    wait right here...
>
> 2. **A key is pressed**
>    The user presses a key while in the source editor window of Xcode, and XVim's
>    implementation of the `-keyDown:` method is called, wherein XVim uses
>    information about the key press (the class that received the message, the
>    key that was pressed, the context of the key press, etc.) to determine the
>    proper ***Evaluator*** thereof.
>
>    A message is sent to the proper Evaluator with the key input information,
>    where the input will be evaluated and further operation will take place.
>
> 3. **The Evaluator receives the key press information**
>    The Evaluator's main job is to keep the current state of key input, as well
>    as perform some resultant operations on the current text view.
>
>    When the Evaluator evaluates (for lack of a better word) the input, but
>    needs further input to determine the resultant operation (ie `g` was
>    pressed, and we need to know what the next key will be to determine what to
>    do), the return value of the message will be the next Evaluator.
>
> 4. **An operation is performed**
>    The evaluator then performs the operation (or tells another object to do
>    so), and then manipulates the current text view (or tells another object to
>    do so) with the results of said operation.

## Xcode internal classes
You can find header files for Xcode's internal classes [here][xc-intern]. They
were generated with a nifty tool called [class-dump][class-dump] that takes
advantage of Objective-C's dynamic runtime to 'dump' information about library
symbols (including classes and their methods, but not implementations).

[`DVTSourceTextView`][dvt-src] in particular has a number of methods that would
be potentially useful in implementing vim-like operations in Xcode.

## Xcode Reverse-Engineering
The following is some information gleaned from reverse-engineering Xcode and
inspecting the generated header files.

### [`IDEWorkspaceWindow`][ide-ww]

[`IDEWorkspaceWindow`][ide-ww] is the main window class for Xcode. Following
general Cocoa design principles, [`IDEWorkspaceWindow`][ide-ww] is mainly
controlled by a window controller: in this case, the controller class is
[`IDEWorkspaceWindowController`][ide-wwc].

If you want to access to current window's controller, use following code:
```objective-c
    [NSClassFromString(@"IDEWorkspaceWindowController")
        performSelector:@selector("workspaceWindowControllerForWindow")];
```
or, if you have included the corresponding header from
[`class-dump`][class-dump], just
```objective-c
    [IDEWorkspaceWindowController workspaceWindowControllerForWindow];
```

### [`IDEEditorArea`][ide-ed]
[`IDEEditorArea`][ide-ed] is the area which includes all of the source editors
(primary, assistant(s)), the debug area, and so on. These children views are
easily accessible through this class. If you want to make changes to any of the
children views, access them through this class.

### [`DVTChooserView`][dvt-choose]
This class can be used as the 'thick border' in Xcode. The top navigation bar is an instance of this class, and we can reuse it to create similar functionality.

If you want to do so, use the following code:
```objective-c
    [[NSClassFromString(@"DVTChooserView") performSelector:@"alloc"] init];
```
or, with the headers (just like a normal Objective-C object):
```objective-c
    [[DVTChooserView alloc] init];
```

You can add subviews to this view just like any other view, but beware:
> you must remove all subviews from your instance of
> [`DVTChooserView`][dvt-choose] before doing so, or some existing subviews will
> obscure or hide your subview.

### Notifications
Xcode also uses a lot of notifications for distributed messaging, and are
distinguishable by their class prefixes `DVT` and `IDE`. Listening to these
notifications is a very useful way to determine what events Xcode is firing off,
as well as what classes to look into to provide functionality.

#### Playing with Notifications
A convenient way to spy on notifications is to create a new Xcode project with
the [Xcode Plugin Template][xcp-template] and add a bit of code to the principal
class.

##### Getting Set Up:
As soon as you open up the new project, build and run it. Your main Xcode
instance will open a new 'child' instance of Xcode, where your plugin will be
loaded.

While in the 'child' instance, go ahead and open up the `Edit` menu, and click
the newly-added `Do Action` menu item.

> ***If you don't see it, or if Xcode is spitting out log messages about how and
> why it couldn't load your plugin, DO NOT FRET!***
>
> Run the following command in your terminal and copy the output into the
> plugin's info.plist, under the DVTPluginCompatibilityUUIDs array.
    ```shell
    $ defaults read /Applications/Xcode.app/Contents/Info DVTPlugInCompatibilityUUID
    ```
>
> Then, delete the built products
    ```shell
    $ cd ~/Library/Application\ Support/Developer/Shared/Xcode/Plug-ins/
    $ rm -r [MY_PLUGIN].plugin
    ```
> and then restart Xcode.

##### Adding the Notification Observer Code

Add the following property to your principal plugin class:
```objective-c
@property(nonatomic, strong) NSMutableSet *notificationSet;
```

Next, add this to the `-initWithBundle:` method:
```objective-c
[[NSNotificationCenter defaultCenter] addObserver:self
                                         selector:@selector(handleNotification:)
                                             name:nil
                                           object:nil];

self.notificationSet = [[NSMutableSet alloc] init];
```

You'll also have to implement the `-handleNotification:` method:
```objective-c
- (void)handleNotification:(NSNotification *)notification
{
    if(![self.notificationSet containsObject:notification.name]) {
        NSLog(@"%@, %@", notification.name, [notification.object class]);
        [self.notificationSet addObject:notification.name];
    }
}
```

Now, every time Xcode posts a notification (that you haven't seen before), a log
message will be printed with the name of the notification and the class of the
object that posted it. Handy!

Play around with different buttons that implement similar functionality, and try
to determine what objects are responsible for that functionality.

Good luck!

[project]: Documents/Users/FeatureList.md
[hooks-src]: XVim/XVimHookManager.h
[op-src]: XVim/
[xvim+load]: XVim/XVim.m
[xvhm-src]: XVim/XVimHookManager.h
[xvhm-hook]: XVim/XVimHookManager.h
[xvim-main-src]: XVim/XVim.m
[swizzle]: http://nshipster.com/method-swizzling/
[dvt-src]: XcodeClasses/Xcode7.0/SharedFrameworks
[dvt-keyDown]: XcodeClasses/Xcode7.0/SharedFrameworks
[xvim-keyDown]: XVim/DVTSourceTextView+XVim.m
[hip-sw]: http://nshipster.com/method-swizzling/
[hipster]: http://nshipster.com
[hip-rt-hack]: http://nshipster.com/associated-objects/
[xc-intern]: XcodeClasses
[class-dump]: http://stevenygard.com/projects/class-dump/
[ide-ww]: XcodeClasses
[ide-wwc]: XcodeClasses
[ide-ed]: XcodeClasses
[dvt-choose]: XcodeClasses
[xcp-template]: XcodeClasses
