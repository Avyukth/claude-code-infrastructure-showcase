# PlantUML C4 Templates

## Overview

PlantUML with the C4-PlantUML library provides rich formatting options and detailed control for C4 diagrams. This resource provides complete, copy-paste-ready templates for all four C4 levels.

## Setup

### Include C4-PlantUML Library

Add this to the top of every C4 diagram:

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml
' or C4_Container.puml, C4_Component.puml, C4_Deployment.puml

' Optional: Customize styling
!define DEVICONS https://raw.githubusercontent.com/tupadr3/plantuml-icon-font-sprites/master/devicons
!define FONTAWESOME https://raw.githubusercontent.com/tupadr3/plantuml-icon-font-sprites/master/font-awesome-5
!include DEVICONS/react.puml
!include FONTAWESOME/server.puml

LAYOUT_WITH_LEGEND()

' Your diagram here

@enduml
```

## Level 1: System Context Templates

### Template 1: Basic System Context

```plantuml
@startuml system-context-basic
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml

LAYOUT_WITH_LEGEND()

title System Context diagram for [System Name]

Person(user, "User", "A user of the system")
System(system, "[System Name]", "Description of what the system does")
System_Ext(ext_system, "External System", "Description of external system")

Rel(user, system, "Uses", "HTTPS")
Rel(system, ext_system, "Reads data from", "REST API/HTTPS")

@enduml
```

### Template 2: Multi-User System Context

```plantuml
@startuml system-context-multi-user
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml

LAYOUT_WITH_LEGEND()

title System Context diagram for E-Commerce Platform

Person(customer, "Customer", "Online shopper")
Person(admin, "Administrator", "System administrator")
Person(seller, "Seller", "Third-party seller")

System(ecommerce, "E-Commerce Platform", "Allows customers to browse and purchase products, sellers to manage inventory, and admins to manage the platform")

System_Ext(payment_gateway, "Payment Gateway", "Stripe payment processing")
System_Ext(email_service, "Email Service", "SendGrid email delivery")
System_Ext(shipping_api, "Shipping API", "FedEx shipping integration")
System_Ext(inventory_system, "Warehouse Inventory System", "External inventory management")

Rel(customer, ecommerce, "Browses products, places orders", "HTTPS")
Rel(admin, ecommerce, "Manages platform", "HTTPS")
Rel(seller, ecommerce, "Manages product listings", "HTTPS")

Rel(ecommerce, payment_gateway, "Processes payments", "REST API/HTTPS")
Rel(ecommerce, email_service, "Sends notification emails", "SMTP/TLS")
Rel(ecommerce, shipping_api, "Calculates shipping, creates labels", "REST API/HTTPS")
Rel(ecommerce, inventory_system, "Syncs inventory levels", "REST API/HTTPS")

@enduml
```

### Template 3: System Context with Boundaries

```plantuml
@startuml system-context-boundaries
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml

LAYOUT_WITH_LEGEND()

title System Context diagram for Healthcare Platform

Enterprise_Boundary(hospital, "Hospital System") {
    Person(doctor, "Doctor", "Medical professional")
    Person(nurse, "Nurse", "Nursing staff")
    System(ehr, "Electronic Health Records", "Patient record management")
}

Person(patient, "Patient", "Healthcare patient")

System_Ext(lab_system, "Laboratory System", "External lab results")
System_Ext(pharmacy, "Pharmacy System", "Prescription management")
System_Ext(insurance, "Insurance Provider", "Insurance verification and claims")

Rel(doctor, ehr, "Manages patient records", "HTTPS")
Rel(nurse, ehr, "Updates patient info", "HTTPS")
Rel(patient, ehr, "Views own records", "HTTPS")

Rel(ehr, lab_system, "Retrieves lab results", "HL7/FHIR")
Rel(ehr, pharmacy, "Sends prescriptions", "HL7/FHIR")
Rel(ehr, insurance, "Verifies coverage, submits claims", "REST API/HTTPS")

@enduml
```

## Level 2: Container Templates

### Template 1: Web Application with Backend

```plantuml
@startuml container-web-app
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

LAYOUT_WITH_LEGEND()

title Container diagram for [System Name]

Person(user, "User", "A user of the system")

System_Boundary(system, "[System Name]") {
    Container(web_app, "Web Application", "React, TypeScript", "Delivers the static content and the single page application")
    Container(api, "API Application", "Node.js, Express", "Provides functionality via a JSON/REST API")
    ContainerDb(database, "Database", "PostgreSQL", "Stores user data, transactions, etc.")
    Container(cache, "Cache", "Redis", "Stores session data and frequently accessed data")
}

