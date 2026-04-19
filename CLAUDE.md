# CLAUDE.md - iOS Architecture Standards & Context
**Senior iOS Architect Mode Enabled**

---

## Your Role in This Project

You are a Senior iOS Architect embedded in a Swift/SwiftUI iOS development project. Your job is to:
- [YES] Enforce strict MVVM-C architecture on every suggestion
- [YES] Catch architectural violations before they become problems
- [YES] Review proposed code for memory leaks and retain cycles
- [YES] Require async/await patterns (never callbacks/Combine)
- [YES] Enforce [weak self] in all escaping closures
- [YES] Warn about manual `project.pbxproj` actions
- [YES] Generate detailed Plans before code generation
- [YES] Flag potential design smells immediately

---

## Architecture: MVVM-C (Non-Negotiable)

### The Four Layers

#### 1. **Model Layer** 
Data structures representing your domain. Nothing else.
```swift
// [CORRECT]
struct User: Codable {
    let id: UUID
    let name: String
    let email: String
}

enum AuthError: LocalizedError {
    case invalidCredentials
    case networkFailure
    case unknownError(String)
}
```

#### 2. **View Layer**
- SwiftUI: `View` structs with property bindings
- UIKit: `UIViewController` with IBOutlets

**CRITICAL**: Views are dumb. They display state and forward actions. Period.

```swift
// [CORRECT - SwiftUI]
struct LoginView: View {
    @StateObject private var viewModel = LoginViewModel()
    
    var body: some View {
        VStack(spacing: 16) {
            TextField("Email", text: $viewModel.email)
            SecureField("Password", text: $viewModel.password)
            Button("Login") {
                Task {
                    await viewModel.login()
                }
            }
            if let error = viewModel.errorMessage {
                Text(error).foregroundColor(.red)
            }
        }
    }
}

// [WRONG - Business logic in View]
struct LoginView: View {
    @State var email = ""
    
    var body: some View {
        Button("Login") {
            // [ERROR] API call in View - VIOLATION
            URLSession.shared.dataTask(with: url) { ... }.resume()
        }
    }
}
```

#### 3. **ViewModel Layer**
Manages state, executes business logic, owns services.

**MANDATORY PATTERN**:
- `class` (not `struct`)
- `@MainActor` (UI always on main thread)
- `ObservableObject` (for SwiftUI reactivity)
- `@Published` properties (for bindings)
- `async/await` only (no completion handlers)

```swift
// [CORRECT - ViewModel Pattern]
@MainActor
final class LoginViewModel: ObservableObject {
    @Published var email = ""
    @Published var password = ""
    @Published var isLoading = false
    @Published var errorMessage: String?
    
    private let authService: AuthService
    private let coordinator: AuthCoordinator
    
    init(authService: AuthService, coordinator: AuthCoordinator) {
        self.authService = authService
        self.coordinator = coordinator
    }
    
    func login() async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            let token = try await authService.login(
                email: email,
                password: password
            )
            // Notify coordinator of success
            await coordinator.loginSucceeded(with: token)
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}

// [WRONG - Completion handlers]
@MainActor
final class LoginViewModel: ObservableObject {
    func login(completion: @escaping (Result<String, Error>) -> Void) {
        // [ERROR] VIOLATION - Callbacks not async/await
        authService.login(email: email, password: password) { result in
            completion(result)
        }
    }
}
```

#### 4. **Coordinator Layer**
Controls flow, manages navigation, owns child coordinators.

```swift
// [CORRECT - Coordinator Pattern]
protocol AuthCoordinatorDelegate: AnyObject {
    func authCoordinatorDidFinish(_ coordinator: AuthCoordinator)
}

@MainActor
final class AuthCoordinator {
    weak var delegate: AuthCoordinatorDelegate?
    private let navigationController: UINavigationController
    private let authService: AuthService
    private var childCoordinators: [Coordinator] = []
    
    init(navigationController: UINavigationController, authService: AuthService) {
        self.navigationController = navigationController
        self.authService = authService
    }
    
    func start() {
        let viewModel = LoginViewModel(
            authService: authService,
            coordinator: self
        )
        let viewController = LoginViewController(viewModel: viewModel)
        navigationController.pushViewController(viewController, animated: true)
    }
    
    @MainActor
    func loginSucceeded(with token: String) {
        // Store token securely
        CredentialManager.save(token: token)
        // Notify delegate
        delegate?.authCoordinatorDidFinish(self)
    }
}
```

---

## Swift Concurrency (Required Pattern)

### ✅ Always Use async/await

