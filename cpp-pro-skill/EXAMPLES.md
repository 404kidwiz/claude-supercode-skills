# C++ Professional - Code Examples & Patterns

## Lock-Free Queue Implementation

```cpp
// Lock-free queue for high-performance scenarios
template<typename T>
class LockFreeQueue {
public:
    struct Node {
        std::atomic<Node*> next{nullptr};
        T data;
        
        template<typename... Args>
        explicit Node(Args&&... args) : data(std::forward<Args>(args)...) {}
    };

    LockFreeQueue() : dummy_(new Node{}), head_(&dummy_), tail_(&dummy_) {}
    
    ~LockFreeQueue() {
        while (Node* head = head_.load()) {
            head_.store(head->next);
            delete head;
        }
    }

    void enqueue(T value) {
        Node* new_node = new Node{std::move(value)};
        Node* prev_tail = tail_.exchange(new_node);
        prev_tail->next.store(new_node);
    }

    bool dequeue(T& result) {
        Node* head = head_.load();
        Node* next = head->next.load();
        
        if (!next) return false;
        
        result = std::move(next->data);
        head_.store(next);
        delete head;
        return true;
    }

private:
    Node* dummy_;
    std::atomic<Node*> head_;
    std::atomic<Node*> tail_;
};

using OrderQueue = LockFreeQueue<Order>;
```

## Custom Smart Pointer with Pool Allocation

```cpp
template<typename T>
class PooledPtr {
public:
    explicit PooledPtr(MemoryPool<T>& pool, T* ptr = nullptr)
        : pool_(&pool), ptr_(ptr) {}

    ~PooledPtr() {
        if (ptr_) {
            ptr_->~T();
            pool_->deallocate(ptr_, 1);
        }
    }

    PooledPtr(const PooledPtr&) = delete;
    PooledPtr& operator=(const PooledPtr&) = delete;

    PooledPtr(PooledPtr&& other) noexcept
        : pool_(other.pool_), ptr_(other.ptr_) {
        other.ptr_ = nullptr;
    }

    PooledPtr& operator=(PooledPtr&& other) noexcept {
        if (this != &other) {
            reset();
            pool_ = other.pool_;
            ptr_ = other.ptr_;
            other.ptr_ = nullptr;
        }
        return *this;
    }

    template<typename... Args>
    static PooledPtr make(MemoryPool<T>& pool, Args&&... args) {
        T* ptr = pool.allocate(1);
        new (ptr) T(std::forward<Args>(args)...);
        return PooledPtr(pool, ptr);
    }

    T& operator*() const { return *ptr_; }
    T* operator->() const { return ptr_; }
    T* get() const noexcept { return ptr_; }

    void reset() {
        if (ptr_) {
            ptr_->~T();
            pool_->deallocate(ptr_, 1);
            ptr_ = nullptr;
        }
    }

    explicit operator bool() const noexcept { return ptr_ != nullptr; }

private:
    MemoryPool<T>* pool_;
    T* ptr_;
};
```

## Testing with Google Test

```cpp
// order_service_test.cpp
#include <gtest/gtest.h>
#include "order_service.hpp"
#include "mock_database.hpp"

using namespace order_system;

class OrderServiceTest : public ::testing::Test {
protected:
    void SetUp() override {
        mock_db_ = std::make_shared<MockDatabase>();
        service_ = std::make_unique<OrderService<MockDatabase>>(mock_db_);
    }

    std::shared_ptr<MockDatabase> mock_db_;
    std::unique_ptr<OrderService<MockDatabase>> service_;
};

TEST_F(OrderServiceTest, CreateOrder_Success) {
    // Arrange
    const std::string customer_id = "customer_123";
    const std::vector<OrderItem> items{
        {"product_1", 2, 29.99},
        {"product_2", 1, 49.99}
    };

    EXPECT_CALL(*mock_db_, insert_order(::testing::_))
        .WillOnce(::testing::Return(true));

    // Act
    auto result = service_->create_order_with_total_check(customer_id, items, 200.0);

    // Assert
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(result.value()->customer_id(), customer_id);
    EXPECT_EQ(result.value()->items().size(), 2);
    EXPECT_DOUBLE_EQ(result.value()->total_amount(), 109.97);
}

TEST_F(OrderServiceTest, CreateOrder_ExceedsMaxTotal) {
    // Arrange
    const std::string customer_id = "customer_123";
    const std::vector<OrderItem> items{
        {"product_1", 10, 100.00}
    };

    // Act
    auto result = service_->create_order_with_total_check(customer_id, items, 50.0);

    // Assert
    ASSERT_FALSE(result.has_value());
    EXPECT_STREQ(result.error().c_str(), "Order total exceeds maximum allowed amount");
}

// Property-based testing with custom generators
class OrderPropertyTest : public ::testing::Test {
protected:
    static auto generate_random_items(int count) -> std::vector<OrderItem> {
        std::vector<OrderItem> items;
        items.reserve(count);
        
        std::random_device rd;
        std::mt19937 gen(rd());
        std::uniform_int_distribution<> qty_dist(1, 10);
        std::uniform_real_distribution<> price_dist(0.01, 1000.0);

        for (int i = 0; i < count; ++i) {
            items.emplace_back(
                "product_" + std::to_string(i),
                qty_dist(gen),
                price_dist(gen)
            );
        }
        
        return items;
    }
};

TEST_F(OrderPropertyTest, TotalCalculation_Consistency) {
    for (int test = 0; test < 100; ++test) {
        const auto items = generate_random_items(10);
        
        // Test SIMD vs scalar calculation
        OrderProcessor processor;
        const double simd_total = processor.calculate_total_simd(items);
        const double scalar_total = std::accumulate(items.begin(), items.end(), 0.0,
            [](double acc, const OrderItem& item) {
                return acc + item.total();
            });

        EXPECT_DOUBLE_EQ(simd_total, scalar_total) << "Test iteration: " << test;
    }
}

// Performance benchmarks
TEST(OrderBenchmark, ParallelProcessing) {
    std::vector<Order> orders;
    orders.reserve(10000);
    
    for (int i = 0; i < 10000; ++i) {
        orders.emplace_back(
            "customer_" + std::to_string(i),
            std::vector<OrderItem>{{"product_1", 1, 29.99}}
        );
    }

    OrderProcessor processor(4);
    
    auto start = std::chrono::high_resolution_clock::now();
    processor.process_orders_parallel(orders);
    auto end = std::chrono::high_resolution_clock::now();
    
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "Parallel processing took: " << duration.count() << "ms\n";
    
    // Verify all orders were processed
    EXPECT_TRUE(std::all_of(orders.begin(), orders.end(),
        [](const Order& order) {
            return order.status() == OrderStatus::Completed;
        }));
}
```

