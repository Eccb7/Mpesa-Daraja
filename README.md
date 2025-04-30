# M-Pesa Daraja API Implementation Examples (Python/Django & Ruby on Rails)

## Introduction

This repository contains detailed guides and example structures for integrating the Safaricom M-Pesa Daraja API into web applications using popular backend frameworks: Python (with Django) and Ruby (with Ruby on Rails).

The goal is to provide a clear path for developers to implement common M-Pesa functionalities, including:

*   **STK Push (Lipa Na M-Pesa Online):** Initiating payment prompts on customer phones.
*   **C2B (Customer To Business):** Handling payments made by customers to your PayBill or Till Number, including URL registration, validation, and confirmation.
*   **B2C (Business To Customer):** Sending money from your business account to customers (e.g., payouts, refunds).
*   **B2B (Business To Business):** Transferring funds between business shortcodes.
*   **Account Balance:** Querying the balance of your M-Pesa shortcode.

A key focus is the implementation of **webhooks (callbacks)**, which are essential for receiving real-time transaction status updates from Safaricom.

**Note:** These guides are based on the structure observed in a Node.js example and expanded based on the official Daraja API documentation. They emphasize secure credential handling and provide detailed steps for each API interaction.

## Frameworks Covered

1.  **Python (Django):** A high-level Python web framework that encourages rapid development and clean, pragmatic design.
2.  **Ruby on Rails:** A popular web application framework written in Ruby, following the model-view-controller (MVC) pattern.

## Example File Structures

Below are the typical file structures and key files created as per the implementation guides. These represent the core logic for M-Pesa integration within a larger application.

### 1. Python (Django) Structure

```
mpesa_project/
├── mpesa_project/         # Django project settings
│   ├── __init__.py
│   ├── settings.py        # Add 'mpesa_api' to INSTALLED_APPS
│   ├── urls.py            # Include mpesa_api.urls
│   ├── wsgi.py
│   └── asgi.py
├── mpesa_api/             # Django app for M-Pesa logic
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations/
│   │   └── __init__.py
│   ├── models.py          # (Optional but recommended) Define models for transactions
│   ├── services.py        # <<< Core M-Pesa API interaction logic, token generation, specific API calls
│   ├── tests.py
│   ├── urls.py            # <<< URL patterns for API endpoints and callbacks
│   └── views.py           # <<< Django views to handle requests and callbacks
├── venv/                  # Virtual environment directory
├── manage.py              # Django management script
└── .env                   # <<< Store sensitive credentials (MPESA_KEY, SECRET, etc.)
```

**Key Django Files:**

*   `.env`: Stores M-Pesa API keys, shortcode, passkey, etc.
*   `mpesa_api/services.py`: Contains functions for getting access tokens, generating passwords, and making calls to specific Daraja endpoints (STK Push, C2B, B2C, B2B, Balance).
*   `mpesa_api/views.py`: Defines view functions that handle incoming HTTP requests to trigger M-Pesa actions and, crucially, to receive and process callbacks (webhooks) from Safaricom.
*   `mpesa_api/urls.py`: Maps URL paths (like `/api/mpesa/stk_push/`, `/api/mpesa/stk_callback/`) to the corresponding views in `views.py`.
*   `mpesa_project/settings.py`: Project configuration, including adding the `mpesa_api` app.
*   `mpesa_project/urls.py`: Main project URL configuration, includes the URLs from the `mpesa_api` app.

### 2. Ruby on Rails Structure

```
mpesa_rails_app/
├── app/
│   ├── controllers/
│   │   └── api/
│   │       └── v1/
│   │           └── mpesa_api_controller.rb  # <<< Controller for M-Pesa actions & callbacks
│   ├── models/                  # (Optional but recommended) Define models for transactions
│   └── services/
│       └── mpesa_service.rb     # <<< Service object for M-Pesa API logic
├── config/
│   ├── environment.rb
│   ├── initializers/
│   └── routes.rb              # <<< Define API routes for M-Pesa endpoints & callbacks
├── lib/
├── log/
├── public/
├── storage/
├── test/
├── tmp/
├── vendor/
├── Gemfile                    # Add 'httparty', 'dotenv-rails'
├── Gemfile.lock
├── Rakefile
├── config.ru
└── .env                       # <<< Store sensitive credentials (MPESA_KEY, SECRET, etc.)
```

**Key Rails Files:**

*   `.env`: Stores M-Pesa API keys, shortcode, passkey, etc.
*   `app/services/mpesa_service.rb`: A service object encapsulating logic for token generation, password creation, and communication with the Daraja API endpoints.
*   `app/controllers/api/v1/mpesa_api_controller.rb`: Handles incoming HTTP requests, calls the `MpesaService`, and manages responses. Contains actions for both initiating requests and receiving callbacks.
*   `config/routes.rb`: Defines the URL routes that map to the controller actions, including API endpoints and callback URLs.
*   `Gemfile`: Lists project dependencies, including `httparty` for HTTP requests and `dotenv-rails` for environment variable management.

## Detailed Guides

For step-by-step implementation details, please refer to the specific guides:

*   **Python/Django:** [`python_django_mpesa_guide.md`](./python_django_mpesa_guide.md)
*   **Ruby on Rails:** [`ruby_rails_mpesa_guide.md`](./ruby_rails_mpesa_guide.md)

These guides cover setup, code implementation for each API endpoint, webhook handling logic, and important security considerations.

