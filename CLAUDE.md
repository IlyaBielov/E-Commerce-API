# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Context

Section **10-e-commerce-api** (Udemy "Section 14: E-Commerce API", 72 lessons, ~8h, lessons 273–344) of a Node.js/Express course. The biggest project of the course, built from an empty `app.js`: users with roles (first registered user = admin), JWT in a **signed httpOnly cookie**, products (admin CRUD + image upload), reviews (one per user per product, average rating via aggregation), orders (server-side totals, fake Stripe), docs via docgen, security packages, deploy.

The starter is the course's code (`app.js` is just a `console.log`; `db/`, `errors/`, `middleware/`, `mockData/` are given), prepared on 2026-11-17: dependencies bumped, `.env` created, `.gitignore` extended. **`README.MD` is a 376-line step-by-step checklist for the whole section** — the lesson order follows it; read the relevant heading before each lesson. No git repo yet — create one (like 03–09) when Ilya first asks to commit; push only when he says so.

A completed reference implementation lives at `../final/` — consult it to see where a lesson is heading. `../final-front-end/` has two optional clients (`front-end/` vanilla, `react-front-end/`) used late in the course to demo cookies. Do **not** copy-paste from `final/`; by default Ilya types the code himself — offer snippets inline or explain. (He often says "роби N" instead — then implement that lesson; see the memory `lesson-pacing`.)

## Commands

- `npm install`
- `npm run dev` — nodemon (local); `npm start` — plain node
- No lint/test setup. Port 5000 is shared with other sections' dev servers — run one at a time.

## Environment

`.env` (gitignored):
- `PORT=5000`
- `MONGO_URL` — Atlas `Cluster0`, DB `10-E-COMMERCE-API` (connection verified 2026-11-17). **This section uses `MONGO_URL`, not `MONGO_URI`** like earlier sections.
- `JWT_SECRET` (generated), `JWT_LIFETIME=1d` — `JWT_SECRET` also signs cookies: `cookieParser(process.env.JWT_SECRET)`.

## Target architecture (`../final/`)

