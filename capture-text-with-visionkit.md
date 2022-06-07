# Capture machine-readable codes and text with VisionKit

Presenter:
- Ron Santos, Input Engineer

Prior to iOS 16, you could use AVFoundation + Vision for data scanning. In iOS 16, we have a new option: the `DataScannerViewController` and the VisionKit framework.

User-facing features include:

- Live camera preview
- Helpful guidance
- Item highlighting
- Tap-to-focus
- Pinch-to-zoom

Developer features include:

- It's a `UIViewController` subclass
- View coordinates
- Region-of-interest
- Text Content Types
- Multiple symbologies

Supported hardware
- Any 2018 or newer iPhone or iPad with a Neural Engine

## To add to your app

### Privacy Usage Description

- iOS asks the user to ask for explicit permission to access the camera. Add a descriptive reason.

1. Go to .plist
2. Look for or add a `Privacy - Camera Usage Description`
3. Input a useful description in the value

### Add to app

- `import VisionKit`
- Use `DataScannerViewController.isSupported` to hide buttons and functionality on devices that do not support it
- Use `DataScannerViewController.isAvailable` to check for availability - i.e. if the user approves the app for camera access & the device is free of restrictions

#### Configure an Instance

To configure an instance:

- Specify the type of data you want to scan - i.e. barcode or text, URL, etc.
- Specify the language you expect - i.e. English - or don't specify. If you don't specify, the user's preferred language is used
- When you present it, present it like any other view controller

```swift
import VisionKit

// Specify the types of data to recognize
let recognizedDataTypes:Set<DataScannerViewController.RecognizedDataType> = [
    .barcode(symbologies: [.qr]),
  	// uncomment to filter on specific languages (e.g., Japanese)
    // .text(languages: ["ja"])
    // uncomment to filter on specific content types (e.g., URLs)
		// .text(textContentType: .URL)
]

// Create the data scanner, present it, and start scanning!
let dataScanner = DataScannerViewController(recognizedDataTypes: recognizedDataTypes)
present(dataScanner, animated: true) {
    try? dataScanner.startScanning()
}
```

#### Configuration Parameters for the DataScannerViewController initializer

- recognizedDataTypes: Text and/or codes and what types of each
- qualityLevel: Balanced, fast, or accurate
- recognizesMultipleItems: Find one item at a time or several
- isHighFrameRateTrackingEnabled: Enable when you want highlights to follow items as closely as possible
- isPinchToZoomEnabled: Allow pinch to zoom
- isGuidanceEnabled: Allow for labels to help the user
- isHighlightingEnabled: Enable system highlighting or draw your own custom highlights

## Detailed Code Examples

### Set a delegate

```swift
// Specify the types of data to recognize
let recognizedDataTypes:Set<DataScannerViewController.RecognizedDataType> = [
    .barcode(symbologies: [.qr]),
    .text(textContentType: .URL)
]

// Create the data scanner, present it, and start scanning!
let dataScanner = DataScannerViewController(recognizedDataTypes: recognizedDataTypes)
dataScanner.delegate = self.present(dataScanner, animated: true) {
    try? dataScanner.startScanning()
}
```

### Handle tap interactions

```swift
func dataScanner(_ dataScanner: DataScannerViewController, didTapOn item: RecognizedItem) {
    switch item {
    case .text(let text):
        print("text: \(text.transcript)")
    case .barcode(let barcode):
        print("barcode: \(barcode.payloadStringValue ?? "unknown")")
    default:
        print("unexpected item")
    }
}
```

- Each `RecognizedItem` has a unique identifier you can use to track it throughout its lifecycle
- Each `RecognizedItem` has a Bounds property, four points, one for each corner

#### Delegates for managing highlighting

Called when recognized items in the scene change

##### `didAdd`: called when items in the scene are newly recognized

```swift
// Dictionary to store our custom highlights keyed by their associated item ID.
var itemHighlightViews: [RecognizedItem.ID: HighlightView] = [:]

// For each new item, create a new highlight view and add it to the view hierarchy.
func dataScanner(_ dataScanner: DataScannerViewController, didAdd addItems: [RecognizedItem], allItems: [RecognizedItem]) {
    for item in addedItems {
        let newView = newHighlightView(forItem: item)
        itemHighlightViews[item.id] = newView
        dataScanner.overlayContainerView.addSubview(newView)
    }
}
```

- Create custom highlights here
- Keep track of highlights using the ID of its associated item
- When adding new views, add them to `overlayContainerView` - they appear above the camera's preview, but below any supplemental chrome

##### `didUpdate`: called when the camera moves, or transcription for recognized text changes

```swift
// Animate highlight views to their new bounds
func dataScanner(_ dataScanner: DataScannerViewController, didUpdate updatedItems: [RecognizedItem], allItems: [RecognizedItem]) {
    for item in updatedItems {
        if let view = itemHighlightViews[item.id] {
            animate(view: view, toNewBounds: item.bounds)
        }
    }
}
```

##### `didRemove`: called when items are removed

```swift
// Remove highlights when their associated items are removed.
func dataScanner(_ dataScanner: DataScannerViewController, didRemove removedItems: [RecognizedItem], allItems: [RecognizedItem]) {
    for item in removedItems {
        if let view = itemHighlightViews[item.id] {
            itemHighlightViews.removeValue(forKey: item.id)
            view.removeFromSuperview()
        }
    }
}
```

### Capturing photos

Asynchronously returns a high-quality UI image

```swift
// Take a still photo and save to the camera roll
if let image = try? await dataScanner.capturePhoto() {
    UIImageWriteToSavedPhotosAlbum(image, nil, nil, nil)
}
```

### If not using custom highlighting

recognizedItem property - async stream continuously updated as the scene changes

```swift
// Send a notification when the recognized items change.
var currentItems: [RecognizedItem] = []

func updateViaAsyncStream() async {
    guard let scanner = dataScannerViewController else { return }
    
    let stream = scanner.recognizedItems
    for await newItems: [RecognizedItem] in stream {
        let diff = newItems.difference(from: currentItems) { a, b in
            return a.id == b.id
        }
        
        if !diff.isEmpty {
            currentItems = newItems
            sendDidChangeNotification()
        }
    }
}
```

## Related Sessions

- Extract document data using Vision: WWDC21
