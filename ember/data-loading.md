---
title: Data Loading and Async Concurrency Patterns
impact: HIGH
impactDescription: Prevents infinite render loops and provides better control of async operations
tags: ember-concurrency, tasks, data-loading, concurrency-patterns, getPromiseState
---

## Data Loading and Async Concurrency Patterns

### Table of Contents

1. [Why Not ember-concurrency for Data Loading?](#why-not-ember-concurrency-for-data-loading)
2. [Initial Data Loading Patterns](#initial-data-loading-patterns)
   - [~~Constructor-based Loading~~](#constructor-based-loading)
   - [Reactive Cached Getter](#reactive-cached-getter)
   - [Reusable Request Component](#reusable-request-component)
3. [Shared Requests Across Multiple Components](#shared-requests-across-multiple-components)
   - [Service-based Request Caching](#service-based-request-caching)
   - [Parent Request with Data Distribution](#parent-request-with-data-distribution)
   - [Request Manager with TTL](#request-manager-with-ttl)
4. [Route-based Data Loading](#route-based-data-loading)
   - [Loading & Error Substates](#loading--error-substates-optional)
   - [Avoid Preloading Data That Isn't Required](#avoid-preloading-data-that-isnt-required)
   - [When to Use `routeDidChange`](#when-to-use-routedidchange)
5. [URL State Management with Query Params](#url-state-management-with-query-params)
6. [User Input Concurrency with ember-concurrency](#user-input-concurrency-with-ember-concurrency)
   - [Task Modifiers](#task-modifiers)
   - [TaskInstance API for Derived Data](#taskinstance-api-for-derived-data)
7. [Quick Reference](#quick-reference)

---

This guide covers patterns for loading and sharing data in Ember applications:

- **Initial data loading**: ~~Constructor~~, `@cached` getter, and Request component patterns
- **Shared requests**: Service caching, parent request distribution, and request managers with TTL
- **Route-based loading**: When to use route model hooks (first-level components only)
- **URL state management**: Query params driving service state for shareable, bookmarkable views
- **User-initiated concurrency**: ember-concurrency for debouncing, throttling, and preventing double-clicks

**Key principle**: Use `getPromiseState` for data loading; use ember-concurrency only for **user-initiated** actions.

### Why Not ember-concurrency for Data Loading?

**Incorrect (using ember-concurrency for data loading):**

```glimmer-ts
// app/components/user-profile.gts
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { task } from 'ember-concurrency';

interface UserProfileSignature {
  Args: {
    userId: string;
  };
}

class UserProfile extends Component<UserProfileSignature> {
  @tracked userData: { name: string } | null = null;
  @tracked error: Error | null = null;

  // WRONG: Setting tracked state inside task
  loadUserTask = task(async () => {
    try {
      const response = await fetch(`/api/users/${this.args.userId}`);
      this.userData = await response.json(); // Anti-pattern!
    } catch (e) {
      this.error = e as Error; // Anti-pattern!
    }
  });

  <template>
    {{#if this.loadUserTask.isRunning}}
      Loading...
    {{else if this.userData}}
      <h1>{{this.userData.name}}</h1>
    {{/if}}
  </template>
}
```

**Why This Is Wrong:**

- Setting tracked state during render can cause infinite render loops
- ember-concurrency adds overhead unnecessary for simple data loading
- Makes component state harder to reason about
- Can trigger multiple re-renders

**Correct (use getPromiseState):**

```glimmer-ts
// app/components/user-profile.gts
import Component from '@glimmer/component';
import { cached } from '@glimmer/tracking';
import { getPromiseState } from 'reactiveweb';

interface UserProfileSignature {
  Args: {
    userId: string;
  };
}

class UserProfile extends Component<UserProfileSignature> {
  @cached
  get userData() {
    const promise = fetch(`/api/users/${this.args.userId}`)
      .then(r => r.json());
    return getPromiseState(promise);
  }

  <template>
    {{#if this.userData.isLoading}}
      <div>Loading...</div>
    {{else if this.userData.error}}
      <div>Error: {{this.userData.error.reason}}</div>
    {{else if this.userData.resolved}}
      <h1>{{this.userData.resolved.name}}</h1>
    {{/if}}
  </template>
}
```

### Initial Data Loading Patterns

For data that should load immediately when a component renders (not triggered by user action), choose based on your needs:

| Pattern           | Use When                                       |
| ----------------- | ---------------------------------------------- |
| ~~Constructor~~   | ~~One-time fetch, no reactive dependencies~~   |
| `@cached` getter  | Fetch depends on reactive args that may change |
| Request component | Declarative loading, consistent UI patterns    |

#### ~~Constructor-based Loading~~

> **Not recommended.** Constructors should not be asynchronous because they are meant to initialize an instance of a class synchronously. While you can technically run a promise in a constructor, the instance will be returned before the asynchronous operation completes, which can lead to potential issues if the rest of your code expects the object to be fully initialized. While returning the promise may seem the solution for this problem, it is not standard practice and can lead to unexpected behavior. Returning a promise (or, in general, any object from another class) from a constructor can be misleading and is considered a bad practice as this leads to unexpected results with inheritance and the `instanceof` operator.
>
> **Use a `@cached` getter or the Request component instead.**

~~Use the constructor when you need to fetch data once when the component is instantiated.~~

~~**Important:** The constructor runs only once when the component is first created. It will **never** re-run when arguments change. If your fetch depends on reactive args that may change over time, use a `@cached` getter instead.~~

```glimmer-ts
// DON'T DO THIS — constructor-based async loading is an anti-pattern
// app/components/dashboard.gts
import Component from '@glimmer/component';
import { getPromiseState } from 'reactiveweb';

interface DashboardStats {
  totalUsers: number;
  activeSessions: number;
}

class Dashboard extends Component {
  #request: Promise<DashboardStats>;

  constructor(owner: unknown, args: object) {
    super(owner, args);
    this.#request = fetch('/api/dashboard/stats').then(r => r.json());
  }

  get stats() {
    return getPromiseState(this.#request);
  }

  <template>
    {{#if this.stats.isLoading}}
      <div class="skeleton-loader">Loading dashboard...</div>
    {{else if this.stats.error}}
      <div class="error">Failed to load: {{this.stats.error.reason}}</div>
    {{else if this.stats.resolved}}
      <div class="dashboard">
        <p>Total users: {{this.stats.resolved.totalUsers}}</p>
        <p>Active sessions: {{this.stats.resolved.activeSessions}}</p>
      </div>
    {{/if}}
  </template>
}
```

#### Reactive Cached Getter

Use `@cached` when the fetch depends on reactive arguments - automatically re-fetches when dependencies change. See the user-profile example above.

#### Reusable Request Component

Create a reusable `Request` component for declarative, composable data loading with named blocks:

```glimmer-ts
// app/components/request.gts
import Component from '@glimmer/component';
import { cached } from '@glimmer/tracking';
import { getPromiseState } from 'reactiveweb';
import type { Error as PromiseError } from 'reactiveweb/get-promise-state';

interface RequestSignature<T> {
  Args: {
    fetch: () => Promise<T>;
  };
  Blocks: {
    loading: [];
    error: [error: PromiseError];
    default: [data: T];
  };
}

export default class Request<T> extends Component<RequestSignature<T>> {
  @cached
  get state() {
    const promise = this.args.fetch();
    return getPromiseState(promise);
  }

  <template>
    {{#if this.state.isLoading}}
      {{#if (has-block "loading")}}
        {{yield to="loading"}}
      {{else}}
        <div>Loading...</div>
      {{/if}}
    {{else if this.state.error}}
      {{#if (has-block "error")}}
        {{yield this.state.error to="error"}}
      {{else}}
        <div>Error: {{this.state.error.reason}}</div>
      {{/if}}
    {{else if this.state.resolved}}
      {{yield this.state.resolved}}
    {{/if}}
  </template>
}
```

**Usage with named blocks:**

```glimmer-ts
// app/components/order-list.gts
import Request from './request';

interface Order {
  id: string;
  status: string;
}

const fetchOrders = (): Promise<Order[]> =>
  fetch('/api/orders').then(r => r.json());

<template>
  <Request @fetch={{fetchOrders}}>
    <:loading>
      <div class="skeleton">
        <div class="skeleton-row"></div>
        <div class="skeleton-row"></div>
      </div>
    </:loading>

    <:error as |error|>
      <div class="error-banner">
        <p>Failed to load orders: {{error.reason}}</p>
        <button type="button">Retry</button>
      </div>
    </:error>

    <:default as |orders|>
      <ul class="order-list">
        {{#each orders as |order|}}
          <li>{{order.id}} - {{order.status}}</li>
        {{/each}}
      </ul>
    </:default>
  </Request>
</template>
```

### Shared Requests Across Multiple Components

When multiple components need the same data simultaneously, avoid duplicate network requests by sharing the promise.

| Pattern         | Use When                                         |
| --------------- | ------------------------------------------------ |
| Service caching | Multiple unrelated components need same data     |
| Parent Request  | Components are in the same subtree               |
| Request manager | Many shared requests with TTL/invalidation needs |

#### Service-based Request Caching

```ts
// app/services/data-cache.ts
import Service from "@ember/service";
import { getPromiseState } from "reactiveweb";

export default class DataCacheService extends Service {
  #requests = new Map<string, Promise<unknown>>();

  fetch<T>(key: string, fetchFn: () => Promise<T>): Promise<T> {
    if (!this.#requests.has(key)) {
      const promise = fetchFn();
      this.#requests.set(key, promise);
    }
    return this.#requests.get(key) as Promise<T>;
  }

  getState<T>(key: string, fetchFn: () => Promise<T>) {
    return getPromiseState(this.fetch(key, fetchFn));
  }

  invalidate(key: string) {
    this.#requests.delete(key);
  }
}
```

**Components sharing the same request:**

```glimmer-ts
// app/components/user-header.gts
import Component from '@glimmer/component';
import { service } from '@ember/service';
import { cached } from '@glimmer/tracking';
import type DataCacheService from '../services/data-cache';

interface User {
  name: string;
  avatar: string;
}

interface UserHeaderSignature {
  Args: {
    userId: string;
  };
}

class UserHeader extends Component<UserHeaderSignature> {
  @service declare dataCache: DataCacheService;

  @cached
  get user() {
    return this.dataCache.getState<User>(
      `user:${this.args.userId}`,
      () => fetch(`/api/users/${this.args.userId}`).then(r => r.json())
    );
  }

  <template>
    {{#if this.user.resolved}}
      <header>
        <img src={{this.user.resolved.avatar}} alt={{this.user.resolved.name}} />
        <h1>{{this.user.resolved.name}}</h1>
      </header>
    {{/if}}
  </template>
}
```

```glimmer-ts
// app/components/user-sidebar.gts - Same pattern, shares the request!
import Component from '@glimmer/component';
import { service } from '@ember/service';
import { cached } from '@glimmer/tracking';
import type DataCacheService from '../services/data-cache';

interface User {
  email: string;
  role: string;
}

interface UserSidebarSignature {
  Args: {
    userId: string;
  };
}

class UserSidebar extends Component<UserSidebarSignature> {
  @service declare dataCache: DataCacheService;

  @cached
  get user() {
    // Same key = same promise = no duplicate request!
    return this.dataCache.getState<User>(
      `user:${this.args.userId}`,
      () => fetch(`/api/users/${this.args.userId}`).then(r => r.json())
    );
  }

  <template>
    {{#if this.user.resolved}}
      <aside>
        <p>Email: {{this.user.resolved.email}}</p>
        <p>Role: {{this.user.resolved.role}}</p>
      </aside>
    {{/if}}
  </template>
}
```

#### Parent Request with Data Distribution

Lift the Request component to a common parent and pass data down:

```glimmer-ts
// app/components/user-page.gts
import Request from './request';
import UserHeader from './user-header';
import UserSidebar from './user-sidebar';

interface User {
  id: string;
  name: string;
  avatar: string;
  email: string;
  role: string;
}

const fetchUser = (userId: string) => (): Promise<User> =>
  fetch(`/api/users/${userId}`).then(r => r.json());

<template>
  <Request @fetch={{(fetchUser @userId)}} as |user|>
    <div class="user-page">
      {{! All children receive the same data - single request }}
      <UserHeader @user={{user}} />
      <UserSidebar @user={{user}} />
    </div>
  </Request>
</template>
```

```glimmer-ts
// app/components/user-header.gts - Receives data as arg, no fetching
import type { TemplateOnlyComponent } from '@ember/component/template-only';

interface User {
  name: string;
  avatar: string;
}

interface UserHeaderSignature {
  Args: {
    user: User;
  };
}

const UserHeader: TemplateOnlyComponent<UserHeaderSignature> = <template>
  <header>
    <img src={{@user.avatar}} alt={{@user.name}} />
    <h1>{{@user.name}}</h1>
  </header>
</template>;

export default UserHeader;
```

#### Request Manager with TTL

For applications with many shared requests needing cache expiration:

```ts
// app/services/request-manager.ts
import Service from "@ember/service";
import { getPromiseState } from "reactiveweb";

interface CacheEntry {
  promise: Promise<unknown>;
  timestamp: number;
}

interface RequestOptions {
  ttl?: number;
}

export default class RequestManagerService extends Service {
  #cache = new Map<string, CacheEntry>();
  #defaultTTL = 5 * 60 * 1000; // 5 minutes

  request<T>(
    key: string,
    fetchFn: () => Promise<T>,
    options: RequestOptions = {},
  ): Promise<T> {
    const ttl = options.ttl ?? this.#defaultTTL;
    const cached = this.#cache.get(key);

    if (cached && Date.now() - cached.timestamp < ttl) {
      return cached.promise as Promise<T>;
    }

    const promise = fetchFn();
    this.#cache.set(key, { promise, timestamp: Date.now() });
    promise.catch(() => this.#cache.delete(key)); // Allow retry on failure

    return promise;
  }

  getState<T>(
    key: string,
    fetchFn: () => Promise<T>,
    options?: RequestOptions,
  ) {
    return getPromiseState(this.request(key, fetchFn, options));
  }

  invalidate(key: string) {
    this.#cache.delete(key);
  }

  invalidatePattern(pattern: string) {
    for (const key of this.#cache.keys()) {
      if (key.includes(pattern)) {
        this.#cache.delete(key);
      }
    }
  }
}
```

**Important: Avoid body stream consumption errors**

A `Response` body can only be consumed once (via `.json()`, `.text()`, etc.). If you cache the raw `fetch()` promise and multiple consumers each call `.json()` on the resolved `Response`, the second consumer will throw `"Response: body stream already read"`. Use `Response.clone()` to ensure each consumer gets its own body stream:

```js
// WRONG - causes "body stream already consumed" errors
getUser(id) {
  if (!this.#cache.has(id)) {
    this.#cache.set(id, fetch(`/api/users/${id}`)); // Response object
  }
  return this.#cache.get(id).then(r => r.json()); // Second call fails!
}

// CORRECT - clone the Response before consuming the body
// Each consumer gets a clone, so the original Response stays unconsumed
getUser(id) {
  if (!this.#cache.has(id)) {
    this.#cache.set(id, fetch(`/api/users/${id}`));
  }
  return this.#cache.get(id).then(r => r.clone().json());
}

// ALSO CORRECT - cache the fully resolved data promise
// Sidesteps the issue entirely since all consumers share the parsed result
getUser(id) {
  if (!this.#cache.has(id)) {
    const promise = fetch(`/api/users/${id}`).then(r => r.json());
    this.#cache.set(id, promise);
  }
  return this.#cache.get(id); // All consumers get same resolved value
}
```

### Route-based Data Loading

Load data in the route's `model` hook and pass it to first-level components. This pattern works well when consumers are direct children of the route template.

**Warnings:**

- Only use for first-level components. For deeply nested components, use a service instead to avoid prop drilling.
- Do not preload data that isn't immediately required. Routes block rendering until `model` resolves.

```ts
// app/routes/user.ts
import Route from "@ember/routing/route";
import { action } from "@ember/object";
import { service } from "@ember/service";
import type RouterService from "@ember/routing/router-service";

interface UserRouteParams {
  user_id: string;
}

export default class UserRoute extends Route {
  @service declare router: RouterService;

  async model({ user_id }: UserRouteParams) {
    const response = await fetch(`/api/users/${user_id}`);
    if (!response.ok)
      throw new Error(`Failed to load user (${response.status})`);
    return response.json();
  }

  // Optional: intercept errors to handle specific cases (e.g., redirect on 403)
  // Return true to bubble up and render the error substate template
  @action
  error(error: Error) {
    if (error.message.includes("403")) {
      this.router.replaceWith("login");
    } else {
      return true;
    }
  }
}
```

```glimmer-ts
// app/templates/user.gts
import UserHeader from '../components/user-header';
import UserProfile from '../components/user-profile';

<template>
  <div class="user-page">
    {{! First-level components - OK to pass model data }}
    <UserHeader @user={{@model}} />
    <UserProfile @user={{@model}} />
  </div>
</template>
```

```glimmer-ts
// app/components/user-header.gts - First-level, receives data directly
import type { TemplateOnlyComponent } from '@ember/component/template-only';

interface UserHeaderSignature {
  Args: {
    user: { name: string };
  };
}

const UserHeader: TemplateOnlyComponent<UserHeaderSignature> = <template>
  <header>
    <h1>{{@user.name}}</h1>
  </header>
</template>;

export default UserHeader;
```

#### Loading & Error Substates (optional)

Since route `model` hooks block rendering until resolved, users see nothing while data loads. Ember provides **loading and error substates** to give feedback during transitions. These are not added to `router.js` — they exist implicitly as part of each route.

**Loading substate:** Ember looks for a loading template while a route's `model` hook is pending. For a route named `user`, it searches in order:

1. `user-loading` (sibling template)
2. `loading` or `application-loading` (application-level fallback)

For nested routes like `user.posts`, Ember searches upward through the hierarchy: `user.posts-loading` → `user.loading` / `user-loading` → `loading` / `application-loading`.

```glimmer-ts
// app/templates/user-loading.gts
// Shown automatically while the user route's model hook is pending
<template>
  <div class="loading-spinner">
    <p>Loading user...</p>
  </div>
</template>
```

```glimmer-ts
// app/templates/loading.gts
// Application-wide fallback for any route without a specific loading template
<template>
  <div class="loading-spinner">
    <p>Loading...</p>
  </div>
</template>
```

**Error substate:** When a route's `model` hook rejects, Ember transitions to the error substate. The rejected error becomes the substate's model. The search hierarchy mirrors loading: `user-error` → `error` / `application-error`.

```glimmer-ts
// app/templates/user-error.gts
// Shown automatically when the user route's model hook rejects
<template>
  <div class="error-page">
    <h2>Failed to load user</h2>
    <p>{{@model.message}}</p>
  </div>
</template>
```

> **Note:** Loading and error substates only apply to route `model` hooks. They do not affect component-level data loading with `getPromiseState`. See the route example above for programmatic error handling with the `error` action.

**When route loading causes prop drilling (avoid this):**

```glimmer-ts
// BAD - Prop drilling through multiple levels
<template>
  <UserPage @user={{@model}}>
    {{! UserPage passes to UserSection, which passes to UserCard, etc. }}
  </UserPage>
</template>
```

For deeply nested consumers, use a service instead (see Service-based Request Caching above).

#### Avoid Preloading Data That Isn't Required

Routes block rendering until the `model` hook resolves. Do not load data for components that may not be displayed (tabs, modals, conditional sections).

**Incorrect (preloading all tab data in route):**

```ts
// app/routes/user.ts
// BAD - Loads ALL data even if user only views one tab
export default class UserRoute extends Route {
  async model({ user_id }: { user_id: string }) {
    const [user, posts, followers, settings] = await Promise.all([
      fetch(`/api/users/${user_id}`).then((r) => r.json()),
      fetch(`/api/users/${user_id}/posts`).then((r) => r.json()),
      fetch(`/api/users/${user_id}/followers`).then((r) => r.json()),
      fetch(`/api/users/${user_id}/settings`).then((r) => r.json()),
    ]);
    return { user, posts, followers, settings };
  }
}
```

**Solution 1: Component loads its own data**

Let each tab component load data when it becomes visible:

```glimmer-ts
// app/components/user-posts-tab.gts
import Component from '@glimmer/component';
import { cached } from '@glimmer/tracking';
import { getPromiseState } from 'reactiveweb';

interface Post {
  title: string;
}

interface UserPostsTabSignature {
  Args: {
    userId: string;
  };
}

class UserPostsTab extends Component<UserPostsTabSignature> {
  @cached
  get posts() {
    // Only fetches when this component renders
    const promise: Promise<Post[]> = fetch(`/api/users/${this.args.userId}/posts`)
      .then(r => r.json());
    return getPromiseState(promise);
  }

  <template>
    {{#if this.posts.isLoading}}
      <div>Loading posts...</div>
    {{else if this.posts.resolved}}
      {{#each this.posts.resolved as |post|}}
        <article>{{post.title}}</article>
      {{/each}}
    {{/if}}
  </template>
}
```

```glimmer-ts
// app/templates/user.gts
import UserPostsTab from '../components/user-posts-tab';
import UserFollowersTab from '../components/user-followers-tab';

<template>
  <div class="user-page">
    <h1>{{@model.name}}</h1>

    <div class="tabs">
      {{#if (eq @controller.activeTab "posts")}}
        {{! Data loads only when tab is active }}
        <UserPostsTab @userId={{@model.id}} />
      {{else if (eq @controller.activeTab "followers")}}
        <UserFollowersTab @userId={{@model.id}} />
      {{/if}}
    </div>
  </div>
</template>
```

**Solution 2: URL params trigger conditional route loading**

Use query params to load only the data needed for the current view:

```ts
// app/routes/user.ts
import Route from "@ember/routing/route";

interface UserRouteParams {
  user_id: string;
  tab?: string;
}

export default class UserRoute extends Route {
  queryParams = {
    tab: { refreshModel: true },
  };

  async model({ user_id, tab }: UserRouteParams) {
    // Always load base user data
    const user = await fetch(`/api/users/${user_id}`).then((r) => r.json());

    // Conditionally load tab data based on URL
    let tabData = null;
    if (tab === "posts") {
      tabData = await fetch(`/api/users/${user_id}/posts`).then((r) =>
        r.json(),
      );
    } else if (tab === "followers") {
      tabData = await fetch(`/api/users/${user_id}/followers`).then((r) =>
        r.json(),
      );
    }

    return { user, tabData };
  }
}
```

```glimmer-ts
// app/templates/user.gts
<template>
  <div class="user-page">
    <h1>{{@model.user.name}}</h1>

    <nav class="tab-nav">
      <LinkTo @route="user" @query={{hash tab="posts"}}>Posts</LinkTo>
      <LinkTo @route="user" @query={{hash tab="followers"}}>Followers</LinkTo>
    </nav>

    {{! tabData contains only what's needed for current tab }}
    {{#if @model.tabData}}
      {{#each @model.tabData as |item|}}
        <div>{{item.name}}</div>
      {{/each}}
    {{/if}}
  </div>
</template>
```

**When to use each approach:**

| Approach             | Use When                                             |
| -------------------- | ---------------------------------------------------- |
| Component loads data | Tab/section may never be viewed, no URL state needed |
| URL params + route   | View state should be shareable/bookmarkable          |

#### When to Use `routeDidChange`

The `routeDidChange` event fires after every completed transition, regardless of how it was triggered. Unlike route lifecycle hooks (`model`, `activate`, `setupController`), which only exist inside route classes, `routeDidChange` is available anywhere you have access to the router service.

Use it for these scenarios:

**1. Cross-cutting concerns that need to know about every navigation**

Things that live outside any single route — analytics, breadcrumbs, document title updates, progress indicators. These need to fire on every transition regardless of route hierarchy.

```ts
// app/services/analytics.ts
import Service from "@ember/service";
import { service } from "@ember/service";
import type RouterService from "@ember/routing/router-service";

export default class AnalyticsService extends Service {
  @service declare router: RouterService;

  constructor(properties?: object) {
    super(properties);
    this.router.on("routeDidChange", () => {
      this.track(this.router.currentURL);
    });
  }

  track(url: string) {
    // Send page view to analytics provider
  }
}
```

**2. Reacting to navigation from a service or component (not a route)**

Route lifecycle hooks (`model`, `activate`, `setupController`) are only available inside route classes. If you need to react to navigation from a service or component, `routeDidChange` is your only clean option.

```ts
// app/services/breadcrumbs.ts
import Service from "@ember/service";
import { service } from "@ember/service";
import { tracked } from "@glimmer/tracking";
import type RouterService from "@ember/routing/router-service";
import type Transition from "@ember/routing/transition";

interface Breadcrumb {
  label: string;
  route: string;
}

export default class BreadcrumbsService extends Service {
  @service declare router: RouterService;

  @tracked items: Breadcrumb[] = [];

  constructor(properties?: object) {
    super(properties);
    this.router.on("routeDidChange", (transition: Transition) => {
      this.items = transition.to?.find(() => true)
        ? this.buildBreadcrumbs(transition)
        : [];
    });
  }

  private buildBreadcrumbs(transition: Transition): Breadcrumb[] {
    // Build breadcrumb trail from transition.to route info chain
    return [];
  }
}
```

**3. Detecting transitions where the model hook was skipped**

When navigating with `transitionTo` and passing an object context instead of an ID, the `model` hook is skipped entirely. `routeDidChange` still fires, making it the only consistent place to intercept "we navigated to this route" regardless of how the transition happened.

**4. Reacting to query param changes without `refreshModel`**

If a query param changes without `refreshModel: true`, no route hooks fire — but `routeDidChange` still fires. Use this when you need a side effect (not a model reload) on query param change.

```ts
// app/services/search-tracking.ts
import Service from "@ember/service";
import { service } from "@ember/service";
import type RouterService from "@ember/routing/router-service";

export default class SearchTrackingService extends Service {
  @service declare router: RouterService;

  constructor(properties?: object) {
    super(properties);
    this.router.on("routeDidChange", () => {
      const url = new URL(this.router.currentURL, window.location.origin);
      const query = url.searchParams.get("q");

      if (query) {
        // Log search term — no model reload needed, just a side effect
        console.debug("Search query changed:", query);
      }
    });
  }
}
```

**When NOT to use `routeDidChange`:**

- For data loading — use `getPromiseState` in components or the route `model` hook instead
- As a substitute for `refreshModel: true` — if you need the model to reload on QP changes, use `refreshModel`
- For route-specific setup — use route lifecycle hooks (`model`, `setupController`, `activate`) directly

### URL State Management with Query Params

For shareable, bookmarkable state (filters, pagination, search), use query params in the route to trigger service requests. Components then consume reactive data from the service.

This pattern enables:

- Shareable URLs with current state
- Browser back/forward navigation
- Bookmarkable filtered views
- Service-based reactivity for deeply nested components

```ts
// app/services/products.ts
import Service from "@ember/service";
import { tracked } from "@glimmer/tracking";
import { cached } from "@glimmer/tracking";
import { getPromiseState } from "reactiveweb";

interface Product {
  name: string;
  price: number;
}

interface ProductsResponse {
  items: Product[];
  totalPages: number;
}

interface ProductFilters {
  category?: string | null;
  sortBy?: string;
  page?: number;
}

export default class ProductsService extends Service {
  @tracked category: string | null = null;
  @tracked sortBy = "name";
  @tracked page = 1;

  #promise: Promise<ProductsResponse> | null = null;
  #lastParams: string | null = null;

  get currentParams(): string {
    return `${this.category}-${this.sortBy}-${this.page}`;
  }

  updateFilters({ category, sortBy, page }: ProductFilters) {
    this.category = category ?? this.category;
    this.sortBy = sortBy ?? this.sortBy;
    this.page = page ?? this.page;
    this.#promise = null; // Invalidate cache
  }

  @cached
  get products() {
    // Invalidate if params changed
    if (this.#lastParams !== this.currentParams) {
      this.#promise = null;
      this.#lastParams = this.currentParams;
    }

    if (!this.#promise) {
      const params = new URLSearchParams({
        category: this.category || "",
        sort: this.sortBy,
        page: String(this.page),
      });
      this.#promise = fetch(`/api/products?${params}`).then((r) => r.json());
    }

    return getPromiseState(this.#promise);
  }
}
```

```ts
// app/routes/products.ts
import Route from "@ember/routing/route";
import { service } from "@ember/service";
import type ProductsService from "../services/products";

interface ProductsRouteParams {
  category?: string;
  sort?: string;
  page?: number;
}

export default class ProductsRoute extends Route {
  @service declare products: ProductsService;

  queryParams = {
    category: { refreshModel: true },
    sort: { refreshModel: true },
    page: { refreshModel: true },
  };

  model({ category, sort, page }: ProductsRouteParams) {
    // Route triggers service update from URL params
    this.products.updateFilters({
      category,
      sortBy: sort || "name",
      page: page || 1,
    });

    // No need to return anything - components read from service
  }
}
```

```ts
// app/controllers/products.ts
import Controller from "@ember/controller";
import { service } from "@ember/service";
import { action } from "@ember/object";
import type ProductsService from "../services/products";

export default class ProductsController extends Controller {
  @service declare products: ProductsService;

  queryParams = ["category", "sort", "page"];

  // Default values
  category: string | null = null;
  sort = "name";
  page = 1;

  @action
  updateCategory(category: string) {
    this.category = category; // Updates URL, triggers route model hook
  }

  @action
  updateSort(sort: string) {
    this.sort = sort;
  }

  @action
  goToPage(page: number) {
    this.page = page;
  }
}
```

```glimmer-ts
// app/components/product-filters.gts
// Can be deeply nested - reads from service, not props
import Component from '@glimmer/component';
import { service } from '@ember/service';
import type ProductsService from '../services/products';

interface ProductFiltersSignature {
  Args: {
    onCategoryChange: (category: string) => void;
    onSortChange: (sort: string) => void;
  };
}

class ProductFilters extends Component<ProductFiltersSignature> {
  @service declare products: ProductsService;

  <template>
    <div class="filters">
      <label>
        Category:
        <select {{on "change" (fn @onCategoryChange (pick "target.value"))}}>
          <option value="">All</option>
          <option value="electronics" selected={{eq this.products.category "electronics"}}>
            Electronics
          </option>
          <option value="clothing" selected={{eq this.products.category "clothing"}}>
            Clothing
          </option>
        </select>
      </label>

      <label>
        Sort by:
        <select {{on "change" (fn @onSortChange (pick "target.value"))}}>
          <option value="name" selected={{eq this.products.sortBy "name"}}>Name</option>
          <option value="price" selected={{eq this.products.sortBy "price"}}>Price</option>
        </select>
      </label>
    </div>
  </template>
}
```

```glimmer-ts
// app/components/product-list.gts
// Deeply nested - consumes reactive data from service
import Component from '@glimmer/component';
import { service } from '@ember/service';
import type ProductsService from '../services/products';

interface ProductListSignature {
  Args: {
    onPageChange: (page: number) => void;
  };
}

class ProductList extends Component<ProductListSignature> {
  @service declare products: ProductsService;

  <template>
    {{#if this.products.products.isLoading}}
      <div class="loading">Loading products...</div>
    {{else if this.products.products.error}}
      <div class="error">Failed to load: {{this.products.products.error.reason}}</div>
    {{else if this.products.products.resolved}}
      <ul class="product-grid">
        {{#each this.products.products.resolved.items as |product|}}
          <li class="product-card">
            <h3>{{product.name}}</h3>
            <p>{{product.price}}</p>
          </li>
        {{/each}}
      </ul>

      <Pagination
        @currentPage={{this.products.page}}
        @totalPages={{this.products.products.resolved.totalPages}}
        @onPageChange={{@onPageChange}}
      />
    {{/if}}
  </template>
}
```

```glimmer-ts
// app/templates/products.gts
import ProductFilters from '../components/product-filters';
import ProductList from '../components/product-list';

<template>
  <div class="products-page">
    <h1>Products</h1>

    {{! Components can be deeply nested - they read from service }}
    <ProductFilters
      @onCategoryChange={{@controller.updateCategory}}
      @onSortChange={{@controller.updateSort}}
    />

    <ProductList @onPageChange={{@controller.goToPage}} />
  </div>
</template>
```

**Benefits of this pattern:**

- URL reflects current state: `/products?category=electronics&sort=price&page=2`
- Users can bookmark/share filtered views
- Browser back/forward works correctly
- No prop drilling - deeply nested components read from service
- Single source of truth for filters and data

### User Input Concurrency with ember-concurrency

Use ember-concurrency for managing **user-initiated** async operations. It provides automatic cancelation, debouncing, and prevents race conditions.

**Incorrect (manual async handling):**

```glimmer-ts
// app/components/search.gts
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { action } from '@ember/object';

interface SearchResult {
  name: string;
}

class Search extends Component {
  @tracked results: SearchResult[] = [];
  @tracked isSearching = false;
  currentRequest: AbortController | null = null;

  @action
  async search(event: Event) {
    const query = (event.target as HTMLInputElement).value;

    // Manual cancelation - error-prone
    if (this.currentRequest) {
      this.currentRequest.abort();
    }

    this.isSearching = true;
    const controller = new AbortController();
    this.currentRequest = controller;

    try {
      const response = await fetch(`/api/search?q=${query}`, {
        signal: controller.signal
      });
      this.results = await response.json();
    } catch (e) {
      if ((e as Error).name !== 'AbortError') {
        console.error(e);
      }
    } finally {
      this.isSearching = false;
    }
  }

  <template>
    <input {{on "input" this.search}} />
    {{#if this.isSearching}}Loading...{{/if}}
  </template>
}
```

**Correct (using ember-concurrency):**

```glimmer-ts
// app/components/search.gts
import Component from '@glimmer/component';
import { restartableTask, timeout } from 'ember-concurrency';
import { on } from '@ember/modifier';

interface SearchResult {
  name: string;
}

class Search extends Component {
  searchTask = restartableTask(async (event: Event) => {
    const query = (event.target as HTMLInputElement).value;
    await timeout(300); // Debounce
    const response = await fetch(`/api/search?q=${query}`);
    return response.json() as Promise<SearchResult[]>; // Return, don't set @tracked
  });

  <template>
    <input type="search" {{on "input" this.searchTask.perform}} />

    {{#if this.searchTask.isRunning}}
      <div>Searching...</div>
    {{/if}}

    {{#if this.searchTask.lastSuccessful}}
      <ul>
        {{#each this.searchTask.lastSuccessful.value as |result|}}
          <li>{{result.name}}</li>
        {{/each}}
      </ul>
    {{/if}}

    {{#if this.searchTask.last.isError}}
      <div>Error: {{this.searchTask.last.error.message}}</div>
    {{/if}}
  </template>
}
```

### Task Modifiers

Choose the right modifier for your concurrency pattern:

```glimmer-ts
import Component from '@glimmer/component';
import { dropTask, enqueueTask, restartableTask } from 'ember-concurrency';

interface FormActionsSignature {
  Args: {
    data: Record<string, unknown>;
  };
}

class FormActions extends Component<FormActionsSignature> {
  // restartableTask: Cancels previous, starts new (search/autocomplete)
  searchTask = restartableTask(async (query: string) => {
    const response = await fetch(`/api/search?q=${query}`);
    return response.json();
  });

  // dropTask: Ignores new requests while running (prevent double-submit)
  saveTask = dropTask(async (data: Record<string, unknown>) => {
    const response = await fetch('/api/save', {
      method: 'POST',
      body: JSON.stringify(data)
    });
    return response.json();
  });

  // enqueueTask: Queues requests sequentially
  processTask = enqueueTask(async (item: Record<string, unknown>) => {
    const response = await fetch('/api/process', {
      method: 'POST',
      body: JSON.stringify(item)
    });
    return response.json();
  });

  <template>
    <button
      {{on "click" (fn this.saveTask.perform @data)}}
      disabled={{this.saveTask.isRunning}}
    >
      {{if this.saveTask.isRunning "Saving..." "Save"}}
    </button>

    {{#if this.saveTask.lastSuccessful}}
      <div>Saved successfully!</div>
    {{/if}}

    {{#if this.saveTask.last.isError}}
      <div>Error: {{this.saveTask.last.error.message}}</div>
    {{/if}}
  </template>
}
```

### TaskInstance API for Derived Data

ember-concurrency provides derived state - no tracked properties needed:

- `task.isRunning` - Boolean if any instance is running
- `task.last` - Most recent TaskInstance (successful or failed)
- `task.lastSuccessful` - Most recent successful TaskInstance (persists during new attempts)
- `taskInstance.value` - The returned value
- `taskInstance.isError` - Boolean if failed
- `taskInstance.error` - The error object

### Quick Reference

**Use getPromiseState for:**

- Component initialization data loading
- Service-based reactive data
- Simple API calls without user concurrency concerns

**Use route model hook for:**

- First-level components only (direct children of route template)
- Avoid for deeply nested components (causes prop drilling)

**Use URL state + service for:**

- Shareable/bookmarkable filtered views
- Query params driving service state
- Deep component trees needing reactive data

**Use ember-concurrency for:**

- User input debouncing (search, autocomplete)
- Form submission (prevent double-click with `dropTask`)
- Polling (user-controlled refresh intervals)
- Multi-step wizards (sequential async operations)

**Key Principles:**

1. **Derive data, don't set it** - Use `task.lastSuccessful.value`, never set `@tracked` inside tasks
2. **Return values from tasks** - Let the TaskInstance API manage state
3. **Choose the right modifier**: `restartableTask` for search, `dropTask` for submit, `enqueueTask` for queues

**When NOT to use ember-concurrency:**

- Component initialization data loading (use `getPromiseState`)
- Setting tracked state inside tasks (causes render loops)
- Route model hooks (return promises directly)

References:

- [ember-concurrency](https://ember-concurrency.com/)

- [reactiveweb](https://github.com/universal-ember/reactiveweb)
