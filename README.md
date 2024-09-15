# Example of Using Riverpod with Flutter Map

---

### **1. Add Riverpod to the project**

In your `pubspec.yaml` file, add the dependency:

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_map: ^3.1.0
  location: ^4.4.0
  http: ^0.13.4
  latlong2: ^0.8.1
  flutter_riverpod: ^2.0.0
```

Then run:

```bash
flutter pub get
```

---

### **2. Create providers to manage the state**

#### **a) Provider for the current location**

```dart
final locationProvider = FutureProvider<LatLng?>((ref) async {
  Location location = Location();

  bool serviceEnabled;
  PermissionStatus permissionGranted;
  LocationData locationData;

  // Check if location services are enabled
  serviceEnabled = await location.serviceEnabled();
  if (!serviceEnabled) {
    serviceEnabled = await location.requestService();
    if (!serviceEnabled) {
      return null;
    }
  }

  // Check permissions
  permissionGranted = await location.hasPermission();
  if (permissionGranted == PermissionStatus.denied) {
    permissionGranted = await location.requestPermission();
    if (permissionGranted != PermissionStatus.granted) {
      return null;
    }
  }

  // Get location
  locationData = await location.getLocation();

  return LatLng(locationData.latitude!, locationData.longitude!);
});
```

#### **b) Provider for markers**

Since we want markers to update when the map position changes, we'll use a `StateNotifierProvider` to manage the state of markers.

```dart
class MarkerNotifier extends StateNotifier<List<Marker>> {
  MarkerNotifier() : super([]);

  Future<void> fetchMarkers(LatLng position) async {
    final response = await http.get(Uri.parse(
        'https://example.com/api/markers?lat=${position.latitude}&lng=${position.longitude}'));

    if (response.statusCode == 200) {
      List<dynamic> data = jsonDecode(response.body);
      List<Marker> newMarkers = data.map((item) {
        return Marker(
          point: LatLng(item['latitude'], item['longitude']),
          width: 80,
          height: 80,
          builder: (ctx) => Icon(Icons.location_on, color: Colors.red),
        );
      }).toList();

      state = newMarkers;
    } else {
      throw Exception('Failed to load markers');
    }
  }
}

final markerProvider = StateNotifierProvider<MarkerNotifier, List<Marker>>((ref) {
  return MarkerNotifier();
});
```

---

### **3. Update `main.dart`**

#### **a) Wrap the app with `ProviderScope`**

```dart
void main() {
  runApp(ProviderScope(child: MyApp()));
}
```

#### **b) Update `MyHomePage` to be a `ConsumerStatefulWidget`**

This allows easy access to the providers.

```dart
class MyHomePage extends ConsumerStatefulWidget {
  MyHomePage({Key? key}) : super(key: key);

  @override
  _MyHomePageState createState() => _MyHomePageState();
}
```

#### **c) Handle initial fetching of location and markers**

```dart
class _MyHomePageState extends ConsumerState<MyHomePage> {
  MapController _mapController = MapController();