## Example Use Cases

### Example 1: High-Performance Trading Engine

**Scenario:** Building a low-latency financial trading engine requiring microsecond-level response times.

**Implementation Approach:**
1. **Lock-Free Architecture**: Used atomic operations and memory ordering for zero-contention data paths
2. **SIMD Optimization**: Implemented vectorized price calculations using AVX-512
3. **Cache Optimization**: Designed data structures for cache-line alignment and prefetching
4. **Coroutine-Based Concurrency**: Used C++20 coroutines for efficient I/O multiplexing

**Performance Results:**
- Order processing latency: 50μs (down from 500μs)
- Throughput: 100K orders/second (up from 20K)
- CPU utilization: Reduced by 40% through better cache locality

### Example 2: Embedded Real-Time Controller

**Scenario:** Developing firmware for a medical device with hard real-time constraints (< 1ms response).

**Implementation Strategy:**
1. **Zero-Allocation Design**: Pre-allocated memory pools, no dynamic allocation in hot paths
2. **constexpr Everything**: Compile-time computation for configuration and validation
3. **Concepts-Based API**: Type-safe interfaces preventing misuse at compile time
4. **Hardware Abstraction**: Portable layer supporting multiple microcontroller platforms

**Key Techniques:**
```cpp
// Compile-time validated configuration
template<RealTimeSystem T>
class Controller {
    static_assert(T::max_latency_ms < 1, "Latency requirement not met");
    
    // Pre-allocated buffer pools
    std::array<Message, 256> message_pool_;
    std::atomic_size_t pool_index_{0};
};
```

### Example 3: Cross-Platform Game Engine Library

**Scenario:** Creating a game engine SDK that compiles to Windows, macOS, Linux, and consoles.

**Architecture Decisions:**
1. **Module-Based Build**: Using C++20 modules for faster compilation and cleaner interfaces
2. **Concept Constraints**: Ensuring platform-specific code meets interface requirements
3. **Modern RAII**: Resource management through smart pointers and RAII wrappers
4. **Error Handling**: Using std::expected for recoverable errors without exceptions

**Results:**
- Compile time reduction: 45% through modules
- Cross-platform compatibility: 95% shared code
- Memory safety: Zero memory-related CVEs in 2 years

## Concepts Example

```cpp
// Using concepts for type-safe templates
template<typename T>
concept Serializable = requires(T t, std::ostream& os) {
    { t.serialize(os) } -> std::same_as<void>;
    { T::deserialize(std::declval<std::istream&>()) } -> std::same_as<T>;
};

template<Serializable T>
void save_to_file(const T& obj, const std::string& filename) {
    std::ofstream file(filename);
    obj.serialize(file);
}

// Compound concepts
template<typename T>
concept OrderComponent = Serializable<T> && requires(T t) {
    { t.id() } -> std::convertible_to<std::string>;
    { t.validate() } -> std::same_as<bool>;
};
```

## Ranges Example

```cpp
// Modern ranges-based data processing
auto get_top_orders_by_value(const std::vector<Order>& orders, size_t n) {
    return orders
        | std::views::filter([](const Order& o) { 
            return o.status() == OrderStatus::Completed; 
        })
        | std::views::transform([](const Order& o) {
            return std::make_pair(o.id(), o.total_amount());
        })
        | std::ranges::to<std::vector>()
        | std::ranges::actions::sort([](auto& a, auto& b) {
            return a.second > b.second;
        })
        | std::views::take(n);
}

// Lazy evaluation with views
auto pending_high_value = orders
    | std::views::filter([](const Order& o) { 
        return o.status() == OrderStatus::Pending && o.total_amount() > 1000; 
    });

// Only evaluate when needed
for (const auto& order : pending_high_value | std::views::take(10)) {
    process(order);
}
```

## std::format Example

```cpp
// Type-safe formatting with std::format
std::string format_order_summary(const Order& order) {
    return std::format(
        "Order #{} | Customer: {} | Items: {} | Total: ${:.2f} | Status: {}",
        order.id(),
        order.customer_id(),
        order.items().size(),
        order.total_amount(),
        magic_enum::enum_name(order.status())
    );
}

// Format with alignment
std::string format_table_row(std::string_view name, double value) {
    return std::format("{:<20} {:>10.2f}", name, value);
}

// Format with localization
std::string format_with_locale(double amount) {
    return std::format(std::locale("en_US.UTF-8"), "{:L}", amount);
}
```
