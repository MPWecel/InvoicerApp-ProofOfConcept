## 1. Application overview:
  InvoicerApp is a POC of an invoicing application using PostgreSQL as a database, ASP.NET WebAPI as a backend REST API gateway and front-end clients (TBD later)

## 2. Requirements:
  ### 2.1 Functional:: The app must allow user to perform the following actions:
    > Create invoices by filling a form, that allows to set a buyer, a seller, and line items of an invoice, fields such as tax and totals need to be autocalculated based on filled form data and predesigned rules. 
      Invoice creation must emit a domain event.
    > Use an existing invoice to generate HTML and PDF files representing that invoice
    > Send the invoice to the buyer (or rather fake it using a fake SMTP like MailHog)
    > Delete an existing invoice
    > List all existing invoices, filter and sort the list, view details of a selected invoice;
    > Generate a periodic report for a selectad range of dates, that would contain the aggregate values such as: number of invoices issued, total value of invoices, etc
    
  ### 2.2 Non-functional
    > Tech stack: .NET version 8 or above, ASP.NET WebAPI, PostgreSQL 16, Docker, frontend: React or Blazor;
    > solution fully containerised using Docker and Docker-compose.
    > environment variables for setting DEV/TEST/PROD environment and appropriate app behaviour depending on those variables. Secrets also read fron ENV
    > Backend app in clean architecture, implementing CQRS and Event Sourcing.
    > Simple authentication and authorisation using JWT tokens - barebones solution is enough;

## 3. Solution architecture::
  ### 3.1 High level overview::
    > backend app follows the Clean Architecture pattern, outer layers depend on the inner, never the other way round.
    > backend app layers:
      >> Domain          -  The innermost layer with no external dependencies. Defines Entities, Value Objects, Domain Events, Enums and Constants
      >> Application     -  Depends on Domain. Defines CQRS Commands/Queries/Handlers (MediatR), DTOs, Interfaces (IInvoiceRepository, IEmailService, IInvoiceGenerator)
      >> Infrastructure  -  Depends on Domain and Application, implements interfaces defined in Application, contains EF Core DbContext, PostgreSQL migrations, MailKit email service, PuppeteerSharp PDF generator, Razor HTML generator, MassTransit/RabbitMQ publisher
      >> API             -  Outermost layer of the backend, exposes API endpoints for client apps to consume and call. Contains: ASP.NET WebAPI controllers, JWT auth middleware, Swagger/Scalar docs, DI composition root, environment config
    > frontend apps:: SPAs in React+Typescript and in Blazor WebAssembly as a comparison.
      >> Blazor WebAssembly SPA Client
      >> React 18 + Typescript SPA using Vite build, React Query for data fetching, Zustand for auth state
  ### 3.2 Planned project structure::
    > Monorepo
    > Folder structure::
      /InvoiceApp
        /docs
        /src
          /InvoiceApp.Domain
          /InvoiceApp.Application
          /InvoiceApp.Infrastructure
          /InvoiceApp.Api
          /InvoiceApp.BlazorClient
          /InvoiceApp.ReactClient
        /docker
          /docker-compose.yml
          /docker-compose.override.yml
        /InvoiceApp.sln
  
## 4.  Domain model::
  ### 4.1 Entities and Value Objects::
    |  #  |  Type        |  Name                 |  Fields                                                                                                        | 
    |=====|==============|=======================|================================================================================================================| 
    |  1  |  Entity      |  Invoice              |  Id, InvoiceNumber, CreatedAt, Status, SupplierId, BuyerId, Items (list), TaxRate, TaxAmount, SubTotal, Total  | 
    |-----|--------------|-----------------------|----------------------------------------------------------------------------------------------------------------| 
    |  2  |  Entity      |  InvoiceItem          |  Id, InvoiceId, Description, Quantity, UnitPrice, LineTotal                                                    | 
    |-----|--------------|-----------------------|----------------------------------------------------------------------------------------------------------------| 
    |  3  |  ValueObject |  Party                |  Name, Code, Email                                                                                             | 
    |-----|--------------|-----------------------|----------------------------------------------------------------------------------------------------------------| 
    |  4  |  DomainEvent |  InvoiceCreatedEvent  |  InvoiceId, InvoiceNumber, SellerEmail, BuyerEmail, CreatedAt                                                  | 
    |-----|--------------|-----------------------|----------------------------------------------------------------------------------------------------------------| 
    |  5  |  Enum        |  InvoiceStatus        |  Draft, Sent, Deleted                                                                                          | 
  ## 4.2 Business rules::
    >  Invoice numbers are auto-generated as INV-YYYYMM-NNNN (sequential per month) 
    >  SubTotal = sum of (Quantity × UnitPrice) across all items 
    >  TaxAmount = SubTotal × TaxRate (default 23% VAT — configurable per invoice)  
    >  Total = SubTotal + TaxAmount  
    >  Invoices in Sent or Deleted status cannot be re-sent  
    >  Soft-delete: invoices are marked Deleted but remain in the database for reporting.

## 5. CQRS Design::
  ### 5.1 Commands::
    > CreateInvoiceCommand  -  Creates an invoice, saves it in database, fires InvoiceCreatedEvent, returns the Id of the created invoice
    > SendInvoiceEmailCommand  -  Generates HTML and PDF from invoice data, sends it via SMTP, updates status, returns an int
    > DeleteInvoiceCommand  -  Performs a soft-delete, returns an int
  
  ### 5.2 Queries::
    > GetInvoicesQuery  -  returns a List of all invoices persisted in the database, except the soft-deleted ones;
    > GetInvoiceByIdQuery  -  returns an invoice DTO pertaining to the invoice of a given id, null if no such id in db
    > GetInvoicePdfQuery  -  generates a pdf file and returns a byte array representing that file
    > GetSalesReportQuery  -  returns a SalesReportDTO that contains aggregate data from invoices from a set date range
  
  ### 5.3 Event flow::
    On invoice creation the following chain executes:
      1. CreateInvoiceCommand dispatched via MediatR 
      2. Handler persists invoice to PostgreSQL (EF Core) 
      3. Handler raises InvoiceCreatedEvent via MediatR Notification 
      4a. InvoiceCreatedEventHandler (in-process): logs to console/DB 
      4b. RabbitMqPublishHandler: publishes to 'invoice.created' exchange via MassTransit

## 6. REST API Design::
  ## 6.1 Authentication::
    >  POST /api/auth/login — returns JWT access token (24h expiry); 
    >  All other endpoints require Authorization: Bearer <token> header 
    >  DEV seed user: seller@invoiceapp.dev / Password1!
  ## 6.2 Endpoints::
    POST    /api/auth/login          No   Authenticate, get JWT token
    GET     /api/invoices            Yes  List invoices. Query: ?sortBy=supplier|date
    GET     /api/invoices/{id}       Yes  Get single invoice detail.
    POST    /api/invoices            Yes  Create invoice.
    DELETE  /api/invoices/{id}       Yes  Soft-delete invoice.
    GET     /api/invoices/{id}/pdf   Yes  Download invoice PDF.
    POST    /api/invoices/{id}/send  Yes  Send invoice email to buyer
    GET     /api/reports/sales       Yes  Sales report. Query: ?from=&to=

## 7. Database schema::
  ## 7.1 Tables::
  
## 8. Infrastructure Services
  ## 8.1 Invoice Generation::

  ## 8.2 Email::

  ## 8.1 Messaging queue::