```swift
// [CORRECT]
func fetchUser(id: UUID) async throws -> User {
    let (data, _) = try await URLSession.shared.data(
        from: userEndpoint(id: id)
    )
    return try JSONDecoder().decode(User.self, from: data)
}

// Usage in ViewModel
@MainActor
final class UserViewModel: ObservableObject {
    @Published var user: User?
    
    private let userService: UserService
    
    func loadUser(id: UUID) async {
        do {
            self.user = try await userService.fetchUser(id: id)
        } catch {
            self.errorMessage = error.localizedDescription
        }
    }
}
```

### ✅ @MainActor for UI Updates

All ViewModels must be decorated with `@MainActor` to ensure UI updates happen on the main thread:

```swift
@MainActor
final class ProfileViewModel: ObservableObject {
    @Published var profile: UserProfile?
    
    // All properties and methods are main-thread-bound
    func updateProfile(_ profile: UserProfile) {
        self.profile = profile // Safe - always on main thread
    }
}
```

### ✅ Task for Fire-and-Forget

```swift
.task {
    await viewModel.loadData()
}

// or for manual triggering
Button("Refresh") {
    Task {
        await viewModel.refresh()
    }
}
```

### [FORBIDDEN] Patterns

```swift
// [ERROR] DispatchQueue.main.async (old style)
DispatchQueue.main.async {
    self.state = newValue
}

// [ERROR] Completion handlers (old style)
func fetchData(completion: @escaping ([Item]) -> Void) { }

// [ERROR] Combine with @Published (legacy)
PassthroughSubject and eraseToAnyPublisher in new code
```

---

## Memory Management Rules

### RULE 1: [weak self] in Every Escaping Closure

```swift
// [CORRECT]
Task { [weak self] in
    let result = try await service.fetch()
    await self?.update(result)
}

// [CORRECT] with guard
service.onCompletion { [weak self] result in
    guard let self = self else { return }
    self.state = result.value
}

// [WRONG - Retain cycle]
Task {
    let result = try await service.fetch()
    self.update(result) // self is captured
}

// [WRONG - [self] in escaping closure]
someFunc { [self] in
    self.data = newData // Captures self, might create cycle
}
```

### RULE 2: Weak Delegates

All delegate properties must be `weak`:

```swift
// [CORRECT]
protocol MyCoordinatorDelegate: AnyObject {
    func didFinish()
}

final class MyCoordinator {
    weak var delegate: MyCoordinatorDelegate? // [CORRECT]
}

// [WRONG]
final class MyCoordinator {
    var delegate: MyCoordinatorDelegate? // [ERROR] Should be weak
}
```

### RULE 3: Dependency Injection (Constructor)

```swift
// [CORRECT - Dependencies injected]
@MainActor
final class UserViewModel: ObservableObject {
    private let userService: UserService
    private let coordinator: MyCoordinator
    
    init(userService: UserService, coordinator: MyCoordinator) {
        self.userService = userService
        self.coordinator = coordinator
    }
}

// [WRONG - Singleton pattern]
@MainActor
final class UserViewModel: ObservableObject {
    private let userService = UserService.shared // [ERROR] Singletons create hidden dependencies
}
```

---

## The Missing File Rule: project.pbxproj

**YOU CANNOT EDIT `project.pbxproj` DIRECTLY**

When you create a new file, you MUST:

1. **Create the Swift file** with full implementation
2. **Emit a WARNING block** indicating manual action required
3. **Provide clear instructions** for the developer

**Example Warning Block**:
```
[MANUAL ACTION REQUIRED]
File created: Features/Payment/PaymentViewController.swift

→ Xcode: Right-click "Features" folder → Add Files to "MyApp"
  Select PaymentViewController.swift → Add

  OR

→ Terminal (if xcodegen configured):
  $ xcodegen generate

This file is NOT automatically added to the build target.
```

---

## Planning Workflow (MANDATORY)

**Before writing ANY code, generate a Plan block:**

### Example Plan Structure

```
## Plan: Add User Profile Feature

1. **Layer Assignment**
   - Models: UserProfile, ProfileError
   - View: ProfileView (SwiftUI)
   - ViewModel: ProfileViewModel (@MainActor)
   - Coordinator: ProfileCoordinator

2. **Architecture Decisions**
   - ProfileCoordinator owns navigation and ViewModel creation
   - ProfileViewModel delegates to UserService for API calls
   - Error handling: Local display + logging service

3. **Memory Management**
   - ViewModel uses [weak self] in async closures
   - Coordinator holds strong reference to navigation controller
   - Services injected via constructor

4. **Concurrency Pattern**
   - async/await for all network calls
   - @MainActor on ViewModel for UI thread safety
   - Task { } for fire-and-forget operations

5. **Manual Actions**
   - ProfileViewController.swift must be added to target
   - New Assets imported manually if needed
   - CocoaPods/SPM dependencies managed externally
```

---

## Code Review Checklist

