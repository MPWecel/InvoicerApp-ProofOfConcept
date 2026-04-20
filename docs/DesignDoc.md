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
    
