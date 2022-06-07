# Hello Swift Charts

Presenter:
- Dominik Moritz, Swift Charts Engineering Manager

Apple's new framework for making informative and delightful charts in SwiftUI

Charts work best when they show additional, useful context around data

Flexible Framework for making Swift charts
Same syntax as SwiftUI

## Make Charts

Build charts by composition
Main components in a bar chart are the Bars (Bar Marks) - visual elements for each item in your data

Food Truck example building individual bars:

```swift
// Remember to import Charts
import Charts
import SwiftUI

struct StylesDetailsChart: View {
    var body: some View {
        // Add a chart
        Chart {
            // Add a BarMark
            BarMark(
                // Set the X value to the name of the pancake
                // First arg is the description of the value, second is the value itself
                x: .value("Name", "Cachapa"),
                // Setting the y value sets the height of the single bar. Set it to sales for this item
                y: .value("Sales", 916)
            )
            // Add a second bar
            BarMark(
                x: .value("Name", "Injera"),
                y: .value("Sales", 850)
            )
        }
    }
}
```

Build a chart for a collection as an array of structs

```swift
import Charts
import SwiftUI

// Create a struct for the collection

struct Pancakes: Identifiable {
    let name: String
    let sales: Int

    var id: String { name }
}

// Define the collection. Could pull from an external source, but creating in code for this example
let sales: [Pancakes] = [
    .init(name: "Cachapa", sales: 916),
    .init(name: "Injera", sales: 850),
    .init(name: "Crepe", sales: 882)
]


struct StylesDetailsChart: View {
    var body: some View {
        Chart {
            // Make the bar chart data-driven with ForEach
            ForEach(sales) { element in
                BarMark(
                    x: .value("Name", element.name),
                    y: .value("Sales", element.sales)
                )
            }
        }
    }
}
```

If ForEach is the only content in the chart, we can put the data directly into the chart initializer:

```swift
import Charts
import SwiftUI

// Create a struct for the collection

struct Pancakes: Identifiable {
    let name: String
    let sales: Int

    var id: String { name }
}

// Define the collection. Could pull from an external source, but creating in code for this example
let sales: [Pancakes] = [
    .init(name: "Cachapa", sales: 916),
    .init(name: "Injera", sales: 850),
    .init(name: "Crepe", sales: 882)
]


struct StylesDetailsChart: View {
    var body: some View {
        Chart(sales) { element in 
            // Put the data directly into the chart initializer
            // Acts like a ForEach
            BarMark(
                x: .value("Name", element.name),
                y: .value("Sales", element.sales)
            )
        }
    }
}
```

When adding more data, the labels may become too close together.
We can transpose the bars to change the orientation of the bars and give the labels more room.

```swift
import Charts
import SwiftUI

// Create a struct for the collection

struct Pancakes: Identifiable {
    let name: String
    let sales: Int

    var id: String { name }
}

// Define the collection. Could pull from an external source, but creating in code for this example
let sales: [Pancakes] = [
    .init(name: "Cachapa", sales: 916),
    .init(name: "Injera", sales: 850),
    .init(name: "Crepe", sales: 882),
    .init(name: "Jian Bing", sales: 753),
    .init(name: "Dosa", sales: 654),
    .init(name: "American", sales: 618)
]


struct StylesDetailsChart: View {
    var body: some View {
        Chart(sales) { element in
            // Transpose sales and name to switch the axis and give name labels more room
            BarMark(
                x: .value("Sales", element.sales),
                y: .value("Name", element.name)
            )
        }
    }
}
```

## Preview Charts in Xcode

- There is a new "Variant" feature that lets you Preview charts in different styles, dark mode vs. light mode, with different fonts/font sizes, etc.

## Accessibility

- Swift Charts exposes data in voiceover
- Voiceover reads the data
- Supports Audiographs

## Time Series

