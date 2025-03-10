# PingOne for Customers Mobile SDK

## Overview

PingOne for Customers Mobile SDK is a set of components and services targeted at enabling organizations to include multifactor authentication (MFA) into native applications.
This solution leverages Ping Identity’s expertise in MFA technology, as a component that can be embedded easily and quickly into a new or existing application. The PingOne for Customers Mobile SDK package comprises of the following components:

* The PingOne for Customers Mobile SDK library for iOS applications.
* A sample app example source code for iOS.
* Mobile Authentication Framework for iOS Developers (integrated into the sample app).

Release notes can be found [here](./release-notes.md).

### Documentation

Reference documentation is available for PingOne for Customers Mobile SDK, describing its capabilities, features, installation and setup, integration with mobile apps, deployment and more: 

* [PingOne for Customers Mobile SDK release notes and admin related documentation](https://docs.pingidentity.com/csh?Product=p1&context=p1mfa_c_introduction)
* [PingOne for Customers Mobile SDK developer documentation](https://apidocs.pingidentity.com/pingone/native-sdks/v1/api/#pingone-mfa-native-sdks)


## Set up a mobile app using the PingOne SDK sample code

### Prerequisites

Prepare the iOS push messaging mandatory data from Apple Developer portal:

* Key ID
* Team ID
* Token .p8 file
* Bundle ID

Refer to: [Establishing a Token-Based Connection to APNs](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server/establishing_a_token-based_connection_to_apns).


### Configure iOS push messaging on the PingOne Portal

#### iOS Push Notification:

When configuring your PingOne SDK application in the PingOne admin web console (**Connections > Applications > {NATIVE application} > Edit > Authenticator**), you should upload your .p8 token and fill in the Key ID, Team ID and Bundle ID. See [Edit an application](https://documentation.pingidentity.com/pingone/p14cAdminGuide/index.shtml#p1_t_editApplication.html) in the administration guide.



### Compatibility

**Note:** PingOne SDK supports the following software versions:

* Xcode 13 and above.
* iOS 12.0 and above.


### Xcode integration

Add the PingOne SDK component into your existing project
1. In your **Project Navigator**, click on your target, and drag **PingOneSDK.xcframework** to **Frameworks, Libraries, and Embedded Content**.
2. Check the **Copy items if needed** checkbox.

    ![](./img/p1_i_xc11-SDKintegrateIntoIDE.png)


3. Integrate the PingOneSDK component into your code:
	* Import the framework into your application initialization code:<br>`import PingOneSDK`

### Pairing

To manually pair the device, call the following method with your pairing key:

```swift
/// Pair device
///
/// - Parameters:
///   - pairingKey: The `String` value
///   - completionHandler: Will return PairingInfo object containing data about pairing resolution, and NSError in case of an error. Documentation for pairing object error codes: https://apidocs.pingidentity.com/pingone/mobile-sdks/v1/api/#pingone-mobile-sdk-for-ios
@objc public static func pair(_ pairingKey: String, completion: @escaping (_ response: PairingInfo?, NSError?) -> Void)
```

To automatically pair the device using OpenID Connect:

1. call this function to get the PingOne SDK mobile payload:
```swift
@objc public static func generateMobilePayload() throws -> String
```
2. pass the received mobile payload on the OIDC request as the value of query param: `mobilePayload`
3. call this function with the ID token after the OIDC authentication completes:
```swift
@objc public static func processIdToken(_ idToken: String, completionHandler: @escaping (_ pairingObject: PairingObject?, _ error: NSError?) -> Void)
```

### Working with push messages in iOS

This section details the steps needed in order to work with push messages in iOS:

* Enable **Push Notifications**: Go to your Project Navigator’s **capabilities** tab. Select **Push Notifications > Enable**.
* Enable **Remote Notifications**: Go to your Project Navigator’s **capabilities** tab. Turn on **Background Modes > Remote notifications**.
* Enable **Push Notifications** in your **Apple Developer Account > Certificates, Identifiers & Profiles > Identifiers > Capabilities > Push Notifications > Enable**.
* Enter your app’s **Deployment Details** settings in your **Apple Developer Account > Certificates, Identifiers & Profiles > Identifiers > Your App ID.** These details are now mandatory prerequisites for Apple to register your device for push notifications.

#### Register device token on PingOne server

In order to receive push notifications from PingOne SDK, use the following code in your `didRegisterForRemoteNotificationsWithDeviceToken` call, passing the deviceToken as is:

```swift
@objc public static func setDeviceToken(_ deviceToken: Data, type: APNSDeviceTokenType, completionHandler: @escaping (_ error: NSError?) -> Void)
```

### Handling Push Notifications

PingOne SDK will only handle push notifications which were issued by the PingOne SDK server. For other push notifications, `NSError` with the code `10002, unrecognizedRemoteNotification` will be returned.

The `APNSDeviceTokenType` should be set like this:

```swift
var deviceTokenType : PingOne.APNSDeviceTokenType = .production
#if DEBUG
deviceTokenType = .sandbox
#endif
```

Inside the following AppDelegate method:

```swift
optional func application(_ application: UIApplication,
didReceiveRemoteNotification userInfo: [AnyHashable : Any],
fetchCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void)
```

Call:

```swift
@objc public static func processRemoteNotification(_ userInfo: [AnyHashable : Any], completionHandler: @escaping (_ notificationObject: NotificationObject?, _ error: NSError?) -> Void)
```

and pass it the `userInfo` as is.

#### Push Notifications Categories:

PingOne SDK uses categories for different notifications. If your app already uses categories, you will need to retrieve the PingOne SDK categories `Set<UNNotificationCategory>`, by calling:    

```swift
@objc public static func getUNNotificationCategories() -> Set<UNNotificationCategory>
```

and add them to your current categories, for example:

```swift
let center  = UNUserNotificationCenter.current()
center.delegate = self
center.requestAuthorization(options: [.sound, .alert, .badge]) { (granted, error) in
    if error == nil
    {
        // Registering UNNotificationCategory more than once results in previous categories being overwritten. PingOne provides the needed categories. The developer may add categories.
        UNUserNotificationCenter.current().setNotificationCategories(PingOne.getUNNotificationCategories())
        DispatchQueue.main.async {
            UIApplication.shared.registerForRemoteNotifications()
        }
    }
}
```


#### Localization

The following keys are returned by the PingOne SDK Remote Notification, with suggested localization:

```swift
"notification.confirm"  = "Approve";
"notification.deny"     = "Deny";
"notification.message"  = "You have a new authentication request.";
"notification.title"    = "New Authentication";
```

**Note:** An example of these keys is provided in the sample app, in the `Localizable.strings` file.

### Keychain Sharing

**Note:** This step is required only if your app uses Keychain Sharing.
Make sure that the first item on your Keychain Groups is `YOUR_BUNDLE_ID` (your private keychain group). This requirement will ensure that the SDK keychain values are private, and are not shared between apps​:


![](./img/p1_i_SDKkeychainSharing.png)

#### Mobile Authentication Framework

The following method starts an authentication process when the user taps "Authentication API" on the main screen. The authentication process is completed by the PingFederate Authentication API.

**Note:** Before running this method, you need to update your `oidcIssuer` and `clientId` in the Config.swift class. See [Authentication API for Developers Mobile iOS code](https://github.com/pingidentity/mobile-authentication-framework-ios)

```swift
func authenticate(With payload: String){
    authnUI.authenticate(presenter: self, payload: payload, dynamicData: "") { [weak self] (serverPayload, accessToken, error) in
        guard let self = self else { return }
        
        if (serverPayload.count > 0) { //Need pairing
           PingOne.processIdToken(serverPayload) { (pairingObject, error) in
               self.stopLoadingAnimation()
               if let pairingObject = pairingObject{
                   self.displayNotificationViewAlert(pairingObject)
               }
               else if let error = error{
                   print(error.localizedDescription)
               }
           }
        } else {
           self.stopLoadingAnimation()
        }
    }
}
```

### PingOne Mobile SDK sample app

The PingOne Mobile SDK bundle provides a sample app that includes all the basic flows in order to help you get started.

### Send logs

```swift
// Call this method if you want to send logs to the PingOne support team.
PingOne.sendLogs { (supportId, error) in
    if let supportId = supportId{
        print("Support ID:\(supportId)")
        }
    }
    else if let error = error{
        print("error sending logs: \(error.debugDescription)")
    }
}
```

### Get one time passcode

```swift
/// Get one time passcode object
/// - Parameter completionHandler: OneTimePasscodeData object if available or error
@objc public static func getOneTimePasscode(_ completionHandler: @escaping (_ oneTimePasscodeData: OneTimePasscodeData?, _ error: Error?) -> Void)
```

### Disable SDK push notifications

```swift
/// Method that will notify the server not to send push messages to this device. Set to false to disable SDK push notifications.
/// - Parameter allowed: a boolean that will tell the server to send push messages or not. Defaults to `true`.
@objc (allowPushNotifications:) public static func pushNotification(allowed: Bool)
```

### Authentication via QR code scanning

PingOne SDK provides an ability to authenticate via scanning the QR code (or typing the code manually). 
The authCode should be passed to the PingOne SDK as is or inside a URI. 
For example: "7F45HGF5", "https://myapp.com/pingonesdk?authentication_code=7F45HGF5", "pingonesdk?authentication_code=7F45HGF5"

```swift
/// Authenticate with code
///
/// PingOne SDK provides an ability to authenticate via scanning the QR code.
/// The retrieved value should be passed to the PingOne SDK using the following API method.
///
/// - Parameters:
///   - authCode: String value of the authentication code, parsed from QR or manual input.
/// - Returns: a completionHandler that contains `authenticationObject`: users list array, clientContext object,
/// userApproval String and status String. In case of an error will return NSError.
@objc public static func authenticate(_ authCode: String, completionHandler: @escaping (_ authenticationObject: AuthenticationObject?, _ error: NSError?) -> Void) 
```
AuthenticationObject return from the authenticate method, contains the following varibles and methods:

```swift   
/// List of users that are paired to this device. Each user object can contain the following String variables: `userId`, `email`, `given`, `family` and `username`.
@objc public var users: [[String: Any]]?
/// String that determines if user approval is required to complete an authentication. Possible values: `REQUIRED` and `NOT_REQUIRED`
@objc public var userApproval: String?
/// Object for passing any data as a String from server to end-user
@objc public var clientContext: String?
/// String value returned from a server when a user calls an authenticate API method. Possible values: can be `CLAIMED`, `EXPIRED`, `DENIED` or `COMPLETED`
@objc public var status: String?
    
/// Approve authentication with code
///
/// If `userApproval` is `REQUIRED` or multiple users may approve the authentication,
/// call this method with a userId of the user, who triggered the method.
/// - Parameter userId: String value of the user id, fetched from the user object.
/// - Returns completionHandler: Will return status String value with one of the following values:
/// `COMPLETED` or `EXPIRED`.
/// Returns NSError in case of an error.
@objc public final func approve(userId: String, completionHandler: @escaping (_ status: String?, _ error: NSError?) -> Void) 

/// Deny authentication with code
///
/// If `userApproval` is `REQUIRED` or multiple users may deny the authentication,
/// call this method with a userId of the user, who triggered the method.
/// - Parameter userId: String value of the user id, fetched from the user object.
/// This field is not mandatory, authentication can be denied without passing this value.
/// - Returns completionHandler: Will return status String value with one of the following values:
/// `DENIED` or `EXPIRED`.
/// Returns NSError in case of an error.
@objc public final func deny(_ userId: String? = nil, completionHandler: @escaping (_ status: String?, _ error: NSError?) -> Void)
```

## Disclaimer

THE SAMPLE CODE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SAMPLE CODE OR THE USE OR OTHER DEALINGS IN
THE SAMPLE CODE.  FURTHERMORE, THIS SAMPLE CODE IS NOT COMMERCIALLY SUPPORTED BY PING IDENTITY BUT QUESTIONS MAY BE ADDRESSED TO PING'S SUPPORT CENTER OR MAY BE OTHERWISE ADDRESSED IN THE RELATED DOCUMENTATION.

Any questions or issues should go to the support center, or may be discussed in the [Ping Identity developer communities](https://community.pingidentity.com/collaborate).
