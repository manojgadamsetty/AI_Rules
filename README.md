# iOS AI God Mode Rules
**Turning AI Assistants into Senior Architects.**

This repository contains battle-tested governance files (`.cursorrules`, `CLAUDE.md`, and `.github/copilot-instructions.md`) designed to fix the most common issues in AI-driven iOS development.

---

![Hero Image](Hero_Image.png)

---

## Why This Exists

AI is a powerful "Pilot," but without a map, it creates technical debt. These rules ensure:

- **Zero "Missing File" Errors**: Forces AI to flag manual Xcode target additions.
- **Architectural Consistency**: Strict enforcement of MVVM-C and Coordinator patterns.
- **Memory Safety**: Automatic detection of retain cycles and [weak self] requirements.
- **2026 Standards**: Optimized for Swift Concurrency, async/await, and iOS 17+.

---

## Quick Start

1. **Clone or copy these files** to your iOS project
2. **Move `.cursorrules`** to your project root (or the contents of cursorrules.md)
3. **Move `CLAUDE.md`** to your project root
4. **Move `copilot-instructions.md`** to your `.github/` folder
5. **Start Coding**: Your AI will now automatically follow the "God Mode" constraints

---

## Repository Topics

If publishing to GitHub, use these tags for discoverability:

`cursor-ai` `claude-code` `github-copilot` `ios-development` `swift` `mvvm-c` `ai-rules` `cursor-rules` `vibe-coding` `agentic-workflows` `developer-tools`

---

## Read the Story

For a deep dive into why these rules exist and the problems they solve, read the companion article:

