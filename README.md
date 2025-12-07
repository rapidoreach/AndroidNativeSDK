# RapidoReach Android Integration Guide

RapidoReach Android SDK **1.1.0** targets Android SDK 35, raises the minimum SDK to 23, and ships a Kotlin-first facade while keeping the legacy Java APIs. This document explains how to consume the `.aar` that lives in this repository and highlights the new callbacks exposed in `/cbofferwallsdk`.

## Requirements
- Android Studio Hedgehog/Koala with AGP 8.x
- `compileSdkVersion` / `targetSdkVersion` **35**, `minSdkVersion` **23**
- An Activity context when initializing the SDK (needed to host the offerwall WebView)

### Get Your API Key
Sign up for a developer account and create a new Android app [here](http://www.rapidoreach.com/register). Copy the API Key from the dashboard—you will use it when bootstrapping the SDK.

## Install the SDK

### Download the SDK
You can depend on the Maven artifact `com.rapidoreach:cbofferwallsdk:1.1.0` or drop the `.aar` from this repository straight into your project:

1. Download the latest `Rapidoreach-<version>.aar` (for example [`Rapidoreach-1.1.0.aar`](Rapidoreach-1.1.0.aar) once published).
2. Copy the file into your app module's `libs/` directory.
3. Update your Gradle files as shown below (replace `1.1.0` with the actual file name you downloaded).

#### Module-level `build.gradle`

```gradle
apply plugin: 'com.android.application'

android {
    compileSdkVersion 35

    defaultConfig {
        applicationId "com.example.app"
        minSdkVersion 23
        targetSdkVersion 35
        versionCode 1
        versionName "1.0"
    }
}

dependencies {
    implementation fileTree(dir: "libs", include: ["*.jar"])
    implementation(name: "Rapidoreach-1.1.0", ext: "aar")
    implementation platform("org.jetbrains.kotlin:kotlin-bom:1.9.22")
    implementation "androidx.appcompat:appcompat:1.6.1"
    implementation "androidx.core:core-ktx:1.12.0"
    implementation "androidx.lifecycle:lifecycle-process:2.6.2"
    implementation "org.jetbrains.kotlinx:kotlinx-serialization-json:1.5.0"
    implementation "com.google.android.gms:play-services-ads-identifier:18.0.1"
    implementation "com.google.android.gms:play-services-appset:16.1.0"
}
```

#### Project-level `build.gradle`
Give Gradle access to the local `libs/` folder:

```gradle
allprojects {
    repositories {
        google()
        mavenCentral()
        flatDir {
            dirs 'libs'
        }
    }
}
```

### ProGuard / R8
Keep the SDK package if you shrink or obfuscate:

```proguard
-keep class com.rapidoreach.rapidoreachsdk.** { *; }
```

## Initialize the SDK

### Kotlin / Compose quick start (RapidoReachSdk facade)
The Kotlin facade mirrors the modern TapResearch surface and wires into the legacy Java implementation:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        RapidoReachSdk.initialize(
            apiToken = BuildConfig.RR_API_KEY,
            userIdentifier = "user-123",
            context = this, // must be an Activity
            sdkReadyCallback = { /* Offerwall can be shown now */ },
            rewardCallback = { rewards -> rewards.forEach { logReward(it) } },
            errorCallback = { error -> Log.e("RapidoReach", "Init failed: ${error.code} ${error.description}") },
            contentCallback = { event -> Log.d("RapidoReach", "Offerwall ${event.type}") },
            initOptions = RrInitOptions(
                placementId = "default",
                navigationBarText = "Earn Rewards",
                navigationBarColor = "#211548",
                navigationBarTextColor = "#FFFFFF"
            )
        )
    }

    fun showOfferwall() {
        if (RapidoReachSdk.canShowContentForPlacement("default")) {
            RapidoReachSdk.showContentForPlacement("default")
        }
    }
}
```

You can later update identity or attributes via `RapidoReachSdk.setUserIdentifier(...)` and `RapidoReachSdk.sendUserAttributes(...)`.

### Java Activity (compat layer)
Apps that already integrate the legacy Java API can continue to do so, but you now get explicit ready/error callbacks:

```java
import com.rapidoreach.rapidoreachsdk.RapidoReach;
import com.rapidoreach.rapidoreachsdk.RapidoReachConfig;
import com.rapidoreach.rapidoreachsdk.RapidoReachErrorListener;
import com.rapidoreach.rapidoreachsdk.RapidoReachRewardListener;
import com.rapidoreach.rapidoreachsdk.RapidoReachSurveyAvailableListener;
import com.rapidoreach.rapidoreachsdk.RapidoReachSurveyListener;
import com.rapidoreach.rapidoreachsdk.RapidoReachSdkReadyListener;

