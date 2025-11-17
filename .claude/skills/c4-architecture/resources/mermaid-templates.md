# Mermaid C4 Templates

## Overview

Mermaid provides native C4 diagram support that renders automatically in GitHub, GitLab, and many documentation platforms. This makes it ideal for documentation-first architectures where diagrams live alongside code.

**Advantages**:
- Native GitHub/GitLab rendering
- Simpler syntax than PlantUML
- No external dependencies
- Great for markdown documentation

**Limitations**:
- Less styling control than PlantUML
- Fewer layout options
- Limited to Context and Container diagrams

## Setup

### In Markdown Files

````markdown
```mermaid
C4Context
  title System Context Diagram
  Person(user, "User")
  System(system, "System")
  Rel(user, system, "Uses")
```
````

### Rendering

- **GitHub/GitLab**: Automatic in markdown files
- **VS Code**: Install Mermaid extension
- **Online**: https://mermaid.live/
- **CLI**: `npm install -g @mermaid-js/mermaid-cli`

## Level 1: System Context Templates

### Template 1: Basic System Context

````markdown
```mermaid
C4Context
  title System Context diagram for Internet Banking System

  Person(customer, "Personal Banking Customer", "A customer of the bank, with personal bank accounts.")

  System(banking_system, "Internet Banking System", "Allows customers to view information about their bank accounts, and make payments.")

  System_Ext(mail_system, "E-mail system", "The internal Microsoft Exchange e-mail system.")
  System_Ext(mainframe, "Mainframe Banking System", "Stores all of the core banking information about customers, accounts, transactions, etc.")

  Rel(customer, banking_system, "Views account balances, and makes payments using")
  Rel_Back(customer, mail_system, "Sends e-mails to")
  Rel_Neighbor(banking_system, mail_system, "Sends e-mail using")
  Rel(banking_system, mainframe, "Gets account information from, and makes payments using")

  UpdateElementStyle(customer, $fontColor="white", $bgColor="blue", $borderColor="blue")
  UpdateRelStyle(customer, banking_system, $textColor="blue", $lineColor="blue")
```
````

### Template 2: Multi-Actor System Context

````markdown
```mermaid
C4Context
  title System Context diagram for E-Commerce Platform

  Person(customer, "Customer", "Online shopper")
  Person(admin, "Administrator", "System administrator")
  Person(seller, "Seller", "Third-party seller")

  System(ecommerce, "E-Commerce Platform", "Allows customers to browse and purchase products, sellers to manage inventory, and admins to oversee the platform")

  System_Ext(payment_gateway, "Payment Gateway", "Stripe payment processing")
  System_Ext(email_service, "Email Service", "SendGrid email delivery")
  System_Ext(shipping_api, "Shipping API", "FedEx shipping integration")

  Rel(customer, ecommerce, "Browses products, places orders", "HTTPS")
  Rel(admin, ecommerce, "Manages platform", "HTTPS")
  Rel(seller, ecommerce, "Manages product listings", "HTTPS")

  Rel(ecommerce, payment_gateway, "Processes payments via", "REST API/HTTPS")
  Rel(ecommerce, email_service, "Sends notifications via", "SMTP/TLS")
  Rel(ecommerce, shipping_api, "Calculates shipping via", "REST API/HTTPS")
```
````

### Template 3: Enterprise Boundary Context

````markdown
```mermaid
C4Context
  title System Context diagram for Healthcare Platform

  Enterprise_Boundary(hospital, "Hospital System") {
    Person(doctor, "Doctor", "Medical professional")
    Person(nurse, "Nurse", "Nursing staff")
  }

  Person_Ext(patient, "Patient", "Healthcare patient")

  System_Boundary(internal, "Internal Systems") {
    System(ehr, "Electronic Health Records", "Patient record management system")
    System(scheduling, "Scheduling System", "Appointment scheduling")
  }

  System_Ext(lab_system, "Laboratory System", "External lab results")
  System_Ext(pharmacy, "Pharmacy System", "Prescription management")

  Rel(doctor, ehr, "Manages patient records via", "HTTPS")
  Rel(nurse, ehr, "Updates patient info via", "HTTPS")
  Rel(doctor, scheduling, "Manages appointments via", "HTTPS")

  Rel(patient, scheduling, "Books appointments via", "HTTPS")
  Rel(patient, ehr, "Views own records via", "HTTPS")

  Rel(ehr, lab_system, "Retrieves lab results from", "HL7/FHIR")
  Rel(ehr, pharmacy, "Sends prescriptions to", "HL7/FHIR")
```
````

