# Bluejay

Bluejay is a simple Swift framework for building reliable Bluetooth apps.

Bluejay's primary goals are:
- Simplify talking to a Bluetooth device
- Make good use of Swift features and conventions
- Make it easier to build Bluetooth apps that are actually reliable

## Requirements

- iOS 10 or above
- Xcode 8.2.1 or above

## Getting started

Install using CocoaPods:

`pod 'Bluejay', :git => 'https://github.com/steamclock/bluejay.git', :branch => 'master'`

Import using:

`import Bluejay`

## Demo

Simulator does not work without a BLE dongle.

1. Prepare two iOS devices, one will act as a virtual BLE peripheral, and the other will run the demo app using the Bluejay API.
2. On the iOS device serving as a virtual BLE peripheral, go to the App Store and download the free [LightBlue Explorer](https://itunes.apple.com/ca/app/lightblue-explorer-bluetooth/id557428110?mt=8) app.
3. Open the LightBlue Explorer app, and tap on the "Create Virtual Peripheral" button located at the very bottom of the peripheral list.
4. For simplicity, choose "Heart Rate" from the base profile list, and finish by tapping the "Save" button located at the top right of the screen.
5. Finally, build and run the BluejayDemo app on the other iOS device, and you will be able to interact with the virtual heart rate peripheral using Bluejay.

Notes:

- You can turn the virtual peripheral on or off in LightBlue Explorer by tapping and toggling the blue checkmark to the left of the virtual peripheral's name in the peripheral list.
	- If the virtual peripheral is not working as expected, you can try to reset it this way.
- By default, the demo app will read and write to the "Body Sensor Location" characteristic, and listen to the "Heart Rate Measurement" characteristic.
	- The heart rate measurement returns only zeroes by default, because it is a virtual peripheral without an actual heart rate detector.
- You can use LightBlue Explorer to do CRUD on various characteristics in the virtual peripheral.

## Usage

The Bluejay interface can be accessed through using the Bluejay singleton:

`fileprivate let bluejay = Bluejay.shared`

### Initialization

Turn on Bluejay at the appropriate time during initialization. When it is appropriate depends on the context and requirement of your app. For example, in the demo app Bluejay is powered on inside `viewDidLoad` of the only existing view controller.

`bluejay.powerOn(withObserver: self, andListenRestorable: self)`

Having to explicitly power on is important because this gives your app an opportunity to make sure that the two critical delegates are instantiated and available before the CoreBluetooth stack is initialized. This will ensure CoreBluetooth's startup and restoration events are being handled.

### Bluetooth Events

The `observer` conforms to the `EventsObservable` protocol, and allows the delegate to react to major connection-related events:

```
public protocol EventsObservable: class {
    func bluetoothAvailable(_ available: Bool)
    func connected(_ peripheral: Peripheral)
    func disconected()
}
```

### Listen Restoration

The `ListenRestorable` is a protocol allowing the restoration of previously active listens should CoreBluetooth decide that a state restoration is necessary.

```
public protocol ListenRestorable: class {
    func didFindRestorableListen(on characteristic: CharacteristicIdentifier) -> Bool
}
```

By default, if there is no `ListenRestorable` delegate available, or if the protocol function returns false, any previously active listens will be cancelled when state restoration occurs.

If the function returns true, then the provided characteristic with a previously active listen must be restored using the `restoreListen` function:

```
extension ViewController: ListenRestorable {
    
    func didFindRestorableListen(on characteristic: CharacteristicIdentifier) -> Bool {
        if characteristic == heartRate {
            bluejay.restoreListen(to: heartRate, completion: { (result: ReadResult<UInt8>) in
                switch result {
                case .success(let value):
                    log.debug("Listen succeeded with value: \(value)")
                case .failure(let error):
                    log.debug("Listen failed with error: \(error.localizedDescription)")
                }
            })
            
            return true
        }
        
        return false
    }
    
}
```

### Scan & Connect

Currently, a successful scan of a peripheral will result in the immediate connection to it. We have plans to separate these two operations and create a dedicated scanning API.

```
bluejay.scan(service: heartRateService) { (result) in
    switch result {
    case .success(let peripheral):
        log.debug("Scan succeeded with peripheral: \(peripheral.name)")
    case .failure(let error):
        log.debug("Scan failed with error: \(error.localizedDescription)")
    }
}
```

### Disconnect

`bluejay.disconnect()`

### Read

```
bluejay.read(from: bodySensorLocation) { (result: ReadResult<IncomingString>) in
    switch result {
    case .success(let value):
        log.debug("Read succeeded with value: \(value.string)")
    case .failure(let error):
        log.debug("Read failed with error: \(error.localizedDescription)")
    }
}
```

### Write

```
bluejay.write(to: bodySensorLocation, value: OutgoingString("Wrist")) { (result) in
    switch result {
    case .success:
        log.debug("Write succeeded.")
    case .failure(let error):
        log.debug("Write failed with error: \(error.localizedDescription)")
    }
}
```

### Listen

```
bluejay.listen(to: heartRate) { (result: ReadResult<UInt8>) in
    switch result {
    case .success(let value):
        log.debug("Listen succeeded with value: \(value)")
    case .failure(let error):
        log.debug("Listen failed with error: \(error.localizedDescription)")
    }
}
```

### Cancel Listen

`bluejay.cancelListen(to: heartRate)`

### Receivable & Sendable

The `Receivable` and `Sendable` protocols provide the blueprints to model the data packets being exchanged between the your app and the BLE peripheral, are are mandatory to type the results and the deliverables when using the `read` and `write` functions.

Examples:

```
struct IncomingString: Receivable {
    
    var string: String
    
    init(bluetoothData: Data) {
        string = String(data: bluetoothData, encoding: .utf8)!
    }
    
}

struct OutgoingString: Sendable {
    
    var string: String
    
    init(_ string: String) {
        self.string = string
    }
    
    func toBluetoothData() -> Data {
        return string.data(using: .utf8)!
    }
    
}
```

## Documentaion

- Link out or straight-up include the API docs