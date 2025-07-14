# E-Commerce Platform Design

**Tags**: #case-study #e-commerce #scalability #microservices
**Date**: 2024-01-01

## 📝 Overview

Thiết kế e-commerce platform có khả năng xử lý hàng triệu users, millions of products, và thousands of transactions per second. System phải đảm bảo high availability, consistency cho inventory, và excellent user experience.

## 🎯 Requirements Analysis

### **Functional Requirements**
```
User Management:
- User registration/authentication
- User profiles and preferences
- Address management

Product Catalog:
- Product browsing and search
- Category management
- Product recommendations
- Reviews and ratings

Shopping Cart:
- Add/remove items
- Save cart across sessions
- Multiple payment methods

Order Management:
- Order placement and tracking
- Inventory management
- Payment processing
- Shipping coordination

Seller/Admin:
- Product management
- Order fulfillment
- Analytics dashboard
```

### **Non-Functional Requirements**
```python
performance_requirements = {
    "response_time": {
        "product_search": "< 200ms",
        "add_to_cart": "< 100ms", 
        "checkout": "< 500ms",
        "order_status": "< 100ms"
    },
    "throughput": {
        "peak_traffic": "100K concurrent users",
        "search_qps": "50K queries/second",
        "orders_per_day": "1M orders"
    },
    "availability": {
        "uptime": "99.9%",
        "planned_downtime": "< 4 hours/month"
    },
    "scalability": {
        "users": "100M registered users",
        "products": "10M active products",
        "geographic": "Global deployment"
    }
}
```

## 🏗️ High-Level Architecture

### **Microservices Architecture**
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Web/Mobile    │    │   API Gateway   │    │  Load Balancer  │
│   Clients       │───▶│                 │───▶│                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                       │
                        ┌──────────────────────────────┼──────────────────────────────┐
                        │                              │                              │
                ┌───────▼────────┐            ┌───────▼────────┐            ┌───────▼────────┐
                │ User Service   │            │Product Service │            │ Order Service  │
                └────────────────┘            └────────────────┘            └────────────────┘
                        │                              │                              │
                ┌───────▼────────┐            ┌───────▼────────┐            ┌───────▼────────┐
                │   User DB      │            │ Product DB +   │            │   Order DB     │
                │   (PostgreSQL) │            │ Search Engine  │            │   (PostgreSQL) │
                └────────────────┘            │ (Elasticsearch)│            └────────────────┘
                                              └────────────────┘
```

### **Service Breakdown**
```python
class ECommerceArchitecture:
    def __init__(self):
        self.services = {
            "user_service": {
                "responsibilities": ["Authentication", "User profiles", "Preferences"],
                "database": "PostgreSQL",
                "cache": "Redis"
            },
            "product_service": {
                "responsibilities": ["Product catalog", "Search", "Categories"],
                "database": "PostgreSQL + Elasticsearch",
                "cache": "Redis + CDN"
            },
            "inventory_service": {
                "responsibilities": ["Stock management", "Reservations"],
                "database": "PostgreSQL",
                "cache": "Redis"
            },
            "cart_service": {
                "responsibilities": ["Shopping cart", "Session management"],
                "database": "Redis",
                "cache": "In-memory"
            },
            "order_service": {
                "responsibilities": ["Order processing", "Order history"],
                "database": "PostgreSQL",
                "cache": "Redis"
            },
            "payment_service": {
                "responsibilities": ["Payment processing", "Refunds"],
                "database": "PostgreSQL",
                "external": ["Stripe", "PayPal"]
            },
            "shipping_service": {
                "responsibilities": ["Shipping calculation", "Tracking"],
                "database": "PostgreSQL",
                "external": ["FedEx", "UPS", "DHL"]
            },
            "notification_service": {
                "responsibilities": ["Email", "SMS", "Push notifications"],
                "queue": "RabbitMQ",
                "external": ["SendGrid", "Twilio"]
            },
            "recommendation_service": {
                "responsibilities": ["Product recommendations", "ML models"],
                "database": "MongoDB + ML Pipeline",
                "cache": "Redis"
            }
        }
```

## 🛍️ Core Service Implementation

### **Product Service with Search**
```python
from elasticsearch import Elasticsearch
import redis

