# Rails Expert - Code Examples & Patterns

## Hotwire Implementation

### Turbo Frame Views

```erb
<!-- app/views/orders/index.html.erb -->
<div class="container mx-auto px-4 py-8">
  <h1 class="text-3xl font-bold mb-8">Orders</h1>
  
  <!-- Filters with Turbo Frame -->
  <%= turbo_frame_tag "orders_filters" do %>
    <%= render 'filters', orders: @orders %>
  <% end %>

  <!-- Orders table with Turbo Frame -->
  <%= turbo_frame_tag "orders_table" do %>
    <%= render 'orders_table', orders: @orders %>
  <% end %>
</div>

<!-- app/views/orders/_filters.html.erb -->
<div class="bg-white p-6 rounded-lg shadow mb-6">
  <%= form_with url: orders_path, method: :get, class: "flex flex-wrap gap-4", data: { turbo_frame: "orders_table" } do |form| %>
    <div class="flex-1 min-w-[200px]">
      <%= form.select :customer_id, 
                     options_from_collection_for_select(Customer.all, :id, :name, params[:customer_id]),
                     { prompt: "All Customers" },
                     { class: "w-full px-3 py-2 border border-gray-300 rounded-md" } %>
    </div>
    
    <div class="flex-1 min-w-[200px]">
      <%= form.select :status,
                     Order.statuses.keys.map { |s| [s.titleize, s] },
                     { prompt: "All Statuses" },
                     { class: "w-full px-3 py-2 border border-gray-300 rounded-md" } %>
    </div>
    
    <div class="flex-1 min-w-[200px]">
      <%= form.date_field :date_range, name: "date_range[start_date]", value: @date_range&.first, 
                         class: "w-full px-3 py-2 border border-gray-300 rounded-md" %>
    </div>
    
    <div class="flex-1 min-w-[200px]">
      <%= form.date_field :date_range, name: "date_range[end_date]", value: @date_range&.last,
                         class: "w-full px-3 py-2 border border-gray-300 rounded-md" %>
    </div>
    
    <div>
      <%= form.submit "Filter Orders", class: "px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700" %>
    </div>
  <% end %>
</div>

<!-- app/views/orders/_orders_table.html.erb -->
<div class="bg-white shadow rounded-lg overflow-hidden">
  <table class="min-w-full divide-y divide-gray-200">
    <thead class="bg-gray-50">
      <tr>
        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Order</th>
        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Customer</th>
        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Items</th>
        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Total</th>
        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Status</th>
        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Created</th>
        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Actions</th>
      </tr>
    </thead>
    <tbody class="bg-white divide-y divide-gray-200" id="orders_list">
      <%= render partial: "order_row", collection: orders, as: :order %>
    </tbody>
  </table>
  
  <!-- Pagination with Turbo Frame -->
  <%= turbo_frame_tag "pagination" do %>
    <%= paginate orders %>
  <% end %>
</div>

<!-- app/views/orders/_order_row.html.erb -->
<tr id="order_<%= order.id %>">
  <td class="px-6 py-4 whitespace-nowrap">
    <%= link_to "##{order.id}", order_path(order), 
                class: "text-blue-600 hover:text-blue-900 font-medium",
                data: { turbo_frame: "_top" } %>
  </td>
  <td class="px-6 py-4 whitespace-nowrap">
    <%= order.customer_name %>
  </td>
  <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
    <%= order.item_count %>
  </td>
  <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
    <%= number_to_currency(order.total_amount) %>
  </td>
  <td class="px-6 py-4 whitespace-nowrap">
    <%= render 'status_badge', status: order.status %>
  </td>
  <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
    <%= order.created_at.strftime("%m/%d/%Y") %>
  </td>
  <td class="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
    <div class="flex justify-end space-x-2">
      <%= render 'order_actions', order: order %>
    </div>
  </td>
</tr>

<!-- app/views/orders/_order_actions.html.erb -->
<% if order.pending? %>
  <%= button_to "Process", update_status_order_path(order, status: :processing), 
                method: :patch,
                class: "px-3 py-1 bg-yellow-600 text-white rounded hover:bg-yellow-700 text-xs",
                form: { data: { turbo_confirm: "Are you sure?" } } %>
<% elsif order.processing? %>
  <%= button_to "Complete", update_status_order_path(order, status: :completed), 
                method: :patch,
                class: "px-3 py-1 bg-green-600 text-white rounded hover:bg-green-700 text-xs",
                form: { data: { turbo_confirm: "Are you sure?" } } %>
<% end %>

<%= link_to "Cancel", order_path(order), 
            method: :delete,
            class: "px-3 py-1 bg-red-600 text-white rounded hover:bg-red-700 text-xs",
            data: { turbo_confirm: "Are you sure you want to cancel this order?" } %>
```

