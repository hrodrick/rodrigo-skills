---
name: kmp-clean-architecture
description: Clean Architecture patterns for Kotlin Multiplatform projects — module structure, dependency rules, UseCases, Repositories, and data layer patterns. Use when creating or modifying Kotlin or Gradle code for any platform or in general.
origin: ECC
---

# Kotlin Multiplatform Clean Architecture

Clean Architecture patterns for Kotlin Multiplatform projects. Covers module boundaries, dependency inversion, UseCase/Repository patterns, and data layer design with SQLDelight, and Ktor.

## When to Activate

- Structuring KMP project modules
- Implementing UseCases, Repositories, or DataSources
- Designing data flow between layers (domain, data, presentation)
- Setting up dependency injection with Koin
- Working with SQLDelight and/or Ktor in a layered architecture

## Module Structure

### Recommended modules Layout

```
project/            # Root project
├── shared/         # Main shared module with common code
├── app-android/    # Android application module
├── app-ios/        # iOS application module
├── app-web/        # Web application module
├── app-desktop/    # Desktop application module
└── server/         # Server-side module
```

Most of the code usually belongs to `shared/`. 

### Recommended sub-modules Layout

```
platform-module/          # A main platform app module (e.g. :shared, :app-android, :app-iOS, :app-desktop, usually we will work on :shared)
├── app/                  # Sub-module: Application entry point (where needed), Application specific code (e.g. Android's Application class, MainActivity, etc)
├── di/                   # Sub-module: Dependency injection modules and wiring (using Koin)
├── core/                 # Sub-module: Reusable utilities, base classes, error types, shared utility extensions,
├── domain/               # Sub-module: Reusable UseCases, domain models, repository interfaces
├── data/                 # Sub-module: Reusable Repository implementations, DataSources, DB, network
├── ui/                   # Sub-module: Reusable Compose components, theme, typography, Modifier extensions
└── feature/              # Folder: Feature sub-modules (usually 1 per high-level screen)
    ├── login/            # Sub-module: Login screen feature (signin screen, signup screen, Modals, etc. ViewModels, UI Models for login screens)
    |   ├── domain/       # Folder: Feature specific UseCases, domain models, repository interfaces
    |   ├── data/         # Folder: Feature specific Repository implementations, DataSources, DB, network
    |   ├── components/   # Folder: Smaller component composables
    |   └── utils/        # Folder: Feature-specific utilities (e.g. extensions, internal reusable code, etc)
    ├── settings/         # Sub-module: Settings screen feature
    |   ├── domain/
    |   ├── data/
    |   ├── components/
    |   └── utils/     
    ├── profile/          # Sub-module: User profile screen feature
    |   ├── domain/
    |   ├── data/
    |   ├── components/
    |   └── utils/     
    └── home/             # Sub-module: Home screen feature
        ├── domain/
        ├── data/
        ├── components/
        └── utils/     
```

### Dependency Rules

The dependency graph across module should follow this pattern:

```
app → di, core, domain, data, ui, [features].
domain → core.
data → domain, core.
core → (nothing).
ui → core.
[features] → ui, domain, core.
```

**Critical**: Only depend on modules when they are really required (e.g. data needs domain to implement the interfaces)

## Domain Layer

### UseCase Pattern

Each UseCase represents one business operation. Use `operator fun invoke` for clean call sites:

```kotlin
class GetItemsByCategoryUseCase(
    private val repository: ItemRepository
) {
    suspend operator fun invoke(category: String): List<Item> {
        return repository.getItemsByCategory(category)
    }
}

// Flow-based UseCase for reactive streams
class ObserveUserProgressUseCase(
    private val repository: UserRepository
) {
    operator fun invoke(userId: String): Flow<UserProgress> {
        return repository.observeProgress(userId)
    }
}
```

### Domain Models

Domain models are plain Kotlin data classes — no framework annotations:

```kotlin
data class Item(
    val id: String,
    val title: String,
    val description: String,
    val tags: List<String>,
    val status: Status,
    val category: String
)

enum class Status { Draft, Active, Archived }
```

### Repository Interfaces

Defined in domain, implemented in data:

```kotlin
interface ItemRepository {
    suspend fun getItemsByCategory(category: String): List<Item>
    suspend fun saveItem(item: Item)
    fun observeItems(): Flow<List<Item>>
}
```

## Data Layer

### Repository Implementation

Coordinates between local and remote data sources, maps data models to domain models:

```kotlin
class ItemRepositoryImpl(
    private val localDataSource: ItemLocalDataSource,
    private val remoteDataSource: ItemRemoteDataSource
) : ItemRepository {

    override suspend fun getItemsByCategory(category: String): List<Item> {
        val remote = remoteDataSource.fetchItems(category)
        localDataSource.insertItems(remote.map { it.toEntity() })
        return localDataSource.getItemsByCategory(category)
            .map { it.toDomain() }
    }

    override suspend fun saveItem(item: Item) {
        localDataSource.insertItems(listOf(item.toEntity()))
    }

    override fun observeItems(): Flow<List<Item>> {
        return localDataSource.observeAll().map { entities ->
            entities.map { it.toDomain() }
        }
    }
}
```

### Mapper Pattern

Keep mappers as extension functions near the data models:

```kotlin
// In data layer
fun ItemEntity.toDomain() = Item(
    id = id,
    title = title,
    description = description,
    tags = tags.split("|"),
    status = Status.valueOf(status),
    category = category
)

fun ItemDto.toEntity() = ItemEntity(
    id = id,
    title = title,
    description = description,
    tags = tags.joinToString("|"),
    status = status,
    category = category
)
```

### SQLDelight (KMP)