## Level 2: Container Templates

### Template 1: Web Application with Backend

````markdown
```mermaid
C4Container
  title Container diagram for Internet Banking System

  Person(customer, "Personal Banking Customer", "A customer of the bank")

  System_Boundary(c1, "Internet Banking System") {
    Container(web_app, "Web Application", "React, TypeScript", "Delivers the static content and the single page application")
    Container(spa, "Single-Page Application", "React, TypeScript", "Provides all of the banking functionality to customers via their web browser")
    Container(mobile_app, "Mobile App", "React Native", "Provides a limited subset of the banking functionality to customers via their mobile device")
    Container(api_app, "API Application", "Node.js, Express", "Provides banking functionality via a JSON/REST API")
    ContainerDb(database, "Database", "PostgreSQL", "Stores user authentication data, access logs, etc.")
  }

  System_Ext(email_system, "E-mail System", "The internal Microsoft Exchange e-mail system")

  Rel(customer, web_app, "Visits bigbank.com/ib using", "HTTPS")
  Rel(customer, spa, "Views account balances and makes payments using")
  Rel(customer, mobile_app, "Views account balances and makes payments using")

  Rel(web_app, spa, "Delivers to the customer's web browser")

  Rel(spa, api_app, "Makes API calls to", "JSON/HTTPS")
  Rel(mobile_app, api_app, "Makes API calls to", "JSON/HTTPS")

  Rel_Back(api_app, database, "Reads from and writes to", "SQL/TCP")
  Rel_Back(api_app, email_system, "Sends e-mail using", "SMTP")
  Rel(api_app, email_system, "Sends e-mails using")

  UpdateRelStyle(customer, web_app, $offsetY="-40")
  UpdateRelStyle(customer, spa, $offsetY="-40")
  UpdateRelStyle(customer, mobile_app, $offsetY="-40")
```
````

### Template 2: Microservices Architecture

````markdown
```mermaid
C4Container
  title Container diagram for E-Commerce Microservices Platform

  Person(customer, "Customer", "Online shopper")

  System_Boundary(platform, "E-Commerce Platform") {
    Container(web_app, "Web Application", "React, Next.js", "Customer-facing web interface")
    Container(mobile_app, "Mobile App", "React Native", "Customer mobile app")

    Container(api_gateway, "API Gateway", "Node.js, Express", "Routes requests to backend services")

    Container(user_service, "User Service", "Node.js, Express", "User accounts and authentication")
    Container(product_service, "Product Service", "Rust, Axum", "Product catalog management")
    Container(order_service, "Order Service", "Node.js, Express", "Order processing and management")
    Container(payment_service, "Payment Service", "Node.js, Express", "Payment processing")

    ContainerDb(user_db, "User Database", "PostgreSQL", "User and authentication data")
    ContainerDb(product_db, "Product Database", "PostgreSQL", "Product catalog data")
    ContainerDb(order_db, "Order Database", "PostgreSQL", "Order and transaction data")

    Container(message_broker, "Message Broker", "RabbitMQ", "Asynchronous event messaging")
    Container(cache, "Cache", "Redis", "Session and data caching")
  }

  System_Ext(payment_gateway, "Payment Gateway", "Stripe API")

  Rel(customer, web_app, "Uses", "HTTPS")
  Rel(customer, mobile_app, "Uses", "HTTPS")

  Rel(web_app, api_gateway, "Makes API calls to", "JSON/REST")
  Rel(mobile_app, api_gateway, "Makes API calls to", "JSON/REST")

  Rel(api_gateway, user_service, "Routes requests to")
  Rel(api_gateway, product_service, "Routes requests to")
  Rel(api_gateway, order_service, "Routes requests to")
  Rel(api_gateway, payment_service, "Routes requests to")

  Rel(user_service, user_db, "Reads from and writes to", "SQL/TCP")
  Rel(product_service, product_db, "Reads from and writes to", "SQL/TCP")
  Rel(order_service, order_db, "Reads from and writes to", "SQL/TCP")

  Rel(user_service, cache, "Caches session data in", "Redis Protocol")
  Rel(product_service, cache, "Caches product data in", "Redis Protocol")

  Rel(order_service, message_broker, "Publishes OrderCreated events to", "AMQP")
  Rel(payment_service, message_broker, "Subscribes to OrderCreated events from", "AMQP")

  Rel(payment_service, payment_gateway, "Processes payments via", "REST API/HTTPS")
```
````