### Stimulus Controllers

```javascript
// app/javascript/controllers/order_status_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["statusBadge", "statusSelect", "orderActions"]
  static values = { orderId: String }

  connect() {
    console.log(`Order status controller connected for order ${this.orderIdValue}`)
  }

  async updateStatus(event) {
    event.preventDefault()
    
    const newStatus = event.currentTarget.dataset.status
    const confirmMessage = this.getConfirmMessage(newStatus)
    
    if (!confirm(confirmMessage)) return
    
    try {
      const response = await this.updateOrderStatus(newStatus)
      await this.handleStatusUpdate(response, newStatus)
    } catch (error) {
      this.handleError(error)
    }
  }

  async updateOrderStatus(status) {
    const formData = new FormData()
    formData.append('status', status)
    
    const response = await fetch(`/orders/${this.orderIdValue}/update_status`, {
      method: 'POST',
      headers: {
        'X-CSRF-Token': this.getMetaValue('csrf-token'),
        'Accept': 'text/vnd.turbo-stream.html'
      },
      body: formData
    })
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }
    
    return response
  }

  async handleStatusUpdate(response, newStatus) {
    const turboStream = await response.text()
    Turbo.renderStreamMessage(turboStream)
    
    // Update status badge
    this.updateStatusBadge(newStatus)
    
    // Update actions
    this.updateOrderActions(newStatus)
    
    // Show notification
    this.showNotification(`Order status updated to ${newStatus}`, 'success')
  }

  updateStatusBadge(status) {
    if (this.hasStatusBadgeTarget) {
      this.statusBadgeTarget.className = this.getBadgeClasses(status)
      this.statusBadgeTarget.textContent = this.formatStatus(status)
    }
  }

  getBadgeClasses(status) {
    const classes = {
      pending: 'px-2 py-1 text-xs font-medium rounded-full bg-yellow-100 text-yellow-800',
      processing: 'px-2 py-1 text-xs font-medium rounded-full bg-blue-100 text-blue-800',
      completed: 'px-2 py-1 text-xs font-medium rounded-full bg-green-100 text-green-800',
      cancelled: 'px-2 py-1 text-xs font-medium rounded-full bg-red-100 text-red-800'
    }
    return classes[status] || classes.pending
  }

  getConfirmMessage(status) {
    const messages = {
      processing: 'Are you sure you want to process this order?',
      completed: 'Are you sure you want to complete this order?',
      cancelled: 'Are you sure you want to cancel this order?'
    }
    return messages[status] || 'Are you sure?'
  }

  formatStatus(status) {
    return status.charAt(0).toUpperCase() + status.slice(1)
  }

  handleError(error) {
    console.error('Status update failed:', error)
    this.showNotification('Failed to update order status', 'error')
  }

  showNotification(message, type) {
    const notification = document.createElement('div')
    notification.className = `fixed top-4 right-4 p-4 rounded-md shadow-lg ${
      type === 'success' ? 'bg-green-100 text-green-800' : 'bg-red-100 text-red-800'
    } z-50`
    notification.textContent = message
    document.body.appendChild(notification)
    
    setTimeout(() => {
      notification.remove()
    }, 3000)
  }

  getMetaValue(name) {
    const element = document.head.querySelector(`meta[name="${name}"]`)
    return element ? element.getAttribute('content') : null
  }
}

// app/javascript/controllers/order_form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["itemsContainer", "totalAmount", "addItemButton"]
  static values = { itemTemplate: String, itemIdCounter: Number }

  connect() {
    this.calculateTotal()
  }

  addItem(event) {
    event.preventDefault()
    
    const template = this.itemTemplateValue.replace(/NEW_ITEM_INDEX/g, this.itemIdCounterValue)
    const tempDiv = document.createElement('div')
    tempDiv.innerHTML = template
    const newItem = tempDiv.firstElementChild
    
    this.itemsContainerTarget.appendChild(newItem)
    this.itemIdCounterValue++
    this.calculateTotal()
  }

  removeItem(event) {
    const item = event.target.closest('.order-item')
    item.remove()
    this.calculateTotal()
  }

  calculateTotal() {
    let total = 0
    
    this.itemsContainerTarget.querySelectorAll('.order-item').forEach(item => {
      const quantity = parseInt(item.querySelector('[data-order-form-target="quantity"]').value) || 0
      const unitPrice = parseFloat(item.querySelector('[data-order-form-target="unitPrice"]').value) || 0
      total += quantity * unitPrice
    })
    
    this.totalAmountTarget.textContent = `$${total.toFixed(2)}`
  }

  quantityChanged(event) {
    this.calculateTotal()
  }

  priceChanged(event) {
    this.calculateTotal()
  }
}
```

