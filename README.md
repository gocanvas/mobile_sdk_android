# GoCanvas SDK for Android

#### Table of contents:
- [Installation](#Installation)
- [Usage](#Usage)
- [Branding](#Branding)

## Installation

GoCanvas SDK requires at minimum Android API 26+.

1. Install via GitHub Packages. Add the following source to your `repositories` section

Create a GitHub access token with `read:packages` scope then replace <github_username> and <github_access_token> with your credentials.


```gradle
repositories {
  //...
  maven {
     name = "GitHubPackages"
     url = uri("https://maven.pkg.github.com/gocanvas/mobile_sdk_android")
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
  implementation("com.gocanvas.sdk:android:<version>")
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
## Usage
### SDK Api
The Sdk provides the following api interface:
```kotlin
interface CanvasSdkApi {
    /**
     * Initiates the SDK with the given [licenseKey]
     */
    fun init(licenseKey: String)

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
     * Returns the submission response in JSON string format.
     * */
    fun getResponse(): String
}
```
It can be accessed using the `CanvasSdk` instance.

### Initiate the SDK

Use the `CanvasSdk.init` method to pass the License key to the sdk.

```kotlin
CanvasSdk.init(licenseKey = <your_license_key>)
```

### Display Form

1. Before displaying the form, make sure you have first registered for `ActivityResult`:
```java
ActivityResultLauncher<Intent> formLauncher = registerForActivityResult(new ActivityResultContracts.StartActivityForResult(),
        new ActivityResultCallback<ActivityResult>() {
          @Override
          public void onActivityResult(ActivityResult result) {
            // Handle the result
          }
        });
```
2. Then you can start displaying the form by calling the `showForm` method on the SDK Api entry point instance as follows:
```java
CanvasSdk.INSTANCE.showForm(formJson, activity, formLauncher);
```

3. Form definition

SDK `formJson` param supports the `format=sync` form definition obtained through the GoCanvas API.

Example: `https://www.gocanvas.com/api/v3/forms/123456?format=sync`

#### Configure Form

You can set additional form configuration by passing to the `showForm` method an instance of `CanvasSdkFormConfig`.

```kotlin
/**
 * Canvas SDK Form config
 * @param formJson the form in JSON string format
 * @param referenceDataJson the reference data in JSON string format.
 * Supports both [JSONObject] & [JSONArray] types in case of a single or multiple reference data
 * associated with the given [formJson].
 * @param prefilledEntries the prefilled entries in JSON string format.
 */
data class CanvasSdkFormConfig(
    val formJson: String,
    val referenceDataJson: String? = null,
    val prefilledEntries: String? = null
)
```

##### Prefilled Entries scope
Acts as support for prefilling the form's entries by passing a list of responses.

1. Prefill all entries based on labels:
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

2. Prefill only specific entries based on form's entry id:
```json
{
  "responses": [
    {
      "entry_id": 123456,
      "value": "Example Value",
      "label": "Example Label"
    }
  ]
}
```

### Receive Form Response

The response is being returned through the `ActivityResultCallback` passed to `formLauncher` on the `showForm` method.

Result code info:
- `Activity.RESULT_OK` - when form flow has been completed successfully
- `Activity.RESULT_CANCELED` - when form flow has been canceled due to user cancellation or through an error

Intent extras key access:
- `CanvasSdkKey.RESPONSE_KEY` - contains the info if the form response exists as `Boolean`
- `CanvasSdkKey.ERROR_NUMBER_KEY` - contains the error code in `String` format
- `CanvasSdkKey.ERROR_MESSAGE_KEY` - contains the error message in `String` format

#### Reading the response:

You can access the form response by calling the `CanvasSdk.getResponse()` method after the result is received from the `formLauncher`.

Example:
```java
ActivityResultLauncher<Intent> formLauncher = registerForActivityResult(new ActivityResultContracts.StartActivityForResult(),
        new ActivityResultCallback<ActivityResult>() {
          @Override
          public void onActivityResult(ActivityResult result) {
            int resultCode = result.getResultCode();
            Intent data = result.getData();
            Log.i("ActivityResult", "resultCode: " + resultCode);
            switch (resultCode) {
              case Activity.RESULT_OK:
                // form completed successfully
                if (data != null && data.hasExtra(RESPONSE_KEY)) {
                  Log.i("ActivityResult", "form has response: " + data.getExtras().get(RESPONSE_KEY));
                  Log.i("ActivityResult", "form response: " + CanvasSdk.INSTANCE.getResponse());
                }
                break;
              case Activity.RESULT_CANCELED:
                // user canceled or an error occurred
                if (data != null && data.hasExtra(ERROR_NUMBER_KEY) && data.hasExtra(ERROR_MESSAGE_KEY)) {
                  Log.i("ActivityResult", "error code: " + data.getExtras().get(ERROR_NUMBER_KEY));
                  Log.i("ActivityResult", "error message: " + data.getExtras().get(ERROR_MESSAGE_KEY));
                }
                break;
            }
          }
        });
```

#### Reading Media files

Images, Drawings, Signatures, Videos, Attachments

1. Entry response with single media file

- the library is returning the local path to file as `String` on the JSON `value` field.

```
{
  ..
  "value": "path_to_image_1",
  "label": "Camera Photo"
} 
```

2. Entry response with multiple media files

- the library is returning a single `String` that concatenates all the paths to file. The paths are delimitated by the “\r\n” separator. It can be accessed on the JSON `value` field

```
{
   ..
   "value": "path_to_image_1\r\n\path_to_image_2\r\n\path_to_image_3"
   "label": "Multi Photo"
}
```

### Resume Form Response

The SDK provides the ability to resume the form response in case of an app crash or of a partially form completion.
In order to do that, you just have to call the `CanvasSdk.showForm` method again by passing the same `formJson` input. The User will be prompted when there is a partially saved form response and he/she will be able to choose between resuming the form or starting a new submission of it.

### Errors
The SDK supports the following error types:
- `INVALID_JSON` - when the `formJson` cannot be parsed to `Form`
- `INVALID_FORM_DEFINITION` - when the `Form` has no sections, sheets or entries
- `INVALID_SAVED_RESPONSE` - when the `Response` cannot be restored after partially form saving
- `REFERENCE_DATA_NOT_SET` - when the `Form` definition contains reference data but was not passed
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
    INVALID_PACKAGE_IDENTIFIER(90005,"Invalid package identifier, please contact your account manager for assistance."),
}
```

## Branding
### Colors

The color system that can be used to create a color scheme that reflects your brand or style.

| Color Attribute           | Theme Color Role                      | Default                                                          | Affected Ui Components                                                                        | 
---------------------------|---------------------------------------|------------------------------------------------------------------|-----------------------------------------------------------------------------------------------| 
| gc_sdk_color_primary      | colorPrimary                          | ![#039de7](https://placehold.co/15x15/039de7/039de7.png) #039de7 | toolbar, dialog buttons, primary button, tint input fields                                    | 
| gc_sdk_color_primary_dark | colorPrimaryDark                      | ![#0077b3](https://placehold.co/15x15/0077b3/0077b3.png) #0077b3 | status bar                                                                                    | 
| gc_sdk_color_accent       | colorAccent                           | ![#5fa3d0](https://placehold.co/15x15/5fa3d0/5fa3d0.png) #5fa3d0 | date & time pickers top area background, input fields selected text                           | 
| gc_sdk_color_secondary    | colorControlActivated, colorSecondary | ![#00BFA5](https://placehold.co/15x15/00BFA5/00BFA5.png) #00BFA5 | progress bar, date & time pickers selected value, checkbox, input fields with captured values | 
| gc_sdk_color_icon         | :colorPrimary                         | ![#039de7](https://placehold.co/15x15/039de7/039de7.png) #039de7 | increment & decrement icons                                                                   | 
| gc_sdk_color_rating_selected       | :colorPrimary                         | ![#039de7](https://placehold.co/15x15/039de7/039de7.png) #039de7 | rating selected icons                                                                         | 

By overriding these color attributes, you can easily change the styles of all the mentioned components used by the sdk.

**Applying your own colors:**

Override the color attributes in your `colors.xml` file

```xml
<resources>
    <color name="gc_sdk_color_primary">...</color>
    <color name="gc_sdk_color_accent">...</color>
    <color name="gc_sdk_color_primary_dark">...</color>
    <color name="gc_sdk_color_secondary">...</color>
    <color name="gc_sdk_color_icon">...</color>
    <color name="gc_sdk_color_rating_selected">...</color>
</resources>
```
