API otrzymuje aktualną lokalizację użytkownika i zwraca listę markerów.

- Przy zmianie położenia mapy aplikacja wysyła nowe zapytanie do API z nową lokalizacją, aby otrzymać aktualną listę markerów.

### **Kroki:**

1. **Dodaj Riverpod do projektu.**
2. **Utwórz providery do zarządzania stanem lokalizacji i markerów.**
3. **Zaktualizuj widoki, aby korzystały z Riverpod.**
4. **Obsłuż aktualizację markerów przy zmianie pozycji mapy.**

---

### **1. Dodaj Riverpod do projektu**

W pliku `pubspec.yaml` dodaj zależność:

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

Następnie uruchom:

```bash
flutter pub get
```

---

### **2. Utwórz providery do zarządzania stanem**

#### **a) Provider dla aktualnej lokalizacji**

```dart
final locationProvider = FutureProvider<LatLng?>((ref) async {
  Location location = Location();

  bool serviceEnabled;
  PermissionStatus permissionGranted;
  LocationData locationData;

  // Sprawdź, czy usługa lokalizacji jest włączona
  serviceEnabled = await location.serviceEnabled();
  if (!serviceEnabled) {
    serviceEnabled = await location.requestService();
    if (!serviceEnabled) {
      return null;
    }
  }

  // Sprawdź uprawnienia
  permissionGranted = await location.hasPermission();
  if (permissionGranted == PermissionStatus.denied) {
    permissionGranted = await location.requestPermission();
    if (permissionGranted != PermissionStatus.granted) {
      return null;
    }
  }

  // Pobierz lokalizację
  locationData = await location.getLocation();

  return LatLng(locationData.latitude!, locationData.longitude!);
});
```

#### **b) Provider dla markerów**

Ponieważ chcemy, aby markery były aktualizowane przy zmianie pozycji mapy, użyjemy `StateNotifierProvider` do zarządzania stanem markerów.

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

### **3. Zaktualizuj `main.dart`**

#### **a) Owiń aplikację w `ProviderScope`**

```dart
void main() {
  runApp(ProviderScope(child: MyApp()));
}
```

#### **b) Zaktualizuj `MyHomePage` do `ConsumerStatefulWidget`**

Pozwoli to na łatwy dostęp do providerów.

```dart
class MyHomePage extends ConsumerStatefulWidget {
  MyHomePage({Key? key}) : super(key: key);

  @override
  _MyHomePageState createState() => _MyHomePageState();
}
```

#### **c) Obsłuż inicjalne pobieranie lokalizacji i markerów**

```dart
class _MyHomePageState extends ConsumerState<MyHomePage> {
  MapController _mapController = MapController();

  @override
  void initState() {
    super.initState();

    // Nasłuchuj na zmiany w kontrolerze mapy
    _mapController.mapEventStream.listen((event) {
      if (event is MapEventMoveEnd) {
        // Pobierz nową listę markerów dla aktualnego centrum mapy
        ref.read(markerProvider.notifier).fetchMarkers(event.center);
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    final locationAsyncValue = ref.watch(locationProvider);

    return Scaffold(
      appBar: AppBar(
        title: Text('Mapa z Markerami'),
      ),
      body: locationAsyncValue.when(
        data: (userLocation) {
          if (userLocation == null) {
            return Center(child: Text('Nie udało się uzyskać lokalizacji.'));
          }

          // Pobierz markery
          final markers = ref.watch(markerProvider);

          return FlutterMap(
            mapController: _mapController,
            options: MapOptions(
              center: userLocation,
              zoom: 13.0,
              onMapReady: () {
                // Pobierz markery dla początkowej lokalizacji
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
        error: (err, stack) => Center(child: Text('Wystąpił błąd: $err')),
      ),
    );
  }
}
```

---

### **4. Obsłuż aktualizację markerów przy zmianie pozycji mapy**