  @override
  void initState() {
    super.initState();

    // Listen to changes in the map controller
    _mapController.mapEventStream.listen((event) {
      if (event is MapEventMoveEnd) {
        // Fetch new markers for the current center of the map
        ref.read(markerProvider.notifier).fetchMarkers(event.center);
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    final locationAsyncValue = ref.watch(locationProvider);

    return Scaffold(
      appBar: AppBar(
        title: Text('Map with Markers'),
      ),
      body: locationAsyncValue.when(
        data: (userLocation) {
          if (userLocation == null) {
            return Center(child: Text('Failed to get location.'));
          }

          // Get markers
          final markers = ref.watch(markerProvider);

          return FlutterMap(
            mapController: _mapController,
            options: MapOptions(
              center: userLocation,
              zoom: 13.0,
              onMapReady: () {
                // Fetch markers for the initial location
                ref.read(markerProvider.notifier).fetchMarkers(userLocation);
              },
            ),
            children: [
              TileLayer(
                urlTemplate:
                    "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
                subdomains: ['a', 'b', 'c'],
              ),
              MarkerLayer(
                markers: [
                  Marker(
                    point: userLocation,
                    width: 80,
                    height: 80,
                    builder: (ctx) => Icon(Icons.my_location, color: Colors.blue),
                  ),
                  ...markers,
                ],
              ),
            ],
          );
        },
        loading: () => Center(child: CircularProgressIndicator()),
        error: (err, stack) => Center(child: Text('Error: $err')),
      ),
    );
  }
}
```

---

### **4. Handle marker updates when the map position changes**

As shown in the code above, we added a listener to the map controller in `initState()`:

```dart
_mapController.mapEventStream.listen((event) {
  if (event is MapEventMoveEnd) {
    // Fetch new markers for the current center of the map
    ref.read(markerProvider.notifier).fetchMarkers(event.center);
  }
});
```

This ensures that every time the user pans or zooms the map, upon completion of the movement (`MapEventMoveEnd`), a new list of markers is fetched for the new center of the map.

---

### **Complete `main.dart` code**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_map/flutter_map.dart';
import 'package:location/location.dart';
import 'package:http/http.dart' as http;
import 'dart:convert';
import 'package:latlong2/latlong.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() {
  runApp(ProviderScope(child: MyApp()));
}

class MyApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Map Demo with Riverpod',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: MyHomePage(),
    );
  }
}

// Provider for the current location
final locationProvider = FutureProvider<LatLng?>((ref) async {
  Location location = Location();

  bool serviceEnabled;
  PermissionStatus permissionGranted;
  LocationData locationData;

  // Check if location services are enabled
  serviceEnabled = await location.serviceEnabled();
  if (!serviceEnabled) {
    serviceEnabled = await location.requestService();
    if (!serviceEnabled) {
      return null;
    }
  }

  // Check permissions
  permissionGranted = await location.hasPermission();
  if (permissionGranted == PermissionStatus.denied) {
    permissionGranted = await location.requestPermission();
    if (permissionGranted != PermissionStatus.granted) {
      return null;
    }
  }

  // Get location
  locationData = await location.getLocation();

  return LatLng(locationData.latitude!, locationData.longitude!);
});

// Notifier for markers
class MarkerNotifier extends StateNotifier<List<Marker>> {
  MarkerNotifier() : super([]);

  Future<void> fetchMarkers(LatLng position) async {
    final response = await http.get(Uri.parse(
        'https://example.com/api/markers?lat=${position.latitude}&lng=${position.longitude}'));

    if (response.statusCode == 200) {
      List<dynamic> data = jsonDecode(response.body);
      List<Marker> newMarkers = data.map((item) {
        return Marker(
          point: LatLng(item['latitude'], item['longitude']),
          width: 80,
          height: 80,
          builder: (ctx) => Icon(Icons.location_on, color: Colors.red),
        );
      }).toList();

      state = newMarkers;
    } else {
      throw Exception('Failed to load markers');
    }
  }
}

final markerProvider = StateNotifierProvider<MarkerNotifier, List<Marker>>((ref) {
  return MarkerNotifier();
});

class MyHomePage extends ConsumerStatefulWidget {
  MyHomePage({Key? key}) : super(key: key);

  @override
  _MyHomePageState createState() => _MyHomePageState();
}

class _MyHomePageState extends ConsumerState<MyHomePage> {
  MapController _mapController = MapController();

  @override
  void initState() {
    super.initState();

    // Listen to changes in the map controller
    _mapController.mapEventStream.listen((event) {
      if (event is MapEventMoveEnd) {
        // Fetch new markers for the current center of the map
        ref.read(markerProvider.notifier).fetchMarkers(event.center);
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    final locationAsyncValue = ref.watch(locationProvider);

    return Scaffold(
      appBar: AppBar(
        title: Text('Map with Markers'),
      ),
      body: locationAsyncValue.when(
        data: (userLocation) {
          if (userLocation == null) {
            return Center(child: Text('Failed to get location.'));
          }

          // Get markers
          final markers = ref.watch(markerProvider);

          return FlutterMap(
            mapController: _mapController,
            options: MapOptions(
              center: userLocation,
              zoom: 13.0,
              onMapReady: () {
                // Fetch markers for the initial location
                ref.read(markerProvider.notifier).fetchMarkers(userLocation);
              },
            ),
            children: [
              TileLayer(
                urlTemplate:
                    "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
                subdomains: ['a', 'b', 'c'],
              ),
              MarkerLayer(
                markers: [
                  Marker(
                    point: userLocation,
                    width: 80,
                    height: 80,
                    builder: (ctx) =>
                        Icon(Icons.my_location, color: Colors.blue),
                  ),
                  ...markers,
                ],
              ),
            ],
          );
        },
        loading: () => Center(child: CircularProgressIndicator()),
        error: (err, stack) => Center(child: Text('Error: $err')),
      ),
    );
  }
}
```

---

### **Additional Notes:**

- **REST API Endpoint:**

  - Update the API URL in the `fetchMarkers` function, replacing `https://example.com/api/markers` with your actual API endpoint.
  - Ensure your API supports `lat` and `lng` query parameters.
- **Error Handling:**

  - You can add additional error handling, such as displaying messages when markers fail to load.
- **Optimization:**

  - To avoid making too many API requests during rapid map movements, consider adding debouncing or throttling to the map position change listener.
- **Platform Permissions:**

  - **Android:** Add permissions to `android/app/src/main/AndroidManifest.xml`:

    ```xml
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    ```
  - **iOS:** Add permissions to `ios/Runner/Info.plist`:

    ```xml
    <key>NSLocationWhenInUseUsageDescription</key>
    <string>We need access to your location to display it on the map.</string>
    ```

---

### **Summary**

We modified the application to:

- Use **Riverpod** to manage the state of the location and markers.
- Send requests to the API with the user's current location.
- Update the list of markers whenever the map position changes.

By leveraging Riverpod, managing the application's state becomes more transparent and scalable.