## Testing Strategies

### Modern Rails Testing

```ruby
# test/services/order_service_test.rb
require "test_helper"

class OrderServiceTest < ActiveSupport::TestCase
  def setup
    @customer = create(:customer)
    @product = create(:product, :in_stock, price: 25.00, available_quantity: 10)
  end

  def test_creates_order_successfully
    service = OrderService.new(
      customer_id: @customer.id,
      items: [
        { product_id: @product.id, quantity: 2, unit_price: 25.00 }
      ]
    )

    assert service.create
    assert service.order.persisted?
    assert_equal @customer.id, service.order.customer_id
    assert_equal 1, service.order.order_items.count
    assert_equal 50.00, service.order.total_amount
    assert_equal 'pending', service.order.status
  end

  def test_fails_with_invalid_customer
    service = OrderService.new(
      customer_id: 99999,
      items: [
        { product_id: @product.id, quantity: 1, unit_price: 25.00 }
      ]
    )

    assert_not service.create
    assert_includes service.errors[:customer_id], "can't be blank"
  end

  def test_fails_with_insufficient_stock
    service = OrderService.new(
      customer_id: @customer.id,
      items: [
        { product_id: @product.id, quantity: 15, unit_price: 25.00 }
      ]
    )

    assert_not service.create
    assert service.errors[:base].include?("Insufficient stock for product #{@product.name}")
  end

  def test_validates_minimal_items
    service = OrderService.new(customer_id: @customer.id, items: [])

    assert_not service.create
    assert_includes service.errors[:items], "is too short (minimum is 1 character)"
  end

  private

  def create(name, traits = {})
    FactoryBot.create(name, *traits)
  end
end

# test/integration/orders_test.rb
require "test_helper"

class OrdersTest < ActionDispatch::IntegrationTest
  def setup
    @customer = create(:customer)
    @product = create(:product, :in_stock, price: 50.00, available_quantity: 5)
    sign_in @customer
  end

  test "can create order with valid data" do
    visit new_order_path
    
    fill_in "Customer", with: @customer.id
    select @product.name, from: "Product"
    fill_in "Quantity", with: "2"
    click_button "Add Item"
    
    click_button "Create Order"
    
    assert_selector ".notice", text: "Order was successfully created"
    assert_text "50.00" # Total amount
  end

  test "cannot create order with insufficient stock" do
    visit new_order_path
    
    fill_in "Customer", with: @customer.id
    select @product.name, from: "Product"
    fill_in "Quantity", with: "10" # More than available
    click_button "Add Item"
    
    click_button "Create Order"
    
    assert_selector ".alert", text: "Insufficient stock"
  end

  test "can update order status via turbo stream" do
    order = create(:order, :pending, customer: @customer)
    
    visit orders_path
    
    within "#order_#{order.id}" do
      click_button "Process"
    end
    
    assert_selector ".notice", text: "Order status updated to processing"
    order.reload
    assert_equal "processing", order.status
  end

  test "pagination works with turbo frames" do
    create_list(:order, 25, customer: @customer)
    
    visit orders_path
    
    assert_selector "tbody tr", count: 20 # Default per page
    
    click_link "2"
    
    assert_selector "tbody tr", count: 5 # Remaining orders
    assert_text "Page 2"
  end

  test "filters work with turbo frames" do
    pending_orders = create_list(:order, 3, :pending, customer: @customer)
    completed_orders = create_list(:order, 2, :completed, customer: @customer)
    
    visit orders_path
    
    select "pending", from: "Status"
    click_button "Filter Orders"
    
    assert_selector "tbody tr", count: 3
    pending_orders.each { |order| assert_text order.id }
    completed_orders.each { |order| assert_no_text order.id }
  end

  private

  def create(name, traits = {})
    FactoryBot.create(name, *traits)
  end
end

# test/system/orders_system_test.rb
require "application_system_test_case"

class OrdersSystemTest < ApplicationSystemTestCase
  def setup
    @customer = create(:customer, email: "customer@example.com", password: "password")
    @product = create(:product, :in_stock, price: 30.00, available_quantity: 10)
  end

  test "complete order workflow with hotwire" do
    sign_in @customer
    visit orders_path
    
    click_link "New Order"
    
    # Add multiple items
    2.times do
      select @product.name, from: "Product"
      fill_in "Quantity", with: "2"
      click_button "Add Item"
      sleep 0.1 # Wait for Turbo Frame to update
    end
    
    # Check total calculation
    assert_text "$120.00" # 2 items * 2 quantity * $30 each
    
    click_button "Create Order"
    
    # Should redirect to order show page
    assert_current_path(order_path(Order.last))
    assert_text "Order was successfully created"
    
    # Test status update
    click_button "Process"
    assert_text "Order status updated to processing"
    
    click_button "Complete"
    assert_text "Order status updated to completed"
    
    # Verify order is completed
    assert_selector ".badge.bg-green-100", text: "completed"
  end

  test "real-time status updates with turbo streams" do
    order = create(:order, :pending, customer: @customer)
    
    sign_in @customer
    visit order_path(order)
    
    # Simulate status update from another session
    using_session(:admin) do
      admin = create(:user, :admin)
      sign_in admin
      
      visit orders_path
      within "#order_#{order.id}" do
        click_button "Process"
      end
      assert_text "processing", count: 2
    end
    
    # Check if status is updated in customer session
    within "#order_#{order.id}" do
      assert_text "processing"
    end
  end

  test "form validation works in real-time" do
    sign_in @customer
    visit new_order_path
    
    # Try to submit empty form
    click_button "Create Order"
    
    assert_text "Customer is required"
    assert_text "Order must contain at least one item"
    
    # Add valid customer
    fill_in "Customer", with: @customer.id
    
    # Try to submit without items
    click_button "Create Order"
    
    assert_no_text "Customer is required"
    assert_text "Order must contain at least one item"
  end

  private

  def create(name, traits = {})
    FactoryBot.create(name, *traits)
  end
end
```