Już w kodzie powyżej dodaliśmy nasłuchiwanie na zdarzenia mapy w `initState()`:

```dart
_mapController.mapEventStream.listen((event) {
  if (event is MapEventMoveEnd) {
    // Pobierz nową listę markerów dla aktualnego centrum mapy
    ref.read(markerProvider.notifier).fetchMarkers(event.center);
  }
});
```

To zapewnia, że za każdym razem, gdy użytkownik przesunie lub zzoomuje mapę, po zakończeniu ruchu (`MapEventMoveEnd`) zostanie pobrana nowa lista markerów dla nowego centrum mapy.

---

### **Kompletny kod `main.dart`**

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
  // Ten widget jest korzeniem Twojej aplikacji.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Map Demo z Riverpod',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: MyHomePage(),
    );
  }
}

// Provider dla aktualnej lokalizacji
final locationProvider = FutureProvider<LatLng?>((ref) async {
  Location location = Location();

  bool serviceEnabled;
  PermissionStatus permissionGranted;
  LocationData locationData;

  // Sprawdź, czy usługa lokalizacji jest włączona
  serviceEnabled = await location.serviceEnabled();
  if (!serviceEnabled) {
    serviceEnabled = await location.requestService();
    if (!serviceEnabled) {
      return null;
    }
  }

  // Sprawdź uprawnienia
  permissionGranted = await location.hasPermission();
  if (permissionGranted == PermissionStatus.denied) {
    permissionGranted = await location.requestPermission();
    if (permissionGranted != PermissionStatus.granted) {
      return null;
    }
  }

  // Pobierz lokalizację
  locationData = await location.getLocation();

  return LatLng(locationData.latitude!, locationData.longitude!);
});

// Notifier dla markerów
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

    // Nasłuchuj na zmiany w kontrolerze mapy
    _mapController.mapEventStream.listen((event) {
      if (event is MapEventMoveEnd) {
        // Pobierz nową listę markerów dla aktualnego centrum mapy
        ref.read(markerProvider.notifier).fetchMarkers(event.center);
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    final locationAsyncValue = ref.watch(locationProvider);

    return Scaffold(
      appBar: AppBar(
        title: Text('Mapa z Markerami'),
      ),
      body: locationAsyncValue.when(
        data: (userLocation) {
          if (userLocation == null) {
            return Center(child: Text('Nie udało się uzyskać lokalizacji.'));
          }

          // Pobierz markery
          final markers = ref.watch(markerProvider);

          return FlutterMap(
            mapController: _mapController,
            options: MapOptions(
              center: userLocation,
              zoom: 13.0,
              onMapReady: () {
                // Pobierz markery dla początkowej lokalizacji
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
        error: (err, stack) => Center(child: Text('Wystąpił błąd: $err')),
      ),
    );
  }
}
```

---

### **Dodatkowe uwagi:****Endpoint REST API:**

- - Zaktualizuj URL API w funkcji `fetchMarkers`, zastępując `https://example.com/api/markers` rzeczywistym adresem.
  - Upewnij się, że API obsługuje parametry `lat` i `lng` w zapytaniu.
- **Obsługa błędów:**

  - Możesz dodać dodatkową obsługę błędów, np. wyświetlanie komunikatów, gdy nie uda się pobrać markerów.
- **Optymalizacja:**

  - Aby uniknąć zbyt wielu zapytań do API podczas szybkiego przesuwania mapy, możesz dodać debouncing lub throttling do funkcji nasłuchującej zmiany pozycji mapy.
- **Uprawnienia platformowe:**

  - **Android:** Dodaj uprawnienia do `android/app/src/main/AndroidManifest.xml`:

    ```xml
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    ```
  - **iOS:** Dodaj uprawnienia do `ios/Runner/Info.plist`:

    ```xml
    <key>NSLocationWhenInUseUsageDescription</key>
    <string>Potrzebujemy dostępu do Twojej lokalizacji, aby pokazać ją na mapie.</string>
    ```
