---
name: vue-to-rust
description: Use when migrating Vue.js frontends to Rust WASM — covers SFC to Leptos/Dioxus components, ref()/reactive() to signals, v-if/v-for to Show/For, Pinia/Vuex to signal stores, vue-router to leptos_router, Vite to Trunk, and incremental WASM replacement. Includes canonical code patterns, common mistakes, and reference implementations.
updated: 2026-07-30
---

# Vue to Rust (WASM) Migration Guide

> **Note**: This skill covers frontend/WASM migration, which is fundamentally different from the backend/runtime language migrations in the other `*-to-rust` skills. While other skills map server-side runtimes (Python, Go, Java, etc.) to Rust native binaries, this skill maps a browser-based frontend framework to Rust compiled to WebAssembly. The target runtimes are Leptos and Dioxus, which run in the browser via WASM, not as native binaries.

## Architecture Mapping

Vue's reactive component model (Vue SFC with `<template>`/`<script>`/`<style>`, reactivity via `ref()`/`reactive()`, virtual DOM diffing) maps to Rust WASM frameworks (Leptos, Dioxus, Yew) that compile to WebAssembly and run in the browser at near-native speed. Where Vue relies on JavaScript's runtime type system and V8's JIT compilation, Rust WASM frameworks use AOT-compiled Rust with fine-grained reactivity (signals) and zero-cost iterator-based rendering — no virtual DOM overhead in Leptos/Dioxus.

The migration is fundamentally a shift from JavaScript's runtime dynamism to Rust's compile-time guarantees, but within the same browser execution environment. Vue's single-file components become Rust component functions; Vue's `ref()`/`computed()` become `RwSignal`/`Memo`; Vue Router becomes `leptos_router`; Pinia/Vuex stores become signal-based reactive stores. The build pipeline shifts from Vite/webpack to Trunk/wasm-pack, and npm dependencies become Cargo crates. For incremental adoption, Rust WASM modules can be imported into an existing Vue app via `wasm-bindgen`, allowing component-by-component replacement. For SSR (Nuxt), Leptos and Dioxus Fullstack provide isomorphic Rust rendering on both server and client.

| Vue Concept | Rust Equivalent |
|---|---|
| Vue SFC (.vue) | Leptos component function / Yew functional component / Dioxus `rsx!` |
| Vue instance / app | `leptos::mount_to_body()` / `yew::Renderer::<App>::new().render()` |
| Vue CLI / Vite | Trunk / `wasm-pack` + webpack plugin / Cargo |
| npm / yarn / pnpm | Cargo + crates.io |
| Vue DevTools | `console_error_panic_hook` / browser dev console |
| `vue-router` | `leptos_router` / `yew-router` / `dioxus-router` |
| Pinia / Vuex | `leptos::RwSignal` store module / `reactive_stores` macro |
| `vue-i18n` | `fluent-rs` / `rust-i18n` / custom compile-time i18n |
| TypeScript | Rust (always typed; no distinction) |
| Nuxt (SSR) | Leptos SSR / Yew Server-Side Rendering / Dioxus Fullstack |

The migration target is WebAssembly, which means: no direct DOM access (uses framework abstracted DOM), no blocking the main thread (use wasm-bindgen-futures), and a compile-step between Rust source and browser JavaScript.

## Type System Mapping

Vue uses JavaScript/TypeScript types. Rust WASM code communicates with JavaScript via `wasm-bindgen`.

