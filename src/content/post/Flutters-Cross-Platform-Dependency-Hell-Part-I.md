---
title: "Flutter's Cross Platform Dependency Hell - Part I"
publishDate: 2024-05-12 00:00:00 +0530
tags: [ flutter,programming ]
description: "Issue with the way flutter handles federated plugins"
slug: Flutters-Cross-Platform-Dependency-Hell-Part-I
---

```
Because no versions of flutter_secure_storage match >9.0.0 <9.1.0 and flutter_secure_storage >=9.1.0 <9.1.1 depends on flutter_secure_storage_web ^1.2.0, flutter_secure_storage >9.0.0 <9.1.1 requires flutter_secure_storage_web ^1.2.0.
And because flutter_secure_storage >=9.1.1 depends on flutter_secure_storage_web ^1.2.1, flutter_secure_storage >9.0.0 requires flutter_secure_storage_web ^1.2.0.
And because facebook_auth_desktop >=1.0.1 depends on flutter_secure_storage ^9.0.0 and flutter_secure_storage 9.0.0 depends on flutter_secure_storage_web ^1.1.1, facebook_auth_desktop >=1.0.1 requires flutter_secure_storage_web ^1.1.1.
Because no versions of flutter_facebook_auth match >6.2.0 <7.0.0 and flutter_facebook_auth 6.2.0 depends on facebook_auth_desktop ^1.0.3, flutter_facebook_auth ^6.2.0 requires facebook_auth_desktop ^1.0.3.
Thus, flutter_facebook_auth ^6.2.0 requires flutter_secure_storage_web ^1.1.1.
And because every version of flutter_secure_storage_web depends on js ^0.6.3 and mixpanel_flutter >=2.3.0 depends on js ^0.7.1, flutter_facebook_auth ^6.2.0 is incompatible with mixpanel_flutter >=2.3.0.
So, because my_app depends on both mixpanel_flutter ^2.3.1 and flutter_facebook_auth ^6.2.0, version solving failed.


You can try the following suggestion to make the pubspec resolve:
* Consider downgrading your constraint on mixpanel_flutter: flutter pub add mixpanel_flutter:^2.2.0
```

It's difficult to resist the urge to bang your head on the desk when you encounter an error like this. This is how
Flutter reports a dependency conflict. A dependency conflict, while frustrating, is nothing uncommon. Even more so when
using a cross-platform framework, where you have to depend on third-party implementations of some popular libraries. But
this one is incredibly ridiculous. But, before diving into it, Lets take a brief look at how packages work in Flutter.

There are two ways to develop a flutter package that relies on native APIs:

**1) The simple approach:**

Put the platform-specific implementations in the package under the platform's folder.
See [mixpanel-flutter](https://github.com/mixpanel/mixpanel-flutter/tree/main).

```
   mixpanel_flutter 2.3.1
    ├── flutter_web_plugins 0.0.0
    │   ├── characters...
    │   ├── collection...
    │   ├── flutter...
    │   ├── material_color_utilities...
    │   ├── meta...
    │   └── vector_math...
    ├── flutter...
    └── js...
```

**2) The recommended way:**

Separate out the platform-specific implementations into different packages. This is called a federated plugin in
Flutter. It consists of the following:

- The **app-facing package**, which the plugin's user adds to their pubspec.
- The **platform-specific package(s)** which has the native code, the app-facing package depeneds on these packages.
- And the **platform interface package**, which declares a common interface that any platform package must implement to
  support the app-facing package.

