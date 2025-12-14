# RapidoReach Android SDK

Rewarded survey offerwall SDK for Android. Integrate RapidoReach to show survey content (offerwall / placements / quick questions), receive reward events, and send user attributes for targeting.

## Get your API key

Create an app in the RapidoReach dashboard and copy your API key.

## Installation (bundled AAR)

This repo ships a prebuilt AAR (recommended for external developers/publishers because it avoids Maven auth).

1) Copy `Rapidoreach-1.0.2.aar` into your app module at `app/libs/`.

2) In your project `build.gradle` / `settings.gradle`, ensure you have `google()` and `mavenCentral()` and add `flatDir`:

```gradle
repositories {
  google()
  mavenCentral()
  flatDir { dirs("libs") }
}
```

3) In `app/build.gradle`, add the AAR + required dependencies:

```gradle
dependencies {
  implementation(name: "Rapidoreach-1.0.2", ext: "aar")

  implementation "androidx.appcompat:appcompat:1.6.1"
  implementation "androidx.core:core-ktx:1.12.0"
  implementation "androidx.lifecycle:lifecycle-process:2.6.2"

  implementation "com.google.android.gms:play-services-ads-identifier:18.0.1"
  implementation "com.google.android.gms:play-services-appset:16.1.0"
}
```

## Quick start (legacy offerwall API)

This is the simplest integration path and works from Java or Kotlin.

### 1) Initialize

Initialize once (app start or after login):

```java
import com.rapidoreach.rapidoreachsdk.RapidoReach;

RapidoReach.initWithApiKeyAndUserIdAndActivityContext("YOUR_API_TOKEN", "YOUR_USER_ID", this);
```

### 2) Listen for events (recommended)

```java
import com.rapidoreach.rapidoreachsdk.RapidoReachRewardListener;
import com.rapidoreach.rapidoreachsdk.RapidoReachSurveyAvailableListener;
import com.rapidoreach.rapidoreachsdk.RapidoReachSurveyListener;
import com.rapidoreach.rapidoreachsdk.RapidoReachErrorListener;

RapidoReach.getInstance().setRapidoReachRewardListener(new RapidoReachRewardListener() {
  @Override public void onReward(int amount) {
    // Client-side reward signal (use server-side callbacks in production whenever possible)
  }
});

RapidoReach.getInstance().setRapidoReachSurveyListener(new RapidoReachSurveyListener() {
  @Override public void onRewardCenterOpened() {}
  @Override public void onRewardCenterClosed() {}
});

RapidoReach.getInstance().setRapidoReachSurveyAvailableListener(new RapidoReachSurveyAvailableListener() {
  @Override public void rapidoReachSurveyAvailable(boolean available) {}
});

RapidoReach.getInstance().setRapidoReachErrorListener(new RapidoReachErrorListener() {
  @Override public void onError(String code, String message) {
    // Surface integration/runtime errors in debug builds
  }
});
```

### 3) Show the offerwall

```java
if (RapidoReach.getInstance().isSurveyAvailable()) {
  RapidoReach.getInstance().showRewardCenter();
}
```

### 4) Forward lifecycle (recommended)

```java
@Override
protected void onResume() {
  super.onResume();
  RapidoReach.getInstance().onResume(this);
}

@Override
protected void onPause() {
  RapidoReach.getInstance().onPause();
  super.onPause();
}
```

## Customizing SDK options (navigation bar)

```java
RapidoReach.getInstance().setNavigationBarText("Earn Rewards");
RapidoReach.getInstance().setNavigationBarColor("#211548");
RapidoReach.getInstance().setNavigationBarTextColor("#FFFFFF");
```

## User identity

If your user logs in/out, update the user identifier:

```java
RapidoReach.getInstance().updateUserIdentifier("NEW_USER_ID");
```

## Placement-based API (recommended)

If you want a guided UX with multiple placements, use `RapidoReachSdk` (Kotlin-first API).

Supported placement helpers:
- `getPlacementDetails(tag)`
- `canShowContentForPlacement(tag)`
- `listSurveys(tag)`
- `hasSurveys(tag)`
- `canShowSurvey(tag, surveyId)`
- `showSurvey(tag, surveyId, customParams?)`
- `showContentForPlacement(tag, customParams?)`