class ProductService:
    def __init__(self):
        self.db = self.get_postgres_connection()
        self.es = Elasticsearch(['localhost:9200'])
        self.cache = redis.Redis()
        self.cdn_base_url = "https://cdn.ecommerce.com"
    
    def create_product(self, product_data):
        # Validate product data
        if not self.validate_product(product_data):
            raise ValueError("Invalid product data")
        
        # Store in primary database
        product_id = self.db.execute("""
            INSERT INTO products (name, description, price, category_id, seller_id)
            VALUES (%(name)s, %(description)s, %(price)s, %(category_id)s, %(seller_id)s)
            RETURNING id
        """, product_data).fetchone()[0]
        
        # Index in Elasticsearch for search
        es_doc = {
            "id": product_id,
            "name": product_data["name"],
            "description": product_data["description"], 
            "price": product_data["price"],
            "category": self.get_category_name(product_data["category_id"]),
            "tags": product_data.get("tags", []),
            "created_at": datetime.utcnow(),
            "searchable_text": f"{product_data['name']} {product_data['description']}"
        }
        
        self.es.index(index="products", id=product_id, body=es_doc)
        
        # Clear relevant caches
        self.cache.delete(f"category:{product_data['category_id']}:products")
        
        return product_id
    
    def search_products(self, query, filters=None, page=1, size=20):
        # Build Elasticsearch query
        search_body = {
            "query": {
                "bool": {
                    "must": [
                        {
                            "multi_match": {
                                "query": query,
                                "fields": ["name^2", "description", "searchable_text"]
                            }
                        }
                    ],
                    "filter": []
                }
            },
            "sort": [
                {"_score": {"order": "desc"}},
                {"created_at": {"order": "desc"}}
            ],
            "from": (page - 1) * size,
            "size": size,
            "highlight": {
                "fields": {
                    "name": {},
                    "description": {}
                }
            }
        }
        
        # Add filters
        if filters:
            if "category" in filters:
                search_body["query"]["bool"]["filter"].append({
                    "term": {"category": filters["category"]}
                })
            
            if "price_range" in filters:
                search_body["query"]["bool"]["filter"].append({
                    "range": {
                        "price": {
                            "gte": filters["price_range"]["min"],
                            "lte": filters["price_range"]["max"]
                        }
                    }
                })
        
        # Execute search
        response = self.es.search(index="products", body=search_body)
        
        # Format results
        products = []
        for hit in response["hits"]["hits"]:
            product = hit["_source"]
            product["score"] = hit["_score"]
            if "highlight" in hit:
                product["highlights"] = hit["highlight"]
            products.append(product)
        
        return {
            "products": products,
            "total": response["hits"]["total"]["value"],
            "page": page,
            "total_pages": (response["hits"]["total"]["value"] + size - 1) // size
        }
    
    def get_product_details(self, product_id):
        # Try cache first
        cache_key = f"product:{product_id}"
        cached = self.cache.get(cache_key)
        if cached:
            return json.loads(cached)
        
        # Query database
        product = self.db.execute("""
            SELECT p.*, c.name as category_name, s.name as seller_name
            FROM products p
            JOIN categories c ON p.category_id = c.id
            JOIN sellers s ON p.seller_id = s.id
            WHERE p.id = %s AND p.active = true
        """, [product_id]).fetchone()
        
        if not product:
            return None
        
        # Get product images
        images = self.db.execute("""
            SELECT url FROM product_images WHERE product_id = %s ORDER BY sort_order
        """, [product_id]).fetchall()
        
        # Get product reviews summary
        review_stats = self.db.execute("""
            SELECT 
                COUNT(*) as review_count,
                AVG(rating) as avg_rating
            FROM product_reviews 
            WHERE product_id = %s
        """, [product_id]).fetchone()
        
        result = {
            **dict(product),
            "images": [f"{self.cdn_base_url}/{img['url']}" for img in images],
            "review_count": review_stats["review_count"],
            "avg_rating": float(review_stats["avg_rating"]) if review_stats["avg_rating"] else 0
        }
        
        # Cache for 1 hour
        self.cache.setex(cache_key, 3600, json.dumps(result, default=str))
        
        return result

