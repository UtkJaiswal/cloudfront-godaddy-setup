# AWS Micro-Frontend Deployment Guide (React, Nginx, Docker & CloudFront)

## Architecture Overview

This document outlines a scalable micro-frontend deployment architecture for hosting a React application on a subpath alongside a backend service.

### Tech Stack

- **CDN / Edge Routing:** CloudFront
- **Reverse Proxy:** EC2 instance running Nginx
- **Frontend:** Dockerized React/Vite application served via Nginx
- **Backend API:** Python/Node.js application
- **DNS:** Route 53 (or any DNS provider)

### Architecture Flow

```text
User Request
      ↓
CloudFront (CDN)
      ↓
EC2 Nginx Reverse Proxy
   ├── Frontend → Docker Container (Nginx)
   └── Backend → API Server
```

---

# Step 1: React Application Configuration (Vite + React Router)

When deploying a React application inside a subpath (example: `/app-name/`) instead of the root (`/`), both **Vite** and **React Router** must be configured properly.

## 1.1 Vite Configuration (`vite.config.ts`)

Vite must know where compiled assets (JS, CSS, images) will be served from.

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react-swc";
import tailwindcss from "@tailwindcss/vite";
import path from "path";

export default defineConfig(({ command }) => {
  const isLocalDev = command === "serve";

  return {
    /**
     * Local Development:
     * http://localhost:5173/
     *
     * Production:
     * https://your-domain.com/app-name/
     */
    base: isLocalDev ? "/" : "/app-name/",

    plugins: [
      react(),
      tailwindcss(),
    ],

    resolve: {
      alias: {
        "@": path.resolve(__dirname, "./src"),
      },
    },
  };
});
```

### Why this is needed

During production builds, Vite generates asset paths.

Without a proper `base` path:

```text
/main.js
/styles.css
```

Instead of:

```text
/app-name/main.js
/app-name/styles.css
```

This breaks the application when deployed under a subpath.

---

## 1.2 React Router Configuration

React Router also needs to understand its base route.

### `App.tsx` or `main.tsx`

```tsx
import { BrowserRouter, Routes, Route } from "react-router-dom";

function App() {
  return (
    <BrowserRouter basename={import.meta.env.BASE_URL}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/dashboard" element={<Dashboard />} />
      </Routes>
    </BrowserRouter>
  );
}