System_Ext(email_system, "Email System", "SendGrid")

Rel(user, web_app, "Uses", "HTTPS")
Rel(web_app, api, "Makes API calls to", "JSON/REST, HTTPS")
Rel(api, database, "Reads from and writes to", "SQL/TCP")
Rel(api, cache, "Reads from and writes to", "Redis Protocol")
Rel(api, email_system, "Sends emails using", "SMTP/TLS")

@enduml
```

### Template 2: Microservices Architecture

```plantuml
@startuml container-microservices
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

LAYOUT_WITH_LEGEND()

title Container diagram for E-Commerce Microservices

Person(customer, "Customer")

System_Boundary(ecommerce, "E-Commerce Platform") {
    Container(web_app, "Web Application", "React, Next.js", "Customer-facing web interface")
    Container(mobile_app, "Mobile App", "React Native", "Customer-facing mobile app")

    Container(api_gateway, "API Gateway", "Node.js, Express", "Routes requests to appropriate services")

    Container(user_service, "User Service", "Node.js, Express", "Manages user accounts and authentication")
    Container(product_service, "Product Service", "Rust, Axum", "Manages product catalog")
    Container(order_service, "Order Service", "Node.js, Express", "Handles order processing")
    Container(payment_service, "Payment Service", "Node.js, Express", "Processes payments")

    ContainerDb(user_db, "User Database", "PostgreSQL", "Stores user data")
    ContainerDb(product_db, "Product Database", "PostgreSQL", "Stores product catalog")
    ContainerDb(order_db, "Order Database", "PostgreSQL", "Stores order data")

    Container(message_broker, "Message Broker", "RabbitMQ", "Asynchronous event messaging")
    Container(cache, "Cache", "Redis", "Session and data caching")
}

System_Ext(payment_gateway, "Payment Gateway", "Stripe")
System_Ext(email_service, "Email Service", "SendGrid")

Rel(customer, web_app, "Uses", "HTTPS")
Rel(customer, mobile_app, "Uses", "HTTPS")

Rel(web_app, api_gateway, "Makes API calls", "JSON/REST, HTTPS")
Rel(mobile_app, api_gateway, "Makes API calls", "JSON/REST, HTTPS")

Rel(api_gateway, user_service, "Routes requests", "JSON/REST, HTTPS")
Rel(api_gateway, product_service, "Routes requests", "JSON/REST, HTTPS")
Rel(api_gateway, order_service, "Routes requests", "JSON/REST, HTTPS")
Rel(api_gateway, payment_service, "Routes requests", "JSON/REST, HTTPS")

Rel(user_service, user_db, "Reads/writes", "SQL/TCP")
Rel(product_service, product_db, "Reads/writes", "SQL/TCP")
Rel(order_service, order_db, "Reads/writes", "SQL/TCP")

Rel(user_service, cache, "Caches session data", "Redis Protocol")
Rel(product_service, cache, "Caches product data", "Redis Protocol")

Rel(order_service, message_broker, "Publishes OrderCreated events", "AMQP")
Rel(payment_service, message_broker, "Subscribes to OrderCreated events", "AMQP")
Rel(payment_service, payment_gateway, "Processes payments", "REST API/HTTPS")

Rel(order_service, email_service, "Sends order confirmations", "SMTP/TLS")

@enduml
```

### Template 3: Event-Driven Architecture

```plantuml
@startuml container-event-driven
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

LAYOUT_WITH_LEGEND()

title Container diagram for Event-Driven Order System

Person(customer, "Customer")
Person(admin, "Administrator")

System_Boundary(system, "Order Management System") {
    Container(web_ui, "Web UI", "React, TypeScript", "Customer and admin interface")
    Container(api_gateway, "API Gateway", "Node.js, Express", "API routing and composition")

    Container(order_service, "Order Service", "Node.js, Express", "Order management")
    Container(inventory_service, "Inventory Service", "Rust, Axum", "Inventory tracking")
    Container(notification_service, "Notification Service", "Node.js, Express", "Customer notifications")
    Container(analytics_service, "Analytics Service", "Python, FastAPI", "Business analytics")

    ContainerDb(order_db, "Order Database", "PostgreSQL", "Order data")
    ContainerDb(inventory_db, "Inventory Database", "PostgreSQL", "Inventory data")
    ContainerDb(analytics_db, "Analytics Database", "TimescaleDB", "Time-series analytics data")

    Container(event_bus, "Event Bus", "Apache Kafka", "Event streaming platform")
    Container(cache, "Cache", "Redis", "Application caching")
}

