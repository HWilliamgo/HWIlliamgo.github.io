> 原文地址 https://blog.stylingandroid.com/package-name-vs-application-id/

All Android developers should understand that the Package Name that we choose for our app is very important. I’m referring to the Package Name of the application itself (which gets declared in the Manifest) rather than the Java package name (although often they will be identical. It was only recently, when I was tackling a somewhat obscure issue as part of my day job, that I realised that there is actually a subtle, but important, difference between the Package Name and the Application ID. In this article we’ll look at the difference between the two.  

[![image](https://upload-images.jianshu.io/upload_images/7177220-f2945e2ab187c3f2.25&resize=150,150&ssl=1?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://i2.wp.com/blog.stylingandroid.com/wp-content/uploads/2017/02/box.png?ssl=1)Let’s begin with a reminder of the basics. The Package Name is, as we’ve already mentioned, defined as part of the Manifest and represents the identity of any given app both on an individual device, and on the Google Play store (or any other app store for that matter). Therefore your package name has to be unique. However it is sometimes useful to be able to produce separate APKs with different identities (e.g. different IDs for debug & release builds so that they can be published as distinct entities in Fabric Beta). To facilitate this we can control the `applicationId` in our `build.gradle` and this enables us have distinct package names. We can see this in a simple project. The common Manifest is defined like this:

``` xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
  xmlns:tools="http://schemas.android.com/tools"
  package="com.stylingandroid.packagename">

  <application
    android:allowBackup="false"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:supportsRtl="true"
    android:theme="@style/AppTheme"
    tools:ignore="GoogleAppIndexingWarning">
    <activity android:name=".MainActivity">
      <intent-filter>
        <action android:name="android.intent.action.MAIN" />

        <category android:name="android.intent.category.LAUNCHER" />
      </intent-filter>
    </activity>
  </application>

</manifest>
```



We have a base package name of `com.stylingandroid.packagename`. Next we have our `build.gradle`:

``` groovy
apply plugin: 'com.android.application'

android {
    compileSdkVersion 25
    buildToolsVersion "25.0.0"
    defaultConfig {
        applicationId "com.stylingandroid.packagename"
        minSdkVersion 23
        targetSdkVersion 25
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            applicationIdSuffix ".release"
        }
        debug {
            applicationIdSuffix ".debug"
        }
    }
}

dependencies {
    compile 'com.android.support:appcompat-v7:25.1.0'
}
```



Here we add a distinct suffix to the `applicationId` for both debug and release builds. It is possible to override the entire `applicationId` (i.e. replace the whole of the package name with the `applicationId`, but we’ll just add a suffix here.

If we now compare the two Manifests which get added to the two APKs we can see how distinct package names are specified in each Manifest:

[![image](https://upload-images.jianshu.io/upload_images/7177220-c917a2a94c30301b.25&resize=708,198&ssl=1?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://i2.wp.com/blog.stylingandroid.com/wp-content/uploads/2017/02/Debug-vs.-Release-manifest.png?ssl=1)

There should be nothing here that should be surprising to the vast majority of Android developers, but from here on it is important to understand the difference between the Package Name – which gets declared up-front in the base Manifest; and the application ID – which gets declared in our `build.gradle`. The package name controls much of what occurs during the build, but the application ID only gets applied right at the very end. For proof of this, take a look [here](https://developer.android.com/studio/build/application-id.html#change_the_package_name), specifically the final note which states:

> Although you may have a different name for the manifest package and the Gradle applicationId, the build tools copy the application ID into your APK’s final manifest file at the **end** of the build

(The emphasis of the word **end** is mine)

So why is this important? Let’s begin by adding some code to our main _Activity_ firstly a simple layout containing two _TextView_s:

``` xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
  xmlns:tools="http://schemas.android.com/tools"
  android:layout_width="match_parent"
  android:layout_height="match_parent"
  android:paddingTop="@dimen/padding_vertical"
  android:paddingBottom="@dimen/padding_vertical"
  android:paddingStart="@dimen/padding_horizontal"
  android:paddingEnd="@dimen/padding_horizontal"
  android:orientation="vertical"
  tools:context="com.stylingandroid.packagename.MainActivity">
 
  <TextView
    android:id="@+id/package_name"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content" />
 
  <TextView
    android:id="@+id/application_id"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content" />
 
</LinearLayout>
```



To the _Activity_ itself we’ll add some code to set these to the package name & the applicationId:

``` java
package com.stylingandroid.packagename;
 
import android.app.Activity;
import android.os.Bundle;
import android.widget.TextView;
 
public class MainActivity extends Activity {
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
 
        TextView packageNameTextView = (TextView) findViewById(R.id.package_name);
        TextView applicationIdTextView = (TextView) findViewById(R.id.application_id);
 
        packageNameTextView.setText(getString(R.string.package_name, BuildConfig.class.getPackage().toString()));
        applicationIdTextView.setText(getString(R.string.application_id, BuildConfig.APPLICATION_ID));
    }
}
```



The three highlighted lines actually explain what happens during the build.

The first thing to note is the Java package name of _MainActivity_ itself. As is normal we have matched this to the package name which we initially defined in the Manifest (shown earlier).

Secondly, when we come to reference our _BuildConfig_, we don’t need to fully qualify the package name, or import the package. That is because the `BuildConfig.java` and `R.java` both got generated in the package name that was specified _originally_ in the Manifest – `com.stylingandroid.packagename`.

Thirdly, although the package name has not been set in the Manifest at the point when our code is compiled, we can still gain access to the final application ID for the current build flavor via the _BuildConfig.APPLICATION_ID_ constant.

If we run this code it confirms this:

[![image](https://upload-images.jianshu.io/upload_images/7177220-4478359c4f97ea01.25&resize=708,530&ssl=1?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://i0.wp.com/blog.stylingandroid.com/wp-content/uploads/2017/02/app.png?ssl=1)

This won’t cause anyone any issues in the vast majority of cases but if, for example, you are generating code from an annotation processor, it can be important to understand when to use the package name over the applicationId and vice versa.

So, why is it left so late to set the applicationID? The simple reason is that it would cause no end of problems if it was set earlier in the process. If the `BuildConfig.java` and `R.java` files was set to the applicationId rather than the package name, then the code which referenced these would need to have different imports for different build variants – in other words you would need to have separate source sets (with huge amounts of duplication) for your different build flavors in order to access these. This would have some quite profound implications for the maintainability of our code.

As I have said, an understanding of precisely how this all works is not necessary for developers to be able to write android apps. But if, like me, you encounter some issues with generating code in the correct java package then this can be invaluable. When I was searching for information on this, the only thing I could find was the page on the official documentation (which I quoted and linked to earlier), so it seemed worth sharing what I learned. Apologies to anyone expecting the usual UI/UX-centric subject matter but hopefully this will be of interest to many as us developers are an inquisitive bunch!

The source code for this article is available [here](https://github.com/StylingAndroid/PackageName).

© 2017, [Mark Allison](https://blog.stylingandroid.com/). All rights reserved.