| Vue / JS / TS Type | Rust Type | Notes |
|---|---|---|
| `string` | `String` | Direct mapping via wasm-bindgen |
| `number` | `f64` / `i32` / `u32` | Choose based on domain; wasm-bindgen bridges to JS number |
| `boolean` | `bool` | Direct mapping |
| `null` / `undefined` | `Option<T>` | `JsValue::NULL` / `JsValue::UNDEFINED` when bridging to JS |
| `Array<T>` | `Vec<T>` / `js_sys::Array` | `Vec` is copied across boundary; `Array` for large data |
| `Object` / `Record<K,V>` | `HashMap<K,V>` / `BTreeMap<K,V>` / struct | For JS interop, `JsValue` with `Reflect::get` |
| `Map<K,V>` | `js_sys::Map` / `HashMap<K,V>` | Native Map via js-sys when needed |
| `Set<T>` | `js_sys::Set` / `HashSet<T>` | |
| `Promise<T>` | `wasm_bindgen_futures::JsFuture` | `.await` a JS Promise from Rust |
| `Function` | `js_sys::Function` | Call JS callbacks via wasm-bindgen |
| `Ref<T>` (template ref) | `NodeRef` (Leptos) / `NodeRef` (Yew) | |
| `Event` | `web_sys::Event` / `web_sys::MouseEvent` | Typed event wrappers via web-sys |
| Template literal type | `&'static str` / const generics | Rust has more powerful compile-time string handling |
| `any` | `JsValue` | Only when bridging to JS; minimize usage |
| `enum` (TypeScript) | `enum` (Rust) / `strum` derive macros | |

### Prop Validation in Rust

```rust
// Vue:
// interface Props { title: string; count?: number; }

// Leptos:
#[component]
fn MyComponent(
    #[prop(into)] title: String,              // String 可从 &str 转换
    #[prop(optional)] count: Option<i32>,      // 可选 prop，默认 None
    #[prop(default = 0)] count_default: i32,   // 带默认值的可选 prop
) -> impl IntoView {
    view! { <div>{title} — count: {count_default}</div> }
}
```

## Memory & Ownership Model

Vue's reactivity system automatically tracks dependencies and cleans up. Rust WASM frameworks use signals and explicit ownership for similar behavior.

| Vue Reactivity | Rust Equivalent |
|---|---|
| `ref()` / `reactive()` | `create_signal()` / `RwSignal::new()` (Leptos) |
| `computed()` | `create_memo()` / `Memo::new()` (Leptos) |
| `watch()` | `create_effect()` (Leptos) / `use_effect()` (Dioxus) |
| `watchEffect()` | `create_effect()` — runs immediately and on every dependency change |
| `shallowRef` | `ArcSwap<T>` / signal with manual `.set()` |
| `toRaw()` / `markRaw()` | Not needed — visibility controlled by pub |
| `readonly()` | `ReadSignal` / `Signal::read_only()` |
| `$ref()` (Vue 3.5) | Signal getter/setter pattern |
| Automatic cleanup on unmount | `on_cleanup()` / `Drop` implementation |
| `provide` / `inject` | `provide_context()` / `use_context()` (Leptos) |

### Reactive Store Translation (Pinia/Vuex)

```rust
// Vue Pinia store:
// const useCounter = defineStore('counter', () => {
//   const count = ref(0);
//   const double = computed(() => count.value * 2);
//   function increment() { count.value++; }
//   return { count, double, increment };
// });

// Rust Leptos — 文件: src/stores/counter.rs
use leptos::prelude::*;

#[derive(Clone)]
pub struct CounterStore {
    pub count: RwSignal<i32>,
    pub double: Memo<i32>,
}

impl CounterStore {
    pub fn new() -> Self {
        let count = RwSignal::new(0);
        let double = Memo::new(move |_| count.get() * 2);
        Self { count, double }
    }

    pub fn increment(&self) {
        self.count.update(|n| *n += 1);
    }
}

// 在组件中使用:
// let store = CounterStore::new();
// provide_context(store.clone());
```

## Concurrency / Async Translation

Vue runs on the browser's single-threaded event loop. Rust WASM uses wasm-bindgen-futures to bridge the JS Promise world with Rust's async/await.

