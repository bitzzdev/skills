# App architecture guide

Reflects Google's commonly recommended modern Android app architecture. Re-check
https://developer.android.com/topic/architecture if the user's task involves an architectural
decision this file doesn't clearly cover, since official guidance evolves.

## Layers

```
UI layer (Compose screens + ViewModels)
   ↓ calls
Domain layer (optional — use cases/interactors, for apps complex enough to need business
   logic shared across multiple ViewModels; skip for simple apps)
   ↓ calls
Data layer (repositories, wrapping local DB / network / datastore sources)
```

For a simple app (a handful of screens, no complex shared business logic), UI layer calling
directly into a data layer's repositories is fine — don't force an unnecessary domain/use-case
layer onto a small app just because it's "the proper architecture." Introduce the domain layer
when business logic needs to be shared/reused across multiple ViewModels or is complex enough to
deserve isolated testing.

## UI layer pattern: ViewModel + UI state + unidirectional data flow

```kotlin
data class ProfileUiState(
    val isLoading: Boolean = false,
    val userName: String = "",
    val errorMessage: String? = null,
)

class ProfileViewModel(
    private val userRepository: UserRepository,
) : ViewModel() {

    private val _uiState = MutableStateFlow(ProfileUiState())
    val uiState: StateFlow<ProfileUiState> = _uiState.asStateFlow()

    fun loadUser(userId: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            try {
                val user = userRepository.getUser(userId)
                _uiState.update { it.copy(isLoading = false, userName = user.name) }
            } catch (e: Exception) {
                _uiState.update { it.copy(isLoading = false, errorMessage = e.message) }
            }
        }
    }
}
```

```kotlin
@Composable
fun ProfileScreen(viewModel: ProfileViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    ProfileContent(
        uiState = uiState,
        onRetry = { viewModel.loadUser(userId = "123") },
    )
}

// Keep the actual UI in a separate stateless composable taking plain data + lambdas —
// makes it trivially previewable and testable without a real ViewModel/DI graph.
@Composable
private fun ProfileContent(
    uiState: ProfileUiState,
    onRetry: () -> Unit,
) {
    // render based on uiState, call onRetry() from a button, etc.
}
```

Key conventions:
- One `UiState` data class per screen, exposed as `StateFlow`, collected with
  `collectAsStateWithLifecycle()` (lifecycle-aware — prefer this over plain `collectAsState()`
  in Compose to avoid collecting while the UI isn't visible).
- Split each screen composable into a "stateful" wrapper that pulls in the ViewModel, and a
  "stateless" content composable taking plain parameters — makes `@Preview` composables trivial
  to write without needing a fake ViewModel/DI setup.
- Events flow up (lambdas passed down, called on user interaction), state flows down (UiState
  passed down from ViewModel to composables) — don't let composables call back into the
  ViewModel by holding a reference to it directly if it can be avoided; pass specific lambdas
  instead, for easier previewing/testing.

## Data layer: repository pattern

```kotlin
class UserRepository(
    private val userDao: UserDao,
    private val apiService: ApiService,
) {
    suspend fun getUser(userId: String): User {
        // e.g.: try local cache first, fall back to network, or always hit network and cache result
        return userDao.getUser(userId) ?: apiService.fetchUser(userId).also { userDao.insert(it) }
    }
}
```

Repositories are the single source of truth abstraction the UI/domain layer depends on — UI code
should not talk to Room DAOs or Retrofit services directly.

## Dependency injection: Hilt vs. Koin vs. manual

- **Hilt** — Google's recommended DI framework for Android, built on Dagger, compile-time
  validated. Good default for most real apps, especially if the user wants the
  "standard"/officially-recommended approach. Requires annotation processing (KSP) and a bit more
  boilerplate/setup than Koin.
- **Koin** — a popular lighter-weight, runtime DI framework, pure Kotlin, no code generation.
  Good choice if the user wants less build-time complexity or is building something small, or
  explicitly prefers it.
- **Manual DI** (a simple container class passing dependencies through constructors) — perfectly
  reasonable for small apps/prototypes; don't force Hilt's setup overhead onto a throwaway/small
  project unless the user wants it.

Ask or infer from project scale which fits, rather than always defaulting to Hilt for every
request — for a quick single-screen prototype, manual DI or no DI framework at all is often the
right call.

## Navigation

Use **Navigation Compose** (`androidx.navigation:navigation-compose`) for multi-screen Compose
apps — the current standard approach, replacing older Fragment-based Navigation for pure-Compose
apps. See `compose-patterns.md` for a navigation graph example.

## Testing layers

- **ViewModel tests** — test `uiState` emissions given fake/mocked repositories, using
  `kotlinx-coroutines-test` for coroutine scheduling control.
- **Repository tests** — test against fake DAOs/API services, not real Room/network, for speed.
- **Compose UI tests** — use `createComposeRule()` and semantics-based matchers
  (`onNodeWithText`, `onNodeWithContentDescription`) rather than testing implementation
  internals.

Only scaffold tests if the user asks for them or the task's scope clearly warrants it (e.g. a
production-quality deliverable) — don't pad every quick prototype request with a full test suite
unasked.
