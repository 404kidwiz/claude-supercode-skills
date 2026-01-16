# C++ Professional - Technical Reference

## Architecture Patterns

### Modern C++ Project Structure

```cpp
// order.hpp - Interface
#pragma once

#include <uuid/uuid.h>
#include <vector>
#include <memory>
#include <chrono>
#include <expected>
#include <string_view>
#include <ranges>
#include <concepts>

namespace order_system {

enum class OrderStatus : uint8_t {
    Pending = 0,
    Processing = 1,
    Completed = 2,
    Cancelled = 3
};

template<typename T>
concept Numeric = std::is_arithmetic_v<T>;

struct OrderItem {
    std::string product_id;
    int quantity;
    double unit_price;
    
    constexpr double total() const noexcept {
        return quantity * unit_price;
    }
};

class Order {
public:
    Order(std::string customer_id, std::vector<OrderItem> items);
    
    [[nodiscard]] const std::string& id() const noexcept { return id_; }
    [[nodiscard]] const std::string& customer_id() const noexcept { return customer_id_; }
    [[nodiscard]] const std::vector<OrderItem>& items() const noexcept { return items_; }
    [[nodiscard]] OrderStatus status() const noexcept { return status_; }
    [[nodiscard]] double total_amount() const noexcept;
    
    void set_status(OrderStatus status) noexcept { status_ = status; }
    [[nodiscard]] std::chrono::system_clock::time_point created_at() const noexcept { return created_at_; }
    
    // Constexpr constructor for compile-time creation
    static constexpr Order create_empty() noexcept {
        return Order{};
    }

private:
    Order() = default; // Private for create_empty
    
    std::string id_;
    std::string customer_id_;
    std::vector<OrderItem> items_;
    OrderStatus status_ = OrderStatus::Pending;
    std::chrono::system_clock::time_point created_at_;
};

// Factory function with proper error handling
using OrderResult = std::expected<std::unique_ptr<Order>, std::string>;

OrderResult create_order(std::string_view customer_id, 
                        std::span<const OrderItem> items);

} // namespace order_system
```

### Template-Based Service Layer

```cpp
// order_service.hpp
#pragma once

#include "order.hpp"
#include <database/connection_pool.hpp>
#include <cache/lru_cache.hpp>
#include <concepts>
#include <mutex>
#include <shared_mutex>

namespace order_system {

template<typename DatabaseConnection>
requires requires(DatabaseConnection conn) {
    { conn.execute_query(std::string_view{}) } -> std::same_as<std::vector<std::unordered_map<std::string, std::string>>>;
    { conn.execute_update(std::string_view{}) } -> std::same_as<int>;
}
class OrderService {
public:
    explicit OrderService(std::shared_ptr<DatabaseConnection> db, 
                         size_t cache_size = 1000)
        : db_(std::move(db))
        , cache_(cache_size) {}

    // Modern function with concepts
    template<Numeric TotalType>
    auto create_order_with_total_check(const std::string& customer_id,
                                       std::vector<OrderItem> items,
                                       TotalType max_total) -> OrderResult {
        const auto total = calculate_total(items);
        
        if (total > max_total) {
            return std::unexpected("Order total exceeds maximum allowed amount");
        }

        return create_order_internal(customer_id, std::move(items));
    }

    // Ranges-based processing
    auto get_orders_by_total_range(double min_total, double max_total) const 
        -> std::vector<Order> {
        
        return orders_
            | std::views::filter([&](const Order& order) {
                return order.total_amount() >= min_total && 
                       order.total_amount() <= max_total;
            })
            | std::ranges::to<std::vector>();
    }

    // Compile-time string formatting with std::format
    std::string generate_order_report(const Order& order) const {
        return std::format(
            "Order Report\n"
            "============\n"
            "Order ID: {}\n"
            "Customer: {}\n"
            "Status: {}\n"
            "Items: {}\n"
            "Total: ${:.2f}\n"
            "Created: {:%Y-%m-%d %H:%M:%S}",
            order.id(),
            order.customer_id(),
            static_cast<int>(order.status()),
            order.items().size(),
            order.total_amount(),
            order.created_at()
        );
    }

private:
    OrderResult create_order_internal(const std::string& customer_id,
                                     std::vector<OrderItem> items);
    
    static constexpr double calculate_total(std::span<const OrderItem> items) noexcept {
        return std::accumulate(items.begin(), items.end(), 0.0,
            [](double acc, const OrderItem& item) {
                return acc + item.total();
            });
    }

    std::shared_ptr<DatabaseConnection> db_;
    mutable LRUCache<std::string, Order> cache_;
    mutable std::shared_mutex cache_mutex_;
    std::vector<Order> orders_;
};

// Type alias for common use case
using PostgreSQLOrderService = OrderService<PostgreSQLConnection>;

} // namespace order_system
```

