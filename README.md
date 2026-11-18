# Django Store API
 
A RESTful e-commerce backend built with **Django** and **Django REST Framework**. It provides product and category management, shopping carts, order processing, product comments, and JWT-based authentication, all behind role-aware permissions.
 
---
 
## ✨ Features
 
- **Catalog**: products and categories with search, ordering, and inventory filtering.
- **Carts**: carts identified by UUID, with nested cart-item endpoints.
- **Orders**: orders are created from a cart; customers see only their own orders, staff see all.
- **Comments**: nested product comments with a moderation status (waiting, approved, not approved).
- **Authentication**: user registration and JWT login via Djoser and Simple JWT, using a custom user model with a unique email.
- **Automatic customer profiles**: a `Customer` record is created for every new user through a `post_save` signal.
- **Fine-grained permissions**: admin-only writes, Django model permissions, and a custom permission for a private-email action.
- **Rich admin panel**: inventory status filters, inline order and cart items, bulk actions, and comment moderation.
- **Developer tooling**: Django Debug Toolbar, plus a factory-based fake-data module (`store/factories.py`).
## 🧰 Tech Stack
 
| Layer          | Technology                                                 |
| -------------- | ---------------------------------------------------------- |
| Language       | Python 3.12+                                               |
| Framework      | Django 6.x, Django REST Framework                          |
| Auth           | Djoser, `djangorestframework-simplejwt`                    |
| Filtering      | `django-filter`, DRF `SearchFilter` / `OrderingFilter`     |
| Routing        | `drf-nested-routers`                                       |
| Database       | MySQL (via `mysqlclient`); SQLite works for local use      |
| Dev & testing  | `django-debug-toolbar`, `factory_boy`, `Faker`             |
 
## 📁 Project Structure
 
```text
django-drf/
├── manage.py
├── config/                  # Project settings and root URL configuration
│   ├── settings.py
│   └── urls.py
├── core/                    # Custom user model (unique email) and Djoser serializers
│   ├── models.py
│   └── serializers.py
└── store/                   # Main e-commerce app
    ├── models.py            # Category, Product, Customer, Order, Cart, Comment, ...
    ├── serializers.py
    ├── views.py             # ViewSets for every resource
    ├── urls.py              # Router with nested routes
    ├── permissions.py       # Custom permission classes
    ├── filters.py           # Product filter set
    ├── paginations.py
    ├── signals/             # order_created signal and customer-profile handler
    ├── factories.py         # factory_boy factories
    ├── management/commands/setup_fake_data.py
    └── admin.py
```
 
## 🚀 Getting Started
 
### Prerequisites
 
- Python **3.12** or newer
- A running **MySQL** server
- On Debian/Ubuntu, the MySQL client headers needed to build `mysqlclient`:
  `sudo apt install pkg-config default-libmysqlclient-dev build-essential`
### Installation
 
```bash
# 1. Clone the repository
git clone https://github.com/sahebi-code/django-drf.git
cd django-drf
 
# 2. Create and activate a virtual environment
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
 
# 3. Install dependencies
pip install -r requirements.txt
```
 
### Configuration
 
Database settings and `SECRET_KEY` live in [`config/settings.py`](config/settings.py). Before running the project:
 
1. Create a MySQL database (the default name is `store2`) and set your own `USER`, `PASSWORD`, and `HOST` in the `DATABASES` block.
2. Replace the development `SECRET_KEY` with your own value.
> **Security:** never commit real credentials. For anything beyond local development, load secrets from environment variables and set `DEBUG = False`.
 
### Run the project
 
```bash
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```
 
The API is now available at `http://127.0.0.1:8000/`:
 
| URL                              | Description                       |
| -------------------------------- | --------------------------------- |
| `http://127.0.0.1:8000/store/`   | Browsable API root                |
| `http://127.0.0.1:8000/admin/`   | Django admin                      |
| `http://127.0.0.1:8000/auth/`    | Djoser authentication endpoints   |
 
## 🔐 Authentication
 
The API uses **JWT**. Note that the header prefix is `JWT` (not `Bearer`), and access tokens are valid for **1 day**.
 
**1. Register**
 
```bash
curl -X POST http://127.0.0.1:8000/auth/users/ \
  -H "Content-Type: application/json" \
  -d '{"username": "jane", "email": "jane@example.com", "password": "a-strong-password", "first_name": "Jane", "last_name": "Doe"}'
```
 
**2. Obtain a token**
 
