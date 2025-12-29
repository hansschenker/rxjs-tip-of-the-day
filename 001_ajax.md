# RxJS Tip of the Day — Creation Operator: `ajax` (Day 1 — 2025-12-28)

The RxJS `ajax` creation function wraps `XMLHttpRequest` and exposes the request as a **cold** `Observable`: the HTTP request starts on **subscribe**, and is **cancellable** via **unsubscribe**.

> Imports used below:
>
> ```ts
> import { ajax } from 'rxjs/ajax';
> import {
>   catchError,
>   debounceTime,
>   distinctUntilChanged,
>   filter,
>   map,
>   startWith,
>   switchMap,
>   takeUntil,
> } from 'rxjs/operators';
> import { fromEvent, interval, merge, of } from 'rxjs';
> ```

---

## 1) Load a resource on demand (GET JSON)

**ajax** — “When the user clicks **Load**, call an API endpoint and emit the JSON response as a stream.”

**Scenario description (short):** A classic “load on click” interaction: convert a click into a request, emit the JSON payload, and keep the stream alive by treating errors as data.

```ts
import { ajax } from 'rxjs/ajax';
import { fromEvent, of } from 'rxjs';
import { catchError, map, startWith, switchMap } from 'rxjs/operators';

type Result<T> =
  | { type: 'loading' }
  | { type: 'ok'; value: T }
  | { type: 'err'; error: unknown };

type Todo = { id: string; title: string; done: boolean };

const loadBtn = document.querySelector<HTMLButtonElement>('#load')!;
const todosUl = document.querySelector<HTMLUListElement>('#todos')!;

const renderTodos = (ul: HTMLUListElement) => (r: Result<Todo[]>) => {
  if (r.type === 'loading') {
    ul.innerHTML = '<li>Loading…</li>';
    return;
  }
  if (r.type === 'err') {
    ul.innerHTML = `<li>Failed: ${String(r.error)}</li>`;
    return;
  }
  ul.innerHTML = r.value.map((t) => `<li>${t.done ? '✅' : '⬜'} ${t.title}</li>`).join('');
};

const loadClicks$ = fromEvent(loadBtn, 'click');

const todosResult$ = loadClicks$.pipe(
  switchMap(() =>
    ajax.getJSON<Todo[]>('/api/todos').pipe(
      map((todos): Result<Todo[]> => ({ type: 'ok', value: todos })),
      startWith({ type: 'loading' } as const),
      catchError((error): ReturnType<typeof of<Result<Todo[]>>> => of({ type: 'err', error }))
    )
  )
);

todosResult$.subscribe(renderTodos(todosUl));
```

---

## 2) Submit a form (POST JSON)

**ajax** — “When the user clicks **Save**, POST the form payload and emit the server acknowledgement as a stream.”

**Scenario description (short):** Convert a form submission into a POST request, and model *loading / success / error* explicitly as stream values.

```ts
import { ajax } from 'rxjs/ajax';
import { fromEvent, of } from 'rxjs';
import { catchError, map, startWith, switchMap } from 'rxjs/operators';

type Result<T> =
  | { type: 'idle' }
  | { type: 'saving' }
  | { type: 'ok'; value: T }
  | { type: 'err'; error: unknown };

type CreateTodo = { title: string };
type CreateTodoAck = { id: string; createdAt: string };

const form = document.querySelector<HTMLFormElement>('#createTodoForm')!;
const statusEl = document.querySelector<HTMLDivElement>('#status')!;

const renderStatus = (el: HTMLElement) => (r: Result<CreateTodoAck>) => {
  if (r.type === 'idle') el.textContent = '';
  if (r.type === 'saving') el.textContent = 'Saving…';
  if (r.type === 'ok') el.textContent = `Created: ${r.value.id} @ ${r.value.createdAt}`;
  if (r.type === 'err') el.textContent = `Save failed: ${String(r.error)}`;
};

const submit$ = fromEvent<SubmitEvent>(form, 'submit');

const createTodoResult$ = submit$.pipe(
  map((ev) => {
    ev.preventDefault();
    const fd = new FormData(form);
    const payload: CreateTodo = { title: String(fd.get('title') ?? '') };
    return payload;
  }),
  switchMap((payload) =>
    ajax<CreateTodoAck>({
      url: '/api/todos',
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: payload,
    }).pipe(
      map((res) => ({ type: 'ok', value: res.response } as const)),
      startWith({ type: 'saving' } as const),
      catchError((error) => of({ type: 'err', error } as const))
    )
  ),
  startWith({ type: 'idle' } as const)
);

createTodoResult$.subscribe(renderStatus(statusEl));
```