## High-Performance Data Processing

```cpp
// order_processor.hpp
#pragma once

#include "order.hpp"
#include <vector>
#include <thread>
#include <atomic>
#include <barrier>
#include <latch>
#include <algorithm>
#include <execution>

namespace order_system {

class OrderProcessor {
public:
    explicit OrderProcessor(size_t num_threads = std::thread::hardware_concurrency())
        : num_threads_(num_threads), stop_flag_(false) {}

    // Parallel processing with std::execution policies
    void process_orders_parallel(std::vector<Order>& orders) {
        std::for_each(std::execution::par_unseq, orders.begin(), orders.end(),
            [](Order& order) {
                if (order.status() == OrderStatus::Pending) {
                    process_single_order(order);
                }
            });
    }

    // SIMD-accelerated total calculation
    struct alignas(32) OrderData {
        double quantities[4];
        double prices[4];
        double totals[4];
    };

    double calculate_total_simd(std::span<const OrderItem> items) {
        if (items.size() < 4) {
            return calculate_total_scalar(items);
        }

        double total = 0.0;
        const size_t simd_chunks = (items.size() / 4) * 4;
        
        // Process 4 items at a time using SIMD
        for (size_t i = 0; i < simd_chunks; i += 4) {
            OrderData data;
            
            for (size_t j = 0; j < 4; ++j) {
                data.quantities[j] = items[i + j].quantity;
                data.prices[j] = items[i + j].unit_price;
            }

            // SIMD multiplication
            for (size_t j = 0; j < 4; ++j) {
                data.totals[j] = data.quantities[j] * data.prices[j];
            }

            total += std::accumulate(std::begin(data.totals), std::end(data.totals), 0.0);
        }

        // Process remaining items
        for (size_t i = simd_chunks; i < items.size(); ++i) {
            total += items[i].total();
        }

        return total;
    }

private:
    static void process_single_order(Order& order) {
        std::this_thread::sleep_for(std::chrono::milliseconds(1));
        order.set_status(OrderStatus::Processing);
        order.set_status(OrderStatus::Completed);
    }

    static double calculate_total_scalar(std::span<const OrderItem> items) {
        return std::accumulate(items.begin(), items.end(), 0.0,
            [](double acc, const OrderItem& item) {
                return acc + item.total();
            });
    }

    size_t num_threads_;
    std::vector<std::thread> workers_;
    std::atomic<bool> stop_flag_;
};

} // namespace order_system
```

## Memory Management

### Memory Pool Implementation

```cpp
// memory_pool.hpp
#pragma once

#include <memory>
#include <vector>
#include <cstddef>
#include <mutex>

namespace order_system {

template<typename T, size_t BlockSize = 1024>
class MemoryPool {
public:
    MemoryPool() : blocks_(), free_list_(nullptr) {}
    
    ~MemoryPool() {
        for (auto block : blocks_) {
            ::operator delete(block);
        }
    }

    T* allocate(size_t n) {
        if (n != 1) {
            return static_cast<T*>(::operator new(n * sizeof(T)));
        }

        std::lock_guard<std::mutex> lock(mutex_);
        
        if (!free_list_) {
            allocate_block();
        }

        T* result = reinterpret_cast<T*>(free_list_);
        free_list_ = free_list_->next;
        
        return result;
    }

    void deallocate(T* ptr, size_t n) noexcept {
        if (n != 1) {
            ::operator delete(ptr);
            return;
        }

        std::lock_guard<std::mutex> lock(mutex_);
        
        Node* node = reinterpret_cast<Node*>(ptr);
        node->next = free_list_;
        free_list_ = node;
    }

private:
    struct Node {
        Node* next;
    };

    void allocate_block() {
        char* block = static_cast<char*>(::operator new(BlockSize * sizeof(T)));
        blocks_.push_back(block);
        
        for (size_t i = 0; i < BlockSize; ++i) {
            Node* node = reinterpret_cast<Node*>(block + i * sizeof(T));
            node->next = free_list_;
            free_list_ = node;
        }
    }

    std::vector<void*> blocks_;
    Node* free_list_;
    std::mutex mutex_;
};

} // namespace order_system
```

