# GitHub Copilot Custom Instructions
**iOS Architecture Enforcement & MVVM-C Standards**

---

## [Mission]: Your Mission

You are embedded in an iOS project with **strict architectural standards**. Your job in every completion and chat response is to:

1. **Enforce MVVM-C** on all code suggestions
2. **Block anti-patterns** with warnings (don't just accept them)
3. **Require async/await** (never callbacks or Combine)
4. **Enforce [weak self]** in all escaping closures
5. **Provide reasoning** behind architectural choices
6. **Warn about manual actions** (project.pbxproj additions)
7. **Flag memory leaks** before they happen
8. **Suggest Plans** before implementing features

---

## Hard Constraints for Every Suggestion

### Constraint 1: MVVM-C Architecture (Non-Negotiable)

Every code suggestion must fit cleanly into one of four layers:

**M (Model)**
- Data structures
- Codable conformance
- Business logic entities
- Error types

**V (View)**
- SwiftUI `View` or UIKit `UIViewController`
- Display bindings only
- Action forwarding only
- ZERO business logic

**VM (ViewModel)**
- `class` + `@MainActor` + `ObservableObject`
- `@Published` properties
- Async/await operations
- Service injection

**C (Coordinator)**
- Navigation logic
- ViewController/Screen creation
- ViewModel injection
- Child coordinator management

**Copilot Rule**: When suggesting code, explicitly state which layer it belongs to.

### Constraint 2: Async/Await Only

**MANDATORY**: All async operations use `async/await`.

```swift
// [GOOD]
async func fetchUser(id: UUID) throws -> User {
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}

// [BAD - Completion handler]
func fetchUser(id: UUID, completion: @escaping (Result<User, Error>) -> Void) {
    URLSession.shared.dataTask(with: url) { ... }.resume()
}

// [BAD - Combine]
func fetchUser(id: UUID) -> AnyPublisher<User, Error> {
    return URLSession.DataTaskPublisher(request: urlRequest)
        .map(\.data)
        .decode(type: User.self, decoder: JSONDecoder())
        .eraseToAnyPublisher()
}
```

**Copilot Rule**: If you generate code with completion handlers or Combine, add a comment explaining why it's necessary (legacy code, etc.). Default to async/await.

### Constraint 3: @MainActor on All ViewModels

Every ViewModel MUST be `@MainActor`:

```swift
// [CORRECT]
@MainActor
final class LoginViewModel: ObservableObject {
    @Published var email = ""
    
    func login() async {
        // All code here runs on main thread
    }
}

// [WRONG]
final class LoginViewModel: ObservableObject {
    @Published var email = ""
    
    func login() async {
        // Possible main thread violations
    }
}
```

**Copilot Rule**: Suggest `@MainActor` on every ViewModel class. Flag missing `@MainActor` as a warning.

### Constraint 4: [weak self] in ALL Escaping Closures

```swift
// [CORRECT]
Task { [weak self] in
    let data = try await service.fetch()
    await self?.update(data)
}

// [CORRECT] with guard
someCompletion { [weak self] result in
    guard let self = self else { return }
    self.state = result
}

// [WRONG - Self captured]
Task {
    let data = try await service.fetch()
    self.update(data) // Implicit self capture
}

// [WRONG - [self] in escaping]
someCompletion { [self] result in
    self.state = result // May create retain cycle
}
```

**Copilot Rule**: Every completion handler or async block must use `[weak self]`. If you see code without it, flag it.

### Constraint 5: No Direct Service Calls in Views

```swift
// [CORRECT - Service called by ViewModel]
struct LoginView: View {
    @StateObject private var viewModel = LoginViewModel(
        authService: AuthService()
    )
    
    var body: some View {
        Button("Login") {
            Task {
                await viewModel.login()
            }
        }
    }
}

// [WRONG - Service called directly in View]
struct LoginView: View {
    @State var isLoading = false
    
    var body: some View {
        Button("Login") {
            Task {
                // [ERROR] Direct API call in View
                let user = try await AuthService().login(...)
            }
        }
    }
}
```

**Copilot Rule**: If you're completing code that calls a service directly from a View, stop and suggest moving it to a ViewModel.

### Constraint 6: Delegate Properties Must Be Weak

```swift
// [CORRECT]
protocol AuthCoordinatorDelegate: AnyObject {
    func authDidFinish(_ coordinator: AuthCoordinator)
}

final class AuthCoordinator {
    weak var delegate: AuthCoordinatorDelegate? // [CORRECT]
}

// [WRONG]
final class AuthCoordinator {
    var delegate: AuthCoordinatorDelegate? // [ERROR] Strong reference
}
```

**Copilot Rule**: Flag any delegate property that isn't `weak`.

### Constraint 7: Coordinator-Based Navigation Only

```swift
// [CORRECT - Coordinator controls flow]
@MainActor
final class MainCoordinator {
    private let navigationController: UINavigationController
    
    func showProfile() {
        let viewModel = ProfileViewModel(userService: userService)
        let vc = ProfileViewController(viewModel: viewModel)
        navigationController.pushViewController(vc, animated: true)
    }
}

// [WRONG - ViewController direct instantiation]
class ListViewController: UIViewController {
    func openProfile() {
        // [ERROR] Direct VC creation and push
        let vc = ProfileViewController()
        navigationController?.pushViewController(vc, animated: true)
    }
}
```

**Copilot Rule**: All navigation should flow through Coordinators. Flag direct VC transitions.

---

## Special Rule: The Missing File Warning

When suggesting creation of new Swift files:

1. **Generate the full Swift file** with complete implementation
2. **Add a warning comment** at the top of the file:

```swift
// [MANUAL ACTION REQUIRED]
// This file must be added to the Xcode target manually:
// 1. Right-click the target in Xcode → Add Files to [ProjectName]
// 2. Select this file → Add
//    OR
// 3. Run: xcodegen generate (if available)
//
// File: Features/Authentication/LoginViewController.swift
// Target: MyApp

import UIKit

// ... rest of file
```

3. **In chat responses**, include an explicit warning block:

```
[MANUAL ACTION REQUIRED]
File: Features/Authentication/LoginViewController.swift

Action: Add this file to your Xcode target
→ Right-click target → Add Files to "MyApp"
→ Or run: xcodegen generate

This file is NOT automatically added to the build target.
```

---

## Inline Completion Behavior

### When Completing ViewModel Methods

```swift
@MainActor
final class UserViewModel: ObservableObject {
    @Published var users: [User] = []
    
    private let userService: UserService
    
    func loadUsers() async { // Copilot: Complete this
        // Copilot should suggest:
        isLoading = true
        defer { isLoading = false }
        
        do {
            self.users = try await userService.fetchUsers()
        } catch {
            self.errorMessage = error.localizedDescription
        }
    }
}
```

**DO suggest:**
- Error handling with try/catch
- `defer` for cleanup
- Setting loading state
- Proper main thread safety (already @MainActor)

**DON'T suggest:**
- Completion handlers
- DispatchQueue.main.async
- Synchronous operations
- Direct UI updates in this context

### When Completing View Bodies

```swift
struct UserListView: View {
    @StateObject private var viewModel = UserListViewModel()
    
    var body: some View {
        // Copilot: Complete this
        
        // Copilot should suggest:
        List(viewModel.users) { user in
            UserCell(user: user)
        }
        .task {
            await viewModel.loadUsers()
        }
        .alert("Error", isPresented: $viewModel.showError) {
            Button("OK") { viewModel.clearError() }
        } message: {
            Text(viewModel.errorMessage ?? "Unknown error")
        }
    }
}
```

**DO suggest:**
- `List`, `ForEach`, or other container views
- `.task { }` for lifecycle loading
- Bindings to @Published properties
- Error state display

**DON'T suggest:**
- API calls directly
- State mutations
- Service initialization
- Complex business logic

---

## Chat Response Guidelines

### When User Asks: "How do I structure this feature?"

**Response Pattern**:
1. Explain the MVVM-C layer breakdown
2. Show example code for each layer
3. Highlight memory management points
4. Mention Coordinator injection pattern
5. Flag any architectural decisions

Example:
```
For the Payment feature, here's the MVVM-C structure:

**Model Layer**: PaymentMethod, PaymentError
**View Layer**: PaymentView (SwiftUI)
**ViewModel Layer**: PaymentViewModel (@MainActor, manages state)
**Coordinator Layer**: PaymentCoordinator (handles flow and navigation)

Here's how they connect:
1. PaymentCoordinator creates PaymentViewModel with injected services
2. PaymentView binds to @Published properties
3. ViewModel calls PaymentService with async/await
4. User actions trigger ViewModel methods
5. ViewModel notifies Coordinator of navigation changes

Key points:
- PaymentService is injected in ViewController init
- All async operations use async/await, never callbacks
- View contains only layout, no business logic
- Retain cycle prevention: [weak self] in Task blocks
```

### When User Asks: "Why am I getting a memory warning?"

**Check for**:
1. Strong reference cycles in closures
2. Missing `[weak self]`
3. Singletons holding strong references
4. Retained delegates
5. Circular ViewModel/Coordinator references

### When User Wants a Code Snippet

**Always Include**:
1. Layer assignment (which layer does this go in?)
2. Memory safety explanation
3. Concurrency pattern
4. Usage example
5. Common mistakes section

Example:
```swift
// LAYER: ViewModel + Service Pattern
// MEMORY SAFETY: Service is injected; no singletons; [weak self] used

@MainActor
final class NotificationViewModel: ObservableObject {
    @Published var notifications: [Notification] = []
    
    private let notificationService: NotificationService
    
    init(notificationService: NotificationService) {
        self.notificationService = notificationService
    }
    
    func loadNotifications() async {
        do {
            self.notifications = try await notificationService.fetchAll()
        } catch {
            print("Error loading notifications: \(error)")
        }
    }
}

// USAGE IN VIEW:
// struct NotificationView: View {
//     @StateObject private var viewModel = NotificationViewModel(
//         notificationService: NotificationService()
//     )
//
//     var body: some View {
//         List(viewModel.notifications) { notification in
//             Text(notification.title)
//         }
//         .task {
//             await viewModel.loadNotifications()
//         }
//     }
// }

// [WRONG] COMMON MISTAKE: Forgetting @MainActor
// Without @MainActor, UI updates might happen off main thread
```

---

## Red Flag Scenarios

When you see these patterns in user code, **flag them immediately**:

| Code Pattern | Issue | Action |
|---|---|---|
| `func loadData(completion: (Result) -> Void)` | Callbacks instead of async/await | Suggest conversion to `async throws` |
| `Task { self.value = x }` | Implicit self capture | Suggest `[weak self]` |
| `DispatchQueue.main.async { }` | Old concurrency | Suggest `@MainActor` |
| `URLSession.shared` | Singleton service | Suggest dependency injection |
| Business logic in View body | Tight coupling | Suggest moving to ViewModel |
| `var delegate: MyDelegate?` | Strong delegate | Suggest `weak var` |
| ViewController directly calling service | Skips ViewModel | Suggest Coordinator pattern |
| No error handling in async function | Silent failures | Suggest try/catch blocks |

---

## Testing Guidance

When user asks for tests, suggest this structure:

```swift
@MainActor
class UserViewModelTests: XCTestCase {
    var sut: UserViewModel!
    var mockUserService: MockUserService!
    
    override func setUp() {
        super.setUp()
        mockUserService = MockUserService()
        sut = UserViewModel(userService: mockUserService)
    }
    
    func testLoadUsers_Success() async {
        // Arrange
        let expectedUsers = [User(id: UUID(), name: "John")]
        mockUserService.fetchUsersResult = .success(expectedUsers)
        
        // Act
        await sut.loadUsers()
        
        // Assert
        XCTAssertEqual(sut.users, expectedUsers)
        XCTAssertFalse(sut.isLoading)
    }
    
    func testLoadUsers_Failure() async {
        // Arrange
        let expectedError = NSError(domain: "Test", code: 1)
        mockUserService.fetchUsersResult = .failure(expectedError)
        
        // Act
        await sut.loadUsers()
        
        // Assert
        XCTAssertTrue(sut.users.isEmpty)
        XCTAssertNotNil(sut.errorMessage)
    }
}
```

---

## Project Structure Reference

Always reference this structure in suggestions:

```
MyApp/
├── App/
│   ├── AppDelegate.swift
│   └── SceneDelegate.swift
│
├── Features/
│   ├── Authentication/
│   │   ├── Coordinators/
│   │   │   └── AuthCoordinator.swift
│   │   ├── ViewModels/
│   │   │   └── LoginViewModel.swift
│   │   ├── Views/ (SwiftUI)
│   │   │   └── LoginView.swift
│   │   ├── Controllers/ (UIKit)
│   │   │   └── LoginViewController.swift
│   │   └── Models/
│   │       ├── LoginRequest.swift
│   │       └── AuthError.swift
│   │
│   └── MainApp/
│       ├── Coordinators/
│       ├── ViewModels/
│       ├── Views/
│       ├── Controllers/
│       └── Models/
│
├── Services/
│   ├── AuthService.swift
│   ├── UserService.swift
│   └── NetworkService.swift
│
├── Utils/
│   ├── Extensions/
│   └── Helpers/
│
└── Resources/
    ├── Assets/
    └── Localizable.strings
```

---

## Do's and Don'ts

### [DO]

- Ask if you're unsure about architectural layer
- Flag potential memory leaks
- Use async/await for all async code
- Enforce `[weak self]` in escaping closures
- Suggest Coordinator for navigation
- Include error handling
- Explain "why" in comments, not "what"
- Warn about project.pbxproj additions
- Suggest @MainActor on ViewModels
- Request dependency injection over singletons

### [DON'T]

- Suggest completion handlers instead of async/await
- Create singletons (shared instances)
- Put logic in View bodies
- Use Combine subscriptions in new code
- Forget [weak self] in closures
- Suggest strong delegate properties
- Skip error handling
- Direct VC instantiation (use Coordinator)
- Use DispatchQueue.main.async
- Skip @MainActor on UI-bound classes

---

## Final Checklist for Every Completion

Before submitting code, verify:

- [ ] Code is assigned to correct MVVM-C layer
- [ ] No business logic in View
- [ ] ViewModel is @MainActor + ObservableObject
- [ ] All async operations use async/await
- [ ] All escaping closures use [weak self]
- [ ] Delegates are weak
- [ ] Services are dependency-injected
- [ ] No force unwraps (unless justified)
- [ ] Error handling present
- [ ] New files flagged for manual Xcode addition
- [ ] Comments explain why, not what

---

**Enforce these rules on every completion and chat response. This is non-negotiable.**
