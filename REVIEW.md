# Code Review: Whitelable-show / Quick My Ticket

## 1. Architecture Overview
- **Type**: Frontend SPA application built with vanilla JavaScript, HTML, CSS.
- **Backend**: Supabase (PostgreSQL, Auth, Edge Functions, Storage).
- **Payment Processing**: Razorpay (integrated via Supabase Edge Functions).
- **Deployment**: Configured for Vercel. A build script (`deploy-theater.js` / `deploy-theater.sh`) injects environment configurations into the `config.js` file for "white-labeled" sub-deployments.

## 2. Frontend Review
### Code Quality
- **Architecture**: The application relies heavily on Vanilla JS without any modern framework like React, Vue, or Svelte. Logic is scattered across multiple global files (`app.js`, `admin.js`, `owner-dashboard.js`).
- **DOM Manipulation**: Frequent usage of `.innerHTML` for DOM manipulation (found ~70 occurrences). While some are hardcoded strings, others map over data arrays (e.g., in `public_owner.js` around line 1536). This approach makes the application highly susceptible to Cross-Site Scripting (XSS) if the data loaded from the database (e.g., user inputs like movie names or theatre names) is not properly sanitized before rendering.
- **State Management**: State seems to be managed via global variables or DOM state. There are also uses of `sessionStorage` for temporary states like `preview_theater_id` and guest sessions.
- **Maintainability**: Without a module bundler (like Webpack/Vite) or a framework, maintaining this codebase as it scales will be difficult.

### Security
- **XSS Vulnerabilities**: Using `.innerHTML` to insert dynamic data is dangerous.
  - **Recommendation**: Switch to `.textContent` or use DOM creation methods (`document.createElement()`) wherever possible. If HTML formatting is required, use a sanitizer library like `DOMPurify`.
- **CSP Headers**: The `vercel.json` file defines strong Content-Security-Policy (CSP) headers, which is excellent and mitigates some XSS risks, but it shouldn't replace proper DOM sanitization.

## 3. Backend (Supabase) Review
### Database & RLS (Row Level Security)
- The application makes extensive use of Row Level Security (RLS) policies to control data access.
- Migrations 026 and 056 indicate recent security hardening (fixing privilege escalations and improving session validation).
- RLS logic like `(auth.jwt() -> 'app_metadata' ->> 'is_admin')::boolean = true` is used properly for admin endpoints.

### Edge Functions
- **`create-order`**: Handles Razorpay order creation. Uses `booking_ids` for idempotency and verifies pricing against the database, which is good. It reads Razorpay keys from environment variables.
- **`verify-payment`**: Handles webhook verification. It securely verifies the Razorpay signature using `crypto.subtle`.
- **`tmdb-proxy`**: Proxies TMDB requests so the TMDB API key is not exposed to the frontend.

## 4. Deployment Pipeline
- The custom deployment script mechanism (`deploy-theater.js`) replaces `%%PLACEHOLDER%%` tokens with actual credentials in `config.js`. This is a somewhat brittle approach compared to using standard build tools and environment variable injection (e.g., Vite/Webpack env vars).
- Secrets aren't hardcoded in the codebase, which is a major positive.

## 5. Summary & Actionable Recommendations
1. **Migrate to a Framework**: The reliance on Vanilla JS and `innerHTML` is unsustainable for a complex booking platform. Consider a gradual migration to a framework (React, Vue) or at least a bundler (Vite) to allow module imports and better state management.
2. **Fix XSS Vectors**: Audit all uses of `.innerHTML` in `admin.js`, `owner-dashboard.js`, and `app.js`. Replace them with safer alternatives.
3. **Refactor Deployments**: Instead of manually parsing and injecting JS files, use standard frontend tooling (e.g. Vite environment variables) to handle different builds per white-label site.
4. **Testing**: There appears to be a lack of automated testing. Add unit and integration tests (e.g., Jest, Playwright) especially for the payment and booking flows.