class InventoryService:
    def __init__(self):
        self.db = self.get_postgres_connection()
        self.cache = redis.Redis()
    
    def check_availability(self, product_id, quantity=1):
        # Check cache first for hot products
        cache_key = f"inventory:{product_id}"
        cached_stock = self.cache.get(cache_key)
        
        if cached_stock is not None:
            return int(cached_stock) >= quantity
        
        # Query database
        stock = self.db.execute("""
            SELECT available_quantity FROM inventory 
            WHERE product_id = %s
        """, [product_id]).fetchone()
        
        if stock:
            available = stock["available_quantity"]
            # Cache for 5 minutes
            self.cache.setex(cache_key, 300, available)
            return available >= quantity
        
        return False
    
    def reserve_inventory(self, product_id, quantity, reservation_id):
        """Reserve inventory for order processing"""
        try:
            self.db.begin()
            
            # Lock row for update
            current_stock = self.db.execute("""
                SELECT available_quantity FROM inventory 
                WHERE product_id = %s FOR UPDATE
            """, [product_id]).fetchone()
            
            if not current_stock or current_stock["available_quantity"] < quantity:
                self.db.rollback()
                return False
            
            # Create reservation
            self.db.execute("""
                INSERT INTO inventory_reservations 
                (id, product_id, quantity, created_at, expires_at)
                VALUES (%s, %s, %s, NOW(), NOW() + INTERVAL '15 minutes')
            """, [reservation_id, product_id, quantity])
            
            # Update available quantity
            self.db.execute("""
                UPDATE inventory 
                SET available_quantity = available_quantity - %s
                WHERE product_id = %s
            """, [quantity, product_id])
            
            self.db.commit()
            
            # Update cache
            self.cache.decr(f"inventory:{product_id}", quantity)
            
            return True
            
        except Exception as e:
            self.db.rollback()
            raise
    
    def confirm_reservation(self, reservation_id):
        """Confirm reservation and move to sold"""
        self.db.execute("""
            UPDATE inventory_reservations 
            SET status = 'confirmed' 
            WHERE id = %s
        """, [reservation_id])
    
    def release_reservation(self, reservation_id):
        """Release expired or cancelled reservation"""
        reservation = self.db.execute("""
            SELECT product_id, quantity FROM inventory_reservations 
            WHERE id = %s
        """, [reservation_id]).fetchone()
        
        if reservation:
            # Return inventory
            self.db.execute("""
                UPDATE inventory 
                SET available_quantity = available_quantity + %s
                WHERE product_id = %s
            """, [reservation["quantity"], reservation["product_id"]])
            
            # Update cache
            self.cache.incr(f"inventory:{reservation['product_id']}", 
                          reservation["quantity"])
            
            # Delete reservation
            self.db.execute("""
                DELETE FROM inventory_reservations WHERE id = %s
            """, [reservation_id])
```

### **Cart Service**
```python
class CartService:
    def __init__(self):
        self.redis = redis.Redis()
        self.product_service = ProductService()
        self.inventory_service = InventoryService()
    
    def add_to_cart(self, user_id, product_id, quantity=1):
        # Validate product exists and is available
        if not self.inventory_service.check_availability(product_id, quantity):
            raise ValueError("Product not available")
        
        cart_key = f"cart:{user_id}"
        
        # Get current cart
        cart_data = self.redis.hget(cart_key, product_id)
        current_quantity = int(cart_data) if cart_data else 0
        
        new_quantity = current_quantity + quantity
        
        # Check total availability
        if not self.inventory_service.check_availability(product_id, new_quantity):
            raise ValueError("Not enough stock available")
        
        # Update cart
        self.redis.hset(cart_key, product_id, new_quantity)
        self.redis.expire(cart_key, 30 * 24 * 3600)  # 30 days
        
        # Track analytics
        self.track_cart_event(user_id, "add_to_cart", {
            "product_id": product_id,
            "quantity": quantity
        })
        
        return self.get_cart(user_id)
    
    def get_cart(self, user_id):
        cart_key = f"cart:{user_id}"
        cart_items = self.redis.hgetall(cart_key)
        
        if not cart_items:
            return {"items": [], "total": 0, "item_count": 0}
        
        # Get product details for each item
        items = []
        total_price = 0
        total_items = 0
        
        for product_id, quantity in cart_items.items():
            product_id = product_id.decode()
            quantity = int(quantity)
            
            product = self.product_service.get_product_details(product_id)
            if product:
                item_total = product["price"] * quantity
                items.append({
                    "product_id": product_id,
                    "product": product,
                    "quantity": quantity,
                    "item_total": item_total
                })
                total_price += item_total
                total_items += quantity
        
        return {
            "items": items,
            "total": total_price,
            "item_count": total_items
        }
    
    def update_quantity(self, user_id, product_id, new_quantity):
        if new_quantity <= 0:
            return self.remove_from_cart(user_id, product_id)
        
        # Check availability
        if not self.inventory_service.check_availability(product_id, new_quantity):
            raise ValueError("Not enough stock available")
        
        cart_key = f"cart:{user_id}"
        self.redis.hset(cart_key, product_id, new_quantity)
        
        return self.get_cart(user_id)
    
    def remove_from_cart(self, user_id, product_id):
        cart_key = f"cart:{user_id}"
        self.redis.hdel(cart_key, product_id)
        
        return self.get_cart(user_id)
    
    def clear_cart(self, user_id):
        cart_key = f"cart:{user_id}"
        self.redis.delete(cart_key)
```

### **Order Processing Service**
```python
import uuid
from enum import Enum