Rel(customer, web_ui, "Uses", "HTTPS")
Rel(admin, web_ui, "Manages", "HTTPS")
Rel(web_ui, api_gateway, "Makes API calls", "JSON/REST, HTTPS")

Rel(api_gateway, order_service, "Manages orders", "JSON/REST, HTTPS")
Rel(api_gateway, inventory_service, "Queries inventory", "JSON/REST, HTTPS")

Rel(order_service, order_db, "Reads/writes", "SQL/TCP")
Rel(inventory_service, inventory_db, "Reads/writes", "SQL/TCP")
Rel(analytics_service, analytics_db, "Writes metrics", "SQL/TCP")

' Event-driven relationships (dashed for async)
Rel(order_service, event_bus, "Publishes OrderCreated, OrderUpdated", "Kafka Protocol", $lineStyle = DashedLine())
Rel(inventory_service, event_bus, "Subscribes to OrderCreated", "Kafka Protocol", $lineStyle = DashedLine())
Rel(notification_service, event_bus, "Subscribes to OrderCreated, OrderUpdated", "Kafka Protocol", $lineStyle = DashedLine())
Rel(analytics_service, event_bus, "Subscribes to all events", "Kafka Protocol", $lineStyle = DashedLine())

Rel(order_service, cache, "Caches order data", "Redis Protocol")
Rel(inventory_service, cache, "Caches inventory levels", "Redis Protocol")

@enduml
```

## Level 3: Component Templates

### Template 1: Layered Architecture Components

```plantuml
@startuml component-layered
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

LAYOUT_WITH_LEGEND()

title Component diagram for Order Service

Container(web_app, "Web Application", "React", "User interface")
ContainerDb(database, "Database", "PostgreSQL")
Container(message_broker, "Message Broker", "RabbitMQ")

Container_Boundary(order_service, "Order Service") {
    Component(api, "API Controller", "Express Router", "Handles HTTP requests, routing")
    Component(order_controller, "Order Controller", "Class", "Order endpoint handlers")
    Component(auth_middleware, "Auth Middleware", "Express Middleware", "JWT authentication")

    Component(order_service_comp, "Order Service", "Class", "Business logic for orders")
    Component(inventory_service, "Inventory Service", "Class", "Inventory management logic")
    Component(notification_service, "Notification Service", "Class", "Notification logic")

    Component(order_repository, "Order Repository", "Class", "Order data access")
    Component(product_repository, "Product Repository", "Class", "Product data access")

    Component(event_publisher, "Event Publisher", "Class", "Publishes domain events")
}

Rel(web_app, api, "Makes API calls to", "JSON/REST, HTTPS")
Rel(api, auth_middleware, "Authenticates with")
Rel(auth_middleware, order_controller, "Routes to")

Rel(order_controller, order_service_comp, "Uses")
Rel(order_controller, inventory_service, "Uses")

Rel(order_service_comp, order_repository, "Uses")
Rel(order_service_comp, event_publisher, "Publishes events via")
Rel(inventory_service, product_repository, "Uses")
Rel(notification_service, event_publisher, "Publishes events via")

Rel(order_repository, database, "Reads/writes", "SQL/TCP")
Rel(product_repository, database, "Reads", "SQL/TCP")
Rel(event_publisher, message_broker, "Publishes to", "AMQP")

@enduml
```

### Template 2: Hexagonal Architecture Components

```plantuml
@startuml component-hexagonal
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

LAYOUT_WITH_LEGEND()

title Component diagram for User Service (Hexagonal Architecture)

Container(web_app, "Web Application", "React")
Container(admin_cli, "Admin CLI", "Node.js CLI")
ContainerDb(database, "Database", "PostgreSQL")
System_Ext(email_service, "Email Service", "SendGrid")

Container_Boundary(user_service, "User Service") {
    ' Ports (Interfaces)
    Component(http_port, "HTTP Port", "Interface", "REST API interface")
    Component(cli_port, "CLI Port", "Interface", "Command-line interface")
    Component(db_port, "Database Port", "Interface", "Data persistence interface")
    Component(email_port, "Email Port", "Interface", "Email sending interface")

    ' Core Domain
    Component(user_domain, "User Domain", "Domain Logic", "User business rules and logic")
    Component(auth_domain, "Authentication Domain", "Domain Logic", "Authentication logic")

    ' Adapters
    Component(http_adapter, "HTTP Adapter", "Express Controller", "Implements HTTP Port")
    Component(cli_adapter, "CLI Adapter", "Commander.js", "Implements CLI Port")
    Component(postgres_adapter, "PostgreSQL Adapter", "Repository", "Implements Database Port")
    Component(sendgrid_adapter, "SendGrid Adapter", "Email Service", "Implements Email Port")
}

