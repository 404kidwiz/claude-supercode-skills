# Rails Expert - Technical Reference

## Architecture Patterns

### Modern Rails Application Structure

The following shows a complete model implementation with modern Rails 7+ patterns:

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  belongs_to :customer
  has_many :order_items, dependent: :destroy
  has_many :products, through: :order_items

  # Modern enum with i18n support
  enum :status, { pending: 'pending', processing: 'processing', completed: 'completed', cancelled: 'cancelled' },
       suffix: true

  # Rich validations with custom error messages
  validates :customer, presence: { message: 'Customer is required' }
  validates :order_items, presence: true, length: { minimum: 1, message: 'Order must contain at least one item' }
  validates :total_amount, numericality: { greater_than: 0 }, allow_nil: true

  # Scopes with proper chaining
  scope :recent, -> { order(created_at: :desc) }
  scope :by_customer, ->(customer_id) { where(customer_id:) }
  scope :with_status, ->(status) { where(status:) }
  scope :in_date_range, ->(start_date, end_date) { where(created_at: start_date..end_date) }

  # Delegates for cleaner interfaces
  delegate :name, :email, to: :customer, prefix: true

  # Callbacks with proper error handling
  before_save :calculate_total_amount
  after_create :send_confirmation_email
  after_update :log_status_change, if: :saved_change_to_status?

  # Money handling with Rails 7 attributes
  attribute :total_amount_cents, :integer, default: 0

  # Virtual attribute for money formatting
  def total_amount
    Money.new(total_amount_cents || 0)
  end

  def total_amount=(value)
    self.total_amount_cents = Money.from_amount(value).cents
  end

  # Business logic methods
  def can_transition_to?(new_status)
    status_transitions = {
      'pending' => ['processing', 'cancelled'],
      'processing' => ['completed', 'cancelled'],
      'completed' => [],
      'cancelled' => []
    }
    status_transitions[status]&.include?(new_status.to_s)
  end

  def transition_to!(new_status)
    unless can_transition_to?(new_status)
      raise ArgumentError, "Cannot transition from #{status} to #{new_status}"
    end

    update!(status: new_status)
  end

  # Calculations with caching
  def item_count
    Rails.cache.fetch("order_#{id}_item_count", expires_in: 1.hour) do
      order_items.sum(:quantity)
    end
  end

  private

  def calculate_total_amount
    self.total_amount_cents = order_items.sum { |item| item.quantity * item.unit_price_cents }
  end

  def send_confirmation_email
    OrderMailer.confirmation(self).deliver_later
  end

  def log_status_change
    Rails.logger.info "Order #{id} status changed from #{saved_change_to_status[0]} to #{status}"
  end
end
```

### Service Objects Pattern

```ruby
# app/services/order_service.rb
class OrderService
  include ActiveModel::Model
  include ActiveModel::Attributes
  include ActiveModel::Validations

  attribute :customer_id, :integer
  attribute :items, array: true

  validates :customer_id, presence: true
  validates :items, presence: true, length: { minimum: 1 }

  attr_reader :order

  def initialize(attributes = {})
    super
    @items = build_items_attributes(attributes[:items] || [])
  end

  def create
    return false unless valid?

    ActiveRecord::Base.transaction do
      @order = Order.create!(customer_id:)
      create_order_items
      calculate_and_save_total
      process_payment
      send_notifications
    end

    true
  rescue ActiveRecord::RecordInvalid, ActiveRecord::RecordNotSaved => e
    errors.add(:base, e.message)
    false
  rescue StandardError => e
    errors.add(:base, 'Order creation failed. Please try again.')
    Rails.logger.error "Order creation failed: #{e.message}"
    false
  end

  private

  def build_items_attributes(items_data)
    items_data.map do |item_data|
      {
        product_id: item_data[:product_id],
        quantity: item_data[:quantity],
        unit_price: item_data[:unit_price]
      }
    end
  end

  def create_order_items
    items.each do |item_data|
      product = Product.find(item_data[:product_id])
      
      unless product.available?(item_data[:quantity])
        raise ArgumentError, "Insufficient stock for product #{product.name}"
      end

      order.order_items.create!(
        product: product,
        quantity: item_data[:quantity],
        unit_price: product.price
      )
    end
  end

  def calculate_and_save_total
    total = order.order_items.sum { |item| item.quantity * item.unit_price_cents }
    order.update!(total_amount_cents: total)
  end

  def process_payment
    PaymentProcessor.new(order).process_payment
  end

  def send_notifications
    OrderMailer.confirmation(order).deliver_later
    ProductStockService.new(order.order_items).reserve_stock
  end