- **app.js** — `trust proxy`, rateLimiter, helmet, cors, xss, mongoSanitize, `express.json()`, `cookieParser(JWT_SECRET)`, `express.static('./public')`, `fileUpload()`, five routers under `/api/v1/{auth,users,products,reviews,orders}`, notFound + errorHandler; `start()` awaits `connectDB(process.env.MONGO_URL)`. `morgan('tiny')` is used during development.
- **models/** — `User` (name, email unique + `validator.isEmail`, password hashed in `pre('save')` only if modified, role `admin|user`, `comparePassword`), `Product` (price in cents, enums for category/company, `user` ref, virtual `reviews`, cascade-delete reviews), `Review` (rating 1–5, compound unique index `{ product, user }`, static `calculateAverageRating` run from post hooks), `Order` (`orderItems` subdocuments, status enum, `clientSecret`, `paymentIntentId`).
- **utils/** — `jwt.js` (`createJWT`, `isTokenValid`, `attachCookiesToResponse` → cookie `token`: httpOnly, signed, `secure` in production, 1 day), `createTokenUser` (`{ name, userId, role }`), `checkPermissions(requestUser, resourceUserId)` (admin or owner, else 403).
- **middleware/authentication.js** — `authenticateUser` (reads `req.signedCookies.token`) and `authorizePermissions(...roles)`.
- **errors/** — adds `UnauthorizedError` (403) next to the given ones.
- **controllers/** — `authController` (register/login/logout), `userController` (getAllUsers, getSingleUser, showCurrentUser, updateUser, updateUserPassword), `productController` (CRUD + `uploadImage` to `public/uploads`), `reviewController` (CRUD + `getSingleProductReviews`), `orderController` (`createOrder` recalculates prices from the DB and calls `fakeStripeAPI`, getAllOrders, getSingleOrder, getCurrentUserOrders, updateOrder → `paid`).
- **routes/** — one file per resource; `/api/v1/products/:id/reviews`, `/api/v1/orders/showAllMyOrders`, `/api/v1/users/showMe`, `/api/v1/users/updateUser`, `/api/v1/users/updateUserPassword` (static paths must be declared before `/:id`).

## Version differences vs the course

Bumped: `mongoose` 9.12.1 (exact; course: 6), `jsonwebtoken` 9, `dotenv` 16, `express` 4.21, `nodemon` 3; scripts `start` = node, `dev` = nodemon; Node 22 (`engines`). The rest stays at the course's majors (`helmet` 4, `express-rate-limit` 5, `express-mongo-sanitize` 2, `express-fileupload` 1.x, `xss-clean`). `npm audit --omit=dev` → 0 vulnerabilities.

**Mongoose 9 — code from the videos that will NOT work as shown:**
- `document.remove()` is gone (removed in Mongoose 7). The course uses `await product.remove()` and `await review.remove()` → use `await doc.deleteOne()`.
- Because of that, the `'remove'` hooks never fire: `ProductSchema.pre('remove', ...)` (cascade-delete reviews) and `ReviewSchema.post('remove', ...)` (recalculate rating) must become `pre/post('deleteOne', { document: true, query: false }, ...)`. This is the one place where silently following the video gives a wrong result with no error.
- No `next` in hooks: write `pre('save', async function () { ... })` and just `return` (the final already does this in `User`).
- `{ new: true }` → `{ returnDocument: 'after' }` in `findOneAndUpdate` (keep `runValidators: true`).
- `new mongoose.Types.ObjectId(...)` needs `new`. For schema refs prefer `mongoose.Schema.Types.ObjectId` (the final mixes `mongoose.Schema.ObjectId` and `mongoose.Types.ObjectId`).
- README mentions "mongoose V6 info" — the old connect options are already gone; `db/connect.js` passes only the url.

Other gotchas to watch for in the lessons:
- The given `middleware/authentication.js` and `middleware/full-auth.js` are **not** the course's final code (Bearer header + `authorizeRoles`, `payload.user.userId`, and they require `../utils/jwt`, which doesn't exist yet). The lessons write `authenticateUser` / `authorizePermissions` from scratch — expect the file to be replaced.
- `errors/unauthorized.js` doesn't exist in the starter; it's added in the "Authorize Permissions" lessons and must be exported from `errors/index.js`.
- Cookies: `signed: true` requires `cookieParser(secret)` and reading from `req.signedCookies`, not `req.cookies`. `secure: true` only in production (needs HTTPS; on Render also `app.set('trust proxy', 1)`). In Postman cookies are stored automatically; with a separate front-end origin cors needs `credentials: true` + an explicit `origin`.
- `checkPermissions` compares `requestUser.userId` (string from the JWT) with `resourceUserId.toString()` (ObjectId) — forgetting `.toString()` makes every non-admin check fail.
- Route order: `/showMe`, `/updateUser`, `/updateUserPassword`, `/showAllMyOrders`, `/uploadImage` before `/:id`, otherwise they hit the `:id` handler and fail with a CastError → 404.
- `uploadImage` writes to `public/uploads/` — the folder must exist; the default product image is `/uploads/example.jpeg` (sample images are in `../final/public/uploads/`). Decide then whether uploads are gitignored (07 pattern: ignore contents, keep `.gitkeep`).
- `mockData/products.json` has no `user` field — it is for Postman bodies, not a populate script. `mockData/orders.json` contains hardcoded product ids that must be replaced with real ones.
- Prices, tax and shipping are in cents. `createOrder` takes `price` from the DB, not from the client — the fix for what 09-stripe-payment left open. Stripe here is a stub (`fakeStripeAPI`); a real `stripe.paymentIntents.create` from section 09 could replace it (outside the course).
- Docs lesson uses the `docgen` CLI binary (Postman collection → `public/index.html`); it's a separate download, not an npm package.
- README says "Deploy on Heroku" + `Procfile`; earlier sections were deployed to **Render** instead (env vars set in the dashboard, deploy on push to `main`). `express-rate-limit` is set to 60 requests / 15 min in the final — easy to hit while testing.

## Course workflow

- When Ilya asks for help implementing something, explain the concept and show the code inline — don't write it to his files unless he explicitly asks ("роби N").
- If he's stuck, compare his code against `../final/` mentally and point out specific differences rather than dumping the whole reference.
- After a lesson he runs `/lesson-explain`; he often asks to commit afterwards; push only when he says so.
