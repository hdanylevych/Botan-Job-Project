# Botan Job Project

[**Botan**](https://apps.apple.com/us/app/ai-plant-identifier-app-botan/id1557446434) is a plant identification app. As part of the **Tonti Laguna Mobile** team, I contributed to its development for over a year. In this repository, I'll detail the key features I worked on, challenges faced, and solutions applied.

## Project Stack <br />
- **Architecture:** MVP + Coordinator. <br />
- **UI:** UIKit, Auto-layout, Storyboards and XIBs. <br />
- **Storage:** CoreData, UserDefaults, File System. <br />
- **Network:** Client-Server architecture with URLSession and Moya. <br />

## 📱 Social Feed Feature

🎥 Below is a video overview showcasing the main screens and interactions associated with the Social Feed feature:

https://github.com/hdanylevych/Botan-Job-Project/assets/36959716/18a227ef-a28c-4d93-b812-1a38d3c1cf4c

### Challenges:

1. **Navigation:**<br />
    - Given the integration of push notifications for events like likes, comments, etc., one of the primary challenges was ensuring that when a user clicked on a notification, they were seamlessly directed to the specific post or even the exact comment within the post where the interaction occurred.

2. **Unpredictability of Future Development:**<br />
    - With the first release being akin to an MVP of the social network, there was an inherent uncertainty regarding its future trajectory. In the event of a tremendous success and user adoption, the company could decide to rapidly develop and scale this feature.

3. **Restrictions of the GetSocial SDK:**<br />
    - Utilizing the GetSocial SDK as our server solution posed challenges due to its inherent restrictions. While third-party SDKs can significantly accelerate development by providing pre-built solutions, they also come with limitations that may not align perfectly with custom requirements or future scalability plans.

### Implementation:

1. **Navigation System Refactoring**:

    - **Legacy Challenge**: The project's segue-based navigation was limiting, especially for the intricate navigational needs of the social feed feature.
    
    - **Refactoring Process**: Mapped out the app's navigational flows, designed specific coordinators, and systematically replaced the old navigation system, testing each transition to ensure stability.

<details closed>
<summary>Coordinator Code Example </summary>
<br />
<details closed>
<summary>Coordinator Protocols </summary>
    
```Swift
protocol Coordinator: AnyObject {
    var parent: ParentCoordinator? { get }
    var navigationController: UINavigationController { get }
    func start()
    func handle(event: AppEvent)
}

protocol ParentCoordinator: AnyObject {
    var sourceEvent: AppEvent? { get }
    
    func handle(event: AppEvent)
    func setActive(coordinator: Coordinator)
}
```
</details>
<details closed>
<summary>BaseCoordinator </summary>
    
```Swift
class BaseCoordinator: ParentCoordinator, Coordinator {
    let navigationController: UINavigationController
    private let _sourceEvent: AppEvent?
    
    private weak var _parent: ParentCoordinator?
    
    var parent: ParentCoordinator? {
        return _parent
    }
    var sourceEvent: AppEvent? {
        _sourceEvent
    }
    
    init(parent: ParentCoordinator, navigationController: UINavigationController, sourceEvent: AppEvent? = nil) {
        self._parent = parent
        self._sourceEvent = sourceEvent
        
        self.navigationController = navigationController
    }
    
    func start() {
        setActive(coordinator: self)
    }

    // updates the activeCoordinator property in AppCoordinator
    func setActive(coordinator: Coordinator) {
        parent?.setActive(coordinator: self)
    }
}
```
</details>
<details closed>
<summary>AppEvent</summary>
    
```Swift
// This universal event added to completly decouple screens from each other
// and communicate throught the events
enum AppEvent {
    // MARK: - SomeModule
    case goToSomeScreen(data: SomeDataStruct)
}
```
</details>
<details closed>
<summary>Coordinator Example</summary>
    
```Swift
class SomeCoordinator: BaseCoordinator {
    override func start() {
        super.start()
        
        // instantiate and configure ViewController
        // also unwrap sourceEvent property to get some initial data
    }
    
    // handle the event or forward to parent
    override func handle(event: AppEvent) {
        switch event {
        case .someEvent:
            handleSomeEvent()
        default:
            parent?.handle(event: event)
        }
    }
    
    private func handleSomeEvent() { }
}
```
</details>
</details>

3. **Modularization and Abstraction of Social Feed Screens:** <br />

    - To mitigate potential risks of directly relying on the GetSocial SDK, I abstracted the SDK by encapsulating its services behind protocols. This effectively decoupled it from native implementations. This approach laid a more flexible foundation for potential future changes, ensuring a smoother transition should we need to switch SDKs or migrate to our own server solution.

    - To cater to the versatile nature of the social feed, each screen was modularized and abstracted. This design not only enabled easy invocation from different parts of the app or directly through push notifications but also provided greater flexibility and potential for future development, paving the way for enhanced features and user experiences.

## 🌱 Plant Card Feature

🎥 Below is a video overview of the Plant Card feature:

https://github.com/hdanylevych/Botan-Job-Project/assets/36959716/1380bcdd-638b-4d8e-a090-da8eab75a606

### Challenges:

1. **Client-Side Communication with Third-party APIs:** <br />

    - At the time of implementation, the project lacked a client-server architecture. All interactions with third-party APIs were done directly from the client side. This posed a challenge especially when dealing with the Wikipedia API. The API, which was both large and somewhat unstable, returned plant information intertwined with HTML code, all within a single string. Parsing this complex response to extract relevant data for the plant card was a considerable challenge.

2. **Data-Driven Content with Distinct States:** <br />

   - The content of the plant card screen had to be both structured in a specific manner and be data-driven. Additionally, the screen presented different content depending on its state — before the card was saved versus after. Balancing this dynamic content presentation with the need for consistent UI/UX design posed a unique challenge.

### Implementation:

1. **UITableView Implementation with Enum-Driven DataSource:** <br />

   - I opted for a UITableView to structure the screen. The dataSource was designed as an array of Enums, where each Enum represents a specific cell type and provides the necessary data for its display.

2. **DataSource Factories for Dynamic States:** <br />

   - As the number of view states expanded to four, I introduced DataSource Factories. These factories dynamically package the dataSource array based on the current state and the provided data, ensuring the screen displays the appropriate content for each state.

3. **Filtering HTML from Wikipedia API Response:** <br />

   - To handle the embedded HTML in the Wikipedia API responses, I utilized **SwiftSoup** to construct a complete HTML page from each string. After analyzing numerous responses, I identified recurring patterns of extraneous HTML tags. Leveraging the DOM structure, I systematically removed these irrelevant tags to obtain a cleaner and more focused data set for the plant card.

<details closed>
<summary>Filter HTML Function Code</summary>

```Swift
func filter(htmlString: String) -> String? {
    do {
        let doc: Document = try SwiftSoup.parseBodyFragment(htmlString)
        
        let selectorsToRemove: [String] = [
            "a[href^=#cite_note]",
            "a[href^=/wiki/Wikipedia]",
            "table",
            ".hatnote",
            "ul",
            "ol",
            "div"
        ]
        
        for selector in selectorsToRemove {
            let elements = try doc.select(selector)
            try elements.remove()
        }
        
        var cleanText = try doc.text()
        cleanText = cleanText.htmlString
        
        return cleanText
        
    } catch Exception.Error(_, let message) {
        print("SwiftSoup error: \(message)")
    } catch {
        print("Unknown error: \(error.localizedDescription)")
    }
    
    return nil
}
```
</details>

## Thank you for reviewing my work on Botan. If you've reached this point, I'd appreciate a moment to connect and discuss further.