Rel(web_app, http_adapter, "Makes API calls", "JSON/REST, HTTPS")
Rel(admin_cli, cli_adapter, "Executes commands", "CLI")

Rel(http_adapter, http_port, "Implements")
Rel(cli_adapter, cli_port, "Implements")

Rel(http_port, user_domain, "Calls")
Rel(http_port, auth_domain, "Calls")
Rel(cli_port, user_domain, "Calls")

Rel(user_domain, db_port, "Uses")
Rel(user_domain, email_port, "Uses")
Rel(auth_domain, db_port, "Uses")

Rel(postgres_adapter, db_port, "Implements")
Rel(sendgrid_adapter, email_port, "Implements")

Rel(postgres_adapter, database, "Reads/writes", "SQL/TCP")
Rel(sendgrid_adapter, email_service, "Sends emails", "SMTP/TLS")

@enduml
```

## Advanced Styling

### Custom Colors and Icons

```plantuml
@startuml container-styled
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

!define DEVICONS https://raw.githubusercontent.com/tupadr3/plantuml-icon-font-sprites/master/devicons
!define FONTAWESOME https://raw.githubusercontent.com/tupadr3/plantuml-icon-font-sprites/master/font-awesome-5

!include DEVICONS/react.puml
!include DEVICONS/nodejs.puml
!include DEVICONS/postgresql.puml
!include DEVICONS/redis.puml
!include FONTAWESOME/server.puml

LAYOUT_WITH_LEGEND()

title Styled Container Diagram

Person(user, "User")

System_Boundary(system, "System") {
    Container(web, "Web App", "React", "Frontend", $sprite="react")
    Container(api, "API", "Node.js", "Backend", $sprite="nodejs")
    ContainerDb(db, "Database", "PostgreSQL", "Data Store", $sprite="postgresql")
    Container(cache, "Cache", "Redis", "Caching", $sprite="redis")
}

Rel(user, web, "Uses", "HTTPS")
Rel(web, api, "Calls", "REST/HTTPS")
Rel(api, db, "Reads/writes", "SQL")
Rel(api, cache, "Caches", "Redis Protocol")

@enduml
```

### Layout Control

```plantuml
@startuml container-layout
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

' Control layout direction
LAYOUT_TOP_DOWN()
' or LAYOUT_LEFT_RIGHT()
' or LAYOUT_LANDSCAPE()

title Container Diagram with Controlled Layout

Person(user, "User")

System_Boundary(system, "System") {
    Container(web, "Web App", "React")
    Container(api, "API", "Node.js")
    ContainerDb(db, "Database", "PostgreSQL")
}

' Explicit positioning hints
Lay_R(web, api)
Lay_D(api, db)

Rel_D(user, web, "Uses", "HTTPS")
Rel_R(web, api, "Calls", "REST")
Rel_D(api, db, "Queries", "SQL")

@enduml
```

## Rendering

### Local Rendering

```bash
# Install PlantUML (macOS)
brew install plantuml

# Render single diagram
plantuml diagram.puml

# Render all diagrams in directory
plantuml diagrams/*.puml

# Specify output format
plantuml -tpng diagram.puml
plantuml -tsvg diagram.puml
```

### VS Code Integration

Install the PlantUML extension:
1. Install extension: `jebbs.plantuml`
2. Open `.puml` file
3. Press `Alt+D` to preview
4. Right-click → "Export Current Diagram"

### CI/CD Integration

```yaml
# .github/workflows/diagrams.yml
name: Render Diagrams

on:
  push:
    paths:
      - 'docs/diagrams/**/*.puml'

jobs:
  render:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup PlantUML
        run: |
          sudo apt-get update
          sudo apt-get install -y plantuml

      - name: Render diagrams
        run: |
          find docs/diagrams -name "*.puml" -exec plantuml {} \;

      - name: Commit rendered diagrams
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add docs/diagrams/**/*.png
          git commit -m "Update rendered diagrams" || exit 0
          git push
```

## Complete Example

```plantuml
@startuml complete-ecommerce
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

LAYOUT_WITH_LEGEND()

title Container Diagram for E-Commerce Platform

Person(customer, "Customer", "Online shopper")
Person(seller, "Seller", "Third-party seller")
Person(admin, "Admin", "Platform administrator")