class OrderStatus(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    PROCESSING = "processing"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class OrderService:
    def __init__(self):
        self.db = self.get_postgres_connection()
        self.cart_service = CartService()
        self.inventory_service = InventoryService()
        self.payment_service = PaymentService()
        self.notification_service = NotificationService()
    
    def create_order(self, user_id, shipping_address, payment_method):
        try:
            # Start distributed transaction
            order_id = str(uuid.uuid4())
            
            # Get cart contents
            cart = self.cart_service.get_cart(user_id)
            if not cart["items"]:
                raise ValueError("Cart is empty")
            
            # Reserve inventory for all items
            reservations = []
            for item in cart["items"]:
                reservation_id = str(uuid.uuid4())
                if not self.inventory_service.reserve_inventory(
                    item["product_id"], item["quantity"], reservation_id
                ):
                    # Rollback previous reservations
                    for res_id in reservations:
                        self.inventory_service.release_reservation(res_id)
                    raise ValueError(f"Cannot reserve {item['product']['name']}")
                reservations.append(reservation_id)
            
            # Calculate order total
            subtotal = cart["total"]
            tax = subtotal * 0.08  # 8% tax
            shipping_cost = self.calculate_shipping(cart["items"], shipping_address)
            total = subtotal + tax + shipping_cost
            
            # Create order record
            self.db.execute("""
                INSERT INTO orders 
                (id, user_id, status, subtotal, tax, shipping_cost, total, 
                 shipping_address, created_at)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, NOW())
            """, [order_id, user_id, OrderStatus.PENDING.value, 
                  subtotal, tax, shipping_cost, total, 
                  json.dumps(shipping_address)])
            
            # Create order items
            for i, item in enumerate(cart["items"]):
                self.db.execute("""
                    INSERT INTO order_items 
                    (order_id, product_id, quantity, unit_price, total_price, reservation_id)
                    VALUES (%s, %s, %s, %s, %s, %s)
                """, [order_id, item["product_id"], item["quantity"],
                      item["product"]["price"], item["item_total"], reservations[i]])
            
            # Process payment
            payment_result = self.payment_service.process_payment(
                order_id, total, payment_method
            )
            
            if payment_result["success"]:
                # Confirm inventory reservations
                for reservation_id in reservations:
                    self.inventory_service.confirm_reservation(reservation_id)
                
                # Update order status
                self.db.execute("""
                    UPDATE orders SET 
                    status = %s, 
                    payment_id = %s,
                    confirmed_at = NOW()
                    WHERE id = %s
                """, [OrderStatus.CONFIRMED.value, payment_result["payment_id"], order_id])
                
                # Clear cart
                self.cart_service.clear_cart(user_id)
                
                # Send confirmation
                self.notification_service.send_order_confirmation(order_id)
                
                # Trigger async processes
                self.trigger_fulfillment_process(order_id)
                
                return {"order_id": order_id, "status": "success"}
            else:
                # Release reservations on payment failure
                for reservation_id in reservations:
                    self.inventory_service.release_reservation(reservation_id)
                
                self.db.execute("""
                    UPDATE orders SET status = %s WHERE id = %s
                """, [OrderStatus.CANCELLED.value, order_id])
                
                raise ValueError("Payment failed")
                
        except Exception as e:
            # Cleanup on any error
            for reservation_id in reservations:
                self.inventory_service.release_reservation(reservation_id)
            raise
    
    def get_order_status(self, order_id, user_id=None):
        query = """
            SELECT o.*, 
                   array_agg(
                       json_build_object(
                           'product_id', oi.product_id,
                           'quantity', oi.quantity,
                           'unit_price', oi.unit_price,
                           'total_price', oi.total_price
                       )
                   ) as items
            FROM orders o
            JOIN order_items oi ON o.id = oi.order_id
            WHERE o.id = %s
        """
        params = [order_id]
        
        if user_id:
            query += " AND o.user_id = %s"
            params.append(user_id)
        
        query += " GROUP BY o.id"
        
        result = self.db.execute(query, params).fetchone()
        
        if not result:
            return None
        
        return dict(result)
```

## 🔗 Related Topics

- [[Microservices Architecture]] - Service decomposition strategy
- [[Database Sharding]] - Scaling database layer
- [[Caching Strategies]] - Performance optimization
- [[Message Queue Systems]] - Async order processing

## 📚 Further Reading

- Amazon's e-commerce architecture
- eBay's scaling journey
- Shopify's platform design
- Payment processing best practices

---

**Key Takeaway**: E-commerce platforms require careful balance giữa consistency (inventory, payments) và performance (search, recommendations). Use microservices cho scalability, implement proper inventory reservations, và ensure data consistency across distributed transactions. 