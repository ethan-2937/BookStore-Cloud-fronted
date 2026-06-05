# BookStore Cloud Frontend

Vue 3 + Vite + Element Plus frontend for the Bookstore Cloud online bookstore.

## Tech Stack

- Vue 3
- Vite
- Element Plus
- Nginx for Docker production serving

## Local Run

The Vite dev server proxies `/api` to `http://localhost:8090`.

```powershell
npm install
npm run dev
```

Frontend dev server runs at:

```text
http://localhost:5173
```

## Docker Run

Start the backend first, then run:

```powershell
docker compose up -d --build
```

Docker frontend runs at:

```text
http://localhost:18080
```

The Nginx container proxies `/api` to `http://host.docker.internal:18090` by default. To use another backend address:

```powershell
$env:API_PROXY_PASS="http://your-backend-host:18090"
docker compose up -d --build
```

## Main Features

- Book browsing with backend pagination
- Category and keyword search
- Cart and order creation
- Pay/cancel orders
- Floating AI customer service chat for book recommendations
- Review purchased books from order items
- User dropdown for orders, support, reviews, and logout
- Admin login for book/order/support management