export default App;
```

### Why this is needed

Without `basename`, refreshing URLs such as:

```text
/app-name/dashboard
```

can lead to React Router throwing a **404 route mismatch**.

---

# Step 2: Docker Configuration

The frontend application runs inside a Docker container using Nginx to serve static assets.

The container itself should remain unaware of the external subpath routing.

## 2.1 Container Nginx Configuration

Create a file:

### `container-nginx.conf`

```nginx
server {
    listen 3000;
    root /usr/share/nginx/html;

    # Prevent nginx from appending internal ports to redirects
    port_in_redirect off;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Why this is needed

React is a Single Page Application (SPA).

If a user refreshes:

```text
/app-name/dashboard
```

Nginx must serve:

```text
/index.html
```

instead of returning:

```text
404 Not Found
```

`try_files` ensures React Router handles routing.

---

# Step 3: EC2 Nginx Reverse Proxy Configuration

The EC2 instance acts as a **traffic router**.

Its responsibility is to:

1. Accept requests from CloudFront
2. Route frontend traffic to Docker
3. Route backend traffic to API service
4. Strip the application subpath before forwarding requests

---

## 3.1 Nginx Configuration

Location:

```text
/etc/nginx/sites-available/default
```

Example configuration:

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    # Redirect exact path to trailing slash
    location = /app-name {
        return 301 https://$host/app-name/;
    }

    # Frontend Route
    location /app-name/ {

        # IMPORTANT:
        # Trailing slash strips the prefix
        proxy_pass http://127.0.0.1:3000/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Backend/API Route
    location / {
        proxy_pass http://127.0.0.1:5000/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## Important Nginx Detail: Trailing Slash Behavior

This line is critical:

```nginx
proxy_pass http://127.0.0.1:3000/;
```

### With trailing slash

Request:

```text
/app-name/dashboard
```

Forwarded to container as:

```text
/dashboard
```

### Without trailing slash

Request:

```text
/app-name/dashboard
```

Forwarded as:

```text
/app-name/dashboard
```

This usually breaks frontend routing.

---

# Step 4: CloudFront Configuration

CloudFront acts as the global CDN and routing layer.

It decides:

- Which requests go to frontend
- Which requests go elsewhere
- What should be cached

---

## 4.1 Add an EC2 Origin

In CloudFront:

### Create Origin

Configuration:

| Setting | Value |
|----------|-------|
| Origin Type | Custom Origin |
| Origin Domain | EC2 Public DNS / Load Balancer DNS |
| Protocol | HTTP Only |
| HTTP Port | 80 |

---

## 4.2 Create a Behavior

Add a behavior for frontend routing.

### Example Configuration

| Setting | Value |
|----------|-------|
| Path Pattern | `/app-name*` |
| Origin | EC2 Origin |
| Viewer Protocol Policy | Redirect HTTP to HTTPS |
| Allowed Methods | GET, HEAD, OPTIONS |
| Cache Policy | CachingOptimized |
| Origin Request Policy | None |

### Why this is needed

CloudFront must bypass the default origin when it sees:

```text
/app-name
```

and instead send requests to the EC2 instance.

---

# Step 5: DNS Configuration

To support:

```text
example.com
```

and

```text
www.example.com
```

both should point to CloudFront.

### Typical Setup

1. Add alternate domain names in CloudFront
2. Create DNS records pointing to CloudFront
3. Enable SSL certificate for all domains

---

# Step 6: Deployment Workflow

Whenever:

- New frontend code is deployed
- Nginx configuration changes
- Static assets are updated

perform cache invalidation.

---

## CloudFront Cache Invalidation

CloudFront aggressively caches:

- JavaScript bundles
- CSS
- Redirects
- 404 pages

After deployment:

1. Open CloudFront
2. Navigate to **Invalidations**
3. Create a new invalidation

### Path

```text
/*
```

Wait for invalidation completion before testing.

---

# Common Debugging Checklist

## React App Loads Blank Page

Check:

- `vite.config.ts` base path
- Browser console errors
- Static asset URLs

---

## Refresh Gives 404

Check:

```nginx
try_files $uri $uri/ /index.html;
```

inside container Nginx.

---

## URLs Include Internal Port

Example:

```text
:3000
```

Check:

```nginx
port_in_redirect off;
```

---

## Frontend Route Not Working

Check:

```tsx
basename={import.meta.env.BASE_URL}
```

in `BrowserRouter`.

---

## Subpath Not Stripped

Check:

```nginx
proxy_pass http://127.0.0.1:3000/;
```

The trailing slash is mandatory.

---

# Recommended Deployment Order

```text
1. Build frontend
2. Build Docker image
3. Deploy container
4. Verify Nginx config
5. Restart nginx
6. Test locally on EC2
7. Verify CloudFront behavior
8. Invalidate cache
9. Final production test
```

---

# Useful Commands

## Restart Nginx

```bash
sudo systemctl restart nginx
```

## Check Nginx Status

```bash
sudo systemctl status nginx
```

## Validate Nginx Config

```bash
sudo nginx -t
```

## Reload Nginx

```bash
sudo systemctl reload nginx
```

## Docker Logs

```bash
docker logs <container-id>
```

## Running Containers

```bash
docker ps
```

---

# Key Learnings / Pitfalls

### 1. Vite Base Path Matters

Wrong base path causes missing JS/CSS files.

---

### 2. React Router Needs Basename

Without it, refreshing nested routes breaks.

---

### 3. Trailing Slash in `proxy_pass` Is Critical

It strips the subpath before forwarding.

---

### 4. CloudFront Caches Errors Too

Sometimes fixes do not appear until invalidation.

---

### 5. SPA Requires `index.html` Fallback

Without it, route refreshes fail.