## Build System Configuration

### CMake with Modern C++20

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(OrderSystem VERSION 1.0.0 LANGUAGES CXX)

# Set C++20 standard
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Compiler-specific optimizations
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    target_compile_options(order_system PRIVATE -O3 -march=native -flto)
    target_link_options(order_system PRIVATE -flto)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    target_compile_options(order_system PRIVATE /O2 /GL)
    target_link_options(order_system PRIVATE /LTCG)
endif()

# Find required packages
find_package(PkgConfig REQUIRED)
find_package(Threads REQUIRED)
find_package(fmt REQUIRED)

# Create main library
add_library(order_system
    src/order.cpp
    src/order_service.cpp
    src/order_processor.cpp
)

target_include_directories(order_system
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
)

target_link_libraries(order_system
    PUBLIC
        fmt::fmt
        Threads::Threads
)

# Add tests
include(CTest)
enable_testing()
add_subdirectory(tests)

# Installation
include(GNUInstallDirs)

install(TARGETS order_system
    EXPORT OrderSystemTargets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(DIRECTORY include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)
```

## Coroutines for Asynchronous Operations

```cpp
#include <coroutine>
#include <future>

namespace order_system {

template<typename T>
struct Promise {
    T value_;
    std::coroutine_handle<> continuation_;
    bool ready_ = false;

    OrderSystemTask get_return_object() {
        return OrderSystemTask{std::coroutine_handle<Promise>::from_promise(*this)};
    }

    std::suspend_always initial_suspend() { return {}; }
    std::suspend_always final_suspend() noexcept { 
        if (continuation_) {
            continuation_.resume();
        }
        return {}; 
    }

    void return_value(T value) {
        value_ = std::move(value);
        ready_ = true;
    }

    void unhandled_exception() { throw; }

    struct awaiter {
        Promise* promise_;
        
        bool await_ready() { return promise_->ready_; }
        
        void await_suspend(std::coroutine_handle<> continuation) {
            promise_->continuation_ = continuation;
        }
        
        T await_resume() { return std::move(promise_->value_); }
    };
};

template<typename T>
struct OrderSystemTask {
    using promise_type = Promise<T>;
    std::coroutine_handle<promise_type> coro_;

    ~OrderSystemTask() { if (coro_) coro_.destroy(); }

    // Move constructor/assignment
    OrderSystemTask(OrderSystemTask&& other) noexcept : coro_(other.coro_) {
        other.coro_ = {};
    }

    OrderSystemTask& operator=(OrderSystemTask&& other) noexcept {
        if (this != &other) {
            if (coro_) coro_.destroy();
            coro_ = other.coro_;
            other.coro_ = {};
        }
        return *this;
    }

    auto operator co_await() {
        return typename promise_type::awaiter{&coro_.promise()};
    }
};

// Usage example
OrderSystemTask<std::vector<Order>> process_orders_async(
    std::shared_ptr<DatabaseConnection> db,
    std::vector<std::string> customer_ids) {
    
    std::vector<Order> results;
    results.reserve(customer_ids.size());

    for (const auto& customer_id : customer_ids) {
        auto orders = co_await db->get_orders_by_customer(customer_id);
        results.insert(results.end(), orders.begin(), orders.end());
    }

    co_return results;
}

} // namespace order_system
```