### Template 3: Event-Driven Architecture

````markdown
```mermaid
C4Container
  title Container diagram for Event-Driven Order System

  Person(customer, "Customer", "Places orders")
  Person(admin, "Administrator", "Manages system")

  System_Boundary(system, "Order Management System") {
    Container(web_ui, "Web UI", "React, TypeScript", "Customer and admin interface")
    Container(api_gateway, "API Gateway", "Node.js, Express", "API routing and composition")

    Container(order_service, "Order Service", "Node.js, Express", "Order management and processing")
    Container(inventory_service, "Inventory Service", "Rust, Axum", "Inventory tracking and management")
    Container(notification_service, "Notification Service", "Node.js, Express", "Customer notifications")
    Container(analytics_service, "Analytics Service", "Python, FastAPI", "Business analytics and reporting")

    ContainerDb(order_db, "Order Database", "PostgreSQL", "Order and customer data")
    ContainerDb(inventory_db, "Inventory Database", "PostgreSQL", "Inventory and product data")
    ContainerDb(analytics_db, "Analytics Database", "TimescaleDB", "Time-series analytics data")

    Container(event_bus, "Event Bus", "Apache Kafka", "Event streaming platform")
    Container(cache, "Cache", "Redis", "Application-level caching")
  }

  System_Ext(email_service, "Email Service", "SendGrid")

  Rel(customer, web_ui, "Uses", "HTTPS")
  Rel(admin, web_ui, "Manages via", "HTTPS")

  Rel(web_ui, api_gateway, "Makes API calls to", "JSON/REST")

  Rel(api_gateway, order_service, "Routes order requests to")
  Rel(api_gateway, inventory_service, "Routes inventory requests to")

  Rel(order_service, order_db, "Reads/writes order data", "SQL/TCP")
  Rel(inventory_service, inventory_db, "Reads/writes inventory", "SQL/TCP")
  Rel(analytics_service, analytics_db, "Writes metrics to", "SQL/TCP")

  Rel(order_service, event_bus, "Publishes OrderCreated events to", "Kafka")
  Rel(inventory_service, event_bus, "Subscribes to OrderCreated from", "Kafka")
  Rel(notification_service, event_bus, "Subscribes to OrderCreated from", "Kafka")
  Rel(analytics_service, event_bus, "Subscribes to all events from", "Kafka")

  Rel(notification_service, email_service, "Sends emails via", "SMTP/TLS")

  Rel(order_service, cache, "Caches order data in", "Redis")
  Rel(inventory_service, cache, "Caches inventory in", "Redis")
```
````

## Styling and Customization

### Custom Colors

````markdown
```mermaid
C4Context
  title Styled System Context Diagram

  Person(user, "User", "System user")
  System(system, "Main System", "Core application")
  System_Ext(external, "External System", "Third-party service")

  Rel(user, system, "Uses")
  Rel(system, external, "Integrates with")

  UpdateElementStyle(user, $fontColor="white", $bgColor="#08427B", $borderColor="#052E56")
  UpdateElementStyle(system, $fontColor="white", $bgColor="#1168BD", $borderColor="#0B4884")
  UpdateElementStyle(external, $fontColor="white", $bgColor="#999999", $borderColor="#6B6B6B")

  UpdateRelStyle(user, system, $textColor="blue", $lineColor="blue", $offsetX="-50")
  UpdateRelStyle(system, external, $textColor="red", $lineColor="red", $offsetY="-40")
```
````

### Relationship Styling

