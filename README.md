# booklog

**Search any book on Google Books, rate it in half-star steps, and read what other people thought.**

```bash
# once the dev server is running (see Quick Start below)
curl "http://localhost:3000/api/books/search?q=dune"
# -> { "items": [ { "googleBooksId", "title", "authors", "coverImage" }, ... ] }
```

booklog is a small review site built with Next.js and MongoDB. The books themselves come from the Google Books API. Only users and their reviews are stored in your database, so there is no catalogue to maintain. The repository is named `biblioteca-digitala`; the app calls itself *booklog* in the UI.

## Features

| Feature | Description |
|---|---|
| Book search | Live search over Google Books (10 results per query) with debounced input and a client-side cache |
| User search | Find reviewers by username prefix from the same search box |
| Half-star ratings | Ratings from 1 to 5 in 0.5 steps, plus an optional written review |
| One review per book | A unique index on user + book rejects duplicates with a `409` |
| Edit and delete | Only the author of a review can change or remove it |
| Popular books | Home page grid ranked by review count, then title |
| Public profiles | `/users/<username>` lists a reader's reviews, best-rated first |
| Login by email or username | JWT sessions that last 7 days; passwords hashed with bcrypt |

## Quick Start

You need Node.js 20.9 or later (the minimum for Next.js 16) and a MongoDB instance. The commands below assume MongoDB is running locally on the default port. For Atlas or another host, swap in your own connection string.

```bash
git clone https://github.com/dragos-penciu/biblioteca-digitala.git
cd biblioteca-digitala
npm install

echo "MONGODB_URI=mongodb://127.0.0.1:27017/booklog" > .env.local
echo "JWT_SECRET=$(openssl rand -hex 32)" >> .env.local

npm run dev
```

Open [http://localhost:3000](http://localhost:3000), create an account at `/register`, then search for a book and review it.

## Usage

The UI covers everything below. The API is plain JSON over HTTP, which is handy for scripting or testing.

Create an account, log in, and post a review:

```bash
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"reader@example.com","username":"reader_one","password":"correct-horse-battery"}'

curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"identifier":"reader_one","password":"correct-horse-battery"}'
```

The login response contains a `token`. Send it as a bearer token on anything that writes:

```bash
curl -X POST http://localhost:3000/api/reviews \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"googleBooksId":"<id from the search results>","rating":4.5,"text":"Slow start, great finish."}'
```

Production build:

```bash
npm run build
npm start
```

## Configuration

Settings are read from environment variables. In development, put them in `.env.local` (already git-ignored).

| Variable | Required | Description |
|---|---|---|
| `MONGODB_URI` | Yes | MongoDB connection string |
| `JWT_SECRET` | Yes | Secret used to sign and verify login tokens. Use a long random value and keep it private |
| `GOOGLE_BOOKS_API_KEY` | No | Google Books API key. Without it requests are sent anonymously and hit shared quota limits sooner |

Validation rules enforced at registration:

| Field | Rule |
|---|---|
| `email` | Must look like an email; stored trimmed and lowercased |
| `username` | 3-20 characters: letters, digits, underscore; stored lowercased |
| `password` | At least 8 characters |

## API Reference

All routes live under `/api`. "Auth" means an `Authorization: Bearer <token>` header from `/api/auth/login`.

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/auth/register` | - | Body: `email`, `username`, `password`. Returns `201`, or `409` if the email or username is taken |
| `POST` | `/auth/login` | - | Body: `identifier` (email or username), `password`. Returns `{ token, username }` |
| `GET` | `/books/search?q=` | - | Up to 10 Google Books results with the best available cover |
| `GET` | `/books/:googleBooksId` | - | Book details, its reviews, `avgRating`, and `reviewCount` |
| `GET` | `/home?limit=` | - | Most-reviewed books with average rating. Default limit 12, maximum 24 |
| `GET` | `/reviews?googleBooksId=` | - | All reviews for a book, newest first |
| `POST` | `/reviews` | Yes | Body: `googleBooksId`, `rating`, optional `text`. `409` if you already reviewed the book |
| `PUT` | `/reviews/:id` | Owner | Body: `rating`, `text` (text is required on edit) |
| `DELETE` | `/reviews/:id` | Owner | Removes the review |
| `GET` | `/users/search?q=` | - | Username prefix match; needs 2+ characters, returns at most 5 |
| `GET` | `/users/:username/reviews` | - | A user's reviews, highest rating first |

Pages: `/` (search and popular books), `/books/:googleBooksId`, `/users/:username`, `/login`, `/register`.

## How It Fits Together

```
Browser ──> Next.js (App Router, route handlers under src/app/api)
                ├──> MongoDB   users, reviews (via Mongoose)
                └──> Google Books API   titles, authors, covers, descriptions
```

Reviews store only the Google Books volume ID. Book metadata is fetched on demand, and the book and profile pages revalidate it hourly.

```
src/
  app/          pages and API route handlers
  components/   search bar, book and review cards, review form
  lib/          database connection, JWT helpers, validators
  models/       Mongoose schemas: User, Review
```

## Troubleshooting

**The Popular grid says there are no popular books.** That is expected on a fresh database. Books appear once they have at least one review.

**Search returns a 502.** Google Books rejected or failed the request. The response includes the upstream status. Adding `GOOGLE_BOOKS_API_KEY` usually fixes quota errors.

**API calls fail right after startup with a connection error.** Check that `MONGODB_URI` points at a reachable database. The connection is opened with Mongoose's `bufferCommands` turned off, so queries fail immediately instead of waiting for a connection.

**Login works but you are logged out within a week.** Tokens expire after 7 days. Log in again.

## Contributing

There is no test suite yet. Before opening a pull request, run the linter and make sure a production build passes:

```bash
npm run lint
npm run build
```

## License

No license file is included in this repository yet. Until one is added, the default is all rights reserved.

## Author

[![dragos-penciu](https://github.com/dragos-penciu.png?size=64)](https://github.com/dragos-penciu)

[dragos-penciu](https://github.com/dragos-penciu)

Built with [Next.js](https://nextjs.org), [React](https://react.dev), [Mongoose](https://mongoosejs.com), [Tailwind CSS](https://tailwindcss.com), and the [Google Books API](https://developers.google.com/books).
