## 1. Application overview::
  InvoicerApp is a POC of an invoicing application using PostgreSQL as a database, ASP.NET WebAPI as a backend REST API gateway and front-end clients (TBD later)

## 2. Requirements::
  ### 2.1 Functional:: The app must allow user to perform the following actions:
    > Create invoices by filling a form, that allows to set a buyer, a seller, and line items of an invoice, fields such as tax and totals need to be autocalculated based on filled form data and predesigned rules. 
      Invoice creation must emit a domain event.
    > Use an existing invoice to generate HTML and PDF files representing that invoice
    > Send the invoice to the buyer (or rather fake it using a fake SMTP like MailHog)
    > Delete an existing invoice
    > List all existing invoices, filter and sort the list, view details of a selected invoice;
    > Generate a periodic report for a selectad range of dates, that would contain the aggregate values such as: number of invoices issued, total value of invoices, etc
    
  ### 2.2 Non-functional::
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
  ### 6.1 Authentication::
    >  POST /api/auth/login — returns JWT access token (24h expiry); 
    >  All other endpoints require Authorization: Bearer <token> header 
    >  DEV seed user: seller@invoiceapp.dev / Password1!
  ### 6.2 Endpoints::
	
	|	#	|	Verb	|	AUTH reqired	|	URL						|	DESCRIPTION										|	
	|=======|===========|===================|===========================|===================================================|	
	|	1	|	POST	|	NO				|	/api/auth/login			|	Authenticate, get JWT token						|	
	|-------|-----------|-------------------|---------------------------|---------------------------------------------------|	
	|	2	|	GET		|	YES				|	/api/invoices			|	List invoices. Query: ?sortBy=supplier/date		|	
	|-------|-----------|-------------------|---------------------------|---------------------------------------------------|	
	|	3	|	GET		|	YES				|	/api/invoices/{id}		|	Get single invoice detail.						|	
	|-------|-----------|-------------------|---------------------------|---------------------------------------------------|	
	|	4	|	POST	|	YES				|	/api/invoices			|	Create invoice.									|	
	|-------|-----------|-------------------|---------------------------|---------------------------------------------------|	
	|	5	|	DELETE	|	YES				|	/api/invoices/{id}		|	Soft-delete an invoice							|	
	|-------|-----------|-------------------|---------------------------|---------------------------------------------------|	
	|	6	|	GET		|	YES				|	/api/invoices/{id}/pdf	|	Download invoice PDF							|	
	|-------|-----------|-------------------|---------------------------|---------------------------------------------------|	
	|	7	|	POST	|	YES				|	/api/invoices/{id}/send	|	Send invoice email to buyer						|	
	|-------|-----------|-------------------|---------------------------|---------------------------------------------------|	
	|	8	|	GET		|	YES				|	/api/reports/sales		|	Sales report. Query: ?from=&to=					|	
	|-------|-----------|-------------------|---------------------------|---------------------------------------------------|	
	
## 7. Database schema::
	### 7.1 Tables::
	
	|	#	|	TABLE				|	COLUMN			|	TYPE					|	NOTES											|	
	|=======|=======================|===================|===========================|===================================================|	
	|	 1	|	Invoices			|	Id				|	UUID PK					|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	 2	|	Invoices			|	Invoice_Number	|	VARCHAR(20)				|	Unique, generated								|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	 3	|	Invoices			|	Created_At		|	TIMESTAMPTZ				|	Index											|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	 4	|	Invoices			|	Status			|	VARCHAR(20)				|	Draft/Sent/Deleted								|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	 5	|	Invoices			|	Tax_Rate		|	DECIMAL(5,4)			|	e.g. 0.2300										|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	 6	|	Invoices			|	Sub_total		|	DECIMAL(18,2)			|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	 7	|	Invoices			|	Tax_Amount		|	DECIMAL(18,2)			|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	 8	|	Invoices			|	Total			|	DECIMAL(18,2)			|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	 9	|	Invoices			|	Supplier_Name	|	VARCHAR(200)			|	Owned Party VO									|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	10	|	Invoices			|	Supplier_Code	|	VARCHAR(50)				|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	11	|	Invoices			|	Supplier_Email	|	VARCHAR(200)			|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	12	|	Invoices			|	Buyer_Name		|	VARCHAR(200)			|	Owned Party VO									|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	13	|	Invoices			|	Buyer_Code		|	VARCHAR(50)				|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	14	|	Invoices			|	Buyer_Email		|	VARCHAR(200)			|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	15	|	Invoice_Items		|	Id				|	UUID PK					|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	16	|	Invoice_Items		|	Invoice_Id		|	UUID FK					|	-> Invoices.Id									|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	17	|	Invoice_Items		|	Description		|	VARCHAR(500)			|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	18	|	Invoice_Items		|	Quantity		|	DECIMAL(10,4)			|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	19	|	Invoice_Items		|	Unit_Price		|	DECIMAL(18,2)			|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	20	|	Invoice_Items		|	Line_Total		|	DECIMAL(18,2)			|	quantity x unit_price							|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	21	|	Domain_Events_Log	|	Id				|	BIGSERIAL PK			|	Audit log										|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	22	|	Domain_Events_Log	|	Event_Type		|	VARCHAR(100)			|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	23	|	Domain_Events_Log	|	Payload			|	JSONB					|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	|	24	|	Domain_Events_Log	|	Occured_At		|	TIMESTAMPTZ				|													|	
	|-------|-----------------------|-------------------|---------------------------|---------------------------------------------------|	
	
	Party data (supplier/buyer) is stored as Owned Entity columns on the invoice table — no separate contacts table. 
	This keeps invoices self-contained and immutable to contact changes.
	
## 8. Infrastructure Services::
	### 8.1 Invoice Generation::
	>	Step 1 — Razor template renders invoice HTML using a shared InvoiceViewModel.: HTML
	>	Step 2 — PuppeteerSharp spins a headless Chromium instance, loads the HTML, and exports to PDF bytes. Chromium binaries are downloaded once at container startup via BrowserFetcher.: PDF
	>	Generated files are not stored on disk; PDF bytes are produced on demand per request (GetInvoicePdfQuery) or attached to email.

	### 8.2 Email::
	>	MailKit library sends SMTP to MailHog on port 1025.
	>	Email body = HTML invoice template. PDF attached as invoice-{number}.pdf.
	>	MailHog web UI runs at http://localhost:8025 for DEV viewing.


	### 8.1 Messaging queue::
	>	MassTransit abstracts the RabbitMQ connection. Configured in Infrastructure layer.
	>	Exchange: invoice.created (fanout). Consumer example included for demo purposes (logs event).
	>	RabbitMQ Management UI at http://localhost:15672 (guest/guest) in DEV.
	
## 9. Authentication & Security::
	>	JWT Bearer tokens issued by the API on successful login.
	>	Token contains: sub (userId), email, role (Seller), expiry (24h).
	>	Both frontends store the token in memory (Blazor: protected localStorage via ILocalStorageService; React: Zustand in-memory store). Never stored in plain localStorage/sessionStorage.
	>	Single seed user created on application startup in DEV via IHostedService.
	>	No user management endpoints — out of scope for this demo.

## 10. Frontend Applications::
	### 10.1 Shared feature set::
	
	### 10.2 Blazor WebAssembly::
	
	### 10.3 React::
	
## 11. Docker & Containerisation::
	### 11.1 Services in docker-compose.yml::
	
	### 11.2 Environment Strategy::
	
## 12. Architectural Decisions Log::



## 13. Implementation Plan::




## 14. Full Tech Stack::




