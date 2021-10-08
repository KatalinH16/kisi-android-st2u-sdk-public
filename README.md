# Kisi's Android ST2U SDK

This project implements Kisi's secure tap to unlock algorithm.

## Requirements

- Android 5.0 at minimum
- Android Studio 2020.3.1

## Integration

### Build files

Add the latest AAR file (check the Releases page on Github) into your app module (`app/libs` by default), then open the `build.gradle` file of your app module and add dependencies needed by the SDK:

```gradle

dependencies {

    implementation fileTree(dir: 'libs', include: ['*.jar', '*.aar'])

    implementation "at.favre.lib:hkdf:1.1.0"
    implementation "io.reactivex.rxjava3:rxjava:3.0.0"
}
```

Sync the project with Gradle files.

### Card emulation

Kisi's Secure Tap to Unlock ("S2U") technology for Android is based on Android's [Host-based Card Emulation](https://developer.android.com/guide/topics/connectivity/nfc/hce) (or HCE), so the integration of the SDK in question consists of tying together the Android's entry point into host card emulation (called [HostApduService](https://developer.android.com/reference/android/nfc/cardemulation/HostApduService)) and Kisi's code.

We need to create a subclass of HostApduService, so let's take a look at how the sample implementation can look like:

```kotlin

class ScramTestService : HostApduService() {

    private lateinit var offlineMode: IOfflineMode

    private var disposable: Disposable? = null

    override fun onCreate() {
        super.onCreate()

        offlineMode = Scram3.with(
            applicationContext,
            loginFetcher = { organizationId: Int? ->
                Maybe.just(
                    Login(
                        id = 42,
                        authenticationToken = "35B8ACFCF1F6AB6604CEB9F9157303A9",
                        phoneKey = "40CA258E7D5850C62068D70784B0DB7D",
                        onlineCertificate = "6D011D6C6F6D6E276BB6FEF7EFA5F87BBC3E7D9B945C786EAE5C086716F4B5EFF901D94A7DF90A98F1D9CEE6984F9588A8EF4CE59D8B20194A254BD7"
                    )
                )
            },
            unlockFailureListener = { }
        )
    }

    override fun processCommandApdu(commandApdu: ByteArray, extras: Bundle?): ByteArray? {
        disposable = offlineMode
            .handle(commandApdu)
            .subscribeWith(
                object : DisposableSingleObserver<ByteArray>() {
                    override fun onSuccess(t: ByteArray) {
                        sendResponseApdu(t)
                    }

                    override fun onError(e: Throwable) {
                        sendResponseApdu(null)
                    }
                }
            )

        return null
    }

    override fun onDeactivated(reason: Int) {
        disposable?.dispose()
        disposable = null
    }
}
```

The key component that you're going to use is an implementation of `IOfflineMode` interface; at the time of this writing, there is only one implementation called `Scram3`. This class requires two parameters to perform its duties:

* An instance of Android's [Context](https://developer.android.com/reference/android/content/Context) class.
* And a callback that returns an instance of Login class wrapped in RxJava's [Maybe](http://reactivex.io/RxJava/3.x/javadoc/io/reactivex/rxjava3/core/Maybe.html).

An instance of `Login` contains 4 properties, all of which you will get while signing the user in via [Kisi's API](https://api.kisi.io/docs/#tag/Logins/paths/~1logins/post):

* `id` corresponds to the `id` field of aforementioned request
* `authenticationToken` corresponds to the `authentication_token` field
* `phoneKey` corresponds to the `phone_key` field of the `scram_credentials` object
* `onlineCertificate` corresponds to the `online_certificate` field of the `scram_credentials` objects

### Manifest

The last thing to do is to add the service to the app's manifest:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application>

        <service
            android:name=".ScramTestService"
            android:exported="true"
            android:permission="android.permission.BIND_NFC_SERVICE">

            <intent-filter>
                <action android:name="android.nfc.cardemulation.action.HOST_APDU_SERVICE" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>

            <meta-data
                android:name="android.nfc.cardemulation.host_apdu_service"
                android:resource="@xml/hce_service" />
        </service>

    </application>
</manifest>
```

`hce_service` is an XML file shipped with Kisi's SDK. It defines the set of properties used by Android OS to determine which of the installed applications will handle an incoming NFC connection. You should not override this file because otherwise, Android OS won't pick your application as a handler of an incoming connection from Kisi's reader.

Build your app and make sure that SDK functions as expected.

### Optional parameters of Scram3

`unlockFailureListener` is an optional parameter that you can provide to be notified about unlocking errors. ' UnlockError` enumeration defines the list of existing errors:

```kotlin
enum class UnlockError {

    PHONE_LOCKED,
    PHONE_LOCK_MISSING
}
```

Both errors correspond to Kisi's implementation of [two-factor authentication](https://www.getkisi.com/features/2fa) for unlocks. If the place administrator has enabled 2FA for premises, we're trying to answer two questions when unlock is happening:

1. Is the screen lock enabled on this device?
2. If it's enabled, is the screen currently locked?

If the answer to any of these two questions is negative, the notification asking the user to either enable the screen lock or unlock the device to prove their identity pops up on the screen. But what if the user has deliberately turned off notifications on their device? In this case, you can use the provided callback to be notified about an error and act accordingly later. E.g., you can show a popup in your application next time the user opens it to let them know that notifications need to be turned on.