```bash
curl -X POST http://127.0.0.1:8000/auth/jwt/create/ \
  -H "Content-Type: application/json" \
  -d '{"username": "jane", "password": "a-strong-password"}'
```
 
**3. Call a protected endpoint**
 
```bash
curl http://127.0.0.1:8000/store/orders/ \
  -H "Authorization: JWT <access_token>"
```
 
## 📡 API Reference
 
### Auth (`/auth/`, provided by Djoser)
 
| Method | Endpoint              | Description                   |
| ------ | --------------------- | ----------------------------- |
| POST   | `/auth/users/`        | Register a new user           |
| GET    | `/auth/users/me/`     | Current user profile          |
| POST   | `/auth/jwt/create/`   | Obtain access and refresh tokens |
| POST   | `/auth/jwt/refresh/`  | Refresh an access token       |
| POST   | `/auth/jwt/verify/`   | Verify a token                |
 
### Store (`/store/`)
 
| Resource   | Endpoint                                  | Methods                      | Notes                                                        |
| ---------- | ----------------------------------------- | ---------------------------- | ------------------------------------------------------------ |
| Products   | `/store/products/`, `/store/products/{id}/` | GET, POST, PUT, PATCH, DELETE | Django model permissions; deleting a product that has order items is rejected |
| Categories | `/store/categories/`, `/store/categories/{id}/` | GET, POST, PUT, PATCH, DELETE | Read-only for non-staff; deleting a category that has products is rejected |
| Comments   | `/store/products/{product_id}/comments/`  | GET, POST, PUT, PATCH, DELETE | Nested under a product                                       |
| Carts      | `/store/carts/`, `/store/carts/{uuid}/`   | POST, GET, DELETE            | Carts are identified by UUID                                 |
| Cart items | `/store/carts/{cart_id}/items/`           | GET, POST, PATCH, DELETE     | Adding an existing product increases its quantity            |
| Customers  | `/store/customers/`                       | GET, POST, PUT, PATCH, DELETE | Admin only                                                   |
|            | `/store/customers/me/`                    | GET                          | The authenticated user's customer profile                    |
|            | `/store/customers/{id}/send_private_email/` | GET                        | Requires the `store.send_private_email` permission           |
| Orders     | `/store/orders/`, `/store/orders/{id}/`   | GET, POST, PATCH, DELETE     | Users see their own orders; staff see all. PATCH and DELETE are admin only |
 
### Product query parameters
 
| Parameter        | Example                               | Description                                  |
| ---------------- | ------------------------------------- | -------------------------------------------- |
| `search`         | `/store/products/?search=laptop`      | Search by product name or category title     |
| `ordering`       | `/store/products/?ordering=-unit_price` | Order by `name`, `unit_price`, or `inventory` |
| `inventory__lt`  | `/store/products/?inventory__lt=10`   | Products with inventory below a value        |
| `inventory__gt`  | `/store/products/?inventory__gt=50`   | Products with inventory above a value        |
 
### Ordering flow
 
1. `POST /store/carts/` creates a cart and returns its UUID.
2. `POST /store/carts/{cart_id}/items/` with `{"product": <id>, "quantity": <n>}` adds products.
3. `POST /store/orders/` with `{"cart_id": "<uuid>"}` converts the cart into an order.
Product responses also include a computed `unit_price_after_tax` field (9% tax).
 
## 🛡️ Roles and Permissions
 
| Role                | Access                                                                                 |
| ------------------- | -------------------------------------------------------------------------------------- |
| Anonymous           | Read categories                                                                        |
| Authenticated user  | Manage carts, place and view own orders, view own customer profile, post comments      |
| User with model permissions | Access products according to their Django `view` / `add` / `change` / `delete` permissions |
| Staff / admin       | Full access to categories, customers, and all orders; update or delete orders          |
 
## 🗂️ Admin Panel
 
The Django admin (`/admin/`) is customized for day-to-day store management:
 
- Products: inventory status labels, a "critical inventory" filter, inline price editing, comment counts linking to filtered comments, and a bulk **Clear inventory** action.
- Orders: inline order items, item counts, and editable status.
- Comments: inline status editing for quick moderation.
- Carts and customers: inline cart items and searchable customer records.
## 👤 Author
 
**sahebi-code**
 
- GitHub: [@sahebi-code](https://github.com/sahebi-code)
- LinkedIn: [mohammad-sahebi](https://www.linkedin.com/in/mohammad-sahebi)
 