```swift
import Charts
import SwiftUI

struct SalesSummary: Identifiable {
    let weekday: Date
    let sales: Int

    var id: Date { weekday }
}

let cupertinoData: [SalesSummary] = [
    /// Monday
    .init(weekday: date(2022, 5, 2), sales: 54),
    /// Tuesday
    .init(weekday: date(2022, 5, 2), sales: 42),
    /// Wednesday
    .init(weekday: date(2022, 5, 2), sales: 88),
    /// Thursday
    .init(weekday: date(2022, 5, 2), sales: 49),
    /// Friday
    .init(weekday: date(2022, 5, 2), sales: 42),
    /// Saturday
    .init(weekday: date(2022, 5, 2), sales: 125),
    /// Sunday
    .init(weekday: date(2022, 5, 2), sales: 67)
]

struct CupertinoDetailsChart: View {
    var body: some View {
        Chart(cupertinoData) { element in
            BarMark(
                x: .value("Day", element.weekday, unit: .day),
                y: .value("Sales", element.sales)
            )
        }
    }
}
```

Make a chart that toggles between data sets:

```swift
import Charts
import SwiftUI

struct SalesSummary: Identifiable {
    let weekday: Date
    let sales: Int

    var id: Date { weekday }
}

let cupertinoData: [SalesSummary] = [
    /// Monday
    .init(weekday: date(2022, 5, 2), sales: 54),
    /// Tuesday
    .init(weekday: date(2022, 5, 2), sales: 42),
    /// Wednesday
    .init(weekday: date(2022, 5, 2), sales: 88),
    /// Thursday
    .init(weekday: date(2022, 5, 2), sales: 49),
    /// Friday
    .init(weekday: date(2022, 5, 2), sales: 42),
    /// Saturday
    .init(weekday: date(2022, 5, 2), sales: 125),
    /// Sunday
    .init(weekday: date(2022, 5, 2), sales: 67)
]

let sfData: [SalesSummary] = [
    /// Monday
    .init(weekday: date(2022, 5, 2), sales: 81),
    /// Tuesday
    .init(weekday: date(2022, 5, 2), sales: 90),
    /// Wednesday
    .init(weekday: date(2022, 5, 2), sales: 52),
    /// Thursday
    .init(weekday: date(2022, 5, 2), sales: 72),
    /// Friday
    .init(weekday: date(2022, 5, 2), sales: 84),
    /// Saturday
    .init(weekday: date(2022, 5, 2), sales: 84),
    /// Sunday
    .init(weekday: date(2022, 5, 2), sales: 137)
]

struct LocationToggleChart: View {
    // Add a State var to store the city for the toggle
    @State var city: City = .cupertino

    // Add a switch statement to toggle the data
    var data: [SalesSummary] {
        switch city {
            case .cupertino:
                return cupertinoData
            case .sanFrancisco:
                return sfData
        }
    }

    var body: some View {
        // Add a picker to the view
        VStack {
            // Swift Charts supports animation, so we can add it here
            // to animate the toggle transition
            Picker("City", selection: $city.animation(.easeInOut)) {
                Text("Cupertino").tag(City.cupertino)
                Text("San Francisco").tag(City.sanFrancisco)
            }
            .pickerStyle(.segmented)
            // Replace specific data with the switch that toggles
            Chart(data) { element in
                BarMark(
                    x: .value("Day", element.weekday, unit: .day),
                    y: .value("Sales", element.sales)
                )
            }
        }
    }
}
```

For the next iteration, make a bar chart with both cities, and iterate to make it a line chart:

```swift
import Charts
import SwiftUI

struct Series: Identifiable {
    let city: String
    let sales: [SalesSummary]

    var id: String { city }
}

let seriesData: [Series] = [
    .init(city: "Cupertino", sales: cupertinoData),
    .init(city: "San Francisco", sales: sfData)
]

struct LocationDetails: View {

    var body: some View {
        Chart(seriesData) { series in
            ForEach(series.sales) { element in
                // Can change this to LineMark for a different visualization
                BarMark(
                x: .value("Day", element.weekday, unit: .day),
                y: .value("Sales", element.sales)
                )
                // Set a foregroundStyle to be derived by the city to set a different color for each city
                .foregroundStyle(by: .value("City", series.city))
                // If we change it to a line mark, we can add point marks to the lines
                .symbol(by: .value("City", series.city))
            }
        }
    }
}
```