| Vue / JS Async | Rust WASM Equivalent |
|---|---|
| `async/await` | `async fn` / `.await` (requires wasm-bindgen-futures executor) |
| `Promise.all([a, b])` | `futures::future::join_all` / `join!(a, b)` |
| `Promise.race([a, b])` | `futures::future::select(a, b)` |
| `setTimeout(fn, ms)` | `gloo_timers::future::TimeoutFuture::new(ms)` |
| `setInterval(fn, ms)` | `gloo_timers::callback::Interval` |
| `fetch()` / `axios` | `reqwest` (WASM feature) / `gloo_net::http::Request` |
| `WebSocket` | `web_sys::WebSocket` / `gloo_net::websocket` |
| `requestAnimationFrame(fn)` | `request_animation_frame()` via wasm-bindgen |
| `EventTarget.addEventListener` | `web_sys::EventTarget::add_event_listener_with_callback` |
| Worker thread | `web_sys::Worker` / `wasm_bindgen_rayon` |
| `AbortController` | `futures::future::AbortHandle` / Drop the future |

### Async Data Fetching Component

```rust
use leptos::prelude::*;
use serde::{Deserialize, Serialize};

#[derive(Clone, Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
    email: String,
}

// Vue:
// const user = ref(null);
// onMounted(async () => user.value = await fetch('/api/user/1').then(r => r.json()));

// Rust Leptos:
#[component]
fn UserProfile(user_id: u64) -> impl IntoView {
    let user = LocalResource::new(
        move || user_id,
        |id| async move {
            let url = format!("/api/user/{id}");
            reqwest::get(&url)
                .await
                .ok()
                .and_then(|r| r.json::<User>().await.ok())
        },
    );

    view! {
        <Suspense fallback=|| view! { <p>"Loading..."</p> }>
            {move || Suspend::new(async move {
                user.await.map(|u| view! {
                    <h2>{u.name.clone()}</h2>
                    <p>{u.email.clone()}</p>
                })
            })}
        </Suspense>
    }
}
```

## Build System & Dependencies

| Vue Tool | Rust Equivalent |
|---|---|
| `npm create vue@latest` | `cargo init` + add leptos/yew/dioxus dependency |
| `vite.config.ts` | `Trunk.toml` / `Cargo.toml [profile.release]` |
| `package.json` | `Cargo.toml` |
| `node_modules/` | `target/` (compilation artifacts) |
| `npm run dev` | `trunk serve` / `dx serve` (Dioxus) |
| `npm run build` | `trunk build --release` / `wasm-pack build` |
| `npm run test` | `wasm-pack test --headless` / `cargo test` |
| `tsconfig.json` | `Cargo.toml` + `rust-toolchain.toml` |
| `.env` / `import.meta.env` | `dotenvy` + compile-time env with `env!()` |
| PostCSS / Tailwind | CSS file linked in `index.html` or `stylist-rs` |
| ESLint / Prettier | `clippy` / `rustfmt` |
| Husky (git hooks) | `cargo-husky` / manual git hooks |

### Sample Cargo.toml for a Vue-to-Rust Migration

```toml
[package]
name = "my-app"
version = "0.1.0"
edition = "2021"

[dependencies]
leptos = { version = "0.7", features = ["csr"] }  # 客户端渲染
leptos_router = { version = "0.7", features = ["csr"] }
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
web-sys = { version = "0.3", features = ["Window", "Document", "Element", "HtmlElement"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
reqwest = { version = "0.12", features = ["json"] }
gloo-net = "0.6"
gloo-timers = { version = "0.3", features = ["futures"] }
console_error_panic_hook = "0.1"
tracing = "0.1"
tracing-wasm = "0.2"

[profile.release]
opt-level = "s"    # 优化 WASM 体积
lto = true
codegen-units = 1
```

## Standard Library & Ecosystem Mapping