**[Why Your AI-Generated Code Is Failing (And the Rulebook That Fixes It)](https://medium.com/@manojgadamsetty/why-your-ai-generated-code-is-failing-and-the-rulebook-that-fixes-it-d9b859d034d6)** on Medium

This article explores real-world failures, architectural anti-patterns, and how these governance rules prevent them.

---

## Overview

This repository contains AI-native configuration files designed to provide consistent architectural guidance across multiple development tools. These files act as a "source of truth" for iOS architecture, ensuring that Copilot, Cursor, and Claude maintain strict adherence to MVVM-C patterns and modern Swift concurrency practices.

---

## Files Included

### 1. `.cursorrules`
**Location**: Root directory  
**Purpose**: Controls Cursor editor behavior and AI completions  

**Key Features**:
- MVVM-C architecture enforcement
- Swift Concurrency (async/await) mandates
- Memory safety rules ([weak self] requirements)
- The Missing File Rule (project.pbxproj warnings)
- Planning workflow requirement
- Code review gates checklist
- Project structure template

**Use Case**: Place in your project root to ensure Cursor AI suggestions follow your architectural standards.

---

### 2. `CLAUDE.md`
**Location**: Root directory  
**Purpose**: Architecture context and standards for Claude models  

**Key Features**:
- Detailed MVVM-C layer explanations with code examples
- Swift Concurrency patterns (async/await, @MainActor)
- Memory management rules (weak delegates, [weak self])
- Common patterns and anti-patterns
- Red flags & anti-patterns table
- Testing philosophy and best practices
- Code review checklist

**Use Case**: Reference this file when using Claude models for iOS development. Claude will automatically follow these architectural standards when this file is loaded into context.

---

### 3. `.github/copilot-instructions.md`
**Location**: `.github` subdirectory  
**Purpose**: Guides GitHub Copilot inline completions and chat behavior  

**Key Features**:
- Hard constraints for every suggestion
- Inline completion behavior guidelines
- Chat response patterns and examples
- Red flag scenarios with corrective actions
- Testing guidance with code examples
- Project structure reference
- Do's and Don'ts summary
- Final checklist for every completion

**Use Case**: GitHub Copilot in VS Code will automatically reference this file to enforce architectural standards during completions and chat interactions.

---

## Architecture Overview: MVVM-C

### The Four Mandatory Layers

#### Model (M)
- Data structures (Codable)
- Business logic entities
- Error types and enums
- No UI references

#### View (V)
- SwiftUI `View` structs or UIKit `UIViewController`
- Display state only
- Forward user actions
- **ZERO business logic**

#### ViewModel (VM)
- State management (`@Published` properties)
- Business logic execution
- Service orchestration
- **MUST be `@MainActor` class implementing `ObservableObject`**
- **MUST use async/await only**

#### Coordinator (C)
- Navigation flow control
- ViewController/View creation
- ViewModel dependency injection
- Child coordinator management

---

## Core Rules & Constraints

### 1. Swift Concurrency (Mandatory)
- ALL async operations use `async/await`
- FORBIDDEN: `DispatchQueue.main.async`
- FORBIDDEN: Completion handlers
- FORBIDDEN: `@escaping` closures without `[weak self]`

### 2. Memory Safety
- Every escaping closure uses `[weak self]`
- All delegate properties are `weak`
- No circular references between ViewModels and Coordinators
- Services injected via constructor (no singletons)

### 3. The Missing File Rule
- AI cannot edit `project.pbxproj`
- New files require manual Xcode addition or xcodegen
- Clear warning blocks must accompany file creation

### 4. Planning Workflow
- Plans provided before code generation
- Architecture decisions explicitly stated
- Memory management approach outlined
- Manual actions clearly documented

---

## Deployment Instructions

### Step 1: Copy Files to Project Root

```bash
# Navigate to your iOS project root
cd /path/to/your/ios/project

# Copy the rule files
cp .cursorrules .
cp CLAUDE.md .
mkdir -p .github
cp .github/copilot-instructions.md .github/
```

### Step 2: Commit to Git

```bash
git add .cursorrules CLAUDE.md .github/copilot-instructions.md
git commit -m "feat: add MVVM-C architecture rule files"
```

### Step 3: Enable in Your Tools

#### Cursor
- Cursor automatically reads `.cursorrules` from project root
- No additional configuration needed
- Cursor will enforce these rules on all AI suggestions

#### GitHub Copilot (VS Code)
- Copilot reads `.github/copilot-instructions.md` automatically
- Ensure `.github` directory is in root
- Inline completions and chat will respect architectural rules

#### Claude
- Load `CLAUDE.md` into context when asking questions
- Or upload to your project in Claude Projects
- Claude will maintain these standards throughout conversation

---

## Enforcement Mechanisms

### What Each Tool Does

| Tool | Behavior |
|------|----------|
| **Cursor** | Refuses to suggest singletons; asks about Coordinator placement before code generation; flags [weak self] violations |
| **Copilot** | Completes @MainActor automatically; suggests async/await patterns; warns about direct service calls in Views |
| **Claude** | Reviews code for memory leaks; suggests architectural improvements; provides detailed Plans before implementation |

---

## Using the Rules Effectively

### Before Code Generation
1. AI must provide a `Plan` block first
2. Plan includes layer assignments and architectural decisions
3. User reviews and approves Plan
4. Code generation follows approved Plan

### During Code Generation
1. Each code block is assigned to an MVVM-C layer
2. Memory safety rules are enforced
3. No singletons or hidden dependencies
4. New files include manual action warnings

### Code Review
- Architecture layer clearly identified
- No business logic in Views
- All async operations use async/await
- All escaping closures use [weak self]
- Dependencies injected, not singleton-accessed

---

## Common Patterns

### ViewModel with Service Injection

```swift
@MainActor
final class UserViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false
    @Published var errorMessage: String?
    
    private let userService: UserService
    
    init(userService: UserService) {
        self.userService = userService
    }
    
    func loadUsers() async {
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

### Coordinator-Based Navigation

```swift
@MainActor
final class MainCoordinator {
    private let navigationController: UINavigationController
    private let userService: UserService
    
    func showUserList() {
        let viewModel = UserListViewModel(userService: userService)
        let vc = UserListViewController(viewModel: viewModel)
        navigationController.pushViewController(vc, animated: true)
    }
}
```

### SwiftUI View with ViewModel

```swift
struct UserListView: View {
    @StateObject private var viewModel = UserListViewModel(
        userService: UserService()
    )
    
    var body: some View {
        List(viewModel.users) { user in
            Text(user.name)
        }
        .task {
            await viewModel.loadUsers()
        }
    }
}
```

---

## Forbidden Patterns (Zero Tolerance)

| Pattern | Why | Fix |
|---------|-----|-----|
| `DispatchQueue.main.async` | Old concurrency | Use `@MainActor` |
| `func callback(completion: @escaping)` | Callback hell | Use `async throws` |
| `URLSession.shared` | Singleton coupling | Inject via constructor |
| `var delegate: MyDelegate?` | Retain cycle | Use `weak var` |
| Business logic in View body | Tight coupling | Move to ViewModel |
| `[self]` in escaping closure | Likely memory leak | Use `[weak self]` |
| `Task { self.value = x }` | Implicit capture | Use `[weak self]` |
| Direct ViewController instantiation | Skips Coordinator | Use Coordinator pattern |

---

## Project Structure Template

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
│   │   ├── Views/
│   │   │   └── LoginView.swift
│   │   ├── Controllers/
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
├── Resources/
│   ├── Assets.xcassets/
│   └── Localizable.strings
│
├── .cursorrules
├── CLAUDE.md
├── .github/
│   └── copilot-instructions.md
└── README.md
```

---

## Best Practices

### Code Generation Workflow

1. **Ask the AI**: "Help me add a user profile feature"
2. **AI provides Plan**: Layer breakdown, coordinator approach, memory management
3. **Review Plan**: Approve or ask for changes
4. **Code Generation**: AI generates full implementation with warnings
5. **Manual Steps**: Add new files to Xcode target
6. **Code Review**: Verify against checklist

### Memory Safety Checklist

- [ ] Every `Task { }` uses `[weak self]`
- [ ] All delegate properties are `weak`
- [ ] Services injected via `init`, never singletons
- [ ] No circular references between layers
- [ ] ViewModels decorated with `@MainActor`

### Architecture Checklist

- [ ] Each file belongs to exactly one layer (M/V/VM/C)
- [ ] Views contain only layout and bindings
- [ ] ViewModels manage all business logic
- [ ] Coordinators manage all navigation
- [ ] Models contain only data structures

---

## Support & Questions

### When to Ask for Architectural Guidance

- Before starting a new feature
- When unsure about layer assignment
- When designing complex state management
- When dealing with circular dependencies
- When working with legacy code

### How to Ask

**To Cursor/Copilot**:
> "I'm building a payment feature. Should this be one Coordinator or multiple? How should I structure the ViewModel?"

**To Claude**:
> "Review this code for memory leaks. Does the architecture follow MVVM-C?"

---

## File Maintenance

### When to Update These Files

1. **New architectural patterns** emerge and are validated
2. **Swift language changes** require updates (e.g., new concurrency features)
3. **Project needs** diverge from current patterns
4. **Tool updates** change how rules are enforced

### Update Process

1. Make changes to all three files for consistency
2. Commit to version control
3. Document changes in commit message
4. Communicate changes to team

---

## References

- [MVVM-C Pattern](https://www.hackingwithswift.com/articles/175/advanced-coordinator-pattern-tutorial-ios)
- [Swift Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [Memory Management in Swift](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/memorymanagement)
- [SwiftUI Best Practices](https://developer.apple.com/wwdc21/10018)

---

## License

These architecture rules are provided as a reference template and can be freely adapted to your project's needs.

---

**Last Updated**: April 2026  
**Architecture**: MVVM-C with Swift Concurrency  
**Target**: Swift 5.9+, iOS 14.0+