end
```

### Payment Processor Pattern

```ruby
# app/services/payment_processor.rb
class PaymentProcessor
  include ActiveModel::Model

  attr_reader :order

  def initialize(order)
    @order = order
  end

  def process_payment
    payment_intent = create_stripe_payment_intent
    order.update!(stripe_payment_intent_id: payment_intent.id)
    
    payment_intent
  rescue Stripe::StripeError => e
    raise PaymentError, "Payment processing failed: #{e.message}"
  end

  def confirm_payment(payment_intent_id)
    payment_intent = Stripe::PaymentIntent.retrieve(payment_intent_id)
    
    if payment_intent.status == 'succeeded'
      order.update!(payment_status: :paid, paid_at: Time.current)
      order.transition_to!(:processing)
    else
      order.update!(payment_status: :failed)
    end
    
    payment_intent
  end

  private

  def create_stripe_payment_intent
    Stripe::PaymentIntent.create(
      amount: order.total_amount_cents,
      currency: 'usd',
      metadata: {
        order_id: order.id,
        customer_id: order.customer_id
      },
      receipt_email: order.customer_email
    )
  end
end
```

## Modern Controllers with Hotwire

```ruby
# app/controllers/orders_controller.rb
class OrdersController < ApplicationController
  before_action :authenticate_user!
  before_action :set_order, only: [:show, :update, :destroy]

  # Index with Turbo Frames for pagination
  def index
    @orders = policy_scope(Order)
                 .includes(:customer, :order_items)
                 .by_customer(params[:customer_id])
                 .with_status(params[:status])
                 .in_date_range(date_range_params)
                 .page(params[:page])
                 .per(20)

    respond_to do |format|
      format.html
      format.turbo_stream { render partial: 'orders', locals: { orders: @orders } }
    end
  end

  def show
    @order_items = @order.order_items.includes(:product)
  end

  def new
    @order_service = OrderService.new
    @products = Product.available.order(:name)
  end

  def create
    @order_service = OrderService.new(order_params)

    if @order_service.create
      redirect_to @order_service.order, notice: 'Order was successfully created.'
    else
      @products = Product.available.order(:name)
      render :new, status: :unprocessable_entity
    end
  end

  def update
    if @order.update(order_update_params)
      respond_to do |format|
        format.html { redirect_to @order, notice: 'Order was successfully updated.' }
        format.turbo_stream { render turbo_stream: turbo_stream.replace(@order, partial: 'orders/order', locals: { order: @order }) }
      end
    else
      respond_to do |format|
        format.html { render :show, status: :unprocessable_entity }
        format.turbo_stream { render turbo_stream: turbo_stream.replace("order_#{@order.id}", partial: 'orders/order_form', locals: { order: @order }) }
      end
    end
  end

  def destroy
    @order.transition_to!(:cancelled)
    redirect_to orders_url, notice: 'Order was successfully cancelled.'
  end

  # Turbo Stream action for real-time status updates
  def update_status
    new_status = params[:status].to_sym
    
    if @order.can_transition_to?(new_status)
      @order.transition_to!(new_status)
      broadcast_order_update
      render turbo_stream: turbo_stream.replace(
        "order_#{@order.id}",
        partial: 'orders/order_row',
        locals: { order: @order }
      )
    else
      render turbo_stream: turbo_stream.replace(
        "order_#{@order.id}_errors",
        partial: 'shared/alert',
        locals: { alert_type: 'error', message: 'Invalid status transition' }
      )
    end
  end

  private

  def set_order
    @order = Order.find(params[:id])
    authorize @order
  end

  def order_params
    params.require(:order_service).permit(:customer_id, items: [:product_id, :quantity, :unit_price])
  end

  def order_update_params
    params.require(:order).permit(:notes)
  end

  def date_range_params
    if params[:date_range].present?
      params[:date_range].values_at(:start_date, :end_date)
    else
      [1.week.ago.to_date, Date.current]
    end
  end

  def broadcast_order_update
    Turbo::StreamsChannel.broadcast_update_to(
      "order_#{@order.id}",
      target: "order_#{@order.id}",
      partial: 'orders/order_row',
      locals: { order: @order }
    )
  end
