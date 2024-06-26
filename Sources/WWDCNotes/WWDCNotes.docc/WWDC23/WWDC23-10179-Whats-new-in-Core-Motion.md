# What’s new in Core Motion

Learn how you can use the latest Core Motion updates to expand how your app uses motion data. Discover how to stream higher-frequency sensor data when recording a HealthKit workout on Apple Watch. We’ll show you how you can get submersion data — including water depth and temperature — during water-based activities like snorkeling. Find out how to stream motion data like attitude, user acceleration, and rotation rate from audio devices like AirPods to connected devices like iPhone and Mac.

@Metadata {
   @TitleHeading("WWDC23")
   @PageKind(sampleCode)
   @CallToAction(url: "https://developer.apple.com/wwdc23/10179", purpose: link, label: "Watch Video (23 min)")

   @Contributors {
      @GitHubUser(RamitSharma991)
   }
}



# Core Motion Overview 

* CoreMotion serves as a central framework to access motion data from inertial sensors.
* Crash detection, fall detection, and spatial audio are just some of the features that rely on improved sensing capabilities.
* Capturing the way a device moves is central to how we experience them. 
* Many of Apple's devices use built-in sensors to help create a notion of their movement through space.
* Apple Watch for example has built-in sensors include: 
  * an accelerometer, which measures acceleration 
  * a gyroscope, which measures rotation
  * a magnetometer, which measures the magnetic field
  * a barometer, which measures pressure. 
  * Together, they help track how the device moves and orients in space.
* Generating an idea of a device's movement is fundamental like steps taken, calories burned.
* It supports experiences that rely on the orientation of the device, like with a stargazing app

## Headphone motion

* Dynamic head tracking relies on the same device motion algorithms that live on iPhone and Apple Watch.
* `CMHeadphoneMotionManager`:
   * Introduced a couple years ago
   * Provided the same data that made dynamic head tracking possible 
   * Head tracking unlocked features like gaming to fitness applications.
   * coming to macOS 14 and can be used to stream device motion from audio products that support spatial audio with dynamic head tracking.
* Use `CMDeviceMotion` for data
* `CMDeviceMotion.SensorLocation` for sensor location
* `CMHeadphoneMotionManagerDelegate` for connection state

### Headphone motion events 

- Adopt the `CMHeadphoneMotionManagerDelegate protocol` to respond to connection state updates. 
- If Automatic Ear Detection is enabled we receive events that impact head tracking.
- If the buds are taken out of ear we get a disconnect event and a connect event when they're put back in.

```swift

func headphoneMotionManagerDidConnect(_ manager: CMHeadphoneMotionManger) {
	//	Tracking started with connect event
}

func headphoneMotionManagerDidDisConnect(_ manager: CMHeadphoneMotionManger) {
	//	Tracking stopped with connect event
}

```

- if automatic head detection is enabled, putting on and taking off over ear headphones will trigger these events.
- Setting up `CMHeadphoneMotionManager`: 
    - make sure that device motion data is available by checking the `isDeviceMotionAvailable` property
    - Assign a delegate to receive the connection events
    - Then, start streaming data. 
    - `CMHeadphoneMotionManager` exposes both a push and a pull interface to grab data.
    - Use `startDeviceMotionUpdates` and specify an operation queue and handler.
    - Users need to authorize your app for motion data using the Motion Usage Description key you add to your Info.plist. 
    - can check the `authorizationStatus` property to confirm whether you've been authorized for motion data
    - Once authorized and data starts streaming, tracking head pose is easy using the attitude information provided with each device motion update. 

```swift

// Start streaming headphone motion
let manager = CMHeadphoneMotionManager()
guard manger.isDeviceMotionAvailable else {
	return
}
manager.delegate = self

manager.startDeviceMotionUpdates(to: queue) { (motion, error) in
	guard let motion else {
		return
	}
//Track head movement using device motion
let currentPose = motion.attitude
  if let startingPose {
	  currentPose.Multiply(byInverseOf: startingPoint)
	}
}

```

- Along with the attitude, user acceleration, and rotation rate data, each device motion update contains sensor location information.
- motion data is delivered to you from one bud at a time. 

### Sensor location
- Use SensorLocation to Identify which bug produced the data 
- The bud streaming the data can be impacted by a number of things, including in-ear state if Automatic Ear Detection is enabled. 

```swift

	public enum SensorLocation: Int {
	
	case ‘default’ = 0
	case headphoneLeft = 1
	case headphoneRight = 2
}

```
## Submersion

**CMWaterSubmersionManager:**