---

## 3) Autocomplete search (GET with query params + cancellation)

**ajax** — “When the user types in **Search**, call an endpoint with the query and emit suggestions, canceling stale requests.”

**Scenario description (short):** A typeahead: debounce input, ignore repeats, call the API for each query, and ensure only the latest request matters.

```ts
import { ajax } from 'rxjs/ajax';
import { fromEvent, of } from 'rxjs';
import {
  catchError,
  debounceTime,
  distinctUntilChanged,
  filter,
  map,
  startWith,
  switchMap,
} from 'rxjs/operators';

type Result<T> =
  | { type: 'idle' }
  | { type: 'loading' }
  | { type: 'ok'; value: T }
  | { type: 'err'; error: unknown };

type Suggestion = { id: string; label: string };

const input = document.querySelector<HTMLInputElement>('#search')!;
const suggestionsUl = document.querySelector<HTMLUListElement>('#suggestions')!;

const renderSuggestions = (ul: HTMLUListElement) => (r: Result<Suggestion[]>) => {
  if (r.type === 'idle') {
    ul.innerHTML = '';
    return;
  }
  if (r.type === 'loading') {
    ul.innerHTML = '<li>Searching…</li>';
    return;
  }
  if (r.type === 'err') {
    ul.innerHTML = `<li>Error: ${String(r.error)}</li>`;
    return;
  }
  ul.innerHTML = r.value.map((s) => `<li data-id="${s.id}">${s.label}</li>`).join('');
};

const query$ = fromEvent<InputEvent>(input, 'input').pipe(
  map(() => input.value.trim()),
  debounceTime(250),
  distinctUntilChanged()
);

const suggestionsResult$ = query$.pipe(
  // Avoid calling the backend for empty queries.
  filter((q) => q.length > 0),
  switchMap((q) =>
    ajax.getJSON<Suggestion[]>('/api/suggest', { 'X-Query': q }).pipe(
      // If your backend expects query params instead of headers, use: `/api/suggest?q=${encodeURIComponent(q)}`
      map((xs) => ({ type: 'ok', value: xs } as const)),
      startWith({ type: 'loading' } as const),
      catchError((error) => of({ type: 'err', error } as const))
    )
  ),
  startWith({ type: 'idle' } as const)
);

suggestionsResult$.subscribe(renderSuggestions(suggestionsUl));
```

---

## 4) Poll a job status until completion

**ajax** — “When the user clicks **Start**, poll a job-status endpoint every 2 seconds and emit status updates until completion.”

**Scenario description (short):** Convert a start action into a polling stream that repeatedly calls the backend, stops when the job completes, and can be canceled.