````markdown
```mermaid
C4Container
  title Container Diagram with Styled Relationships

  Person(user, "User")

  System_Boundary(system, "System") {
    Container(web, "Web App", "React")
    Container(api, "API", "Node.js")
    ContainerDb(db, "Database", "PostgreSQL")
  }

  Rel(user, web, "Uses", "HTTPS")
  Rel(web, api, "Calls", "REST/HTTPS")
  Rel(api, db, "Queries", "SQL/TCP")

  UpdateRelStyle(user, web, $textColor="green", $lineColor="green", $offsetY="-40")
  UpdateRelStyle(web, api, $textColor="blue", $lineColor="blue")
  UpdateRelStyle(api, db, $textColor="red", $lineColor="red", $offsetX="-50")
```
````

### Boundary Grouping

````markdown
```mermaid
C4Container
  title Container Diagram with Boundaries

  Person(customer, "Customer")

  Enterprise_Boundary(company, "Company Systems") {
    System_Boundary(frontend, "Frontend") {
      Container(web, "Web Application", "React")
      Container(mobile, "Mobile App", "React Native")
    }

    System_Boundary(backend, "Backend Services") {
      Container(api, "API Gateway", "Node.js")
      Container(service1, "Service A", "Node.js")
      Container(service2, "Service B", "Rust")
    }

    System_Boundary(data, "Data Layer") {
      ContainerDb(db1, "Database A", "PostgreSQL")
      ContainerDb(db2, "Database B", "PostgreSQL")
    }
  }

  Rel(customer, web, "Uses")
  Rel(customer, mobile, "Uses")
  Rel(web, api, "Calls")
  Rel(mobile, api, "Calls")
  Rel(api, service1, "Routes to")
  Rel(api, service2, "Routes to")
  Rel(service1, db1, "Reads/writes")
  Rel(service2, db2, "Reads/writes")
```
````

## Complete Examples

### Example 1: Full Stack E-Commerce

````markdown
```mermaid
C4Container
  title E-Commerce Platform - Container Diagram

  Person(customer, "Customer", "Browses and purchases products")
  Person(seller, "Seller", "Manages product inventory")
  Person(admin, "Admin", "Administers platform")

  System_Boundary(platform, "E-Commerce Platform") {
    Container(customer_web, "Customer Web App", "React, Next.js", "Shopping interface")
    Container(seller_portal, "Seller Portal", "React, TypeScript", "Inventory management")
    Container(admin_panel, "Admin Panel", "React, TypeScript", "Platform administration")

    Container(api_gateway, "API Gateway", "Kong", "API routing, rate limiting, auth")

    Container(catalog_svc, "Catalog Service", "Rust, Axum", "Product catalog and search")
    Container(order_svc, "Order Service", "Node.js, Express", "Order processing")
    Container(payment_svc, "Payment Service", "Node.js, Express", "Payment handling")
    Container(user_svc, "User Service", "Node.js, Express", "User and authentication")

    ContainerDb(catalog_db, "Catalog DB", "PostgreSQL", "Product data")
    ContainerDb(order_db, "Order DB", "PostgreSQL", "Order data")
    ContainerDb(user_db, "User DB", "PostgreSQL", "User data")

    Container(search_engine, "Search Engine", "Elasticsearch", "Product search index")
    Container(cache, "Cache", "Redis", "Session and data cache")
    Container(queue, "Message Queue", "RabbitMQ", "Event messaging")
  }

  System_Ext(payment_provider, "Payment Provider", "Stripe")
  System_Ext(email_service, "Email Service", "SendGrid")
  System_Ext(cdn, "CDN", "CloudFlare")

  Rel(customer, cdn, "Accesses via", "HTTPS")
  Rel(seller, seller_portal, "Manages products via", "HTTPS")
  Rel(admin, admin_panel, "Administers via", "HTTPS")

  Rel(cdn, customer_web, "Delivers")

  Rel(customer_web, api_gateway, "API calls", "JSON/REST")
  Rel(seller_portal, api_gateway, "API calls", "JSON/REST")
  Rel(admin_panel, api_gateway, "API calls", "JSON/REST")

  Rel(api_gateway, catalog_svc, "Routes to")
  Rel(api_gateway, order_svc, "Routes to")
  Rel(api_gateway, payment_svc, "Routes to")
  Rel(api_gateway, user_svc, "Routes to")

  Rel(catalog_svc, catalog_db, "Reads/writes", "SQL")
  Rel(order_svc, order_db, "Reads/writes", "SQL")
  Rel(user_svc, user_db, "Reads/writes", "SQL")

  Rel(catalog_svc, search_engine, "Indexes products in", "HTTP/JSON")
  Rel(catalog_svc, cache, "Caches data in", "Redis")
  Rel(user_svc, cache, "Caches sessions in", "Redis")

  Rel(order_svc, queue, "Publishes OrderCreated to", "AMQP")
  Rel(payment_svc, queue, "Subscribes to OrderCreated from", "AMQP")

  Rel(payment_svc, payment_provider, "Processes via", "REST/HTTPS")
  Rel(order_svc, email_service, "Sends confirmations via", "SMTP")

  UpdateElementStyle(customer, $fontColor="white", $bgColor="#08427B")
  UpdateElementStyle(seller, $fontColor="white", $bgColor="#08427B")
  UpdateElementStyle(admin, $fontColor="white", $bgColor="#08427B")
```
````