| Vue / JS Library | Rust Equivalent |
|---|---|
| `vue-router` | `leptos_router` / `yew-router` / `dioxus-router` |
| Pinia / Vuex | `leptos::RwSignal` / reactive stores |
| `@vueuse/core` (useMouse, useStorage) | `gloo-events` / `web-sys` + signal |
| `axios` / `fetch()` | `reqwest` (WASM) / `gloo-net::http` |
| `lodash` | `itertools` + std collections |
| `dayjs` / `date-fns` | `chrono` (with wasm-bindgen feature) / `time` |
| `vee-validate` / `zod` | `validator` crate / `garde` |
| `vue-i18n` | `fluent-rs` / `rust-i18n` / custom compile-time `phf` map |
| `marked` (markdown) | `pulldown-cmark` (compile-time safe) |
| hammer.js (gestures) | `web-sys` TouchEvent / `gloo-events` |
| `echarts` / `chart.js` | `plotters` (renders to WASM canvas) / `d3`-alike via web-sys |
| `highlight.js` | `syntect` (server side) / `tree-sitter` highlighting |
| `socket.io-client` | `gloo-net::websocket` / `web_sys::WebSocket` |
| `localStorage` | `web_sys::Storage` / `gloo-storage` |
| `IndexedDB` | `web_sys::IdbDatabase` / `rexie` crate |
| `crypto-js` | `web_sys::SubtleCrypto` / `ring` / `aes-gcm` crate |
| `@vue/test-utils` | `leptos::testing` / wasm-bindgen-test |
| `vitest` | `wasm-bindgen-test` / `cargo test` (headless browser) |

## Canonical Patterns

### Pattern 1: v-if / v-show to Conditional Rendering

```rust
// Vue:
// <div v-if="isVisible">Content</div>
// <div v-show="isVisible">Content</div>

// Rust Leptos:
use leptos::prelude::*;

#[component]
fn Conditional() -> impl IntoView {
    let (is_visible, set_visible) = signal(true);

    view! {
        // v-if 等价 — 从 DOM 中添加/移除
        <Show when=move || is_visible.get()
            fallback=|| view! { <p>"Hidden with v-if"</p> }
        >
            <p>"Shown"</p>
        </Show>
        // v-show 等价 — CSS display 切换
        <p style:display=move || if is_visible.get() { "block" } else { "none" }>
            "Shown with v-show style"
        </p>
    }
}
```

### Pattern 2: v-for to Iterator Map

```rust
// Vue:
// <li v-for="(item, index) in items" :key="item.id">{{ item.name }}</li>

// Rust Leptos:
use leptos::prelude::*;

#[derive(Clone, Debug, PartialEq, Eq)]
struct ListItem { id: u64, name: String }

#[component]
fn ItemList(items: Vec<ListItem>) -> impl IntoView {
    view! {
        <ul>
            {items.into_iter().map(|item| view! {
                <li key={item.id}>{item.name.clone()}</li>
            }).collect_view()}
        </ul>
    }
}

// 动态列表（响应式）:
#[component]
fn DynamicList() -> impl IntoView {
    let (items, set_items) = signal::<Vec<ListItem>>(vec![]);

    // Leptos 0.7 使用 For 组件进行高效的 keyed diff:
    view! {
        <For
            each=move || items.get()
            key=|item: &ListItem| item.id
            children=|item: ListItem| view! { <li>{item.name}</li> }
        />
    }
}
```

### Pattern 3: v-model Two-Way Binding

```rust
// Vue:
// <input v-model="username" />

// Rust Leptos — 受控输入:
use leptos::{prelude::*, html::Input};

#[component]
fn TextInput() -> impl IntoView {
    let (value, set_value) = signal(String::new());

    view! {
        <input type="text"
            prop:value=move || value.get()
            on:input=move |ev| {
                set_value.set(event_target_value(&ev));
            }
        />
        <p>"You typed: " {move || value.get()}</p>
    }
}
```

### Pattern 4: Component Emits as Callback Props

```rust
// Vue:
// const emit = defineEmits<{ 'update:modelValue': [value: string] }>();
// emit('update:modelValue', newValue);
//
// 父组件: <Child @update:model-value="handleUpdate" />

// Rust Leptos:
#[component]
fn Child<F>(on_change: F) -> impl IntoView
where
    F: Fn(String) + 'static,
{
    view! {
        <button on:click=move |_| on_change("new value".into())>
            "Trigger"
        </button>
    }
}

#[component]
fn Parent() -> impl IntoView {
    let (msg, set_msg) = signal(String::new());
    view! {
        <Child on_change=move |v| set_msg.set(v) />
        <p>{move || msg.get()}</p>
    }
}
```