```sql
-- Item.sq
CREATE TABLE ItemEntity (
    id TEXT NOT NULL PRIMARY KEY,
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    tags TEXT NOT NULL,
    status TEXT NOT NULL,
    category TEXT NOT NULL
);

getByCategory:
SELECT * FROM ItemEntity WHERE category = ?;

upsert:
INSERT OR REPLACE INTO ItemEntity (id, title, description, tags, status, category)
VALUES (?, ?, ?, ?, ?, ?);

observeAll:
SELECT * FROM ItemEntity;
```

### Ktor Network Client (KMP)

```kotlin
class ItemRemoteDataSource(private val client: HttpClient) {

    suspend fun fetchItems(category: String): List<ItemDto> {
        return client.get("api/items") {
            parameter("category", category)
        }.body()
    }
}

// HttpClient setup with content negotiation (usually done within an "api" class that can be injected in the datasource. e.g. MyAppApi)
val httpClient = HttpClient {
    install(ContentNegotiation) { json(Json { ignoreUnknownKeys = true }) }
    install(Logging) { level = LogLevel.HEADERS }
    defaultRequest { url("https://api.example.com/") } // URLs should be obtained from a buildConfig constant
}
```

## Dependency Injection

### Koin (KMP-friendly)

```kotlin
// Domain module
val domainModule = module {
    factory { GetItemsByCategoryUseCase(get()) }
    factory { ObserveUserProgressUseCase(get()) }
}

// Data module
val dataModule = module {
    single<ItemRepository> { ItemRepositoryImpl(get(), get()) }
    single { ItemLocalDataSource(get()) }
    single { ItemRemoteDataSource(get()) }
}

// Presentation module
val presentationModule = module {
    viewModelOf(::ItemListViewModel)
    viewModelOf(::DashboardViewModel)
}

// And so on
```

## Error Handling

### Exceptionize-all pattern

Let unhandled Exceptions propagate upwards the coroutine context and catch them on the Viewmodels when they make their respective call.

For managed errors, catch and throw custom exceptions that represents the error itself.
Example: Trying to login on the `LoginRepositoryImpl` with an existing user will receive an exception/error response from the backend api, it should be cache dby the `LoginRepositoryImpl`, and rethrown as a custom `UserAlreadyExistsException`.

```kotlin

// In Repository - intercept 3rd party exceptions and Results and then retrow custom exceptions
override suspend fun loginWithProvider(provider: CustomAuthProvider, email: String?, password: String?): UserInfo {
    try {
        when (val result = googleLoginClient.startLogin()) {
            GoogleLoginResult.Cancelled -> throw AuthException.LoginCancelledAuthException()
            is GoogleLoginResult.Error -> throw AuthException.SocialMediaLoginException(result.exception)
            GoogleLoginResult.Success -> { } // nothing to do needed
        }
    } catch (e: AuthRestException) {
        when (e.errorCode) {
            AuthErrorCode.InvalidCredentials -> throw AuthException.InvalidCredentialsException()
            AuthErrorCode.EmailAddressInvalid -> throw AuthException.InvalidEmailException()
            AuthErrorCode.EmailNotConfirmed -> throw AuthException.EmailNotConfirmedException()
            else -> throw e
        }
    }
}

// In ViewModel — catch relevant errors and map to UI state
private fun login(provider: CustomAuthProvider, email: String?, password: String?) = viewModelScope.launch(uncaughtExceptionsHandler) {
    logger.i { "Logging in with ${provider.id}" }
    updateUIState { state.copy(isLoading = true) }
    try {
        userRepository.loginWithProvider(provider, email, password)
        logger.i { "Login successful with ${provider.id}" }
        updateUIState { state.copy(isLoading = false) }
        postOneShotEvent(LoginSideEffect.OnLoginSuccess)
    } catch (_: AuthException.LoginCancelledAuthException) {
        logger.i { "Login with ${provider.id} cancelled by user" }
        updateUIState { state.copy(isLoading = false) }
    } catch (_: AuthException.EmailNotConfirmedException) {
        updateUIState { state.copy(isLoading = false) }
        postOneShotEvent(LoginSideEffect.OnLoginSuccessWithUnConfirmedEmail(
            email = state.email.textFieldState.text.toString()
        ))
    } catch (_: AuthException.InvalidCredentialsException) {
        updateUIState { state.copy(isLoading = false, password = state.password.copy(errorMessage = Res.string.invalid_credentials)) }
    } catch (_: AuthException.InvalidEmailException) {
        updateUIState { state.copy(isLoading = false, email = state.email.copy(errorMessage = Res.string.email_regex_unmatched)) }
    } catch (ex: Exception) {
        logger.e(ex) { "Login failed with ${provider.id} with an unhandled exception: ${ex.message}" }
        appGlobalsService.showSnackbar(message = Res.string.error_generic)
        updateUIState { state.copy(isLoading = false) }
    }
}
```

## Convention Plugins (Gradle)

Use convention plugins to reduce build file code duplication:

## Anti-Patterns to Avoid

- Importing Android framework or platform specific classes in `domain` — keep it pure Kotlin
- Exposing database entities or DTOs to the UI layer — always map to domain models
- Putting business logic in ViewModels — extract to UseCases
- Using `GlobalScope` or unstructured coroutines — use `viewModelScope` or structured concurrency
- Using hardocded coroutine scopes/dispatchers such as `Dispatchers.IO` or `Dispatchers.Main` — Inject a CoroutineContext or CoroutineDispatcher at the constructor level (name it accordingly to its purpose)
- Fat repository implementations — split into focused repositories or review if there is code that should be on a  DataSources instead.
- Circular module dependencies — if A depends on B, B must not depend on A.

## References

See skill: `compose-multiplatform-patterns` for UI patterns.