## Rendering Options

### GitHub/GitLab

Mermaid diagrams render automatically in markdown files:

```markdown
# System Architecture

Our system follows a microservices architecture:

```mermaid
C4Container
  [your diagram here]
```

See the diagram above for details.
```

### VS Code

1. Install "Markdown Preview Mermaid Support" extension
2. Open markdown file with Mermaid diagram
3. Use markdown preview (`Cmd+Shift+V`)

### Command Line

```bash
# Install mermaid-cli
npm install -g @mermaid-js/mermaid-cli

# Render to PNG
mmdc -i diagram.mmd -o diagram.png

# Render to SVG
mmdc -i diagram.mmd -o diagram.svg

# Custom theme
mmdc -i diagram.mmd -o diagram.png -t forest
```

### Online Editor

Visit https://mermaid.live/ for:
- Live preview
- Export to PNG/SVG
- Share diagrams
- Embed in docs

## CI/CD Integration

### GitHub Actions

```yaml
name: Render Mermaid Diagrams

on:
  push:
    paths:
      - 'docs/**/*.mmd'

jobs:
  render:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install Mermaid CLI
        run: npm install -g @mermaid-js/mermaid-cli

      - name: Render diagrams
        run: |
          find docs -name "*.mmd" -exec mmdc -i {} -o {}.png \;

      - name: Commit rendered diagrams
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add docs/**/*.png
          git commit -m "Update rendered diagrams" || exit 0
          git push
```

## Best Practices

### DO:
✅ Use Mermaid for documentation-first architectures
✅ Store `.mmd` files alongside markdown docs
✅ Use meaningful titles with `title` keyword
✅ Specify technology stacks in container descriptions
✅ Use System_Boundary for logical grouping
✅ Add relationship labels with protocols
✅ Leverage GitHub's native rendering
✅ Keep diagrams under 30-40 elements for readability

### DON'T:
❌ Use for complex diagrams (PlantUML better for that)
❌ Forget to specify relationship protocols
❌ Mix abstraction levels in single diagram
❌ Use vague relationship labels
❌ Over-style (keep it simple)
❌ Create diagrams wider than readable in GitHub

## Comparison: Mermaid vs PlantUML

| Feature | Mermaid | PlantUML |
|---------|---------|----------|
| **GitHub Rendering** | Native | Requires plugin |
| **Syntax Complexity** | Simple | More complex |
| **Styling Options** | Limited | Extensive |
| **Layout Control** | Auto | More control |
| **C4 Support** | Context, Container | All 4 levels |
| **Learning Curve** | Easy | Moderate |
| **Best For** | Docs, simple diagrams | Complex, detailed |

## Related Resources

- [PlantUML C4 Templates](./plantuml-templates.md) - Alternative with more features
- [Complete C4 Examples](./complete-examples.md) - Full system examples
- [Diagramming Best Practices](./best-practices.md) - General C4 guidance

---

**Resource Coverage**: Complete Mermaid templates for Context and Container levels
**Advantages**: Native GitHub rendering, simple syntax, great for documentation
**Use When**: Documentation-first approach, moderate complexity diagrams
