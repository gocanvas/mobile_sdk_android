# GoCanvas SDK for Android

#### Table of contents:

- [Introduction](#introduction)
- [Installation](#installation)
- [Migrating from 6.2 and earlier](#migrating-from-62-and-earlier)
- [Common](#common)
  - [Initialize the SDK](#initialize-the-sdk)
  - [Configuration values](#configuration-values)
  - [Register the form launcher](#register-the-form-launcher)
  - [Receive the form result](#receive-the-form-result)
  - [Prefilled entries](#prefilled-entries)
  - [Errors](#errors)
  - [Clear data](#clear-data)
  - [Branding](#branding)
  - [Multi-Language](#multi-language)
- [Userless](#userless)
  - [Userless API](#userless-api)
  - [Userless integration steps](#userless-integration-steps)
- [User-based](#user-based)
  - [User-based API](#user-based-api)
  - [Result format](#result-format)
  - [User-based integration steps](#user-based-integration-steps)
  - [Other calls](#other-calls)
  - [Working offline](#working-offline)
  - [User-based error codes](#user-based-error-codes)

## Introduction

Welcome to the GoCanvas Mobile SDK for Android! The SDK lets you bring the power of GoCanvas forms into your own mobile app, so your users can fill out forms, capture photos, signatures and other media, and complete submissions without leaving your application.

The SDK can be used in two modes:

- **[Userless](#userless)** - your app provides the form JSON (plus optional reference and prefilled data) and receives the submission back. No GoCanvas user is involved.

  Use Userless mode when you create and maintain the forms you display to your users, or when you fetch the form definitions yourself using the GoCanvas API. Your app is responsible for storing and processing the submissions how you see fit.

- **[User-based](#user-based)** - the SDK works on behalf of a GoCanvas user: it syncs the user's forms, reference data, dispatches and handoffs, and uploads their submissions.

  Use User-based mode when your customers manage their own forms on the GoCanvas website. The SDK downloads and manages everything the user needs to access their assigned forms, and pushes the resulting submissions up to GoCanvas for you.

Both modes share the setup described in [Common](#common).

## Installation

GoCanvas SDK requires at minimum Android API 26+. Compile SDK API 36.

1. Install via GitHub Packages. Add the following source to your `repositories` section

Create a GitHub access token with `read:packages` scope then replace <github_username> and <github_access_token> with
your credentials.

```gradle
repositories {
  //...
  maven {
     name = "GitHubPackages"
     url = uri("https://maven.pkg.github.com/gocanvas/android_sdk_package")
     credentials {
         username = "<github_username>"
         password = "<github_access_token>"
     }
  }
```

2. Add the library to the `dependencies` section:

```gradle
dependencies {
  //...
  implementation("com.gocanvas.sdk:android:6.3.0")
  //...
}
```

3. Make sure you have these repositories declared:

```gradle
repositories {
  google()
  mavenCentral()
  maven { url = uri("https://jitpack.io") }
  //...
}
```

4. Make sure you have the AndroidX & Jetifier enabled inside the `gradle.properties` file:

```gradle.properties
android.useAndroidX=true
android.enableJetifier=true
```

## Migrating from 6.2 and earlier

Existing integrations keep working in [Userless](#userless) mode: `init`, `showForm`, the form launcher and the intent extras are unchanged. The breaking change is the format of the JSON returned by `getResponse()`.

### The response is now wrapped in `error` and `data`

Up to 6.2, `getResponse()` returned the submission itself. Starting with 6.3, it returns a JSON with an `error` and a `data` property, and the submission that was previously returned is now in the `data` property.

Before (6.2 and earlier):

```json
{
  "guid": "f4c7a1d9-8b1e-4d7f-9d5c-2d9e6b4e5a8f",
  "last_sheet_id": "12345",
  "status": "closed",
  "form": { "id": 4639578, "version": 1 },
  "responses": [ ... ]
}
```

After (6.3):

```json
{
  "error": null,
  "data": {
    "guid": "f4c7a1d9-8b1e-4d7f-9d5c-2d9e6b4e5a8f",
    "last_sheet_id": "12345",
    "status": "closed",
    "form": { "id": 4639578, "version": 1 },
    "responses": [ ... ]
  }
}
```

The content of `data` has the same properties as the 6.2 response, see [Read the response](#step-5-read-the-response).

### What to change

1. Read the submission from the `data` property instead of the root of the JSON:

```kotlin
val response = CanvasSdk.getResponse()
if (response.isNotEmpty()) {
    // 6.2: val submission = JSONObject(response)
    val submission = JSONObject(response).getJSONObject("data")
    val status = submission.getString("status")
    val responses = submission.getJSONArray("responses")
}
```

2. When you store an `open` response to [resume it later](#prefill-with-partially-response), store and pass back only the `data` object, not the whole JSON returned by `getResponse()`. If the whole JSON is passed as `prefilledEntries`, it is not recognized: the user is warned and the form opens without the saved values.

```kotlin
val submission = JSONObject(CanvasSdk.getResponse()).getJSONObject("data")
if (submission.getString("status") == "open") {
    savedResponse = submission.toString() // later passed as prefilledEntries
}
```

3. Keep handling the form flow errors through the intent extras. In Userless mode, `error` is always `null`: errors are still reported with `Activity.RESULT_CANCELED` and the `ERROR_NUMBER_KEY` / `ERROR_MESSAGE_KEY` extras, see [Receive the form result](#receive-the-form-result). `getResponse()` returns an empty string when there is no result, so check for it before parsing.

## Common

This section describes what both modes share. All the methods are called on the `CanvasSdk` instance (`CanvasSdk.INSTANCE` from Java).

### Initialize the SDK

```kotlin
fun init(licenseKey: String)
```

| Parameter | Description |
|---|---|
| `licenseKey` | the license key provided by GoCanvas for your application. Not needed in [User-based](#user-based) mode, where a blank string (`""`) can be passed |

Returns nothing. Call it once, before any other SDK call (for example in `Application.onCreate`):

```kotlin
CanvasSdk.init(licenseKey = "<your_license_key>")
```

An invalid license key is reported when a [Userless](#userless) form is shown, as an `INVALID_LICENSE_KEY` or `INVALID_PACKAGE_IDENTIFIER` [error](#errors).

> **Note:** The license key is only used in Userless mode. In [User-based](#user-based) mode it is not needed, so you can pass a blank string:
>
> ```kotlin
> CanvasSdk.init(licenseKey = "")
> ```

### Configuration values

```kotlin
fun addConfigValue(key: String, value: Any)
```

| Parameter | Description |
|---|---|
| `key` | one of the configuration keys below |
| `value` | the value for that key, of the type listed below |

Returns nothing. Call it after `init` and before showing a form (for `MOBILE_DOMAIN`, before the first `sync`).

| Key | Value type | Default | Used by |
|---|---|---|---|
| `MOBILE_INTERFACE_THEME` | `CanvasSdkInterfaceTheme` (`SYSTEM`, `LIGHT`, `DARK`) | `SYSTEM` | both modes, see [Light & Dark mode](#light--dark-mode) |
| `MOBILE_LANGUAGE` | `java.util.Locale` | the system language, with `English (US)` fallback | both modes, see [Multi-Language](#multi-language) |
| `MOBILE_DOMAIN` | `String`, the GoCanvas server host (e.g. `"www.gocanvas.com"`) | the GoCanvas production server | User-based, see [Set the server](#step-2-set-the-server-optional) |

### Register the form launcher

In both modes the form is displayed in a separate activity, and the result is delivered through an `ActivityResultLauncher` that you pass to `showForm` / `showSubmission`. Register it before showing a form (for example in `onCreate`):

```java
ActivityResultLauncher<Intent> formLauncher = registerForActivityResult(new ActivityResultContracts.StartActivityForResult(),
        new ActivityResultCallback<ActivityResult>() {
            @Override
            public void onActivityResult(ActivityResult result) {
                // Handle the result, see "Receive the form result"
            }
        });
```

### Receive the form result

When the form flow ends, the `ActivityResultCallback` receives:

Result code:

- `Activity.RESULT_OK` - when the form flow has been completed successfully (the user tapped "Submit")
- `Activity.RESULT_CANCELED` - when the form flow has been canceled by the user, saved with "Save and Close", or ended with an error

Intent extras:

- `CanvasSdkKey.RESPONSE_KEY` - `Boolean`, `true` when a result is available through `getResponse()`. It is set when the form is submitted in both modes; after "Save and Close" it is set only in Userless mode, so in User-based mode call `getResponse()` directly
- `CanvasSdkKey.ERROR_NUMBER_KEY` - the error code, as a `String`, when the form could not be displayed (see [Errors](#errors))
- `CanvasSdkKey.ERROR_MESSAGE_KEY` - the error message, as a `String`

Then read the result with:

```kotlin
fun getResponse(): String
```

It takes no parameters and returns a JSON string with an `error` and a `data` property, or an empty string when the flow ended without a result (e.g. an error, or the user canceled). The content of `data` depends on the mode:

- Userless: the full submission, including the entries' values, see [Read the response](#step-5-read-the-response)
- User-based: only the submission header, see [Read the result](#step-9-read-the-result)

```java
ActivityResultLauncher<Intent> formLauncher = registerForActivityResult(new ActivityResultContracts.StartActivityForResult(),
        new ActivityResultCallback<ActivityResult>() {
            @Override
            public void onActivityResult(ActivityResult result) {
                Intent data = result.getData();
                if (data != null && data.hasExtra(ERROR_NUMBER_KEY)) {
                    Log.i("ActivityResult", "error code: " + data.getExtras().get(ERROR_NUMBER_KEY));
                    Log.i("ActivityResult", "error message: " + data.getExtras().get(ERROR_MESSAGE_KEY));
                    return;
                }
                String response = CanvasSdk.INSTANCE.getResponse();
                if (!response.isEmpty()) {
                    Log.i("ActivityResult", "result code: " + result.getResultCode() + ", response: " + response);
                }
            }
        });
```

### Prefilled entries
<a id="prefilled-entries-scope"></a>

Both modes accept an optional `prefilledEntries` JSON string that prefills the form's entries. The following formats are supported:

1. Prefill all entries based on labels:
<a id="prefill-by-label"></a>

```json
{
  "responses": [
    {
      "value": "Example Value",
      "label": "Example Label"
    }
  ]
}
```

2. Prefill only specific entries based on form's entry id (Userless only):

```json
{
  "responses": [
    {
      "entry_id": 123456,
      "value": "Example Value",
      "label": "Example Label", // Label is Optional
      // Required for List loop
      "id": 391647,
      "multi_key": "Example Multi Key"
    }
  ]
}
```

3. Prefill with Partially Response (Userless only): resumes a response previously returned by `getResponse()` with status `open`
<a id="prefill-with-partially-response"></a>

```json
{
  "guid": "12345-abcd",
  "last_sheet_id": "12345",
  "status": "open",
  "form": {
    "id": 6789,
    "version": 1
  },
  "responses": [
    {
      "entry_id": 12345,
      "guid": "55EB..",
      "displayed": true,
      "multi_key": "Multi Key Example Value",
      "key_id": 789789,
      "multi_key_id": 345345,
      "value": "Example Value",
      "label": "Example Label"
    }
  ]
}
```

### Errors

Errors are reported:

- by the form flows, with `Activity.RESULT_CANCELED` and the `ERROR_NUMBER_KEY` / `ERROR_MESSAGE_KEY` intent extras, see [Receive the form result](#receive-the-form-result)
- by the User-based calls, in the `error` property of the returned JSON, see [Result format](#result-format)

The following errors are shared by both modes:

- `INVALID_JSON` - when the form definition cannot be parsed
- `INVALID_FORM_DEFINITION` - when the form has no sections, sheets or entries
- `INVALID_SAVED_RESPONSE` - when the saved response cannot be restored
- `REFERENCE_DATA_NOT_SET` - when the form uses reference data that was not provided
- `INVALID_LICENSE_KEY` - when the license key provided is not valid
- `INVALID_PACKAGE_IDENTIFIER` - when the license key provided is not valid for the app in use

Each error has associated an error code and a message as follows:

```kotlin
enum class CanvasSdkErrorType(val statusCode: Int, val errorDescription: String) {
    INVALID_JSON(90000, "Unable to parse form definition."),
    INVALID_FORM_DEFINITION(90001, "Form definition is invalid."),
    INVALID_SAVED_RESPONSE(90002, "Unable to resume response."),
    REFERENCE_DATA_NOT_SET(90003, "Unable to show form. Reference data was not set."),
    INVALID_LICENSE_KEY(90004, "License key is invalid, please contact your account manager for assistance."),
    INVALID_PACKAGE_IDENTIFIER(90005, "Invalid package identifier, please contact your account manager for assistance."),
}
```

The User-based mode adds its own errors, see [User-based error codes](#user-based-error-codes).

### Clear data

```kotlin
fun clearData()
```

Takes no parameters and returns nothing. Deletes all the data the SDK has stored on the device. It can be used in both modes and must not be called on the main thread.

### Branding

#### Colors

The color system that can be used to create a color scheme that reflects your brand or style.

| Color Attribute                       | Default (Light)                                                  | Default (Dark)                                                   | Affected Ui Components                                                                                                                                     |
|---------------------------------------|------------------------------------------------------------------|------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| gc_sdk_color_primary                  | ![#0e51c4](https://placehold.co/15x15/0e51c4/0e51c4.png) #0e51c4 | ![#039de7](https://placehold.co/15x15/039de7/039de7.png) #039de7 | brand color: buttons, links, icons, selected controls (checkbox, radio button, chip, segmented, rating), input fields with captured values                 |
| gc_sdk_color_secondary                | ![#ecf1fa](https://placehold.co/15x15/ecf1fa/ecf1fa.png) #ecf1fa | ![#000d12](https://placehold.co/15x15/000d12/000d12.png) #000d12 | table of contents button background                                                                                                                        |
| gc_sdk_color_background_light         | ![#f5f6f6](https://placehold.co/15x15/f5f6f6/f5f6f6.png) #f5f6f6 | ![#151b1e](https://placehold.co/15x15/151b1e/151b1e.png) #151b1e | screen background, action-icon background on top bar                                                                                                       |
| gc_sdk_color_background_dark          | ![#0b1215](https://placehold.co/15x15/0b1215/0b1215.png) #0b1215 | ![#ffffff](https://placehold.co/15x15/ffffff/ffffff.png) #ffffff | inverse (dark) background surface                                                                                                                          |
| gc_sdk_color_secondary_container      | ![#ffffff](https://placehold.co/15x15/ffffff/ffffff.png) #ffffff | ![#0b1215](https://placehold.co/15x15/0b1215/0b1215.png) #0b1215 | background for: input fields, top bar, checkbox, radio button, chip, segmented, rating, table/list header & list item, grid rows (1st, 3rd, …)             |
| gc_sdk_color_secondary_container_low  | ![#f5f8fd](https://placehold.co/15x15/f5f8fd/f5f8fd.png) #f5f8fd | ![#000609](https://placehold.co/15x15/000609/000609.png) #000609 | section background in bottom sheets (sheet menu, photo source, review items), pressed/selected list item                                                   |
| gc_sdk_color_secondary_container_high | ![#fafafa](https://placehold.co/15x15/fafafa/fafafa.png) #fafafa | ![#10171a](https://placehold.co/15x15/10171a/10171a.png) #10171a | grid rows (2nd, 4th, …)                                                                                                                                    |
| gc_sdk_color_background_progress      | ![#d8d9da](https://placehold.co/15x15/d8d9da/d8d9da.png) #d8d9da | ![#32383a](https://placehold.co/15x15/32383a/32383a.png) #32383a | progress bar inactive color                                                                                                                                |
| gc_sdk_color_progress                 | ![#0e51c4](https://placehold.co/15x15/0e51c4/0e51c4.png) #0e51c4 | ![#039de7](https://placehold.co/15x15/039de7/039de7.png) #039de7 | progress bar active color                                                                                                                                  |
| gc_sdk_color_error                    | ![#c40e51](https://placehold.co/15x15/c40e51/c40e51.png) #c40e51 | ![#e63a79](https://placehold.co/15x15/e63a79/e63a79.png) #e63a79 | error icon, text & border                                                                                                                                  |

By overriding these color attributes, you can easily change the styles of all the mentioned components used by the sdk.

#### Light & Dark mode

**Applying your own colors:**

1. Enable the theme mode with the SDK API (default is set to `SYSTEM`)

Call the following method after `CanvasSdk.init()`

```kotlin
// You can also set CanvasSdkInterfaceTheme to LIGHT or DARK 
CanvasSdk.addConfigValue("MOBILE_INTERFACE_THEME", CanvasSdkInterfaceTheme.LIGHT)
```

2. Add the following colors defined in your `default colors.xml` file:

```xml

<resources>
    <color name="gc_sdk_color_primary">...</color>
    <color name="gc_sdk_color_secondary">...</color>
    <color name="gc_sdk_color_background_light">...</color>
    <color name="gc_sdk_color_background_dark">...</color>
    <color name="gc_sdk_color_secondary_container">...</color>
    <color name="gc_sdk_color_secondary_container_low">...</color>
    <color name="gc_sdk_color_secondary_container_high">...</color>
    <color name="gc_sdk_color_background_progress">...</color>
    <color name="gc_sdk_color_progress">...</color>
    <color name="gc_sdk_color_error">...</color>
</resources>
```

3. Add the following colors for Dark Mode in your `values-night colors.xml` file:

```xml

<resources>
    <color name="gc_sdk_color_primary">...</color>
    <color name="gc_sdk_color_secondary">...</color>
    <color name="gc_sdk_color_background_light">...</color>
    <color name="gc_sdk_color_background_dark">...</color>
    <color name="gc_sdk_color_secondary_container">...</color>
    <color name="gc_sdk_color_secondary_container_low">...</color>
    <color name="gc_sdk_color_secondary_container_high">...</color>
    <color name="gc_sdk_color_background_progress">...</color>
    <color name="gc_sdk_color_progress">...</color>
    <color name="gc_sdk_color_error">...</color>
</resources>
```

### Multi-Language

Supported languages:

| Language                | Resources       |
|-------------------------|-----------------|
| English (US)            | values          |
| English (UK)            | values-en-rGB   |
| Chinese (Simplified)    | values-zh-rCN   |
| Chinese (Traditional)   | values-zh-rTW   |
| Czech                   | values-cs       |
| Danish                  | values-da       |
| Dutch                   | values-nl       |
| Finnish                 | values-fi       |
| French                  | values-fr       |
| French (Canada)         | values-fr-rCA   |
| German                  | values-de       |
| Hindi                   | values-hi       |
| Hungarian               | values-hu       |
| Lithuanian              | values-lt       |
| Malay                   | values-ms       |
| Norwegian (Bokmål)      | values-nb       |
| Polish                  | values-pl       |
| Portuguese              | values-pt       |
| Slovak                  | values-sk       |
| Spanish                 | values-es       |
| Spanish (Latin America) | values-b+es+419 |
| Swedish                 | values-sv       |
| Turkish                 | values-tr       |

Default behavior: `System` with `English (US)` fallback

Configuration usage:
```kotlin
val locale: Locale = provideLocale() // provide your own desired locale 
CanvasSdk.addConfigValue("MOBILE_LANGUAGE", locale)
```

#### Strings 

You can update the `Submit` button text by overriding the resource into your `strings.xml` file

```xml
    <string name="submit_button_title">...</string>
```

## Userless

In Userless mode your application provides the form definition (plus optional reference data and prefilled entries) and receives the completed submission back, including the entries' values. The SDK does not keep the submission once it has been returned: storing and uploading it is up to your application.

### Userless API

```kotlin
interface CanvasSdkApi {
    //... init, addConfigValue and clearData, see "Common"

    /**
     * Displays the form into another activity.
     * @param formJson the form in JSON string format
     * @param activity the parent activity
     * @param formLauncher the result launcher that receives the response
     */
    fun showForm(formJson: String, activity: Activity, formLauncher: ActivityResultLauncher<Intent>)

    /**
     * Displays the form into another activity.
     * @param formConfig the form configuration to set complex form info
     * @param activity the parent activity
     * @param formLauncher the result launcher that receives the response
     */
    fun showForm(formConfig: CanvasSdkFormConfig, activity: Activity, formLauncher: ActivityResultLauncher<Intent>)

    /**
     * Returns the result of the last `showForm` flow: a JSON with an `error` and a `data` property,
     * where `data` contains the full submission, including the entries' values.
     * See [Read the response](#step-5-read-the-response).
     * */
    fun getResponse(): String
}
```

### Userless integration steps

1. [Initialize the SDK](#initialize-the-sdk) (see Common)
2. [Register the form launcher](#register-the-form-launcher) (see Common)
3. [Get the form definition](#step-3-get-the-form-definition)
4. [Show the form](#step-4-show-the-form)
5. [Read the response](#step-5-read-the-response)

```kotlin
class MainActivity : AppCompatActivity() {

    private val formLauncher = registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
        val response = CanvasSdk.getResponse() // { "error": null, "data": { "guid": ..., "status": ..., "responses": [...] } }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        CanvasSdk.init(licenseKey = "<your_license_key>")
    }

    private fun openForm(formJson: String, referenceDataJson: String?) {
        CanvasSdk.showForm(CanvasSdkFormConfig(formJson, referenceDataJson), this, formLauncher)
    }
}
```

#### Step 3: Get the form definition

The SDK accepts the `format=sync` form definition obtained through the GoCanvas API:

Example: `https://www.gocanvas.com/api/v3/forms/123456?format=sync`

When the form uses reference data, fetch it through the GoCanvas API as well and pass it together with the form (see the next step); otherwise the form fails with `REFERENCE_DATA_NOT_SET`.

#### Step 4: Show the form

```kotlin
fun showForm(formJson: String, activity: Activity, formLauncher: ActivityResultLauncher<Intent>)
fun showForm(formConfig: CanvasSdkFormConfig, activity: Activity, formLauncher: ActivityResultLauncher<Intent>)
```

| Parameter | Description |
|---|---|
| `formJson` | the form definition from [Step 3](#step-3-get-the-form-definition), as a JSON string |
| `formConfig` | the form definition plus its reference data and prefilled entries, see below |
| `activity` | the activity that shows the form |
| `formLauncher` | the launcher registered as described in [Register the form launcher](#register-the-form-launcher), which receives the result |

Returns nothing: the result is delivered through `formLauncher` when the form flow ends.

```java
CanvasSdk.INSTANCE.showForm(formJson, activity, formLauncher);
```

Use `CanvasSdkFormConfig` to pass the reference data or prefilled entries:

```kotlin
/**
 * Canvas SDK Form config
 * @param formJson the form in JSON string format
 * @param referenceDataJson the reference data in JSON string format.
 * Supports both [JSONObject] & [JSONArray] types in case of a single or multiple reference data
 * associated with the given [formJson].
 * @param prefilledEntries the prefilled entries in JSON string format, see [Prefilled entries](#prefilled-entries).
 */
data class CanvasSdkFormConfig(
    val formJson: String,
    val referenceDataJson: String? = null,
    val prefilledEntries: String? = null
)
```

If the form cannot be displayed, the result code is `Activity.RESULT_CANCELED` and the intent extras contain one of the [shared errors](#errors).

##### App crash or unexpected form closing

If the app crashes or closes unexpectedly, upon launching the SDK for a form the SDK will check for a "leftover" response from a previous unexpected close. If a "leftover" response exists for the form the user will be prompted if they wish to continue the "leftover" response or discard it and continue.

#### Step 5: Read the response

When the result is received by `formLauncher`, call `CanvasSdk.getResponse()`. It returns a JSON with an `error` and a `data` property, where `data` contains the submission:

```json
{
  "error": null,
  "data": {
    "guid": "f4c7a1d9-8b1e-4d7f-9d5c-2d9e6b4e5a8f",
    "last_sheet_id": "12345",
    "status": "closed",
    "form": {
      "id": 4639578,
      "version": 1
    },
    "responses": [
      { "entry_id": 543728076, "guid": "a1b2c3d4-...", "displayed": true, "value": "api.user@example.com", "label": "Email" },
      { "entry_id": 543728079, "guid": "e5f6a7b8-...", "displayed": true, "multi_key": "Priority", "value": "High", "label": "Level" }
    ]
  }
}
```

| Property | Description |
|---|---|
| `guid` | the submission's unique id |
| `last_sheet_id` | the last screen the user visited |
| `status` | `closed` when the user tapped "Submit", or `open` when the user elected to "Save and Close" |
| `form` | the `id` and `version` of the form |
| `responses` | one item per entry, with its `entry_id`, entry `guid`, `label`, `value` and whether it was `displayed`. Entries inside list loops also have a `multi_key` |

An `open` response can be resumed later by passing it back as [prefilled entries](#prefill-with-partially-response).

##### Reading Media files

Images, Drawings, Signatures, Videos, Attachments

As in 6.2, the `value` of a media entry contains the local path(s) of its file(s). All the paths point to files in the SDK's directory inside your app's private storage (on Android 10+, compressed videos are moved there too, instead of being left in the app's external `Movies` directory).

1. Entry response with single media file

- the library is returning the local path to file as `String` on the JSON `value` property.

```
{
  ..
  "value": "path_to_image_1",
  "label": "Camera Photo"
} 
```

2. Entry response with multiple media files

- the library is returning a single `String` that concatenates all the paths to file. The paths are delimitated by the
  “\r\n” separator. It can be accessed on the JSON `value` property

```
{
   ..
   "value": "path_to_image_1\r\n\path_to_image_2\r\n\path_to_image_3"
   "label": "Multi Photo"
}
```

## User-based

Starting with 6.3.0, the SDK can work on behalf of a GoCanvas user. Using the user's OAuth token, the SDK downloads the forms, reference data, dispatches and handoffs assigned to that user, stores them on the device, and uploads the submissions the user completes. Your application no longer needs to fetch form definitions or reference data itself, and the entries' values are never returned to it.

### User-based API

```kotlin
interface CanvasSdkApi {
    //... init, addConfigValue and clearData, see "Common"

    /**
     * Registers a provider the SDK calls to obtain a new token when the stored one has expired.
     * @param tokenProvider returns the token JSON of the synced user, in the same format passed to `sync`.
     * It is called off the main thread and must not call the other SDK methods.
     * See [Register a token provider](#step-4-register-a-token-provider-optional).
     */
    fun setTokenProvider(tokenProvider: () -> String)

    /**
     * Uploads the completed submissions, then downloads the user's forms, reference data,
     * dispatches and handoffs. The first successful sync stores the user on the device.
     * @param username the user's GoCanvas username (email)
     * @param token the user's OAuth token JSON, see [Obtain the user's token](#step-3-obtain-the-users-token)
     * @param uploadOnly when `true`, only uploads the completed submissions
     * @return a JSON with an `error` and a `data` property. On success `data` contains the counts of
     * `forms_added`, `forms_removed`, `forms_retired`, `dispatches_added` and `dispatches_removed`
     * (empty when `uploadOnly` is `true`). See [Sync](#step-5-sync).
     *
     * NOTE: this must not be run on the main thread
     */
    fun sync(username: String, token: String, uploadOnly: Boolean = false): String

    /**
     * Returns the forms stored on the device, each with its submissions.
     * @param username the synced user's username
     * @return a JSON with an `error` and a `data` property. On success `data` contains the `forms`
     * array, and each form contains its `submissions` array.
     * See [Get the forms and submissions](#step-6-get-the-forms-and-submissions).
     *
     * NOTE: this must not be run on the main thread
     */
    fun getFormsAndSubmissions(username: String): String

    /**
     * Starts a new submission for a form stored on the device by `sync`.
     * @param username the synced user's username
     * @param formId the `id` of the form, as returned by `getFormsAndSubmissions`
     * @param activity the parent activity
     * @param formLauncher the result launcher that receives the response
     * @param prefilledEntries the prefilled entries in JSON string format, see
     * [Prefill all entries based on labels](#prefill-by-label). The `label` is mandatory.
     */
    fun showForm(username: String, formId: Int, activity: Activity, formLauncher: ActivityResultLauncher<Intent>, prefilledEntries: String? = null)

    /**
     * Opens a saved submission, dispatch or handoff stored on the device by `sync`.
     * @param username the synced user's username
     * @param guid the `guid` of the submission, as returned by `getFormsAndSubmissions`
     * @param activity the parent activity
     * @param formLauncher the result launcher that receives the response
     * @param prefilledEntries the prefilled entries in JSON string format, see
     * [Prefill all entries based on labels](#prefill-by-label). The `label` is mandatory.
     */
    fun showSubmission(username: String, guid: String, activity: Activity, formLauncher: ActivityResultLauncher<Intent>, prefilledEntries: String? = null)

    /**
     * Returns the result of the last `showForm` / `showSubmission` flow: a JSON with an `error` and a
     * `data` property, where `data` contains only the submission header. See [Read the result](#step-9-read-the-result).
     * */
    fun getResponse(): String

    /**
     * Deletes a saved submission from the device.
     * @param username the synced user's username
     * @param guid the `guid` of the submission, as returned by `getFormsAndSubmissions`
     * @return a JSON with an `error` and a `data` property. On success `data` contains the deleted
     * submission. Dispatches, handoffs and completed submissions cannot be deleted and return a
     * `SUBMISSION_CANNOT_BE_DELETED` error.
     *
     * NOTE: this must not be run on the main thread
     */
    fun deleteSubmission(username: String, guid: String): String

    /**
     * Returns `true` when any submission on the device has `pendingUpload: true`, i.e. it will be
     * uploaded (or finish uploading) on the next `sync`.
     *
     * NOTE: this must not be run on the main thread
     */
    fun hasPendingSubmissions(): Boolean
}
```

`sync`, `getFormsAndSubmissions`, `deleteSubmission` and `hasPendingSubmissions` are synchronous and access the network or the local database, so they must be called off the main thread.

### Result format

Except for `hasPendingSubmissions`, all user-based calls return (or deliver through `getResponse()`) a `String` containing a JSON with an `error` and a `data` property.

On success, `error` is `null` and `data` contains the result of the call:

```json
{ "error": null, "data": { } }
```

On failure, `error` contains the error `code`, the user-facing `message` (see [User-based error codes](#user-based-error-codes)) and, when available, a `debug_message`:

```json
{ "error": { "code": 90010, "message": "The provided token is not valid for the specified user.", "debug_message": "Token is valid for 'john.doe@example.com' but sync was requested for 'jane.doe@example.com'" }, "data": {} }
```

The `debug_message` holds developer-facing details about what failed. It is not translated and its content may change, so use it for logging only: don't show it to the user or parse it.

### User-based integration steps

1. [Initialize the SDK](#initialize-the-sdk) (see Common). The license key is not needed, so a blank string can be passed
2. [Set the server](#step-2-set-the-server-optional) (optional)
3. [Obtain the user's token](#step-3-obtain-the-users-token)
4. [Register a token provider](#step-4-register-a-token-provider-optional) (optional)
5. [Sync](#step-5-sync)
6. [Get the forms and submissions](#step-6-get-the-forms-and-submissions)
7. [Register the form launcher](#register-the-form-launcher) (see Common)
8. [Show a form or a submission](#step-8-show-a-form-or-a-submission)
9. [Read the result](#step-9-read-the-result)
10. [Sync again](#step-10-sync-again) to upload the completed submissions and receive the latest changes

```kotlin
class MainActivity : AppCompatActivity() {

    private val username = "user@company.com"

    private val formLauncher = registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
        val submission = CanvasSdk.getResponse() // { "error": null, "data": { "guid": ..., "status": ..., "pendingUpload": ... } }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        CanvasSdk.init(licenseKey = "") // the license key is not needed in User-based mode
        CanvasSdk.setTokenProvider { myAuthService.currentTokenJson() }

        lifecycleScope.launch {
            // sync and getFormsAndSubmissions must not run on the main thread
            val syncResult = withContext(Dispatchers.IO) { CanvasSdk.sync(username, myAuthService.currentTokenJson()) }
            val formsAndSubmissions = withContext(Dispatchers.IO) { CanvasSdk.getFormsAndSubmissions(username) }
            // list the forms and submissions, then open one:
            CanvasSdk.showForm(username, formId = 123456, activity = this@MainActivity, formLauncher = formLauncher)
        }
    }
}
```

#### Step 2: Set the server (optional)

The SDK connects to the GoCanvas production server by default. To set the GoCanvas server explicitly, add the `MOBILE_DOMAIN` config value after `init` and before the first `sync`:

```kotlin
CanvasSdk.addConfigValue("MOBILE_DOMAIN", "www.gocanvas.com")
```

The user's OAuth token must be issued by the same server.

#### Step 3: Obtain the user's token

Obtain an OAuth token for the user from GoCanvas. The token is the JSON returned by the GoCanvas OAuth endpoint and must contain an `access_token` issued for the user's `username`:

```json
{ "access_token": "<access_token>", "expires_in": 7200, "created_at": 1727172902 }
```

Pass the token to `sync` as you received it. The SDK uses `created_at` and `expires_in` to detect that the token has expired; a token without both properties is never treated as expired, so the [token provider](#step-4-register-a-token-provider-optional) is not called for it.

#### Step 4: Register a token provider (optional)

```kotlin
fun setTokenProvider(tokenProvider: () -> String)
```

| Parameter | Description |
|---|---|
| `tokenProvider` | returns a valid token JSON for the synced user, in the same format passed to `sync` |

Returns nothing. By default, when the stored token expires, the SDK calls return a `NETWORK_ERROR`. To let the SDK refresh the token by itself, register a token provider:

```kotlin
CanvasSdk.setTokenProvider {
    // return a valid token JSON for the synced user
    myAuthService.currentTokenJson()
}
```

The provider is called when the stored token has expired, and the returned token is stored for the next calls. Please note:

- the provider is called off the main thread and blocks the calling thread until it returns, so it must not wait on the main thread
- it must not call the other SDK methods
- the most recent registration wins

#### Step 5: Sync

```kotlin
fun sync(username: String, token: String, uploadOnly: Boolean = false): String
```

| Parameter | Description |
|---|---|
| `username` | the user's GoCanvas username (email) |
| `token` | the user's token JSON from [Step 3](#step-3-obtain-the-users-token) |
| `uploadOnly` | when `true`, only uploads the completed submissions, which makes the sync faster when you don't need to refresh the forms and submissions list |

`sync` uploads the completed submissions, then downloads the user's forms, reference data, dispatches and handoffs. It must not be called on the main thread.

- The first successful `sync` stores the user on the device. All the other calls must use the same `username`.
- When the first `sync` of a user fails, no user is stored: the SDK stays in Userless mode and the user-based calls return `NO_SYNCED_USER` until a `sync` succeeds.
- Only one user can be synced at a time. To switch to a different user, [`clearData`](#clear-data) must be called first.
- The first sync of a user downloads every assigned form, which can take a while for a large account. Later syncs only download what has changed and are much faster. The call returns only when the whole sync has finished, so we recommend showing a progress UI while it runs.
- Only one sync can run at a time; calling `sync` while another sync is running returns a `SYNC_IN_PROGRESS` error.

Returns a JSON (see [Result format](#result-format)) where, on success, `data` contains a summary of the changes:

```json
{
  "error": null,
  "data": {
    "forms_added": 2,
    "forms_removed": 0,
    "forms_retired": 1,
    "dispatches_added": 1,
    "dispatches_removed": 0
  }
}
```

When a form has been retired or is no longer assigned to the user but still has submissions on the device, it is kept as `retired`: new submissions can no longer be created, but existing submissions can be opened and completed.

With `uploadOnly = true`, `data` is empty. When the sync fails, `error` is set and `data` contains the counts collected before the failure.

When a different user is already synced on the device, `sync` returns a `DIFFERENT_USER` error and `data` contains that user, so you can ask whether to clear their data first:

```json
{
  "error": { "code": 90011, "message": "A different user is already synced. Clear data before syncing a new user." },
  "data": { "name": "Jane Doe", "email": "jane@company.com" }
}
```

Possible errors: `INVALID_TOKEN`, `TOKEN_USER_MISMATCH`, `DIFFERENT_USER`, `NETWORK_ERROR`, `SYNC_FAILED`, `SYNC_IN_PROGRESS`.

#### Step 6: Get the forms and submissions

```kotlin
fun getFormsAndSubmissions(username: String): String
```

| Parameter | Description |
|---|---|
| `username` | the synced user's username |

It must not be called on the main thread. Returns a JSON (see [Result format](#result-format)) where, on success, `data` contains the forms stored on the device, each with its submissions:

```json
{
  "error": null,
  "data": {
    "forms": [
      {
        "id": 123456,
        "type": "form",
        "name": "Site Inspection",
        "version": 3,
        "status": "published",
        "email_options": 2,
        "workflow_enabled": true,
        "image_path": "/data/user/0/<app>/.../img_987_1.png",
        "submissions": [
          {
            "guid": "0d1b5c2e-...",
            "name": "*** 09/24/26 10:15:02",
            "status": "open",
            "last_saved_at": 1727172902,
            "dispatch_id": null,
            "deletable": true,
            "pendingUpload": false,
            "rejected": false
          }
        ]
      }
    ]
  }
}
```

Use the form `id` to start a new submission with `showForm`, and the submission `guid` to open it with `showSubmission` or delete it with `deleteSubmission`.

The form properties:

| Property | Description |
|---|---|
| `id` | the form id, passed to `showForm` |
| `type` | always `form` |
| `name` | the form's display name |
| `version` | the form version stored on the device |
| `status` | `published` or `retired`, see below |
| `email_options` | the form's email setting, as configured in GoCanvas |
| `workflow_enabled` | `true` when the form is part of a workflow (its submissions can be handed off) |
| `image_path` | the local path of the form's image |
| `submissions` | the form's submissions stored on the device, see below |

The form `status` can contain one of the following values:
1. `published` - new submissions can be started with `showForm`
2. `retired` - the form has been retired or is no longer assigned; new submissions can no longer be created, but existing submissions can be opened and completed

The submission `status` can contain one of the following values:
1. `open` - the submission can still be edited: the user has elected to "Save and Close" it, or it is a dispatch or handoff not submitted yet
2. `closed` - the user has submitted the submission; it can no longer be edited and is waiting to be uploaded, or its media files are still uploading

Once a submission and all its media files are uploaded, the submission is removed from the device and is no longer returned. If its media files do not finish uploading, it stays `closed` until a subsequent `sync` uploads them.

The submission properties:

| Property | Description |
|---|---|
| `guid` | the submission id, passed to `showSubmission` and `deleteSubmission` |
| `name` | the submission's display name. For a handoff, its name as received from GoCanvas, falling back to the submission name or number when it has none |
| `status` | `open` or `closed`, see above |
| `last_saved_at` | when the submission was last saved, in epoch seconds, or `null` |
| `dispatch_id` | the dispatch id when the submission is a dispatch, otherwise `null` |
| `deletable` | whether `deleteSubmission` can delete it (dispatches, handoffs and completed submissions cannot be deleted) |
| `pendingUpload` | `true` when the submission is `closed`, so it will be uploaded (or finish uploading) on the next `sync` |
| `rejected` | `true` when the submission is a handoff rejected back to the user; it then also contains the `rejector` and the `rejection_note` |

Possible errors: `NO_SYNCED_USER`, `INVALID_USER`.

#### Step 8: Show a form or a submission

```kotlin
fun showForm(username: String, formId: Int, activity: Activity, formLauncher: ActivityResultLauncher<Intent>, prefilledEntries: String? = null)
fun showSubmission(username: String, guid: String, activity: Activity, formLauncher: ActivityResultLauncher<Intent>, prefilledEntries: String? = null)
```

- `showForm` - starts a new submission for the form with the specified `formId`
- `showSubmission` - opens the saved submission, dispatch or handoff with the specified `guid`. Only `open` submissions can be opened; a `closed` one returns a `SUBMISSION_NOT_EDITABLE` error

| Parameter | Description |
|---|---|
| `username` | the synced user's username |
| `formId` | `showForm` only: the form `id`, as returned by `getFormsAndSubmissions` |
| `guid` | `showSubmission` only: the submission `guid`, as returned by `getFormsAndSubmissions` |
| `activity` | the activity that shows the form |
| `formLauncher` | the launcher registered as described in [Register the form launcher](#register-the-form-launcher), which receives the result |
| `prefilledEntries` | optional, see [Prefill all entries based on labels](#prefill-by-label); the `label` is mandatory and must match the label of the form's entry |

Returns nothing: the result is delivered through `formLauncher` when the form flow ends. Both methods work with the data stored by `sync`, so you don't pass a form JSON or reference data.

Signatures and drawings on dispatches and handoffs are downloaded during `sync`, so they can be viewed immediately when opening the submission. Other media files are downloaded when the user taps on the field.

If the form or submission cannot be displayed, the result code is `Activity.RESULT_CANCELED` and the intent extras contain the error in `CanvasSdkKey.ERROR_NUMBER_KEY` and `CanvasSdkKey.ERROR_MESSAGE_KEY`. The possible errors are:

- `NO_SYNCED_USER` - when no user has been synced yet
- `INVALID_USER` - when `username` does not match the synced user
- `INVALID_JSON` - when the form is not found on the device
- `INVALID_SAVED_RESPONSE` - when the submission is not found on the device
- `FORM_RETIRED` - when starting a new submission for a retired form
- `INVALID_FORM_DEFINITION` - when the form has no sections, sheets or entries
- `REFERENCE_DATA_NOT_SET` - when the form's reference data is not stored on the device
- `SUBMISSION_NOT_EDITABLE` - `showSubmission` only: when the submission is `closed` and can no longer be edited

#### Step 9: Read the result

When the result is received by `formLauncher`, call `CanvasSdk.getResponse()`. It returns a JSON (see [Result format](#result-format)) where `data` contains the submission header, with the same properties as in [`getFormsAndSubmissions`](#step-6-get-the-forms-and-submissions):

```json
{
  "error": null,
  "data": {
    "guid": "0d1b5c2e-...",
    "name": "*** 09/24/26 10:15:02",
    "status": "closed",
    "last_saved_at": 1727172902,
    "dispatch_id": null,
    "deletable": false,
    "pendingUpload": true,
    "rejected": false
  }
}
```

Unlike the [Userless](#step-5-read-the-response) `getResponse()`, the entries' values are not returned: they are stored on the device and uploaded by the next `sync`.

#### Step 10: Sync again

Call [`sync`](#step-5-sync) again to upload the completed submissions (`pendingUpload: true`) and receive the latest forms, dispatches and handoffs. Use `uploadOnly = true` to only upload them.

### Other calls

#### deleteSubmission

```kotlin
fun deleteSubmission(username: String, guid: String): String
```

| Parameter | Description |
|---|---|
| `username` | the synced user's username |
| `guid` | the submission `guid`, as returned by `getFormsAndSubmissions` |

Deletes a saved submission from the device. It must not be called on the main thread. Returns a JSON (see [Result format](#result-format)) where, on success, `data` contains the deleted submission, with the same properties as in `getFormsAndSubmissions` (an empty object when no submission has that `guid`). Dispatches, handoffs and completed submissions cannot be deleted and return a `SUBMISSION_CANNOT_BE_DELETED` error.

Possible errors: `NO_SYNCED_USER`, `INVALID_USER`, `SUBMISSION_CANNOT_BE_DELETED`, `INVALID_SAVED_RESPONSE` (when the submission could not be deleted from the device).

#### hasPendingSubmissions

```kotlin
fun hasPendingSubmissions(): Boolean
```

Takes no parameters and returns `true` when any submission on the device has `pendingUpload: true` (its `status` is `closed`), so it will be uploaded or finish uploading on the next `sync`. It must not be called on the main thread.

### Working offline

Once a user is synced, a network connection is needed only for `sync` and for viewing some media files of dispatches and handoffs:

- `getFormsAndSubmissions`, `showForm`, `showSubmission`, `deleteSubmission`, `hasPendingSubmissions` and `clearData` work offline, using the data stored on the device
- completed submissions stay on the device (`pendingUpload: true`) until the next successful `sync`
- photos, videos and attachments of dispatches and handoffs are downloaded only when the user taps on the field to view them while filling out the submission, so a network connection and a valid token are needed at that moment. Signatures and drawings are already on the device after `sync`

### User-based error codes

In addition to the [shared errors](#errors), the user-based calls can return:

- `INVALID_USER` - when the `username` does not match the synced user
- `INVALID_TOKEN` - when the token has no `access_token`
- `NETWORK_ERROR` - when a request fails or the server rejects it
- `SYNC_FAILED` - when the sync fails for any other reason
- `TOKEN_USER_MISMATCH` - when the token belongs to a different user than `username`
- `DIFFERENT_USER` - when another user is already synced on the device
- `NO_SYNCED_USER` - when a call needs a synced user and none is synced yet
- `FORM_RETIRED` - when starting a new submission for a retired form
- `SYNC_IN_PROGRESS` - when `sync` is called while another sync is running
- `SUBMISSION_CANNOT_BE_DELETED` - when the submission is a dispatch, a handoff or waiting to upload
- `SUBMISSION_NOT_EDITABLE` - when `showSubmission` is called for a `closed` submission
- `USER_DATA_CONFLICT` - when the Userless `showForm` is called for a form that is already synced for the user, or that uses reference data already synced for the user

```kotlin
enum class CanvasSdkErrorType(val statusCode: Int, val errorDescription: String) {
    //... shared errors, see "Errors"
    INVALID_USER(90006, "The provided username does not match the user currently synced with the SDK."),
    INVALID_TOKEN(90007, "The provided token is invalid or missing an access_token."),
    NETWORK_ERROR(90008, "A network error occurred, please try again."),
    SYNC_FAILED(90009, "Unable to complete sync, please try again."),
    TOKEN_USER_MISMATCH(90010, "The provided token is not valid for the specified user."),
    DIFFERENT_USER(90011, "A different user is already synced. Clear data before syncing a new user."),
    NO_SYNCED_USER(90012, "You must have a synced user first."),
    FORM_RETIRED(90013, "This form has been retired and can no longer be used."),
    SYNC_IN_PROGRESS(90014, "A sync is already in progress."),
    SUBMISSION_CANNOT_BE_DELETED(90015, "This submission cannot be deleted and needs to be unassigned."),
    SUBMISSION_NOT_EDITABLE(90016, "This submission has been completed and can no longer be edited."),
    USER_DATA_CONFLICT(90017, "Unable to show form."),
}
```
