---
title: Why Next.js feels like Flask, rewritten in TypeScript
published: 2026-08-17
description: A year of writing Flask backends made Next.js feel familiar on day one - same layouts, same server-first rendering, same routing tricks, new syntax.
tags: [Next.js, Flask, React, WebDev]
category: WebDev
draft: false
---

I spent over a year writing backend code in Flask. When I started a new project, I decided to try Next.js. The first thing I noticed was déjà vu. This felt like Flask, rewritten in TypeScript with a newer take on rendering.

React itself works well for me - a tidy component library, nothing more. Next.js is what I fell for, the App Router most of all. The deeper I dig, the more it looks like a classic server-side framework, with extras Flask would never build on its own.

My Flask experience isn't from tutorials. I built [FileServer](https://github.com/whaiman/FileServer) on it (the project is moving to FastAPI now, since the old code stopped scaling). That work is what showed me how much these two tools overlap in concept.

## Templating and layouts

In Flask with Jinja, you build a `base.html` with a `{% block content %}` block, and `users.html` extends that template. Header, footer, navigation - written once, not copied on every page.

Next.js does the same job with `layout.tsx`. You set it at the folder level, and every page inside wraps in it, no extra code needed. App Router lets you nest layouts at any depth - `/admin` gets its own layout on top of the shared one. Flask can do this too, through several levels of `{% extends %}`, but by hand it feels less clean.

## Server rendering by default

My first guess: TSX is Jinja with a different syntax. Close, but not exact.

A component in Next.js defaults to a **Server Component**. It renders on the server, and the browser gets finished HTML, the same way it does after Jinja.

A component turns client-side only when you add `'use client'` at the top of the file. That's the real parallel with Flask. Flask never gives you the choice - everything runs on the server. Next.js gives you the choice, and you can mix server markup with interactive pieces in the browser.

## Data fetching

In Flask, you write the data-fetching logic right in the route, then pass it to the template:

```python
@app.route('/users')
def users():
    users = db.get_all_users()
    return render_template('users.html', users=users)
```

In Next.js, Server Components let you do the same thing inside the component itself, with no extra API endpoint to build:

```tsx
// app/users/page.tsx
export default async function UsersPage() {
  const users = await db.user.findMany();
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

It still feels like writing backend code.

## Dynamic routes

Flask: `@app.route('/user/<int:user_id>')`, and `user_id` arrives in the function as an argument.

Next.js uses the same idea with square brackets in the folder name: `app/user/[id]/page.tsx`, and you pull it out as `params.id`. Different syntax, same idea - a piece of the URL becomes a variable.

## Form handling

In Flask, you build a form, set `method="POST"`, and catch the data on the backend through `request.form`.

Next.js added **Server Actions**. You write an async function on the server and attach it to the `<form>` tag with no extra wiring:

```tsx
async function createUser(formData: FormData) {
  const name = formData.get('name');
  await db.user.create({ data: { name } });
}

<form action={createUser}>
  <input name="name" />
  <button type="submit">Create</button>
</form>
```

It's the same old POST request, packaged in newer syntax. You don't need a separate API route for simple mutations.

## API and HTML in one app

A single Flask app holds `/users`, which renders HTML, next to `/api/users`, which runs `jsonify(...)` and returns plain JSON. One server, two kinds of response.

Next.js works the same way: `app/users/page.tsx` returns a JSX page, and the neighboring `app/api/users/route.ts` returns JSON. This is also how I plan to build the FastAPI backend for FileServer - API on one side, frontend on the other, an idea I picked up from the Flask version of the project.

## Request lifecycle: middleware

Who sees a request before it reaches your code? In Flask, `@app.before_request` handles that - a function that runs before every route. Useful for checking auth or logging.

Next.js has `middleware.ts` at the project root for the same job. It catches the request before it reaches the page, and that's where you handle redirects or check a cookie. It matches `before_request` almost exactly.

## Grouping routes

Next.js has Route Groups - a folder in parentheses like `(auth)` that groups routes logically without showing up in the URL.

Flask solves the same problem with Blueprints: move the auth routes into one blueprint, the admin routes into another, then register both on the main app. The mechanics differ, but the motivation matches - don't dump every route into one flat pile.

## Caching and static files

In Flask, to keep from hitting the database on every request, you add a `@cache.cached(timeout=50)` decorator.

Next.js builds this into the core, under the name ISR (Incremental Static Regeneration). You export a config variable:

```tsx
export const revalidate = 60;
```

## Redirects and request context

In Flask, you use `redirect(url_for('login'))` to send a user elsewhere. For headers, you import the global `request` object.

In Next.js server code, you do the same thing through dedicated functions instead:

```tsx
import { redirect } from 'next/navigation';
import { headers } from 'next/headers';

if (!session) redirect('/login');

const token = headers().get('Authorization');
```

## The takeaway

People often file Next.js under "yet another SPA framework." Under the hood, App Router runs a solid server framework that does what Flask does, then lets you drop in client-side JavaScript exactly where you need it. If you know Flask well, Next.js won't take long to pick up. The concepts stay the same. Only the syntax changed.