Kotlin example:

```kotlin
import com.rapidoreach.rapidoreachsdk.RapidoReachSdk
import com.rapidoreach.rapidoreachsdk.RrInitOptions

RapidoReachSdk.initialize(
  apiToken = "YOUR_API_TOKEN",
  userIdentifier = "YOUR_USER_ID",
  context = this,
  rewardCallback = { rewards -> /* list of RrReward */ },
  errorCallback = { err -> /* RrError */ },
  sdkReadyCallback = { /* ready */ },
  contentCallback = { event -> /* RrContentEvent */ },
  initOptions = RrInitOptions(
    navigationBarColor = "#211548",
    navigationBarTextColor = "#FFFFFF",
    navigationBarText = "Earn Rewards",
    placementId = null,
    resetProfiler = false,
    clearPreviousAttributes = false,
  )
)

RapidoReachSdk.listSurveys("default") { result ->
  result.onSuccess { surveys ->
    val firstId = surveys.firstOrNull()?.surveyIdentifier ?: return@onSuccess
    RapidoReachSdk.showSurvey(
      tag = "default",
      surveyId = firstId,
      customParams = mapOf("source" to "home_screen"),
      contentCallback = { /* RrContentEvent */ },
      errorCallback = { /* RrError */ },
    )
  }
}
```

Java helpers (for listing surveys / quick questions without Kotlin callbacks):

```java
import com.rapidoreach.rapidoreachsdk.RapidoReachSdk;

RapidoReachSdk.listSurveysJava(
  "default",
  surveys -> { /* List<RrSurvey> */ },
  error -> { /* String */ }
);
```

## Quick Questions

Kotlin:

```kotlin
RapidoReachSdk.fetchQuickQuestions("default") { payloadResult ->
  payloadResult.onSuccess { payload ->
    val data = payload.data
    val enabled = data["enabled"] as? Boolean ?: false
    val questions = data["quick_questions"] as? List<Map<String, Any?>> ?: emptyList()
    if (!enabled || questions.isEmpty()) return@onSuccess

    val firstId = questions.firstOrNull()?.get("id")?.toString().orEmpty()
    if (firstId.isEmpty()) return@onSuccess

    RapidoReachSdk.answerQuickQuestion("default", firstId, "yes") { _ -> }
  }
}
```

Java:

```java
RapidoReachSdk.fetchQuickQuestionsJava(
  "default",
  payload -> { /* RrQuickQuestionPayload */ },
  error -> { /* String */ }
);
```

## User attributes

Send user attributes for better targeting/eligibility:

```kotlin
RapidoReachSdk.sendUserAttributes(
  mapOf("country" to "US", "premium" to true),
  false,
) { err -> /* RrError? */ }
```

## Backends / environments

If you use a staging/regional backend, you can point the SDK to a different base URL:

```java
RapidoReach.getInstance().setApiEndpoint("https://your-backend.example");
```

You can also read the current base URL:

```java
String url = RapidoReach.getProxyBaseUrl();
```

## Debug logging

```java
import com.rapidoreach.rapidoreachsdk.RapidoReachLogger;

RapidoReachLogger.setDebug(true);
```

## Proguard / R8

```pro
-keep class com.rapidoreach.rapidoreachsdk.** { *; }
```

## Maven (GitHub Packages) distribution (optional)

This repository can publish the prebuilt `.aar` files to GitHub Packages (Maven) via CI.

- Publish workflow: `.github/workflows/publish-github-packages.yml`
- Trigger: push a tag like `v1.0.2` (it publishes `Rapidoreach-1.0.2.aar`)
- Maven coordinates: `com.rapidoreach:rapidoreach:<version>`

### Consuming from GitHub Packages

GitHub Packages (Maven) requires authentication to install packages.

```gradle
repositories {
  maven {
    url = uri("https://maven.pkg.github.com/<OWNER>/<REPO>")
    credentials {
      username = "<YOUR_GITHUB_USERNAME>"
      password = "<YOUR_GITHUB_PAT>" // needs read:packages
    }
  }
  google()
  mavenCentral()
}

dependencies {
  implementation "com.rapidoreach:rapidoreach:1.0.2"
}
```
