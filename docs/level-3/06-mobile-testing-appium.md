# 06 · Mobile Testing with Appium

Appium extends the exact automation model you already know from Selenium
WebDriver — find an element, act on it, assert — to native and hybrid
mobile apps on Android and iOS. If Level 1's `driver.findElement(By.id(...))`
is comfortable, Appium's API will feel almost identical; the differences are
in locators, drivers, and what "the page" even means on a phone.

## 1. How Appium fits together

- **Appium Server** — a Node.js process implementing the WebDriver protocol
  (the same protocol RemoteWebDriver used in Module 04's Selenium Grid).
- **Driver** — `UiAutomator2` for Android, `XCUITest` for iOS; translates
  WebDriver calls into platform-native automation.
- **Capabilities** — a map telling Appium which app, which device/emulator,
  which platform — the mobile equivalent of `ChromeOptions`.
- Your Java test talks to Appium Server exactly like it talks to a Selenium
  Grid hub: over HTTP, via a `RemoteWebDriver` subtype.

## 2. Setup

```xml
<dependency>
    <groupId>io.appium</groupId>
    <artifactId>java-client</artifactId>
    <version>9.3.0</version>
    <scope>test</scope>
</dependency>
```

```java
package com.example.mobile;

import io.appium.java_client.android.AndroidDriver;
import io.appium.java_client.android.options.UiAutomator2Options;
import java.net.URL;

public class AppiumDriverFactory {

    public static AndroidDriver create() throws Exception {
        UiAutomator2Options options = new UiAutomator2Options()
                .setDeviceName("Pixel_7_API_34")
                .setApp("/path/to/app-debug.apk")
                .setAutomationName("UiAutomator2")
                .setNewCommandTimeout(java.time.Duration.ofSeconds(60));

        return new AndroidDriver(new URL("http://127.0.0.1:4723"), options);
    }
}
```

`AndroidDriver` implements `WebDriver`, so everything from Level 1 —
explicit waits, `findElement`, Page Objects from Level 2 — carries over
unchanged in structure; only the locator strategy and the driver type
change.

## 3. Mobile locators

```java
import io.appium.java_client.AppiumBy;
import org.openqa.selenium.By;

// Resource id (Android) -- most stable, prefer this first
driver.findElement(AppiumBy.id("com.example.app:id/username_field"));

// Accessibility id -- works cross-platform (Android content-desc / iOS accessibilityIdentifier)
driver.findElement(AppiumBy.accessibilityId("login-button"));

// Android UiAutomator selector -- powerful but verbose and Android-only
driver.findElement(AppiumBy.androidUIAutomator(
        "new UiSelector().text(\"Sign in\")"));

// XPath -- last resort, same trade-offs as in Selenium (Level 1 Module 08)
driver.findElement(By.xpath("//android.widget.Button[@text='Sign in']"));
```

Prefer, in order: resource-id / accessibility-id, then UiAutomator selector,
then XPath — the same "specific over brittle" hierarchy taught for CSS
selectors in Level 1, just with mobile-specific tools at the top instead of
`id`/`css`.

## 4. A login test

```java
package com.example.mobile;

import io.appium.java_client.android.AndroidDriver;
import io.appium.java_client.AppiumBy;
import org.junit.jupiter.api.*;
import org.openqa.selenium.support.ui.WebDriverWait;
import java.time.Duration;

import static org.junit.jupiter.api.Assertions.*;

class MobileLoginTest {

    private AndroidDriver driver;
    private WebDriverWait wait;

    @BeforeEach
    void setUp() throws Exception {
        driver = AppiumDriverFactory.create();
        wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }

    @Test
    void validLoginShowsDashboard() {
        driver.findElement(AppiumBy.accessibilityId("username-field")).sendKeys("alice");
        driver.findElement(AppiumBy.accessibilityId("password-field")).sendKeys("correct-horse");
        driver.findElement(AppiumBy.accessibilityId("login-button")).click();

        wait.until(d -> ((AndroidDriver) d)
                .findElements(AppiumBy.accessibilityId("dashboard-title"))
                .size() > 0);

        assertTrue(driver.findElement(AppiumBy.accessibilityId("dashboard-title")).isDisplayed());
    }
}
```

The explicit wait matters even more on mobile than web: app transitions
involve animations and sometimes a fresh Activity/ViewController load, both
slower and less predictable than a web page's DOM update.

## 5. Gestures

```java
import io.appium.java_client.android.nativekey.AndroidKey;
import io.appium.java_client.android.nativekey.KeyEvent;
import org.openqa.selenium.interactions.PointerInput;
import org.openqa.selenium.interactions.Sequence;
import java.util.List;
import java.time.Duration;

// Native back button
driver.pressKey(new KeyEvent(AndroidKey.BACK));

// Swipe up (e.g. to scroll a list) using W3C Actions
PointerInput finger = new PointerInput(PointerInput.Kind.TOUCH, "finger");
Sequence swipeUp = new Sequence(finger, 0)
    .addAction(finger.createPointerMove(Duration.ZERO, PointerInput.Origin.viewport(), 500, 1500))
    .addAction(finger.createPointerDown(PointerInput.MouseButton.LEFT.asArg()))
    .addAction(finger.createPointerMove(Duration.ofMillis(300), PointerInput.Origin.viewport(), 500, 300))
    .addAction(finger.createPointerUp(PointerInput.MouseButton.LEFT.asArg()));
driver.perform(List.of(swipeUp));
```

## 6. iOS differs mainly in the options object and locators

```java
import io.appium.java_client.ios.IOSDriver;
import io.appium.java_client.ios.options.XCUITestOptions;

XCUITestOptions options = new XCUITestOptions()
        .setDeviceName("iPhone 15")
        .setApp("/path/to/App.app")
        .setAutomationName("XCUITest");

IOSDriver iosDriver = new IOSDriver(new URL("http://127.0.0.1:4723"), options);
iosDriver.findElement(AppiumBy.accessibilityId("login-button"));  // accessibilityId works on both platforms
```

Writing tests against `AppiumBy.accessibilityId(...)` wherever the app
exposes one (rather than platform-specific id/UIAutomator selectors) is what
lets the same test class run against both platforms with only the driver
factory swapped — mirroring the cross-browser pattern from Module 04.

I did not run any of this in this environment — no Android SDK, emulator,
Appium Server, or iOS toolchain available headlessly here. This entire
module is reviewed against Appium's `java-client` 9.x documented API and
mirrors the WebDriver patterns already verified working (structurally) in
Level 1/2's Selenium modules; treat it as reviewed, not executed.

## 7. Testing traps

!!! warning "Trap 1 — resource-ids that change per build"
    Some Android build tools obfuscate/randomize resource ids across
    release builds. A locator hard-coded against a debug-build id silently
    breaks against the release APK. Prefer `accessibilityId`/content-desc,
    which app developers set deliberately and should keep stable.

!!! warning "Trap 2 — emulator cold-start flakiness"
    The first test after an emulator boots is often slower and less
    reliable than the tenth — animations haven't settled, background
    services are still starting. A "flaky" first test that passes on retry
    is frequently this, not a real defect; warm the emulator with a no-op
    launch before the real suite runs.

!!! warning "Trap 3 — session not properly torn down"
    Skipping `driver.quit()` (e.g. a test throwing before `@AfterEach` runs
    due to a setup exception) leaves the app session open, and the next
    test's `create()` call either hangs waiting for a device slot or grabs a
    dirty app state. Always tear down in `@AfterEach`, and consider a
    `finally` around risky setup.

!!! warning "Trap 4 — testing on one device profile only"
    A layout assumption that holds on a Pixel 7 emulator (tall aspect ratio,
    specific density) can break on a smaller or older device where the same
    element is off-screen and needs a scroll first. Run the real suite
    across at least a small/medium/large device matrix before trusting it.

!!! warning "Trap 5 — permission dialogs blocking the flow"
    A location/camera/notification permission dialog appearing mid-test (OS
    popups aren't part of your app's element tree) causes every subsequent
    `findElement` to time out looking for something the dialog is covering.
    Grant permissions via capabilities
    (`autoGrantPermissions: true` on Android) or handle the dialog
    explicitly rather than hoping it never appears.

## Cheat sheet

| Task | Code |
|---|---|
| Android capabilities | `UiAutomator2Options().setDeviceName(...).setApp(...)` |
| iOS capabilities | `XCUITestOptions().setDeviceName(...).setApp(...)` |
| Connect to Appium Server | `new AndroidDriver(new URL("http://127.0.0.1:4723"), options)` |
| Resource id locator (Android) | `AppiumBy.id("pkg:id/name")` |
| Cross-platform locator | `AppiumBy.accessibilityId("name")` |
| UiAutomator selector | `AppiumBy.androidUIAutomator("new UiSelector()...")` |
| Native back | `driver.pressKey(new KeyEvent(AndroidKey.BACK))` |
| Swipe/gesture | W3C `Sequence` + `PointerInput` |
| Auto-grant permissions | `autoGrantPermissions: true` capability |
| Always tear down | `driver.quit()` in `@AfterEach` |

## Exercise

1. Install Appium Server and an Android emulator (or document that you're
   reviewing only), write `AppiumDriverFactory`, and confirm a session opens
   against a sample APK.
2. Write `MobileLoginTest` against a real or sample app, using
   `accessibilityId` locators throughout.
3. Add a swipe-to-scroll gesture to reach an element below the fold, and
   assert it becomes visible/clickable after the swipe.
4. Deliberately skip granting a permission your app requests on launch, run
   the suite, and capture the exact timeout/error `findElement` produces —
   this is Trap 5 made concrete.
5. Design (in comments, no device required if you don't have one) a small
   device matrix — at minimum a small-screen and large-screen Android
   profile — and list which of your existing locators you'd expect to break
   on the small-screen profile and why.
