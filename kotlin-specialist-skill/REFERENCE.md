# Kotlin Specialist - Technical Reference

This document contains detailed workflows, technical specifications, and advanced patterns for Kotlin development.

## Workflow: Implement Coroutines-Based Data Layer with Flow

**Goal:** Build reactive repository pattern with StateFlow for UI state management.

### Step 1: Define Data Models

```kotlin
import kotlinx.serialization.Serializable

@Serializable
data class Product(
    val id: String,
    val name: String,
    val price: Double,
    val stock: Int
)

sealed class UiState<out T> {
    object Idle : UiState<Nothing>()
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}
```

### Step 2: Create Repository with Flow

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

class ProductRepository(
    private val apiClient: ApiClient,
    private val dispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    private val _products = MutableStateFlow<UiState<List<Product>>>(UiState.Idle)
    val products: StateFlow<UiState<List<Product>>> = _products.asStateFlow()
    
    private val _selectedProduct = MutableStateFlow<Product?>(null)
    val selectedProduct: StateFlow<Product?> = _selectedProduct.asStateFlow()
    
    // Search products with debounce
    fun searchProducts(query: String): Flow<List<Product>> = flow {
        delay(300) // Debounce
        val result = apiClient.searchProducts(query)
        emit(result.getOrDefault(emptyList()))
    }.flowOn(dispatcher)
    
    suspend fun loadProducts() {
        _products.value = UiState.Loading
        
        withContext(dispatcher) {
            apiClient.getProducts()
                .onSuccess { data ->
                    _products.value = UiState.Success(data)
                }
                .onFailure { error ->
                    _products.value = UiState.Error(error.message ?: "Unknown error")
                }
        }
    }
    
    fun observeProduct(productId: String): Flow<Product?> = flow {
        while (currentCoroutineContext().isActive) {
            val product = apiClient.getProduct(productId).getOrNull()
            emit(product)
            delay(5000) // Poll every 5 seconds
        }
    }.flowOn(dispatcher)
    
    fun selectProduct(product: Product) {
        _selectedProduct.value = product
    }
}
```

### Step 3: Create ViewModel (Android)

```kotlin
// Android ViewModel
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope

class ProductViewModel(
    private val repository: ProductRepository
) : ViewModel() {
    
    val products: StateFlow<UiState<List<Product>>> = repository.products
    val selectedProduct: StateFlow<Product?> = repository.selectedProduct
    
    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()
    
    val searchResults: StateFlow<List<Product>> = searchQuery
        .debounce(300)
        .distinctUntilChanged()
        .flatMapLatest { query ->
            if (query.isBlank()) flowOf(emptyList())
            else repository.searchProducts(query)
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
    
    init {
        loadProducts()
    }
    
    fun loadProducts() {
        viewModelScope.launch {
            repository.loadProducts()
        }
    }
    
    fun search(query: String) {
        _searchQuery.value = query
    }
    
    fun selectProduct(product: Product) {
        repository.selectProduct(product)
    }
}
```

### Step 4: Consume in UI (Jetpack Compose)

```kotlin
import androidx.compose.runtime.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items

@Composable
fun ProductScreen(viewModel: ProductViewModel) {
    val uiState by viewModel.products.collectAsState()
    val searchQuery by viewModel.searchQuery.collectAsState()
    val searchResults by viewModel.searchResults.collectAsState()
    
    Column {
        TextField(
            value = searchQuery,
            onValueChange = { viewModel.search(it) },
            placeholder = { Text("Search products...") }
        )
        
        when (uiState) {
            is UiState.Idle -> Text("Pull to refresh")
            is UiState.Loading -> CircularProgressIndicator()
            is UiState.Success -> {
                val products = (uiState as UiState.Success).data
                LazyColumn {
                    items(products) { product ->
                        ProductItem(product) { viewModel.selectProduct(product) }
                    }
                }
            }
            is UiState.Error -> {
                Text("Error: ${(uiState as UiState.Error).message}")
            }
        }
    }
}
```

### Step 5: Handle Cancellation Properly

```kotlin
class ProductViewModel : ViewModel() {
    private var pollingJob: Job? = null
    
    fun startPolling(productId: String) {
        pollingJob?.cancel() // Cancel previous polling
        pollingJob = viewModelScope.launch {
            repository.observeProduct(productId)
                .catch { e -> 
                    Log.e("ProductVM", "Polling error", e) 
                }
                .collect { product ->
                    // Update UI with latest product data
                }
        }
    }
    
    override fun onCleared() {
        super.onCleared()
        pollingJob?.cancel() // Cleanup
    }
}
```

**Expected Outcome:**
- Reactive UI updates with StateFlow (single source of truth)
- Automatic debouncing and deduplication for search
- Proper lifecycle-aware coroutine scoping
- Clean separation: Repository (data) → ViewModel (business logic) → UI (presentation)

**Verification:**
- Search responds after 300ms pause (no network spam)
- Rotating device preserves state (ViewModel survives config changes)
- Leaving screen cancels polling job (no memory leaks)
- Check with LeakCanary and Profiler

---

## Pattern: expect/actual for Platform-Specific Implementations

**Use case:** Access platform-specific APIs (file system, notifications, sensors) from shared code.

```kotlin
// commonMain/Platform.kt
expect class PlatformStorage {
    suspend fun saveData(key: String, value: String)
    suspend fun loadData(key: String): String?
    suspend fun clearAll()
}

// androidMain/Platform.kt
import android.content.Context
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.core.stringPreferencesKey
import androidx.datastore.preferences.preferencesDataStore
import kotlinx.coroutines.flow.first

actual class PlatformStorage(private val context: Context) {
    private val Context.dataStore by preferencesDataStore(name = "settings")
    
    actual suspend fun saveData(key: String, value: String) {
        context.dataStore.edit { prefs ->
            prefs[stringPreferencesKey(key)] = value
        }
    }
    
    actual suspend fun loadData(key: String): String? {
        return context.dataStore.data.first()[stringPreferencesKey(key)]
    }
    
    actual suspend fun clearAll() {
        context.dataStore.edit { it.clear() }
    }
}

// iosMain/Platform.kt
import platform.Foundation.NSUserDefaults

actual class PlatformStorage {
    private val defaults = NSUserDefaults.standardUserDefaults
    
    actual suspend fun saveData(key: String, value: String) {
        defaults.setObject(value, forKey = key)
    }
    
    actual suspend fun loadData(key: String): String? {
        return defaults.stringForKey(key)
    }
    
    actual suspend fun clearAll() {
        defaults.dictionaryRepresentation().keys.forEach { key ->
            defaults.removeObjectForKey(key as String)
        }
    }
}
```

**Customization points:**
- Add encryption for sensitive data (use `expect/actual` for platform crypto APIs)
- Extend with type-safe keys using inline classes
- Add Flow-based observers for data changes

---

## Pattern: Ktor Custom Plugin

**Use case:** Reusable middleware for logging, authentication, rate limiting.

```kotlin
// Custom request timing plugin
val RequestTimingPlugin = createApplicationPlugin(name = "RequestTiming") {
    onCall { call ->
        val startTime = System.currentTimeMillis()
        
        call.response.pipeline.intercept(ApplicationSendPipeline.After) {
            val duration = System.currentTimeMillis() - startTime
            call.response.headers.append("X-Response-Time", "${duration}ms")
            application.log.info("${call.request.uri} took ${duration}ms")
        }
    }
}

// Usage
fun Application.module() {
    install(RequestTimingPlugin)
}

// Rate limiting plugin
data class RateLimitConfig(
    val maxRequests: Int = 100,
    val windowMs: Long = 60_000
)

val RateLimitPlugin = createApplicationPlugin(
    name = "RateLimit",
    createConfiguration = ::RateLimitConfig
) {
    val requestCounts = mutableMapOf<String, MutableList<Long>>()
    
    onCall { call ->
        val clientId = call.request.headers["X-API-Key"] ?: call.request.origin.remoteHost
        val now = System.currentTimeMillis()
        
        val timestamps = requestCounts.getOrPut(clientId) { mutableListOf() }
        timestamps.removeIf { it < now - pluginConfig.windowMs }
        
        if (timestamps.size >= pluginConfig.maxRequests) {
            call.respond(HttpStatusCode.TooManyRequests, "Rate limit exceeded")
            finish()
        } else {
            timestamps.add(now)
        }
    }
}

// Usage with custom config
install(RateLimitPlugin) {
    maxRequests = 50
    windowMs = 30_000 // 30 seconds
}
```

**Customization points:**
- Use Redis for distributed rate limiting
- Add IP-based vs API-key-based strategies
- Implement exponential backoff headers

---

## Pattern: Structured Concurrency with supervisorScope

**Use case:** Run parallel tasks where one failure doesn't cancel others.

```kotlin
class DataSyncManager {
    suspend fun syncAll(): SyncResult = supervisorScope {
        val userDeferred = async { syncUsers() }
        val productsDeferred = async { syncProducts() }
        val ordersDeferred = async { syncOrders() }
        
        val userResult = runCatching { userDeferred.await() }
        val productResult = runCatching { productsDeferred.await() }
        val orderResult = runCatching { ordersDeferred.await() }
        
        SyncResult(
            users = userResult.getOrNull(),
            products = productResult.getOrNull(),
            orders = orderResult.getOrNull(),
            errors = listOfNotNull(
                userResult.exceptionOrNull(),
                productResult.exceptionOrNull(),
                orderResult.exceptionOrNull()
            )
        )
    }
    
    private suspend fun syncUsers(): List<User> = withContext(Dispatchers.IO) {
        // Sync logic
    }
    
    // ... other sync methods
}

data class SyncResult(
    val users: List<User>?,
    val products: List<Product>?,
    val orders: List<Order>?,
    val errors: List<Throwable>
) {
    val isFullSuccess: Boolean get() = errors.isEmpty()
    val isPartialSuccess: Boolean get() = errors.isNotEmpty() && 
        (users != null || products != null || orders != null)
}
```

**Customization points:**
- Use `coroutineScope` instead if any failure should cancel all
- Add retry logic with exponential backoff
- Implement progress tracking with SharedFlow

---

## Dispatcher Guidelines

| Dispatcher | Use For | Thread Pool |
|------------|---------|-------------|
| **Dispatchers.Main** | UI updates only | Main/UI thread |
| **Dispatchers.IO** | Network, database, file I/O | Up to 64 threads |
| **Dispatchers.Default** | CPU-intensive work | Number of CPU cores |
| **Dispatchers.Unconfined** | Advanced use only (testing) | No thread pool |

**Best Practice:**
```kotlin
// CORRECT: Use appropriate dispatchers
viewModelScope.launch {
    val data = withContext(Dispatchers.IO) {
        database.query() // I/O operations on IO dispatcher
    }
    
    val result = withContext(Dispatchers.Default) {
        heavyComputation(data) // CPU work on Default dispatcher
    }
    
    // Automatically back on Main dispatcher
    updateUI(result)
}
```