- Available on Apple Watch Ultra running watchOS 9.0
- Uses the built-in barometer to tracks metrics during water-based activities.
- Further CMWaterSubmersionManagerDelegate is used for:
  - Submersion and depth state
  - Surface air pressure
  - Water Depth
  - Temperature 
  - Add the Shallow Depth and Pressure capability 
  - Configure Auto Launch settings

```swift

// start tracking water submersion state
guard CMWaterSubmersionManager.waterSubmersionAvailable else {
	return
}

let waterSubmersionManager = CMWaterSubmersionManager()

//Assign a delegate to recieve events and measurement updates
waterSubmersionManager.delegate = self 

```

***Getting submersion updates with CMWaterSubmersionManagerDelegate***

- *Submersion event*

```swift

func(_ manager: CMWaterSubmersionManager, didUpdate event: CMWaterSubmersionEvent) {

	var submerged: Bool ?
	switch event.state {
	case .unknown:
		submerged = nil
	case .notSubmerged:
		submerged = false
	case .submerged:
		submerged = true
	@unknown default:
		fatalError(“Unknown submersion event: \(event.state)”)
	}
	// Handle water submersion update
}

```
- *Error*

```swift

func manager(_ manager: CMWaterSubmersionManager, errorOccurred error: Error) {
	// Handle error received when there are problems delivering submersion updates
}

```
- *Temperature update*

```swift

func manager(_ manager: CMWaterSubmersionManager, didUpdate measurement: CMWaterTemperature) {

	let temp = measurement.temperature
	let uncertainty = measurement.temperatureUncertainty
	let currentTemperature = “\(temp.value) * \(uncertainty.value) \(temp.unit)” 
	// Handle current water temperature update
}

```
- *Measurement update*


```swift

func manager(_ manager: CMWaterSubmersionManager, didUpdate measurement: CMWaterSubmersion) {
	var current Depth: String
	if let depth = measurement.dept {
		currentDepth = “\(depth.value) \(depth.unit)”
	} else {
		currentDepth = “None”
	}
	// Handle new measurement update
	// Similar to depth, pressure and surface contain both a value and unit
}
```

## Depth State

- Out of water, its the `notSubmerged` state.
- Above 1 meter under water its `submergedShallow` state. 
- Beyond 1 meter, its the `submergedDeep` state.
- Shallow Depth and Pressure, ensures the app to stay within depth zones that minimize the risk of decompression sickness.
- It keeps the maximum depth at 6 meters, and prompts when you’re close to that.
- 6 meters, you'll enter the `approachingMaxDepth` state.
- Beyond 6 meters, you're in the `pastMaxDepth` state. 
- Data is vended down to 6 meters, plus some uncertainty in the `pastMaxDepth` state. 
- Beyond that, you're in the `sensorDepthError` state.


## Batched sensors

### Comparing motion interface:

### How sensor data is delivered

- Device motion algorithms fuse data from the built-in accelerometer and gyroscope to provide an easy way to track the way a device, moving through space. 
- CMMotionManager delivers samples on a per-sample basis to your app in real time.
- The maximum supported frequency is 100 Hz hence it's a great choice for low latency requirements, like UI components that rely on the instantaneous attitude of the device.

### how high rate data is delivered using the new CMBatchedSensorManager

- CMBatchedSensorManager provides batches of sensor data on a fixed schedule, delivering a batch of data per second. 
- Higher rate data is delivered at a lower overhead to your app.
- That's 800 Hz accelerometer and 200 Hz device motion, compared to 100 Hz with the existing CMMotionManager.
- Access some data streams that power the features that keep us safe, like fall and crash detection.
- If your app has workout-centric features that can benefit from high rate data, but without very tight latency requirements, then CMBatchedSensorManager is well suited.


### Using high- rate motion data

- Evaluating a swing:
    - A swing has a couple different phases.
    - Pre-swing setup, the actual swing and then the post-impact follow-through.
    - time to contact is also an important metric for swing quality. 
    - Δt = time to contact, from (start of swing to Impact)
- First detect the point of impact between the bat and the ball using 800 Hz accelerometer.
- Then detect the start of the swing using rotation along gravity with 200 Hz device motion.
- Compute time to contact.
- Then use CMBatchedSensorManager to start streaming and processing data.
- this a workout-centric API, you need to have an active HealthKit workout session to get data.
- Swift async support, it's easy to receive batches of sensor data and process each batch. 
- Make sure you evaluate for conditions to exit the loop.

**Pick the right interface for you**

- `CMMotionManager` or `CMBatchedSensorManager`
- 100 Hz maximum or 200 Hz device motion and 800 Hz accelerometer 
- Per-sample dispatch or batched delivery schedule
- Low-latency requirements/Motion-based features outside of workouts or workout-centric features that can benefit from high-rate data. It's available on Apple Watch Series 8 and Ultra.
