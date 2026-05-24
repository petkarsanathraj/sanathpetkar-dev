---
title: "USB Serial in React Native (Android): 4 gotchas nobody warns you about"
description: "Most React Native USB-serial libraries on npm are broken on Android 12+. Here's what actually works in 2026, distilled from a year of running this code on field devices."
publishedAt: 2026-05-24
tags: ["react-native", "android", "kotlin", "usb-serial", "native-modules"]
readingTimeMin: 11
---

A year ago I built a React Native app that talks to industrial measurement equipment over USB-OTG. The app runs on Android tablets in the field. Field engineers plug a serial cable into the tablet, the cable runs to a tester box, and the app streams readings live, GPS-tags them, and writes them to a CSV at the end of the day.

It was supposed to be a two-week project. The actual work was three days of UI and **three weeks of fighting Android's USB stack and the broken state of every React Native USB library on npm**.

Here are the four non-obvious things I had to discover the hard way. If you ever need to read USB-serial data in React Native on Android — for IoT, medical, automotive, industrial, hobby — this is what you need to know in 2026.

> **The working code lives in [`react-native-usb-serial-android`](https://github.com/petkarsanathraj/react-native-usb-serial-android)** — extracted from the production app and sanitized. MIT licensed. Drop it into your project, or read its source alongside this post.

## Why this is harder than it should be

If you search npm for `react-native usb serial` you'll find three or four libraries. Most of them were written before Android 12 (API 31, released 2021) and haven't been updated since. They look fine in their README. They install cleanly. They build.

Then your app crashes on any modern phone the moment you call `requestPermission()`.

The reason is a security model change Android made in API 31 around `PendingIntent` mutability. Old libraries pass the wrong flags, and the OS throws an `IllegalArgumentException` before you ever get to talk to the USB device. The crash is silent in most error reporting because it happens inside the system's USB permission flow, not your app's main thread.

So if you need USB-serial in React Native on Android in 2026, you have three options: fork an unmaintained library, write the native module yourself, or use a maintained one. This post is essentially the brain-dump from option two.

## The setup: `usb-serial-for-android` + a thin RN bridge

I am not going to write a USB-serial driver from scratch. There is no point — [`mik3y/usb-serial-for-android`](https://github.com/mik3y/usb-serial-for-android) is a battle-tested Java library that handles the actual chipset protocols (FTDI, CDC/ACM, CH340, CP210x, Prolific). It is the de-facto standard on Android and most of the npm libraries wrap it too.

The job is just to expose it to JavaScript through a React Native native module:

```kotlin
class UsbSerialModule(private val reactContext: ReactApplicationContext) :
    ReactContextBaseJavaModule(reactContext), SerialInputOutputManager.Listener {

    private var port: UsbSerialPort? = null
    private var ioManager: SerialInputOutputManager? = null
    private var readingActive = false

    override fun getName(): String = "UsbSerialModule"
}
```

Register it in a `ReactPackage`, wire it up via React Native autolinking, and on the JS side you reach for it via `NativeModules.UsbSerialModule`. None of that is interesting. The interesting parts are the four bugs I am about to show you.

## Gotcha #1: `FLAG_MUTABLE` on Android 12+ or your app crashes on permission request

When your app first tries to open a USB device, Android requires user permission via a system dialog. You request permission like this:

```kotlin
val permissionIntent = PendingIntent.getBroadcast(
    reactContext,
    0,
    Intent(ACTION_USB_PERMISSION).apply { setPackage(reactContext.packageName) },
    PendingIntent.FLAG_UPDATE_CURRENT,        // ← BROKEN on Android 12+
)
usbManager.requestPermission(device, permissionIntent)
```

That second-to-last line crashes on Android 12 and up. Starting at API 31, Android requires every `PendingIntent` to **explicitly declare** whether it is mutable or immutable. If you do not pass one of `FLAG_MUTABLE` or `FLAG_IMMUTABLE`, the system throws an `IllegalArgumentException` the moment you try to use the intent.

USB permission specifically needs `FLAG_MUTABLE` because the system writes the permission-granted boolean into the intent's extras before broadcasting it back to you. An immutable intent would arrive with no payload, and you would have no way to tell whether permission was granted.

The fix is version-gated:

```kotlin
val permissionIntent = PendingIntent.getBroadcast(
    reactContext,
    0,
    Intent(ACTION_USB_PERMISSION).apply { setPackage(reactContext.packageName) },
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE
    } else {
        PendingIntent.FLAG_UPDATE_CURRENT
    }
)
```

If you write `FLAG_IMMUTABLE` here — which is what a lot of "fixed" forks did, copy-pasted from notification tutorials — the request goes through but the response handler will see permission granted as `false` every single time, because the broadcast intent's extras are empty. You will spend hours wondering why the user is "denying" a permission dialog they never saw.

The takeaway: USB permission requires a **mutable** `PendingIntent` on API 31+. Hard-code this and forget about it.

## Gotcha #2: UTF-8 will silently corrupt your serial stream

This one cost me an evening.

When data arrives from the USB device, `SerialInputOutputManager` hands you a `ByteArray`. The natural thing to do is decode it to a string and emit it to JavaScript:

```kotlin
override fun onNewData(data: ByteArray) {
    val text = String(data)                   // ← BROKEN
    sendEvent("usbData", text)
}
```

`String(data)` without an explicit charset uses the platform default, which on Android is UTF-8. The problem: serial protocols are not UTF-8. They are arbitrary bytes — 0x00 through 0xFF — that may or may not represent text at all. Many bytes are not valid UTF-8 sequences. When the decoder hits one, it silently replaces it with `U+FFFD` (the `` replacement character).

If your protocol has any byte above 0x7F — most binary framing, checksums, sensor reading bytes do — you will start seeing `` characters appear in your stream. Even worse, **a single invalid byte can corrupt the next several characters** if it looks like the start of a multi-byte UTF-8 sequence. Your parser logic will silently miss readings, or interpret them wrong, or worst of all, occasionally appear to work.

The fix is to use a charset that has a 1-to-1 byte → char mapping. **ISO-8859-1 maps every byte 0–255 to a unique Unicode code point**, no exceptions. There are no invalid sequences. You can losslessly round-trip arbitrary bytes through a JavaScript string and re-encode them on the JS side later.

```kotlin
override fun onNewData(data: ByteArray) {
    // 1-to-1 byte → char mapping. No replacement characters.
    val text = String(data, StandardCharsets.ISO_8859_1)
    sendEvent("usbData", text)
}
```

On the JS side, if you need the raw bytes back, you can re-encode each character to its code point:

```ts
function stringToBytes(s: string): Uint8Array {
  const bytes = new Uint8Array(s.length);
  for (let i = 0; i < s.length; i++) bytes[i] = s.charCodeAt(i) & 0xff;
  return bytes;
}
```

Lossless. If your protocol is line-based ASCII, you can also just keep treating the string as text — every byte under 0x80 is identical in UTF-8 and ISO-8859-1.

If you remember one thing from this post, remember this one. UTF-8 on a binary stream is the kind of bug that ships to production and surfaces three months later as "weird data in the field."

## Gotcha #3: Watch for unplugs or your "connected" state lies

USB devices get yanked. A field engineer pulls the cable mid-reading. A user knocks the OTG dongle out of the port. If your app does not actively listen for this, your JavaScript state happily reports `connected: true` long after the kernel has marked the device gone, and your next read attempt will throw a generic IO exception with no useful message.

Register a `BroadcastReceiver` for `ACTION_USB_DEVICE_DETACHED` in your module's `initialize()`:

```kotlin
override fun initialize() {
    super.initialize()
    reactContext.registerReceiver(
        usbDetachReceiver,
        IntentFilter(UsbManager.ACTION_USB_DEVICE_DETACHED),
        Context.RECEIVER_NOT_EXPORTED,
    )
}

private val usbDetachReceiver = object : BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        if (intent?.action == UsbManager.ACTION_USB_DEVICE_DETACHED) {
            sendEvent("usbStatus", "USB device detached")
            sendEventBool("usbConnected", false)
            ioManager?.stop()
            port?.close()
        }
    }
}
```

The `RECEIVER_NOT_EXPORTED` flag is required on API 34+ for receivers that should only get system broadcasts — without it your app will refuse to register. Forget that flag and you will spend an afternoon debugging a phantom `SecurityException`.

On the JS side, your `useUsbSerial()` hook subscribes to the `usbConnected` event and updates state. The user pulls the cable, your indicator dot goes grey, your "Disconnect" button becomes redundant, and any in-flight reads error out cleanly with a meaningful "USB stopped" message.

## Gotcha #4: bridging the async permission callback back to a JS Promise

This one is more of a design problem than a bug, and it is the part of native-module work that trips up developers who haven't done it before.

In JS land, you want the API to look like:

```ts
await UsbSerial.connect({ baudRate: 9600 });
// either we are connected now, or this threw
```

But on the native side, the permission flow is asynchronous **across a process boundary**. Your `connect()` method calls `usbManager.requestPermission()`. That triggers the system to draw a dialog, the user maybe-taps-allow, the system fires a broadcast back to your receiver, the receiver gets called on a system handler thread, and only then do you know whether you can actually open the port.

Meanwhile, your `connect()` method has long since returned. The `Promise` is dangling. You have to plumb the eventual result back to JavaScript through the React Native bridge.

The pattern that works:

```kotlin
private var connectPromise: Promise? = null
private var pendingBaudRate: Int = 9600
private var pendingDataBits: Int = 8
private var pendingStopBits: Int = 1

@ReactMethod
fun connect(baudRate: Int, dataBits: Int, stopBits: Int, promise: Promise) {
    connectPromise = promise

    val drivers = UsbSerialProber.getDefaultProber().findAllDrivers(usbManager)
    if (drivers.isEmpty()) {
        promise.reject("NO_DEVICE", "No USB serial device found")
        return
    }

    val device = drivers[0].device
    if (usbManager.hasPermission(device)) {
        openConnection(drivers[0].ports.first(), baudRate, dataBits, stopBits, promise)
    } else {
        // Save the connect args because the receiver callback won't have them
        pendingBaudRate = baudRate
        pendingDataBits = dataBits
        pendingStopBits = stopBits
        usbManager.requestPermission(device, permissionIntent)
        // promise stays unresolved — receiver will resolve/reject it later
    }
}

private fun handlePermissionResult(intent: Intent?) {
    val granted = intent?.getBooleanExtra(UsbManager.EXTRA_PERMISSION_GRANTED, false) ?: false
    if (!granted) {
        connectPromise?.reject("PERMISSION_DENIED", "USB permission denied")
        return
    }

    val driver = UsbSerialProber.getDefaultProber().findAllDrivers(usbManager).first()
    val connection = usbManager.openDevice(driver.device)
    openConnection(driver.ports.first(), connection, pendingBaudRate, pendingDataBits, pendingStopBits, connectPromise!!)
}
```

The two things to internalize:

1. **The `Promise` is stored on the module instance**, not in a local. Module instances live for the lifetime of the React context, so the promise survives the trip from `connect()` invocation all the way through the system permission dialog.
2. **Connection parameters are saved alongside the promise** because the broadcast receiver fires in its own context — it does not have access to whatever locals `connect()` had.

If you forget step 2 your "connect at 115200 baud" call will silently fall back to the default 9600 the first time the user has to grant permission, and inexplicably work correctly on every subsequent connect. That is a fun one to diagnose.

## What about iOS?

The honest answer: not without manufacturer cooperation.

Apple does not let third-party iOS apps talk to arbitrary USB or serial devices. To communicate with a Lightning or USB-C accessory, the device manufacturer has to:

1. Enroll in Apple's [Made for iPhone (MFi)](https://mfi.apple.com/) program.
2. Implement a custom protocol identifier in the device firmware.
3. Ship the protocol via Apple's External Accessory framework on the app side.

For one-off industrial or maker hardware, this is almost never economically viable. If a freelance client asks me to "build the same thing for iPhone too," I tell them the truth: their hardware probably doesn't ship from an MFi-licensed manufacturer, so iOS is not on the table without significant external work. Saves both sides a lot of grief.

The library I extracted is Android-only by design and the README says so on the first screen.

## The library

If you want to skip past all of this and just have it work, the code lives at:

**[github.com/petkarsanathraj/react-native-usb-serial-android](https://github.com/petkarsanathraj/react-native-usb-serial-android)**

It is MIT licensed, comes with a runnable example app that has a built-in demo mode (so you can see the UI behave without plugging anything in), and uses the same code I run in production on field devices.

Hook usage:

```tsx
import { useUsbSerial } from "react-native-usb-serial-android";

export function Reader() {
  const { connected, status, connect, startReading } = useUsbSerial({
    onData: (chunk) => console.log("USB:", chunk),
  });

  return (
    <Button
      title="Connect"
      onPress={async () => {
        await connect({ baudRate: 9600 });
        startReading();
      }}
    />
  );
}
```

That is the entire surface area you usually need.

---

If you are working on something hardware-adjacent in React Native and the available libraries are not cutting it — or you need someone to write the Kotlin/Swift bridge for you — [get in touch](/#contact). This is the kind of work I do for a living.