### Pattern 5: Lifecycle Hooks to on_cleanup / on_mount

```rust
// Vue:
// onMounted(() => { init(); return () => cleanup(); });

// Rust Leptos:
use leptos::prelude::*;

#[component]
fn Lifecycled() -> impl IntoView {
    // onMounted — 在 DOM 挂载后运行
    on_mount(move || {
        // init();
    });

    // onUnmounted — 在组件移除时清理
    on_cleanup(|| {
        // cleanup();
    });
    // 或通过 Drop trait 自动清理资源

    view! { <div>"Component"</div> }
}
```

### Pattern 6: Vue Transition to Leptos AnimatedShow

```rust
// Vue:
// <Transition name="fade">
//   <p v-if="show">Hello</p>
// </Transition>

// Rust Leptos:
use leptos::prelude::*;

#[component]
fn Animated() -> impl IntoView {
    let (show, set_show) = signal(true);

    view! {
        <AnimatedShow
            when=move || show.get()
            show_class="fade-enter-active"
            hide_class="fade-leave-active"
        >
            <p>"Hello"</p>
        </AnimatedShow>
    }
}
```

### Pattern 7: Scoped CSS

```rust
// Vue:
// <style scoped>
// .title { color: red; }
// </style>

// Rust Leptos — 使用 stylist-rs:
use stylist::yew::styled_component;
use yew::prelude::*;

#[styled_component]
fn MyComponent() -> Html {
    html! {
        <div class={css!(r#"
            .title { color: red; }
        "#)}>
            <h1 class="title">{"Hello"}</h1>
        </div>
    }
}

// 或 Leptos 中使用 wasm-bindgen 导入外部 CSS module
```

## FFI & Incremental Migration

When migrating a Vue app, you can run Vue and Rust WASM side by side in the same page.

| Strategy | Tool | When to Use |
|---|---|---|
| WASM module in Vue app | `wasm-pack build --target bundler` | Incrementally replace Vue components |
| Vue renders Rust-WASM component | Custom element via `wasm-bindgen` | Wrap Rust component as `<my-wasm-component>` |
| Rust app calls legacy JS | `js_sys` / `web_sys` / wasm-bindgen imports | Leverage existing JS libraries |
| Micro-frontend approach | Separate entry points, shared router | Large app with isolated Rust modules |
| Full rewrite | Pure Rust WASM framework | Small-to-medium apps |

### Calling Rust WASM from Vue

```rust
// Rust — 编译为 WASM 模块，暴露给 JS:
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct Calculator { value: f64 }

#[wasm_bindgen]
impl Calculator {
    pub fn new() -> Self { Self { value: 0.0 } }

    pub fn add(&mut self, n: f64) -> f64 {
        self.value += n;
        self.value
    }

    pub fn get_value(&self) -> f64 { self.value }
}
```

```typescript
// Vue component calling WASM:
import init, { Calculator } from '../wasm-pkg/my_app';

const calc = shallowRef<Calculator | null>(null);

onMounted(async () => {
  await init();
  calc.value = Calculator.new();
});

function add(n: number) {
  calc.value?.add(n);
}
```

### Migration Order

1. Move computation-heavy functions to Rust WASM first (sorting, filtering, string processing).
2. Extract state management from Pinia/Vuex into `wasm-bindgen`-exposed Rust structs.
3. Replace leaf components (buttons, inputs, cards) with WASM-rendered equivalents.
4. Rewrite page-level components using a Rust framework.
5. Switch routing to Rust-side routing with history API.
6. Remove Vue dependency entirely.

## Common Mistakes

### Mistake 1: Blocking the Main Thread with Heavy Computation