## Example Use Cases

### Example 1: Hotwire-Powered Real-Time Dashboard

**Scenario:** Need to add real-time updates to a legacy Rails admin panel.

**Implementation:**
1. Replaced jQuery with Turbo Streams
2. Implemented Action Cable for live updates
3. Created Turbo Frames for partial page updates
4. Added Stimulus controllers for interactive elements
5. Reduced page reloads by 90%

**Results:**
- Dashboard updates in real-time (<100ms latency)
- Eliminated full page refreshes
- Improved perceived performance dramatically
- User satisfaction increased 60%

### Example 2: Database Optimization at Scale

**Scenario:** A Rails app with 10M+ records experiencing N+1 query issues.

**Implementation:**
1. Audited all queries with Query Monitor
2. Implemented eager loading (includes/preload)
3. Added counter caches where needed
4. Optimized database indexes
5. Implemented query result caching

**Results:**
- Average page load reduced from 2.3s to 280ms
- Database CPU reduced by 65%
- Query count reduced from 150+ to <20 per page
- Application scales 5x better under load

### Example 3: Modular Rails Architecture

**Scenario:** Monolithic Rails app becoming unmaintainable.

**Implementation:**
1. Extracted domain logic into Service Objects
2. Created Query Objects for complex database queries
3. Implemented Form Objects for complex validations
4. Used View Components for reusable UI
5. Established clear module boundaries

**Results:**
- Code organization improved significantly
- Testing become easier with focused objects
- Developer productivity increased 40%
- Onboarding time reduced by 50%
