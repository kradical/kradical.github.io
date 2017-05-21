---
layout: post
title: iOS Locations
---

Everybody has a highly accurate tracking device on themselves at almost all times. To take advantage of this many organizations are prioritizing location data, and location based applications. The application I am currently working on attempts to make business expenses simple and less time consuming by using location data to track possible expenses and prefill sensible values. This post will walk through some simple use cases and have example code for how to do locations in swift (3.1) on iOS. I will also attempt to clarify things and present gotcha's that I have encountered.

I try to separate out location functionality from my app delegate and the rest of my app. A stripped down example of the class I would use:

```swift
import UIKit
import CoreLocation

class LocationManager: NSObject, CLLocationManagerDelegate {
    static let shared = LocationManager()
    private override init() {}

    private let locationManager = CLLocationManager()
    var location: CLLocation? { return self.locationManager.location }

    func initLocationManager() {
        self.locationManager.delegate = self

        self.locationManager.requestAlwaysAuthorization()
        self.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        print("didUpdateLocations \(locations)")
    }

    func locationManager(_ manager: CLLocationManager, didChangeAuthorization status: CLAuthorizationStatus) {
        if status == .authorizedAlways {
            self.initLocationManager()
        } else {
            self.locationManager.requestAlwaysAuthorization()
        }
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        print("locationManager error: \(error)", withLevel: .warning)
    }

    func stopUpdatingLocation() {
        self.locationManager.stopUpdatingLocation()
    }

    func startUpdatingLocation() {
        self.locationManager.startUpdatingLocation()
    }
}
```

TODO go over authorization plist keys and functions
TODO go over filter options
TODO go over accuracy options
TODO show a custom filter
TODO do an example of a common use