```rust
// 错误 — 阻塞浏览器 UI 线程:
fn heavy_processing(input: &[u8]) -> Vec<u8> {
    // 耗时 500ms 的处理 — 页面在此期间无响应
    input.iter().map(|b| b.wrapping_mul(3)).collect()
}

// 正确 — 使用 setTimeout 或 requestAnimationFrame 分段处理:
async fn heavy_processing_non_blocking(input: Vec<u8>) -> Vec<u8> {
    let chunk_size = 1000;
    let mut result = Vec::with_capacity(input.len());
    for chunk in input.chunks(chunk_size) {
        result.extend(chunk.iter().map(|b| b.wrapping_mul(3)));
        // 每个 chunk 后让出控制权给浏览器
        gloo_timers::future::TimeoutFuture::new(0).await;
    }
    result
}
```

### Mistake 2: Passing Large Objects Across WASM Boundary Frequently

```rust
// 错误 — 每次渲染都跨边界传递大数据:
#[wasm_bindgen]
pub fn render_table(data: Vec<Row>) -> Vec<u8> { /* ... */ }

// 正确 — 在 Rust 端维护数据，只传递变更:
#[wasm_bindgen]
pub struct TableEngine { rows: Vec<Row>, render_cache: Vec<u8> }

#[wasm_bindgen]
impl TableEngine {
    pub fn update_row(&mut self, idx: usize, row: Row) {
        self.rows[idx] = row;
        // 只重渲染变更的行
    }
}
```

### Mistake 3: Cloning Large Structures in Signal Updates

```rust
// 错误 — 每次信号更新都克隆整个 Vec:
let (items, set_items) = signal(vec![1i32; 10000]);

// 事件处理中:
set_items.update(|v| {
    let mut new_vec = v.clone();  // 不必要的克隆
    new_vec.push(42);
    new_vec
});

// 正确 — 在原位更新:
set_items.update(|v| v.push(42));  // Vec::push 直接修改
// 或使用 StoredValue 存储大结构并用内部可变性
```

### Mistake 4: Forgetting to Register the Panic Hook

```rust
// 错误 — WASM 中 panic 输出不可读:
fn main() {
    mount_to_body(|| view! { <App /> }); // panic 时只有 "unreachable"
}

// 正确 — 在主函数开头注册 hook:
fn main() {
    console_error_panic_hook::set_once();   // 友好的 panic 消息
    tracing_wasm::set_as_global_default();  // tracing 输出到 console
    mount_to_body(|| view! { <App /> });
}
```

### Mistake 5: Using std::time::Instant::now() in WASM

```rust
// 错误:
let start = std::time::Instant::now(); // WASM 中时钟精度很低

// 正确 — 使用浏览器 Performance API:
fn now_ms() -> f64 {
    web_sys::window()
        .and_then(|w| w.performance())
        .map(|p| p.now())
        .unwrap_or(0.0)
}
```

## Reference Implementations

| Project | Description | Migration Pattern |
|---|---|---|
| [Leptos Hacker News Example](https://github.com/leptos-rs/leptos/tree/main/examples/hackernews) | Complete SPA in Leptos | Full Rust WASM rewrite pattern |
| [Yew Realworld Example](https://github.com/jetli/rust-yew-realworld) | Medium clone in Yew | Full SPA migration from Vue/React |
| [Dioxus Tailwind](https://github.com/DioxusLabs/dioxus/tree/main/examples/tailwind) | Tailwind CSS in Dioxus | Build system equivalent to Vite + PostCSS |
| [r3bl-view](https://github.com/r3bl-org/r3bl-open-core) | TUI and GUI framework in Rust | Component architecture patterns |
| [Perseus](https://github.com/framesurge/perseus) | SSR framework with state management | Nuxt-to-Rust equivalent |
| [sycamore](https://github.com/sycamore-rs/sycamore) | Fine-grained reactivity, no VDOM | Vue 3 reactivity model analogue |

## Cross-Reference

- **nodejs-to-rust** — For migrating Express/Fastify backends and npm build tooling
- **go-to-rust** — For migrating API services that Vue frontends connect to
- **php-to-rust** — For migrating Laravel backends paired with Vue frontends
- **java-to-rust** — For migrating Spring Boot backends paired with Vue frontends