```ts
import { ajax } from 'rxjs/ajax';
import { fromEvent, interval, merge, of } from 'rxjs';
import { catchError, map, startWith, switchMap, takeUntil } from 'rxjs/operators';

type Result<T> =
  | { type: 'idle' }
  | { type: 'polling' }
  | { type: 'ok'; value: T }
  | { type: 'err'; error: unknown };

type JobStatus =
  | { state: 'queued' | 'running'; progressPct: number }
  | { state: 'done'; resultUrl: string };

const startBtn = document.querySelector<HTMLButtonElement>('#startJob')!;
const stopBtn = document.querySelector<HTMLButtonElement>('#stopJob')!;
const statusEl = document.querySelector<HTMLDivElement>('#jobStatus')!;

const renderJob = (el: HTMLElement) => (r: Result<JobStatus>) => {
  if (r.type === 'idle') el.textContent = '';
  if (r.type === 'polling') el.textContent = 'Polling…';
  if (r.type === 'err') el.textContent = `Error: ${String(r.error)}`;
  if (r.type === 'ok') {
    const s = r.value;
    el.textContent =
      s.state === 'done' ? `Done → ${s.resultUrl}` : `${s.state} (${s.progressPct}%)`;
  }
};

const start$ = fromEvent(startBtn, 'click');
const stop$ = fromEvent(stopBtn, 'click');

const jobStatusResult$ = start$.pipe(
  switchMap(() =>
    interval(2000).pipe(
      // Cancel polling when user clicks Stop.
      takeUntil(stop$),
      switchMap(() =>
        ajax.getJSON<JobStatus>('/api/job/status').pipe(
          map((s) => ({ type: 'ok', value: s } as const)),
          catchError((error) => of({ type: 'err', error } as const))
        )
      ),
      startWith({ type: 'polling' } as const)
    )
  ),
  // Clicking Stop resets UI to idle.
  takeUntil(merge(of(null), stop$)),
  startWith({ type: 'idle' } as const)
);

jobStatusResult$.subscribe(renderJob(statusEl));
```

---

## 5) Download a file (GET binary response)

**ajax** — “When the user clicks **Download**, request a file as a `Blob` and emit a downloadable object URL.”

**Scenario description (short):** Use `ajax` to request non-JSON content (e.g., PDF/CSV), then create an object URL at the subscribe boundary to trigger a browser download.

```ts
import { ajax } from 'rxjs/ajax';
import { fromEvent, of } from 'rxjs';
import { catchError, map, startWith, switchMap } from 'rxjs/operators';

type Result<T> =
  | { type: 'idle' }
  | { type: 'downloading' }
  | { type: 'ok'; value: T }
  | { type: 'err'; error: unknown };

const downloadBtn = document.querySelector<HTMLButtonElement>('#download')!;
const downloadStatus = document.querySelector<HTMLDivElement>('#downloadStatus')!;

const renderDownload = (el: HTMLElement) => (r: Result<{ url: string; filename: string }>) => {
  if (r.type === 'idle') el.textContent = '';
  if (r.type === 'downloading') el.textContent = 'Downloading…';
  if (r.type === 'err') el.textContent = `Download failed: ${String(r.error)}`;
  if (r.type === 'ok') el.textContent = `Ready: ${r.value.filename}`;
};

const clicks$ = fromEvent(downloadBtn, 'click');

const downloadResult$ = clicks$.pipe(
  switchMap(() =>
    ajax<Blob>({
      url: '/api/reports/monthly.pdf',
      method: 'GET',
      responseType: 'blob',
    }).pipe(
      map((res) => {
        const blob = res.response;
        const url = URL.createObjectURL(blob);
        return { type: 'ok', value: { url, filename: 'monthly.pdf' } } as const;
      }),
      startWith({ type: 'downloading' } as const),
      catchError((error) => of({ type: 'err', error } as const))
    )
  ),
  startWith({ type: 'idle' } as const)
);

// Side effects at the subscribe boundary: trigger the browser download.
const triggerDownload = ({ url, filename }: { url: string; filename: string }) => {
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  a.click();
  URL.revokeObjectURL(url);
};

downloadResult$.subscribe((r) => {
  renderDownload(downloadStatus)(r);
  if (r.type === 'ok') triggerDownload(r.value);
});
```

---

## Notes you should internalize from these examples

- `ajax(...)` is **cold**: no request is made until you `subscribe()`.
- **Cancellation is built-in**: unsubscribing aborts the underlying `XMLHttpRequest`.
- Prefer modeling “loading/success/error” as **data** (values) so the stream stays alive.

---

## Project credit

This repository is maintained as **RxJS Tip of the Day**.

Mentor: **ChatGPT 5.2**.

