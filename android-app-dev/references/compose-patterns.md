# Jetpack Compose patterns

## Basic screen scaffold with Material 3

```kotlin
@Composable
fun HomeScreen(
    onNavigateToDetail: (String) -> Unit,
) {
    Scaffold(
        topBar = {
            TopAppBar(title = { Text("Home") })
        },
    ) { innerPadding ->
        LazyColumn(
            modifier = Modifier
                .padding(innerPadding)
                .fillMaxSize(),
            contentPadding = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp),
        ) {
            items(items = itemList, key = { it.id }) { item ->
                ItemCard(item = item, onClick = { onNavigateToDetail(item.id) })
            }
        }
    }
}
```

Always pass a stable `key` to `items()` in lazy lists when items can be reordered/inserted/
removed — without it, Compose may misattribute item state/animations across recompositions.

## State hoisting

```kotlin
@Composable
fun SearchBar(
    query: String,
    onQueryChange: (String) -> Unit,
) {
    TextField(value = query, onValueChange = onQueryChange, placeholder = { Text("Search") })
}

// Caller owns the actual state:
@Composable
fun SearchScreen() {
    var query by rememberSaveable { mutableStateOf("") }
    SearchBar(query = query, onQueryChange = { query = it })
}
```

Prefer `rememberSaveable` over plain `remember` for UI state that should survive configuration
changes (rotation) and process death, unless the state is something you can cheaply recompute
(then `remember` alone is fine).

## Navigation graph (Navigation Compose)

```kotlin
@Composable
fun AppNavHost(navController: NavHostController = rememberNavController()) {
    NavHost(navController = navController, startDestination = "home") {
        composable("home") {
            HomeScreen(onNavigateToDetail = { id -> navController.navigate("detail/$id") })
        }
        composable(
            route = "detail/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.StringType }),
        ) { backStackEntry ->
            val itemId = backStackEntry.arguments?.getString("itemId") ?: return@composable
            DetailScreen(itemId = itemId, onBack = { navController.popBackStack() })
        }
    }
}
```

For type-safe navigation (avoiding hand-built string routes), check current Navigation Compose
docs for the `@Serializable` route object pattern — this has become the recommended approach in
recent Navigation Compose versions, replacing hand-written string route templates; verify the
exact current API since it's evolved across releases.

## Material 3 theming

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true, // Material You dynamic color, Android 12+
    content: @Composable () -> Unit,
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S ->
            if (darkTheme) dynamicDarkColorScheme(LocalContext.current)
            else dynamicLightColorScheme(LocalContext.current)
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        content = content,
    )
}
```

Gate `dynamicColor` behind an SDK version check — dynamic color APIs require API 31+ (Android
12).

## Previews

```kotlin
@Preview(showBackground = true)
@Composable
private fun HomeScreenPreview() {
    AppTheme {
        HomeScreenContent(uiState = ProfileUiState(userName = "Preview User"))
    }
}
```

Write previews against the stateless "Content" composable (see `architecture-guide.md`), not the
stateful wrapper that requires a real ViewModel — keeps previews fast and dependency-free.
Add `@Preview(uiMode = Configuration.UI_MODE_NIGHT_YES)` alongside a light-mode preview to check
dark theme, and consider a landscape/large-screen preview for anything where layout adapts.

## Common layout recipes

- `Column`/`Row` + `Arrangement.spacedBy(Xdp)` for consistent gaps instead of manual `Spacer`s
  between every item.
- `Modifier.weight()` inside `Row`/`Column` for proportional sizing between siblings.
- `BoxWithConstraints` or `Modifier.windowInsetsPadding` for edge-to-edge / adaptive layouts —
  check current guidance on edge-to-edge display (enabled by default in recent Android versions)
  since insets handling has shifted across API levels.
- For adaptive layouts across phone/tablet/foldable, check
  `androidx.compose.material3.adaptive` (Material 3 Adaptive library) rather than hand-rolling
  breakpoints, if the app needs to support larger screens well.

## Side-effects

Use the correct Compose side-effect API for the situation rather than reaching for `LaunchedEffect`
for everything:
- `LaunchedEffect(key)` — run a suspend function tied to a composable's lifecycle, re-launched
  when `key` changes (e.g. triggering a one-time load when a screen's ID argument changes).
- `DisposableEffect(key)` — for side effects needing explicit cleanup (registering/unregistering
  a listener).
- `rememberCoroutineScope()` — to launch coroutines from event callbacks (e.g. a button's
  `onClick`) rather than from composition itself.
- Avoid triggering side effects directly in a composable's body (outside these APIs) — that runs
  on every recomposition, not just meaningful state changes.
