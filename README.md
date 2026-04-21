# 🚌 Bus Ticket Booking System — Class Diagrams (Mermaid)

> **Use for presentations:** open this file in any Mermaid-compatible viewer (GitHub preview, VS Code + Mermaid plugin, Typora, Obsidian, or paste into https://mermaid.live). Each diagram renders automatically.

> **Architecture:** Distributed 2-tier Spring Boot split — **Backend** (`:8080`, REST API + JPA + MySQL) and **Frontend** (`:8081`, Thymeleaf + RestTemplate). Browser never talks to backend directly.

---

## Table of Contents

1. [System Overview (High-Level)](#1-system-overview-high-level)
2. [Backend — Domain Entities](#2-backend--domain-entities-jpa)
3. [Backend — Layered Architecture (one module deep dive: Booking)](#3-backend--layered-architecture-booking-module-deep-dive)
4. [Backend — All REST Controllers & Services](#4-backend--all-rest-controllers--services)
5. [Backend — Security & Config](#5-backend--security--config-round-4-hardening)
6. [Frontend — API Client Hierarchy](#6-frontend--api-client-hierarchy)
7. [Frontend — View Controllers](#7-frontend--view-controllers)
8. [Frontend — DTO Layer](#8-frontend--dto-layer)
9. [Cross-System Integration (Frontend ↔ Backend)](#9-cross-system-integration-frontend--backend)
10. [Request Flow — Booking a Seat (Sequence Diagram)](#10-request-flow--booking-a-seat-sequence-diagram)
11. [Security Flow — HTTP Basic + CORS (Sequence Diagram)](#11-security-flow--http-basic--cors-sequence-diagram)

---

## 1. System Overview (High-Level)

```mermaid
flowchart LR
    Browser["👤 Browser<br/>localhost:8081"]

    subgraph FE["🟦 FRONTEND (Spring Boot :8081)"]
        direction TB
        Thymeleaf[Thymeleaf Templates]
        VC[View Controllers]
        AC[API Clients]
        RT["RestTemplate<br/>+ Basic Auth"]
        Thymeleaf --> VC
        VC --> AC
        AC --> RT
    end

    subgraph BE["🟩 BACKEND (Spring Boot :8080)"]
        direction TB
        RC["REST Controllers<br/>/api/**"]
        SVC[Services]
        REPO[JPA Repositories]
        SEC["SecurityConfig<br/>HTTP Basic + CORS"]
        RC --> SVC
        SVC --> REPO
        SEC -.guards.-> RC
    end

    DB[("🗄️ MySQL<br/>busticketbooking")]

    Browser -->|"HTTPS (view flow)"| FE
    RT -->|"REST + Basic Auth<br/>Authorization: Basic ..."| RC
    REPO -->|JDBC| DB

    style FE fill:#e3f2fd,stroke:#1976d2
    style BE fill:#e8f5e9,stroke:#388e3c
    style DB fill:#fff3e0,stroke:#f57c00
    style Browser fill:#fce4ec,stroke:#c2185b
```

---

## 2. Backend — Domain Entities (JPA)

```mermaid
classDiagram
    direction LR

    class Agency {
        -Integer agencyId
        -String name
        -String phone
        -String email
        -LocalDate foundedDate
    }

    class Office {
        -Integer officeId
        -String officeName
        -String phone
        -Agency agency
        -Address address
    }

    class Bus {
        -Integer busId
        -String registrationNumber
        -String type
        -Integer capacity
        -Office office
    }

    class Driver {
        -Integer driverId
        -String name
        -String licenseNumber
        -String phone
        -Office office
    }

    class Customer {
        -Integer customerId
        -String name
        -String email
        -String phone
        -Address address
    }

    class Address {
        -Integer addressId
        -String address
        -String city
        -String state
        -String pincode
    }

    class Route {
        -Integer routeId
        -String fromCity
        -String toCity
        -BigDecimal distance
    }

    class Trip {
        -Integer tripId
        -LocalDateTime departureTime
        -LocalDateTime arrivalTime
        -Integer availableSeats
        -BigDecimal fare
        -LocalDateTime tripDate
        -Route route
        -Bus bus
        -Driver driver1
        -Driver driver2
        -Address boardingAddress
        -Address droppingAddress
    }

    class Booking {
        -Integer bookingId
        -Integer seatNumber
        -BookingStatus status
        -Trip trip
        -Customer customer
    }

    class Payment {
        -Integer paymentId
        -BigDecimal amount
        -PaymentStatus paymentStatus
        -LocalDateTime paymentDate
        -Booking booking
        -Customer customer
    }

    class Review {
        -Integer reviewId
        -Integer rating
        -String comment
        -LocalDateTime reviewDate
        -Customer customer
        -Trip trip
    }

    class BookingStatus {
        <<enum>>
        Available
        Booked
        Cancelled
    }

    class PaymentStatus {
        <<enum>>
        PENDING
        COMPLETED
        FAILED
    }

    Agency "1" --o "N" Office
    Office "1" --o "N" Bus
    Office "1" --o "N" Driver
    Customer "1" --o "1" Address
    Office "1" --o "1" Address
    Trip "N" --> "1" Route
    Trip "N" --> "1" Bus
    Trip "N" --> "1" Driver : driver1
    Trip "N" --> "0..1" Driver : driver2
    Trip "N" --> "1" Address : boarding
    Trip "N" --> "1" Address : dropping
    Booking "N" --> "1" Trip
    Booking "N" --> "1" Customer
    Booking "1" --> "1" BookingStatus
    Payment "N" --> "1" Booking
    Payment "N" --> "1" Customer
    Payment "1" --> "1" PaymentStatus
    Review "N" --> "1" Customer
    Review "N" --> "1" Trip
```

---

## 3. Backend — Layered Architecture (Booking module deep-dive)

```mermaid
classDiagram
    direction TB

    class BookingController {
        <<@Controller>>
        -BookingService bookingService
        -TicketPdfService ticketPdfService
        -TripService tripService
        +initiateBooking(BookingRequestDTO) BookingResponseDTO
        +getBookingsForTrip(Integer) List~Map~
        +apiGetBookingById(Integer) Map
        +downloadTicketPdf(Integer) ResponseEntity~byte[]~
        +downloadGroupBookingTicket(String) ResponseEntity~byte[]~
        +showSeatSelection(Integer) String
        +bookSeats(Integer, List, Integer) String
    }

    class BookingService {
        <<@Service>>
        -BookingRepository bookingRepository
        -TripRepository tripRepository
        +initiateBooking(BookingRequestDTO) BookingResponseDTO
        +getBookingsForTrip(Integer) List~Booking~
        +getBookingById(Integer) Booking
        +buildConfirmationContext(Integer) ConfirmationContext
        -validateSeatNumbers(List) void
        -getValidatedTrip(Integer, int) Trip
        -processSeats(BookingRequestDTO, Trip) SeatBookingResult
        -createOrUpdateBooking(...) Booking
    }

    class BookingRepository {
        <<@Repository>>
        +findById(Integer) Optional~Booking~
        +findByTrip_TripId(Integer) List~Booking~
        +findByIdWithTripDetails(Integer) Optional~Booking~
        +findByTrip_TripIdAndSeatNumber(...) Optional~Booking~
    }

    class TicketPdfService {
        <<@Service>>
        +generateTicketPdf(Integer) byte[]
        +generateGroupBookingTicket(List) byte[]
    }

    class BookingRequestDTO {
        <<DTO>>
        +Integer tripId
        +List~Integer~ seatNumbers
        +Integer customerId
    }

    class BookingResponseDTO {
        <<DTO>>
        +List~Integer~ bookingIds
        +String message
        +BigDecimal totalFare
        +Integer customerId
    }

    class SeatBookingResult {
        <<record>>
        +List~Integer~ bookingIds
        +BigDecimal totalFare
    }

    BookingController --> BookingService
    BookingController --> TicketPdfService
    BookingService --> BookingRepository
    BookingService ..> SeatBookingResult : produces
    BookingController ..> BookingRequestDTO : consumes
    BookingController ..> BookingResponseDTO : returns
    BookingService ..> BookingRequestDTO : consumes
    BookingService ..> BookingResponseDTO : returns
```

---

## 4. Backend — All REST Controllers & Services

```mermaid
classDiagram
    direction LR

    class AgencyController
    class AgencyOfficeController
    class BusController
    class DriverController
    class CustomerController
    class AddressController
    class RouteController
    class TripController
    class BookingController
    class PaymentController
    class ReviewController

    class AgencyService
    class AgencyOfficeService
    class BusService
    class DriverService
    class CustomerService
    class AddressService
    class RouteService
    class TripService
    class BookingService
    class PaymentService
    class ReviewService
    class TicketPdfService

    class GlobalExceptionHandler {
        <<@RestControllerAdvice>>
        +handleResourceNotFound(...) ResponseEntity
        +handleBadRequest(...) ResponseEntity
        +handleValidation(...) ResponseEntity
        +handleGeneric(...) ResponseEntity
    }

    AgencyController --> AgencyService
    AgencyOfficeController --> AgencyOfficeService
    BusController --> BusService
    BusController --> AgencyOfficeService
    DriverController --> DriverService
    CustomerController --> CustomerService
    AddressController --> AddressService
    RouteController --> RouteService
    TripController --> TripService
    TripController --> RouteService
    TripController --> BusService
    TripController --> DriverService
    TripController --> AddressService
    BookingController --> BookingService
    BookingController --> TicketPdfService
    BookingController --> TripService
    PaymentController --> PaymentService
    PaymentController --> BookingService
    PaymentController --> TicketPdfService
    ReviewController --> ReviewService
    ReviewController --> CustomerService
    ReviewController --> TripService

    GlobalExceptionHandler ..> AgencyController : advises
    GlobalExceptionHandler ..> BookingController : advises
    GlobalExceptionHandler ..> PaymentController : advises
```

---

## 5. Backend — Security & Config (Round 4 Hardening)

```mermaid
classDiagram
    direction TB

    class SecurityConfig {
        <<@Configuration>>
        -String allowedOrigins
        +apiSecurityFilterChain(HttpSecurity) SecurityFilterChain
        +webSecurityFilterChain(HttpSecurity) SecurityFilterChain
        +corsConfigurationSource() CorsConfigurationSource
    }

    class SecurityFilterChain_API {
        <<@Order(1)>>
        securityMatcher /api/**
        csrf disabled
        httpBasic enabled
        sessionPolicy STATELESS
    }

    class SecurityFilterChain_Web {
        <<@Order(2)>>
        everything else
        csrf enabled
        formLogin /login
        redirects anonymous
    }

    class CorsConfigurationSource {
        <<@Bean>>
        +allowedOrigins APP_CORS_ORIGINS
        +allowedMethods GET POST PUT DELETE OPTIONS
        +allowCredentials true
        +path /api/**
    }

    class InMemoryUserDetailsManager {
        <<built-in>>
        +user APP_ADMIN_USER
        +password APP_ADMIN_PASSWORD
        +role ADMIN
    }

    SecurityConfig --> SecurityFilterChain_API : produces bean
    SecurityConfig --> SecurityFilterChain_Web : produces bean
    SecurityConfig --> CorsConfigurationSource : produces bean
    SecurityFilterChain_API ..> CorsConfigurationSource : uses
    SecurityFilterChain_API ..> InMemoryUserDetailsManager : authenticates against
    SecurityFilterChain_Web ..> InMemoryUserDetailsManager : authenticates against
```

---

## 6. Frontend — API Client Hierarchy

```mermaid
classDiagram
    direction TB

    class AbstractApiClient {
        <<abstract>>
        #RestTemplate restTemplate
        #String baseUrl
        #url(String) String
        #get(String, Class) T
        #getList(String, TypeReference) List~T~
        #post(String, B, Class) T
        #put(String, B, Class) T
        #delete(String) void
        #getBytes(String) byte[]
        -translate(Exception) BackendException
        -extractMessage(String, String) String
    }

    class BookingApiClient {
        +createBooking(BookingRequestDTO) BookingResponseDTO
        +getByTrip(Integer) List~Map~
        +getById(Integer) Map
        +downloadTicket(Integer) byte[]
        +downloadGroupTicket(List) byte[]
    }

    class TripApiClient {
        +getAllTrips() List~TripDTO~
        +getById(Integer) TripDTO
        +searchTrips(String, String) List~TripDTO~
    }

    class CustomerApiClient {
        +getAll() List~CustomerResponseDTO~
        +getById(Integer) CustomerResponseDTO
        +create(CustomerRequestDTO) CustomerResponseDTO
    }

    class PaymentApiClient {
        +getAll() List~PaymentResponseDTO~
        +getById(Integer) PaymentResponseDTO
        +processPayment(PaymentRequestDTO) PaymentResponseDTO
        +downloadTicketByPaymentId(Integer) byte[]
        +downloadGroupTicket(List) byte[]
    }

    class BusApiClient
    class DriverApiClient
    class AgencyApiClient
    class AgencyOfficeApiClient
    class AddressApiClient
    class RouteApiClient
    class ReviewApiClient

    class BackendException {
        <<RuntimeException>>
        -int status
        -String message
        +getStatus() int
    }

    AbstractApiClient <|-- BookingApiClient
    AbstractApiClient <|-- TripApiClient
    AbstractApiClient <|-- CustomerApiClient
    AbstractApiClient <|-- PaymentApiClient
    AbstractApiClient <|-- BusApiClient
    AbstractApiClient <|-- DriverApiClient
    AbstractApiClient <|-- AgencyApiClient
    AbstractApiClient <|-- AgencyOfficeApiClient
    AbstractApiClient <|-- AddressApiClient
    AbstractApiClient <|-- RouteApiClient
    AbstractApiClient <|-- ReviewApiClient
    AbstractApiClient ..> BackendException : throws
```

---

## 7. Frontend — View Controllers

```mermaid
classDiagram
    direction LR

    class HomePageController {
        +home() String
        +login() String
    }

    class BookingViewController {
        -BookingApiClient bookingApi
        -TripApiClient tripApi
        -CustomerApiClient customerApi
        -BusApiClient busApi
        +listAvailableTrips(Model) String
        +showSeatSelection(Integer) String
        +bookSeats(...) String
        +downloadBookingTicket(Integer) ResponseEntity
        +downloadBookingGroupTicket(String) ResponseEntity
    }

    class PaymentViewController {
        -PaymentApiClient paymentApi
        -BookingApiClient bookingApi
        -CustomerApiClient customerApi
        -TripApiClient tripApi
        +listPayments(Model) String
        +showCheckout(Integer) String
        +processPaymentView(...) String
        +downloadPaymentTicket(Integer) ResponseEntity
    }

    class TripViewController
    class CustomerViewController
    class AddressViewController
    class AgencyViewController
    class OfficeViewController
    class BusViewController
    class DriverViewController
    class RouteViewController
    class ReviewViewController

    class MemberController {
        -MemberRegistry registry
        -OperationExecutor executor
        -RestTemplate restTemplate
        +listMembers() String
        +memberDetail(Integer) String
        +executeOperation(...) String
        +downloadPdf(...) ResponseEntity
    }

    BookingViewController --> BookingApiClient
    BookingViewController --> TripApiClient
    BookingViewController --> CustomerApiClient
    BookingViewController --> BusApiClient
    PaymentViewController --> PaymentApiClient
    PaymentViewController --> BookingApiClient
    PaymentViewController --> CustomerApiClient
    PaymentViewController --> TripApiClient
```

---

## 8. Frontend — DTO Layer

```mermaid
classDiagram
    direction TB

    class TripDTO {
        +Integer tripId
        +Integer routeId
        +Integer busId
        +LocalDateTime departureTime
        +LocalDateTime arrivalTime
        +Integer availableSeats
        +BigDecimal fare
        +String fromCity
        +String toCity
        +String busType
    }

    class BookingRequestDTO {
        +Integer tripId
        +List~Integer~ seatNumbers
        +Integer customerId
    }

    class BookingResponseDTO {
        +List~Integer~ bookingIds
        +String message
        +BigDecimal totalFare
        +Integer customerId
    }

    class PaymentRequestDTO {
        +Integer bookingId
        +Integer customerId
        +BigDecimal amount
    }

    class PaymentResponseDTO {
        +Integer paymentId
        +Integer bookingId
        +Integer customerId
        +BigDecimal amount
        +PaymentStatus paymentStatus
        +LocalDateTime paymentDate
    }

    class PaymentGroup {
        +List~Integer~ paymentIds
        +List~Integer~ bookingIds
        +Integer customerId
        +BigDecimal totalAmount
        +PaymentStatus paymentStatus
        +Integer seatCount
        +getFirstPaymentId() Integer
    }

    class CustomerRequestDTO
    class CustomerResponseDTO
    class BusRequestDTO
    class BusResponseDTO
    class DriverRequestDTO
    class DriverResponseDTO
    class AgencyRequestDTO
    class AgencyResponseDTO
    class AddressDTO
    class RouteDTO
    class ReviewDTO
    class OfficeRequestDTO
    class OfficeResponseDTO

    class BookingStatus {
        <<enum>>
        Available
        Booked
        Cancelled
    }

    class PaymentStatus {
        <<enum>>
        PENDING
        COMPLETED
        FAILED
    }

    BookingResponseDTO ..> BookingStatus
    PaymentResponseDTO ..> PaymentStatus
    PaymentGroup ..> PaymentStatus
    PaymentGroup "1" --o "N" PaymentResponseDTO : groups
```

---

## 9. Cross-System Integration (Frontend ↔ Backend)

```mermaid
classDiagram
    direction LR

    namespace Frontend {
        class RestClientConfig {
            <<@Configuration>>
            -long connectTimeoutMs
            -long readTimeoutMs
            -String backendUsername
            -String backendPassword
            +restTemplate(RestTemplateBuilder) RestTemplate
        }

        class AbstractApiClient {
            <<abstract>>
            #RestTemplate restTemplate
            #String baseUrl
        }

        class BookingApiClient {
            +createBooking(BookingRequestDTO) BookingResponseDTO
        }

        class BookingViewController_FE {
            +bookSeats(...) String
        }
    }

    namespace Backend {
        class SecurityFilterChain_API_BE {
            <<@Order(1)>>
            httpBasic
            STATELESS
        }

        class BookingController_BE {
            <<@RestController>>
            +initiateBooking(BookingRequestDTO) BookingResponseDTO
        }

        class BookingService_BE {
            <<@Service>>
            +initiateBooking(...) BookingResponseDTO
        }

        class BookingRepository_BE {
            <<@Repository>>
        }
    }

    class MySQL_DB {
        <<database>>
        busticketbooking
        JDBC
    }

    RestClientConfig ..> AbstractApiClient : provides RestTemplate
    BookingViewController_FE --> BookingApiClient
    BookingApiClient --|> AbstractApiClient
    AbstractApiClient -->|"HTTP POST /api/bookings<br/>Authorization: Basic ..."| SecurityFilterChain_API_BE : "REST over HTTP"
    SecurityFilterChain_API_BE -->|"Authorized"| BookingController_BE
    BookingController_BE --> BookingService_BE
    BookingService_BE --> BookingRepository_BE
    BookingRepository_BE -->|JDBC| MySQL_DB
```

---

## 10. Request Flow — Booking a Seat (Sequence Diagram)

```mermaid
sequenceDiagram
    autonumber
    participant U as 👤 User (Browser)
    participant VC as 🟦 BookingViewController<br/>(Frontend :8081)
    participant BAC as 🟦 BookingApiClient
    participant RT as 🟦 RestTemplate<br/>(+ Basic Auth interceptor)
    participant SEC as 🟩 SecurityFilterChain<br/>(Backend :8080)
    participant BC as 🟩 BookingController
    participant BS as 🟩 BookingService
    participant BR as 🟩 BookingRepository
    participant DB as 🗄️ MySQL

    U->>VC: POST /view/bookings/book<br/>(tripId, seats, customerId)
    VC->>VC: build BookingRequestDTO
    VC->>BAC: createBooking(dto)
    BAC->>RT: postForObject(url, dto, BookingResponseDTO)
    RT->>RT: add header Authorization: Basic base64(user:pwd)
    RT->>SEC: POST /api/bookings
    SEC->>SEC: validate HTTP Basic creds
    alt creds invalid
        SEC-->>RT: 401 Unauthorized
        RT-->>BAC: throws BackendException(401)
        BAC-->>VC: propagate
        VC-->>U: redirect with error flash
    else creds valid
        SEC->>BC: forward to controller
        BC->>BS: initiateBooking(dto)
        BS->>BS: validateSeatNumbers()
        BS->>BR: findByIdForUpdate(tripId)
        BR->>DB: SELECT ... FOR UPDATE
        DB-->>BR: Trip row
        BR-->>BS: Trip
        BS->>BS: processSeats() per seat
        loop for each seat
            BS->>BR: save(booking)
            BR->>DB: INSERT booking
        end
        BS->>BR: save(trip) [decremented seats]
        BR->>DB: UPDATE trip
        BS-->>BC: BookingResponseDTO
        BC-->>SEC: 200 OK + JSON body
        SEC-->>RT: response
        RT-->>BAC: BookingResponseDTO
        BAC-->>VC: return dto
        VC-->>U: redirect /view/bookings/confirmation/{id}
    end
```

---

## 11. Security Flow — HTTP Basic + CORS (Sequence Diagram)

```mermaid
sequenceDiagram
    autonumber
    participant RT as 🟦 Frontend RestTemplate
    participant CORS as 🟩 CorsFilter
    participant AUTH as 🟩 BasicAuthenticationFilter
    participant AUTHZ as 🟩 AuthorizationFilter
    participant CTL as 🟩 REST Controller

    Note over RT,CTL: Two chains: /api/** uses this one;<br/>everything else uses webSecurityFilterChain (form login)

    RT->>CORS: GET /api/trips<br/>Origin: http://localhost:8081<br/>Authorization: Basic YWRtaW46cHdk
    CORS->>CORS: check origin against<br/>APP_CORS_ORIGINS allowlist

    alt origin blocked
        CORS-->>RT: reject (no CORS headers)
    else origin allowed
        CORS->>AUTH: forward with CORS headers
        AUTH->>AUTH: decode base64(user:pwd)
        AUTH->>AUTH: check InMemoryUserDetailsManager

        alt creds invalid
            AUTH-->>RT: 401 WWW-Authenticate: Basic realm=...
        else creds valid
            AUTH->>AUTHZ: set SecurityContext
            AUTHZ->>AUTHZ: check /api/** → authenticated() ✓
            AUTHZ->>CTL: forward to controller
            CTL-->>AUTHZ: 200 OK + JSON
            AUTHZ-->>AUTH: response
            AUTH-->>CORS: response
            CORS-->>RT: 200 OK<br/>Access-Control-Allow-Origin: http://localhost:8081<br/>Access-Control-Allow-Credentials: true
        end
    end
```

---

## 📝 Tips for Your Presentation

1. **Start with Diagram 1 (System Overview)** — gives the big picture in 10 seconds.
2. **Show Diagram 2 (Entities)** next — proves you understand the domain.
3. **Use Diagram 3 (Booking module layered)** to demonstrate clean architecture.
4. **Diagram 6 (API Client hierarchy)** proves DRY / inheritance on the frontend.
5. **Close with Diagrams 10 + 11 (sequence diagrams)** — these turn static boxes into a runtime story, which is what examiners remember.
6. **Skip Diagrams 4, 7, 8** if time is short — they're reference material, not storytelling material.

## How to Render

| Tool | How |
|---|---|
| **GitHub** | Commit the `.md` — Mermaid renders automatically in preview. |
| **VS Code** | Install extension `Markdown Preview Mermaid Support`, then `Ctrl+Shift+V`. |
| **Typora / Obsidian** | Native support — opens straight away. |
| **Export to PNG/SVG** | Paste each diagram at https://mermaid.live, click "Actions → PNG / SVG". |
| **In PowerPoint** | Export each as PNG from mermaid.live, paste into slides. |

---

**File location:** `C:\Users\Sarthak\OneDrive\Desktop\Bus-Ticket-Class-Diagrams.md`

**Total diagrams:** 11 (1 flowchart + 7 class diagrams + 2 sequence diagrams + 1 cross-system integration diagram)

**Coverage:** All 12 backend REST controllers, 11 backend services, 11 JPA entities, 11 frontend API clients, 12 frontend view controllers, 24+ DTOs, Security (Round-4) config, and full request-path trace from browser to MySQL.