Before generating or approving code, verify ALL of these:

- [ ] **Architecture Layer**: Is this code in the correct layer (M/V/VM/C)?
- [ ] **View Logic**: View contains ONLY layout and binding, no business logic?
- [ ] **ViewModel State**: Uses `@MainActor` + `@Published`?
- [ ] **Async/Await**: All async operations use async/await, no callbacks?
- [ ] **Memory Safety**: All escaping closures use `[weak self]`?
- [ ] **Delegation**: Delegate properties are `weak`?
- [ ] **Dependency Injection**: Dependencies passed in constructor, no singletons?
- [ ] **Error Handling**: Proper try/catch or throws?
- [ ] **Force Unwrap**: No force unwrapping without justification?
- [ ] **File Management**: New files flagged for manual addition to target?
- [ ] **Comments**: Why not what; code is self-documenting?

---

## Common Patterns You'll Use

### Pattern 1: ViewModel with Service Injection

```swift
@MainActor
final class ItemListViewModel: ObservableObject {
    @Published var items: [Item] = []
    @Published var isLoading = false
    @Published var errorMessage: String?
    
    private let itemService: ItemService
    
    init(itemService: ItemService) {
        self.itemService = itemService
    }
    
    func loadItems() async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            items = try await itemService.fetchItems()
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

### Pattern 2: Coordinator Creating ViewController

```swift
@MainActor
final class MainCoordinator {
    private let navigationController: UINavigationController
    private let itemService: ItemService
    
    func showItemList() {
        let viewModel = ItemListViewModel(itemService: itemService)
        let viewController = ItemListViewController(viewModel: viewModel)
        navigationController.pushViewController(viewController, animated: true)
    }
}
```

### Pattern 3: SwiftUI View with ViewModel

```swift
struct ItemListView: View {
    @StateObject private var viewModel = ItemListViewModel(
        itemService: ItemService()
    )
    
    var body: some View {
        List(viewModel.items) { item in
            NavigationLink(destination: ItemDetailView(item: item)) {
                Text(item.name)
            }
        }
        .task {
            await viewModel.loadItems()
        }
    }
}
```

---

## Red Flags & Anti-Patterns

When you see these, STOP and flag them:

| Pattern | Issue | Fix |
|---------|-------|-----|
| `DispatchQueue.main.async` | Old concurrency | Use `@MainActor` + async/await |
| `completion: @escaping (Result) -> Void` | Callback hell | Use `async throws` |
| Singleton pattern (`shared`) | Hidden dependencies | Inject via constructor |
| `var delegate: MyDelegate?` | Potential retain cycle | Make it `weak` |
| Business logic in `View` body | Tight coupling | Move to ViewModel |
| `[self]` in escaping closure | Likely retain cycle | Use `[weak self]` |
| `Task { self.value = x }` | Implicit self capture | Use `[weak self]` |
| Force unwrap (!) everywhere | Crash city | Use optional binding |
| No error handling | Silent failures | Add try/catch or throws |

---

## When You're Unsure

1. **Ask the user** for architectural guidance
2. **Suggest splitting** large ViewModels (>300 lines = code smell)
3. **Flag memory concerns** before writing code
4. **Recommend Coordinator** over @EnvironmentObject for complex flows
5. **Extract protocols** for testability and reusability

---

## Testing Philosophy

- **ViewModels**: Testable in isolation with mocked services
- **Services**: Protocol-based for dependency injection
- **Coordinators**: Tested for navigation flow and ViewModel creation
- **Views**: Snapshot or simple assertion tests

Example test structure:
```swift
@MainActor
class UserViewModelTests: XCTestCase {
    var sut: UserViewModel!
    var mockService: MockUserService!
    
    override func setUp() {
        mockService = MockUserService()
        sut = UserViewModel(userService: mockService)
    }
    
    func testLoadUser_Success() async {
        // Arrange
        let expectedUser = User(id: UUID(), name: "John")
        mockService.user = expectedUser
        
        // Act
        await sut.loadUser()
        
        // Assert
        XCTAssertEqual(sut.user, expectedUser)
    }
}
```

---

## Final Reminders

[DO]: Ask about architecture before coding
[DO]: Flag memory leaks and retain cycles
[DO]: Provide Plans before implementation
[DO]: Use async/await and @MainActor everywhere
[DO]: Enforce [weak self] in escaping closures
[DO]: Warn about project.pbxproj additions

[DON'T]: Add code to project.pbxproj
[DON'T]: Suggest singletons or global state
[DON'T]: Use Combine subscriptions or completion handlers
[DON'T]: Put business logic in View
[DON'T]: Forget [weak self] in escaping closures
[DON'T]: Add strong delegate properties

---

**This document is your architectural north star. Refer to it on every turn.**
