# Running the app

- Create a file `.env` in the root of the application directory, containing the database url, e.g.:

``` echo "DATABASE_URL=postgres://username:password@localhost/dbname" > .env ```

- Compile the application with `cargo build`. For a release version, use `cargo build --release`.
  Please note that this application requires rustc 1.43.0-nightly or newer.

# Using the app

There are two endpoints which the server provides for the creation and management of shortlinks.

## Create

Send a POST request to /create, with a JSON body consisting of the following attributes:
- target: The target URL to which the shortlink should point
- name (optional): The name of the shortlink (everything after the slash)

If no name is provided, one will be generated randomly. Names are limited to 128 characters.

## Stats

Statistics are available for each short link at /stats/{name}. This endpoint returns a JSON object
with the following attributes:
- name: The name of the shortlink
- created_on: The date and time (UTC) at which the shortlink was created
- total_visits: The total number of visits via the shortlink since creation
- visits_per_day: The total number of visits per day (UTC), as an object with keys as dates and
  values as integers.
- unique_visitors: The number of unique IP addresses which have accessed the shortlink since
  creation

There is also the third, trivial endpoint; sending a GET request to /{name} will redirect to the
associated target, if there is one.

# Design

There are two main libraries which form the core of this application: actix for the server layer,
and diesel for the database layer. This division of layers is more or less reflected in the file
structure: src/main.rs contains the server code using actix-web, and src/lib.rs contains the
database code.

Actix was chosen because it is known as the fastest Rust server framework currently in production;
Rocket is similarly easy to use, but is synchronous by design, whereas Actix uses the Tokio library
and framework to provide an asynchronous runtime. Diesel is simply the easiest Rust ORM to use,
providing a type-safe DSL instead of requiring the writing of raw SQL. It is not quite fully
featured (as you may see by the occasional raw SQL in the src/lib.rs), but it is close enough for
practical purposes.

Diesel is quite opinionated about its layout; the file src/schema.rs was automatically generated by
the Diesel CLI during development (although it is editable further, if need be), and the migrations/
directory is in a format which Diesel expects -- one directory per migration, containing files
up.sql and down.sql.

I attempted, when implementing each feature, to write it at the layer which would give the best
performance. You will see in the migrations file migrations/postgres/2020-03-18-162114_stats/up.sql,
for example, that I wrote triggers to log the creation times of each newly-created shortlink,
instead of doing so on the Rust level; I also wrote a function `get_stat` which aggregates the
visit statistics by date so I wouldn't have to copy the entire table into Rust. This does leave some
functionality implicit when looking at the Rust code, which is a downside.

One design decision I want to point out is the use of two tables for shortlinks: one for "canonical"
or auto-generated shortlinks, and one for "custom" shortlinks where the name is provided. The
requirements made clear that these two classes of shortlinks were to be treated differently, and as
I experimented with the logic involved, I felt that this was the most appropriate design (as opposed
to, for example, a single table with a boolean `is_custom` column). This does have one interesting
side effect: there are no foreign keys keeping the `stats` table in sync with the shortlinks, hence
the need for trigger functions.

I tried to be fairly strict about separation of concerns between the two layers of the application.
Database errors are non-recoverable, and do not propagate up to the server layer; they simply crash
the thread. I could do some work to turn those into 500's, and that might be a good future
extension, but those types of errors should be rare enough (now that development is done) that I
would consider it a low priority.

# Testing

I ran into several difficulties attempting to write an automated test suite, mostly centered around
the fact that Diesel makes it very difficult to write backend-agnostic code. The tests I wrote at
the bottom of src/lib.rs test the creation functionality; I tested the remainder of the
functionality from the command line using HTTPie.

I would have liked to have organized my tests a little better, but technical limitations made that
difficult.

# Extra credit

The number of unique visitors is included in the /stats endpoint.
The app is deployed to http://bitly.arandur.me