end
```

## API Resources

```ruby
# app/resources/order_resource.rb
class OrderResource < ApplicationResource
  attributes :id, :status, :total_amount, :created_at, :updated_at
  
  has_one :customer
  has_many :order_items
  
  attribute :formatted_total_amount do
    number_to_currency(object.total_amount)
  end
  
  attribute :item_count do
    object.order_items.sum(:quantity)
  end
  
  filter :status, apply: ->(records, value) {
    records.where(status: value) if value.present?
  }
  
  filter :customer_id, apply: ->(records, value) {
    records.where(customer_id: value) if value.present?
  }
  
  filter :date_range, apply: ->(records, value) {
    if value.present? && value[:start_date].present? && value[:end_date].present?
      records.where(created_at: value[:start_date]..value[:end_date])
    end
  }
  
  sort :created_at, :total_amount
end
```

## Background Jobs

```ruby
# app/jobs/order_confirmation_job.rb
class OrderConfirmationJob < ApplicationJob
  queue_as :default
  
  def perform(order)
    # Send email confirmation
    OrderMailer.confirmation(order).deliver_now
    
    # Update inventory
    order.order_items.each do |item|
      product = item.product
      product.decrement!(:available_quantity, item.quantity)
      
      # Create inventory record
      InventoryTransaction.create!(
        product: product,
        quantity: -item.quantity,
        transaction_type: :order,
        reference_id: order.id
      )
    end
    
    # Trigger webhooks
    OrderWebhookService.new(order).trigger_creation_webhooks
  rescue StandardError => e
    Rails.logger.error "Failed to process order confirmation for order #{order.id}: #{e.message}"
    raise
  end
end

# app/jobs/order_processing_job.rb
class OrderProcessingJob < ApplicationJob
  queue_as :high_priority
  
  def perform(order_id)
    order = Order.find(order_id)
    
    # Process payment
    PaymentProcessor.new(order).process_payment
    
    # Update order status
    order.transition_to!(:processing)
    
    # Send notifications
    OrderMailer.processing(order).deliver_later
    OrderStatusNotificationService.new(order).notify_status_change
  end
end
```

## GraphQL API

```ruby
# app/graphql/types/query_type.rb
module Types
  class QueryType < Types::BaseObject
    field :orders, [Types::OrderType], null: false do
      argument :customer_id, ID, required: false
      argument :status, String, required: false
      argument :limit, Integer, required: false, default_value: 20
      argument :offset, Integer, required: false, default_value: 0
    end
    
    def orders(customer_id: nil, status: nil, limit:, offset:)
      orders = Order.includes(:customer, :order_items)
      
      orders = orders.where(customer_id:) if customer_id.present?
      orders = orders.where(status:) if status.present?
      
      orders.limit(limit).offset(offset)
    end
    
    field :order, Types::OrderType, null: true do
      argument :id, ID, required: true
    end
    
    def order(id:)
      Order.find(id)
    end
  end
end

# app/graphql/types/order_type.rb
module Types
  class OrderType < Types::BaseObject
    field :id, ID, null: false
    field :status, String, null: false
    field :total_amount, Float, null: false
    field :formatted_total_amount, String, null: false
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    field :customer, Types::CustomerType, null: false
    field :order_items, [Types::OrderItemType], null: false
    
    def formatted_total_amount
      number_to_currency(object.total_amount)
    end
  end
end
```