For example, See the dependency tree
of [google sign in](https://github.com/flutter/packages/tree/main/packages/google_sign_in)
package

```
├── google_sign_in 6.2.1
│   ├── google_sign_in_android 6.1.23
│   │   ├── flutter...
│   │   └── google_sign_in_platform_interface...
│   ├── google_sign_in_ios 5.7.6
│   │   ├── flutter...
│   │   └── google_sign_in_platform_interface...
│   ├── google_sign_in_platform_interface 2.4.5
│   │   ├── flutter...
│   │   └── plugin_platform_interface...
│   ├── google_sign_in_web 0.12.4
│   │   ├── google_identity_services_web 0.3.1+1
│   │   │   ├── meta...
│   │   │   └── web...
│   │   ├── web 0.5.1
│   │   ├── flutter...
│   │   ├── flutter_web_plugins...
│   │   ├── google_sign_in_platform_interface...
│   │   └── http...
│   └── flutter...
```

Now, let's go back to the original issue. `my_app` is a flutter app **targeting only Android & iOS**. It depends on two
packages `flutter_facebook_auth` & `mixpanel_flutter` . The full dependency tree looks like this:

```
my_app 1.0.0+1
├── flutter_facebook_auth 6.2.0
│   ├── facebook_auth_desktop 1.0.3
│   │   ├── flutter_secure_storage 9.1.1
│   │   │   ├── flutter_secure_storage_linux 1.2.1
│   │   │   │   ├── flutter...
│   │   │   │   └── flutter_secure_storage_platform_interface...
│   │   │   ├── flutter_secure_storage_macos 3.1.0
│   │   │   │   ├── flutter...
│   │   │   │   └── flutter_secure_storage_platform_interface...
│   │   │   ├── flutter_secure_storage_platform_interface 1.1.1
│   │   │   │   ├── flutter...
│   │   │   │   └── plugin_platform_interface...
│   │   │   ├── flutter_secure_storage_web 1.2.1
│   │   │   │   ├── flutter...
│   │   │   │   ├── flutter_secure_storage_platform_interface...
│   │   │   │   ├── flutter_web_plugins...
│   │   │   │   └── js...
│   │   │   ├── flutter_secure_storage_windows 3.1.2
│   │   │   │   ├── ffi 2.1.2
│   │   │   │   ├── path_provider 2.1.3
│   │   │   │   │   ├── path_provider_android 2.2.4
│   │   │   │   │   └── ....
│   │   │   │   └── path...
│   │   │   ├── flutter...
│   │   │   └── meta...
│   │   ├── http 1.2.1
│   │   │   ├── http_parser 4.0.2
│   │   │   └── meta...
│   │   ├── flutter...
│   │   └── flutter_facebook_auth_platform_interface...
│   ├── flutter_facebook_auth_platform_interface 5.0.0
│   │   ├── plugin_platform_interface 2.1.8
│   │   │   └── meta...
│   │   └── flutter...
│   ├── flutter_facebook_auth_web 5.0.1
│   │   ├── flutter...
│   │   ├── flutter_facebook_auth_platform_interface...
│   │   ├── flutter_web_plugins...
│   │   └── js...
│   └── flutter...
└── mixpanel_flutter 2.3.1
    ├── flutter_web_plugins 0.0.0
    │   ├── characters...
    │   ├── collection...
    │   ├── flutter...
    │   ├── material_color_utilities...
    │   ├── meta...
    │   └── vector_math...
    ├── flutter...
    └── js...
```

Notice how adding a single package leads to a cascade of redundant dependencies meant for different platforms, which are
not even the targets of `my_app`!

Observe that `mixpanel_flutter` depends on `js ^0.7.0`. While `my_app` also has a transitive dependency
on `facebook_auth_desktop` which in turn has a transitive dependency on `flutter_secure_storage_web`!
And since `flutter_secure_storage_web` uses `js ^0.6.3`, it's incompatible with  `mixpanel_flutter`. Thus causing the
dependency conflict.

By now, you must have realized why this situation is so ridiculous. Why does `my_app` need to depend
on `facebook_auth_desktop` and `flutter_secure_storage_web` at all, when it doesn't even target desktop or web
platforms? Because of this behavior, I am left to solve a dependency conflict arising out of redundant dependencies that
will never be used in the project.

Thankfully, I found a hack to fix the issue in the package's GitHub repo. Adding the latest version of the `js` package
in the pubspec under `dependency_overrides` fixes the issue. Nevertheless, the problem should have never arisen; Flutter
could have intelligently pruned the dependency tree by ignoring platform-specific dependencies not meant for my target
platforms.