public class MainActivity extends AppCompatActivity
        implements RapidoReachRewardListener, RapidoReachSurveyListener, RapidoReachSurveyAvailableListener {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        RapidoReachConfig config = new RapidoReachConfig.Builder(BuildConfig.RR_API_KEY, BuildConfig.RR_USER_ID)
                .placementId("default")
                .navigationBarText("RapidoReach")
                .navigationBarColor("#211548")
                .navigationBarTextColor("#FFFFFF")
                .build();
        RapidoReach.initialize(config, this);

        RapidoReach.getInstance().setRapidoReachRewardListener(this);
        RapidoReach.getInstance().setRapidoReachSurveyListener(this);
        RapidoReach.getInstance().setRapidoReachSurveyAvailableListener(this);
        RapidoReach.getInstance().setRapidoReachSdkReadyListener(new RapidoReachSdkReadyListener() {
            @Override
            public void onSdkReady() {
                // SDK bootstrapped, safe to enable the "Earn" button.
            }

            @Override
            public void onSdkError(String errorCode, String message) {
                Log.e("RapidoReach", "SDK failed: " + errorCode + " " + message);
            }
        });
        RapidoReach.getInstance().setRapidoReachErrorListener(new RapidoReachErrorListener() {
            @Override
            public void onError(String errorCode, String message) {
                Log.e("RapidoReach", "Error: " + errorCode + " " + message);
            }
        });

        Button openOfferwall = findViewById(R.id.buttonOpenOfferwall);
        openOfferwall.setOnClickListener(view -> {
            if (RapidoReach.getInstance().isReady() && RapidoReach.getInstance().isSurveyAvailable()) {
                RapidoReach.getInstance().showRewardCenter();
            }
        });
    }

    @Override protected void onResume() {
        super.onResume();
        RapidoReach.getInstance().onResume(this);
    }

    @Override protected void onPause() {
        super.onPause();
        RapidoReach.getInstance().onPause();
    }

    @Override public void onReward(int amount) { Log.d("RapidoReach", "Reward: " + amount); }
    @Override public void onRewardCenterOpened() { Log.d("RapidoReach", "Offerwall opened"); }
    @Override public void onRewardCenterClosed() { Log.d("RapidoReach", "Offerwall closed"); }
    @Override public void rapidoReachSurveyAvailable(boolean available) { /* enable/disable UI */ }
}
```

## Reward Center
Call `RapidoReachSdk.showContentForPlacement("default")` (Kotlin) or `RapidoReach.getInstance().showRewardCenter()` (Java) once `sdkReadyCallback` / `RapidoReachSdkReadyListener` fires and `isSurveyAvailable()` returns `true`. The SDK converts payout amounts to your in-app currency according to the exchange rate configured in the dashboard.

## Reward Callbacks

### Server-side callbacks
Configure the callback URL in the RapidoReach dashboard so every reward is confirmed directly on your backend. We POST the `userIdentifier` you provided during initialization along with the reward payload.

### Client-side callbacks
You can also react instantly in-app by implementing the reward listener (only credit the user once if you also use the server callback):

```java
public class RewardActivity extends Activity implements RapidoReachRewardListener {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        RapidoReach.getInstance().setRapidoReachRewardListener(this);
    }

    @Override
    public void onReward(int rewardAmount) {
        Log.d("RapidoReach", "Reward earned: " + rewardAmount);
        // Update currency ledger / show toast
    }
}
```

## Reward Center Events
Listen for entry/exit to pause media, resume gameplay, etc.:

```java
public class MyActivity extends Activity implements RapidoReachSurveyListener {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        RapidoReach.getInstance().setRapidoReachSurveyListener(this);
    }

    @Override public void onRewardCenterOpened() { Log.d("RapidoReach", "Offerwall opened"); }
    @Override public void onRewardCenterClosed() { Log.d("RapidoReach", "Offerwall closed"); }
}
```

## Survey Availability Callback

```java
public class MyActivity extends Activity implements RapidoReachSurveyAvailableListener {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        RapidoReach.getInstance().setRapidoReachSurveyAvailableListener(this);
    }

    @Override
    public void rapidoReachSurveyAvailable(boolean surveyAvailable) {
        Log.d("RapidoReach", "Surveys available: " + surveyAvailable);
    }
}
```

## Placements, Surveys, and Quick Questions (Kotlin APIs)
The Kotlin facade exposes placement helpers that talk to the backend v2 endpoints:

```kotlin
val placementTag = "default"

RapidoReachSdk.getPlacementDetails(placementTag) { result ->
    result.onSuccess { details -> Log.d("RapidoReach", "Placement currency: ${details.currencyName}") }
}

RapidoReachSdk.listSurveys(placementTag) { result ->
    result.onSuccess { surveys ->
        val topSurvey = surveys.firstOrNull()
        if (topSurvey != null) {
            RapidoReachSdk.showSurvey(
                tag = placementTag,
                surveyId = topSurvey.surveyIdentifier,
                customParameters = mapOf("source" to "store"),
                contentListener = { event -> Log.d("RapidoReach", "Survey ${event.type}") },
                errorListener = { error -> Log.e("RapidoReach", "Survey failed ${error.code}") }
            )
        }
    }
}

RapidoReachSdk.fetchQuickQuestions(placementTag) { result ->
    result.onSuccess { payload ->
        val qqId = payload.data["question_id"] as? String ?: payload.data["id"] as? String
        if (qqId != null) {
            RapidoReachSdk.answerQuickQuestion(placementTag, qqId, answer = "A") { answerResult ->
                answerResult.onFailure { Log.e("RapidoReach", "QQ failed: ${it.message}") }
            }
        }
    }
}

RapidoReachSdk.sendUserAttributes(
    attributes = mapOf("level" to 12, "vip" to true),
    clearPrevious = false
) { error -> error?.let { Log.e("RapidoReach", "Attr update failed ${it.code}") } }
```

## Testing the SDK
New apps start in **Test** mode, which guarantees survey availability so you can validate initialization, placement readiness, and reward callbacks. Remember to switch your app to **Live** in the dashboard before releasing, or real users will not receive surveys.

## Customizing the SDK UI
Match the navigation bar to your brand via `RapidoReachConfig` or `RrInitOptions`:

```java
RapidoReach.getInstance().setNavigationBarText("Demo App");
RapidoReach.getInstance().setNavigationBarColor("#17b4b3");
RapidoReach.getInstance().setNavigationBarTextColor("#FFFFFF");
```

You can also pass these values during initialization (see the Kotlin sample above) to ensure they apply before the first render.