System_Boundary(platform, "E-Commerce Platform") {
    Container(web, "Web Application", "React, Next.js, TypeScript", "Customer and seller portal")
    Container(admin_portal, "Admin Portal", "React, TypeScript", "Platform administration")

    Container(gateway, "API Gateway", "Node.js, Express", "API routing and rate limiting")

    Container(user_svc, "User Service", "Node.js, Express", "User and auth management")
    Container(product_svc, "Product Service", "Rust, Axum", "Product catalog")
    Container(order_svc, "Order Service", "Node.js, Express", "Order processing")
    Container(payment_svc, "Payment Service", "Node.js, Express", "Payment handling")

    ContainerDb(user_db, "User DB", "PostgreSQL", "User and auth data")
    ContainerDb(product_db, "Product DB", "PostgreSQL", "Product catalog")
    ContainerDb(order_db, "Order DB", "PostgreSQL", "Orders and transactions")

    Container(broker, "Message Broker", "RabbitMQ", "Event messaging")
    Container(cache, "Cache", "Redis", "Session and data cache")
    Container(search, "Search Engine", "Elasticsearch", "Product search")
}

System_Ext(payment_gw, "Payment Gateway", "Stripe")
System_Ext(email, "Email Service", "SendGrid")
System_Ext(storage, "Object Storage", "AWS S3")

' User interactions
Rel(customer, web, "Browses, purchases", "HTTPS")
Rel(seller, web, "Manages products", "HTTPS")
Rel(admin, admin_portal, "Administers platform", "HTTPS")

' Frontend to Gateway
Rel(web, gateway, "API calls", "JSON/REST, HTTPS")
Rel(admin_portal, gateway, "API calls", "JSON/REST, HTTPS")

' Gateway to Services
Rel(gateway, user_svc, "Routes auth requests", "JSON/REST, HTTPS")
Rel(gateway, product_svc, "Routes product requests", "JSON/REST, HTTPS")
Rel(gateway, order_svc, "Routes order requests", "JSON/REST, HTTPS")
Rel(gateway, payment_svc, "Routes payment requests", "JSON/REST, HTTPS")

' Services to Databases
Rel(user_svc, user_db, "Reads/writes user data", "SQL/TCP")
Rel(product_svc, product_db, "Reads/writes products", "SQL/TCP")
Rel(order_svc, order_db, "Reads/writes orders", "SQL/TCP")

' Services to Cache
Rel(user_svc, cache, "Caches sessions", "Redis Protocol")
Rel(product_svc, cache, "Caches product data", "Redis Protocol")

' Services to Search
Rel(product_svc, search, "Indexes products", "HTTP/JSON")

' Event-driven communication
Rel(order_svc, broker, "Publishes OrderCreated", "AMQP", $lineStyle = DashedLine())
Rel(payment_svc, broker, "Subscribes to OrderCreated", "AMQP", $lineStyle = DashedLine())
Rel(payment_svc, broker, "Publishes PaymentProcessed", "AMQP", $lineStyle = DashedLine())
Rel(order_svc, broker, "Subscribes to PaymentProcessed", "AMQP", $lineStyle = DashedLine())

' External integrations
Rel(payment_svc, payment_gw, "Processes payments", "REST/HTTPS")
Rel(order_svc, email, "Sends confirmations", "SMTP/TLS")
Rel(product_svc, storage, "Stores product images", "S3 API/HTTPS")

@enduml
```

## Best Practices

### DO:
✅ Include C4-PlantUML library at top of every diagram
✅ Use `LAYOUT_WITH_LEGEND()` for automatic legend
✅ Add meaningful titles with `title` keyword
✅ Use dashed lines for async relationships (`$lineStyle = DashedLine()`)
✅ Specify technology stack in container descriptions
✅ Use System_Boundary for grouping containers
✅ Version control `.puml` source files
✅ Render both PNG and SVG for different uses

### DON'T:
❌ Mix C4 levels in single diagram (Context + Container)
❌ Use vague relationship labels ("uses", "connects")
❌ Forget to specify protocols in relationships
❌ Overcomplicate with too many elements
❌ Use non-standard C4 notation
❌ Store only rendered images without source

## Related Resources

- [Mermaid C4 Templates](./mermaid-templates.md) - Alternative diagram format
- [Complete C4 Examples](./complete-examples.md) - Full system examples
- [Diagramming Best Practices](./best-practices.md) - General C4 guidance

---

**Resource Coverage**: Complete PlantUML templates for all C4 levels
**Includes**: Basic templates, advanced patterns, styling, rendering
**Ready to Use**: Copy-paste templates with detailed examples
