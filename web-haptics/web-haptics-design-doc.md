# **Web Haptics API Design Doc**

## This Document is Public

*Authors: [akyereboah@microsoft.com](mailto:akyereboah@microsoft.com)*
*August 2026*

---

# **One-page overview**
**WICG Explainer**: https://github.com/WICG/web-haptics/blob/main/readme.md
### **Summary**

The Web Haptics API gives web content a way to request **semantic, intent-driven
haptic feedback** rather than programming raw vibration durations like the legacy
`navigator.vibrate()`. The API has **two parts** that share one effect vocabulary
and one target-selection model:

1. **Imperative API (JS)**: `navigator.playHaptics(effect, intensity)`, for
   interactions that need runtime logic.
2. **Declarative API (CSS)**: a nested `@haptic` at-rule that fires an effect
   when its style rule starts matching, with no JavaScript.

### **Platforms**

Currently targeting Windows, planned for Mac, Linux, ChromeOS, WebKit, Android, Android WebView.

### **Team**
* akyereboah@microsoft.com
* luhua@microsoft.com
* limzh@microsoft.com
* rob.paveza@microsoft.com
* devexpwa@microsoft.com


### **Bug**

[531787872](https://issues.chromium.org/issues/531787872)

### **Code affected**

Blink bindings & modules (`third_party/blink/renderer/modules/haptics/`),
renderer-facing mojom (`third_party/blink/public/mojom/haptics/`), the
browser-process `content::HapticsService` intermediary, Windows backend, and
interface binder (`content/browser/`), permissions-policy and runtime-flag
registration. (A device-service backend for non-Windows platforms is deferred, see
[D5](#d5-device-service-backend-non-windows-platforms).)

---

# **Design**

*The Web Haptics feature is split into the two API parts described above. All
current design work is in **imperative API**.
**Declarative CSS** is a placeholder.*

## Imperative API: `navigator.playHaptics()`

### D0. Background and end-to-end data flow

`navigator.playHaptics(effect, intensity)` is a call that returns `undefined`.
The renderer does an early, fast-path check, then forwards the request
over a Mojo interface to `content::HapticsService`, a **browser-process**
intermediary that enforces policies and appropriate gating
before invoking the platform haptics backend.

Data flow for a single call:

```
Web page (JS)
  navigator.playHaptics("tick", 0.5)
        │
        ▼
Blink renderer  (third_party/blink/renderer/modules/haptics/)
  HapticsController::PlayHaptics   
    • fenced-frame check
    • "haptics" permissions policy check
    • user-activation check
    • clamp intensity to [0,1]

        │  blink.mojom.HapticsService::PlayHaptics(effect, intensity)
        ▼  (Mojo IPC)

Browser process  (content/browser/haptics/)
  content::HapticsService   (authoritative re-checks; DocumentService)
    • fenced-frame re-check
    • active-document re-check
    • "haptics" permissions policy re-check
    • user-activation re-check
    • re-clamp intensity
        │  in-process C++ call (no device-service hop on Windows)
        ▼
  HapticsManagerImplWin   (runs on the UI thread)
        │
        ▼
Windows.Devices.Haptics.InputHapticsManager (WinRT API)
  → most-recent input device's SimpleHapticsController → hardware vibration
```

*A device-service backend (a second, browser↔utility-process pipe) is **not**
built for the Windows milestone; see [D5](#d5-device-service-backend-non-windows-platforms).*

### D1. JS surface: IDL, effect enum, runtime flag

The API is a partial `Navigator` interface, gated by the `WebHaptics` runtime
flag and implemented by `HapticsController`.

`third_party/blink/renderer/modules/haptics/navigator_haptics.idl`:

```webidl
// The predefined, semantic haptic effect names.
enum HapticEffect { "hint", "edge", "tick", "align" };

[
    ImplementedAs=HapticsController,
    RuntimeEnabled=WebHaptics,
    MeasureAs=NavigatorPlayHaptics
] partial interface Navigator {
    undefined playHaptics(HapticEffect effect, optional double intensity = 1);
};
```

Because `playHaptics()` returns `undefined` and exposes no success signal, web
  content cannot observe whether a device actually played (see [Privacy](#privacy-considerations)).

### D2. Blink renderer: `HapticsController`

`HapticsController` is a `Supplement<Navigator>` that applies the API's gating
rules and forwards to the browser over Mojo. These renderer-side checks are a
fast path only, not the security boundary: `content::HapticsService` re-checks
them in the browser ([D4](#d4-process-routing-and-the-contenthapticsservice-intermediary)).

`third_party/blink/renderer/modules/haptics/haptics_controller.cc`:

```cpp
void HapticsController::PlayHaptics(const V8HapticEffect& effect,
                                    double intensity) {
  LocalDOMWindow* window = GetSupplementable()->DomWindow();
  if (!window) return;

  LocalFrame* frame = window->GetFrame();
  if (!frame) return;

  // Like navigator.vibrate(), haptics are suppressed inside fenced frames.
  if (frame->IsInFencedFrameTree()) return;

  // Gated by the "haptics" permissions policy (default allowlist "self").
  if (!window->IsFeatureEnabled(
          network::mojom::PermissionsPolicyFeature::kHaptics)) {
    return;
  }

  // The imperative API requires sticky user activation.
  if (!frame->HasStickyUserActivation()) return;

  // Normalize intensity defensively (the IDL default is 1.0).
  intensity = std::clamp(intensity, 0.0, 1.0);

  if (!haptics_service_.is_bound()) {
    window->GetBrowserInterfaceBroker().GetInterface(
        haptics_service_.BindNewPipeAndPassReceiver(
            window->GetTaskRunner(TaskType::kMiscPlatformAPI)));
  }
  // One-way mojo call to content::HapticsService; no reply callback.
  haptics_service_->PlayHaptics(ToMojoEffect(effect), intensity);
}
```

* **Gating order**: fenced-frame → permissions policy → sticky activation.
Every early-return silently ignores the call.
* **Use counter**: usage is metered declaratively by `[MeasureAs=NavigatorPlayHaptics]`
  on the IDL operation ([D1](#d1-js-surface-idl-effect-enum-runtime-flag)), which the generated binding records before this
  method runs.
* **Lazy remote**: the `HeapMojoRemote<blink::mojom::HapticsService>` is bound on
  first use and reset in `ContextDestroyed()`.
* **Enum translation**: `ToMojoEffect()` converts the V8 enum to the mojo enum
  1:1 (`kHint`/`kEdge`/`kTick`/`kAlign`).

### D3. Mojo interface: `blink.mojom.HapticsService`

The renderer-facing interface lives in `third_party/blink/public/mojom/haptics/` (because for the Windows milestone the only Mojo pipe is
renderer ↔ browser). `content::HapticsService` is the browser-side receiver.

`third_party/blink/public/mojom/haptics/haptics.mojom`:

```mojom
module blink.mojom;

enum HapticEffect { kHint, kEdge, kTick, kAlign };

// Implemented by content::HapticsService in the browser process, which
// re-checks policy before invoking the platform backend.
interface HapticsService {
  // Plays |effect| at |intensity| (normalized [0.0, 1.0]) on the most recent
  // input device. Fire-and-forget: no reply is sent, and no success signal is
  // exposed, to avoid fingerprinting.
  PlayHaptics(HapticEffect effect, double intensity);
};
```

A single `HapticEffect`/`PlayHaptics` shape is reused end-to-end. A separate
device-service mojom is not defined pre-emptively ([D5](#d5-device-service-backend-non-windows-platforms)).

`PlayHaptics` is a **one-way** method. The API returns nothing
to JS and the renderer never inspects a response.

### D4. Process routing and the `content::HapticsService` intermediary

The renderer never reaches the platform backend directly. The per-frame binder
maps `blink.mojom.HapticsService` to `content::HapticsService`, a browser-process
object that owns the security boundary. It is a
[`DocumentService`](https://source.chromium.org/chromium/chromium/src/+/main:content/public/browser/document_service.h),
so its lifetime is tied to the document and the pipe, and it exposes
`render_frame_host()` for the re-checks.

`content/browser/browser_interface_binders.cc`:

```cpp
map->Add<blink::mojom::HapticsService>(base::BindRepeating(
    [](RenderFrameHost* host,
       mojo::PendingReceiver<blink::mojom::HapticsService> receiver) {
      HapticsServiceImpl::Create(host, std::move(receiver));
    }));
```

`content/browser/haptics/haptics_service_impl.cc`:

```cpp
// static
void HapticsServiceImpl::Create(
    RenderFrameHost* rfh,
    mojo::PendingReceiver<blink::mojom::HapticsService> receiver) {
  // DocumentService owns itself; deleted when the document or pipe goes away.
  new HapticsServiceImpl(*rfh, std::move(receiver));
}

void HapticsServiceImpl::PlayHaptics(blink::mojom::HapticEffect effect,
                                     double intensity) {
  //Re-verify every gate from the untrusted renderer.
  if (render_frame_host().IsNestedWithinFencedFrame()) return;
  // Drop calls from a document that is no longer active (for example one in
  // the back/forward cache or pending deletion): its pipe stays live and its
  // sticky activation still passes, so this is checked explicitly.
  if (!render_frame_host().IsActive()) return;
  if (!render_frame_host().IsFeatureEnabled(
          network::mojom::PermissionsPolicyFeature::kHaptics)) {
    return;
  }
  if (!render_frame_host().HasStickyUserActivation()) return;

  // Reject non-finite values: std::clamp() passes NaN through unchanged.
  if (!std::isfinite(intensity)) return;
  intensity = std::clamp(intensity, 0.0, 1.0);

#if BUILDFLAG(IS_WIN)
  // In-process capability call; no device-service hop needed on Windows.
  GetOrCreateHapticsManagerWin().PlayHaptics(effect, intensity);
#endif
}
```

**Why re-check in the browser.** The renderer's checks ([D2](#d2-blink-renderer-hapticscontroller)) run in a process the
page could compromise. `content::HapticsService` re-derives fenced-frame status, active-document
state, permissions policy, and user activation from browser-side state (`RenderFrameHost`), which the
page cannot forge. The active-document check matters because a `DocumentService` pipe outlives a
navigation while the document sits in the back/forward cache, so a queued call could otherwise run
for a page the user has left. Only after those pass does it make requests to the hardware.

**Why an in-process call on Windows.** `InputHapticsManager::GetForCurrentThread`
returns the haptics manager for the input queue of the *current* thread that owns
the top-level `HWND` receiving pointer input, which in Chromium is the browser
**UI thread**. Because `content::HapticsService` and the Windows backend both live
in the browser process, the intermediary reaches the backend with a direct C++ call;
there is no second Mojo pipe to a utility process. A browser↔device-service pipe
would only be introduced for a backend that must live in the device service (a
future non-Windows platform; see [D5](#d5-device-service-backend-non-windows-platforms)).

### D5. Device-service backend (non-Windows platforms)

**Not built for the Windows milestone.** The Windows backend runs in the browser
process ([D4](#d4-process-routing-and-the-contenthapticsservice-intermediary)), so
there is no browser↔device-service pipe. A device-service `HapticsManager` and a
cross-platform no-op base are **deferred** until a platform needs a backend hosted
in the device service.

The design will follow a two-pipe model:

* `content::HapticsService` stays the browser-side security boundary and the sole
  thing the renderer talks to.
* After its re-checks pass, instead of an in-process call it forwards over a
  second Mojo pipe (browser ↔ device service) to a device-service
  `mojom::HapticsManager`. That pipe may reuse `blink.mojom.HapticsService`, or
  a dedicated `device.mojom` interface can be added under
  `services/device/public/mojom/`.
* The device-service implementation is an **unguarded** capability: it
  trusts its (browser-process) client and does no policy checks, because those
  already happened in `content::HapticsService`. `PlayHaptics` would clamp
  defensively and drive the platform API, exactly as `HapticsManagerImplWin` does
  today.

See [Followup work](#followup-work).

### D6. Windows backend: `HapticsManagerImplWin`

`content/browser/haptics/haptics_manager_impl_win.{h,cc}` is the real
implementation, using the `Windows.Devices.Haptics.InputHapticsManager` API.

#### D6.1 Threading model and ownership

`HapticsManagerImplWin` is a plain browser-process object, not a Mojo
receiver. `content::HapticsService` ([D4](#d4-process-routing-and-the-contenthapticsservice-intermediary))
owns one lazily and calls it in-process on the UI thread. A `SEQUENCE_CHECKER`
guards every method:

```cpp
// Owned by content::HapticsService; created lazily on first PlayHaptics.
HapticsManagerImplWin& HapticsServiceImpl::GetOrCreateHapticsManagerWin() {
  DCHECK_CURRENTLY_ON(BrowserThread::UI);
  if (!win_backend_)
    win_backend_ = std::make_unique<HapticsManagerImplWin>();
  return *win_backend_;
}
```

All WinRT calls happen on the one thread that owns the input queue (the UI
thread), and the backend's lifetime is bounded by its owning
`content::HapticsService`, hence by the document.

#### D6.2 Statics resolution (`EnsureStatics`)

`EnsureStatics()` acquires the `InputHapticsManager` statics and the known-waveform
statics. It records that resolution was attempted so a failure is not retried on every call:

```cpp
HRESULT hr = base::win::GetActivationFactory<
    haptics::IInputHapticsManagerStatics,
    RuntimeClass_Windows_Devices_Haptics_InputHapticsManager>(
    &input_haptics_statics_);
if (FAILED(hr) || !input_haptics_statics_) { /* unsupported → no-op */ }

hr = base::win::GetActivationFactory<
    haptics::IKnownSimpleHapticsControllerWaveformsStatics2,
    RuntimeClass_Windows_Devices_Haptics_KnownSimpleHapticsControllerWaveforms>(
    &known_waveforms2_);
// Failure here is non-fatal; PlayHaptics falls back to a device waveform.
```

`InputHapticsManager` requires UniversalApiContract 19.0 (Windows 11 24H2+), so
this is where the OS floor is enforced: on earlier builds the class does not
exist, `EnsureStatics()` fails, and `PlayHaptics` is a clean no-op.

#### D6.3 Controller priming

**Problem:** On the very first `PlayHaptics` call after startup, the WinRT
haptics stack is cold, `get_CurrentHapticsController` returns null (or an empty
`SupportedFeedback` set) because the OS has not yet associated the haptic device
with the thread's input queue. The result is that the **first** button click
produces no vibration; only the second and later clicks work.

**Fix:** Perform one **throwaway** controller lookup at construction so the
association is established (and cached) before any real request arrives. Because
`Create` runs on the UI thread, the constructor is a safe place to do this:

```cpp
HapticsManagerImplWin::HapticsManagerImplWin() {
  PrimeHapticsController();
}

void HapticsManagerImplWin::PrimeHapticsController() {
  DCHECK_CURRENTLY_ON(BrowserThread::UI);
  if (!EnsureStatics()) return;

  ComPtr<haptics::IInputHapticsManager> manager;
  if (FAILED(input_haptics_statics_->GetForCurrentThread(&manager)) || !manager)
    return;

  // Discarded on purpose: the side effect is that the platform establishes and
  // caches the current haptics controller for this thread's input queue, so the
  // first real PlayHaptics sees a warm controller instead of a cold, null one.
  ComPtr<haptics::ISimpleHapticsController> controller;
  manager->get_CurrentHapticsController(&controller);
}
```

#### D6.4 Waveform detection, semantic mapping, and fallback

A haptic device only plays waveforms it advertises in its
`SimpleHapticsController.SupportedFeedback` set. There are real devices **that do
not implement** the Hover/Collide/Step/Align waveforms at all. When the preferred
semantic waveform is not available, Windows' own convention is to fall back to a
**default waveform chosen by the input-device type**. The Web Haptics API mirrors this so the fallback still feels intentional and consistent with the rest of the platform:

| Device type (`HapticDeviceType`) | Default fallback waveform |
|:---|:---|
| `Mouse`, `Touchpad` | **Hover** |
| `Pen` | **Click** |

(`Generic`/`None`/unknown default to **Hover**, the broadly-available choice.)

**Device type** is read from the same `InputHapticsManager`:

```cpp
haptics::HapticDeviceType device_type = haptics::HapticDeviceType_None;
manager->get_CurrentHapticsControllerDeviceType(&device_type);
```

**Default waveform for the current device type.** `Hover` lives on the
SDK-provided `Statics2` (already resolved in [D6.2](#d62-statics-resolution-ensurestatics)); `Click` lives on the base
`IKnownSimpleHapticsControllerWaveformsStatics`, obtained by `QueryInterface`-ing
the same activation factory:

```cpp
std::optional<uint16_t> HapticsManagerImplWin::DefaultWaveformForDevice(
    haptics::HapticDeviceType device_type) {
  if (!known_waveforms2_) return std::nullopt;
  UINT16 value = 0;
  if (device_type == haptics::HapticDeviceType_Pen) {
    // Click lives on the base statics interface, QI'd from the same factory.
    ComPtr<haptics::IKnownSimpleHapticsControllerWaveformsStatics> base;
    if (FAILED(known_waveforms2_.As(&base)) || !base) return std::nullopt;
    return SUCCEEDED(base->get_Click(&value)) ? std::optional<uint16_t>(value)
                                              : std::nullopt;
  }
  // Mouse, Touchpad, Generic, None -> Hover.
  return SUCCEEDED(known_waveforms2_->get_Hover(&value))
             ? std::optional<uint16_t>(value) : std::nullopt;
}
```

Enumerate supported waveforms (used to decide the primary target):

```cpp
std::vector<uint16_t> GetSupportedWaveforms(
    haptics::IInputHapticsManager* manager) {
  std::vector<uint16_t> waveforms;
  ComPtr<haptics::ISimpleHapticsController> controller;
  if (FAILED(manager->get_CurrentHapticsController(&controller)) || !controller)
    return waveforms;
  ComPtr<collections::IVectorView<haptics::SimpleHapticsControllerFeedback*>>
      feedbacks;
  if (FAILED(controller->get_SupportedFeedback(&feedbacks)) || !feedbacks)
    return waveforms;
  unsigned size = 0;
  feedbacks->get_Size(&size);
  for (unsigned i = 0; i < size; ++i) {
    ComPtr<haptics::ISimpleHapticsControllerFeedback> feedback;
    if (FAILED(feedbacks->GetAt(i, &feedback)) || !feedback) continue;
    UINT16 wf = 0;
    if (SUCCEEDED(feedback->get_Waveform(&wf))) waveforms.push_back(wf);
  }
  return waveforms;
}
```

**Selection strategy inside `PlayHaptics`.** The primary target is the semantic
waveform when the device supports it; the device-type default is passed as the
built-in fallback argument of `TrySendHapticWaveformWithIntensity`, which the OS
substitutes automatically whenever the primary is not in the supported set:

```cpp
std::vector<uint16_t> supported = GetSupportedWaveforms(manager.Get());

auto is_supported = [&](uint16_t w) {
  return std::find(supported.begin(), supported.end(), w) != supported.end();
};

// 1) Preferred: the correct Windows waveform for this effect (Hover/Collide/
//    Step/Align).
std::optional<uint16_t> preferred = WaveformForEffect(effect);

// 2) Device-type default: Mouse/Touchpad -> Hover, Pen -> Click.
std::optional<uint16_t> device_default = DefaultWaveformForDevice(device_type);

// Nothing resolved at all -> nothing to send, return.
if (!preferred && !device_default) return;

// Primary = semantic if the device advertises it, else the device-type default.
uint16_t target = (preferred && is_supported(*preferred))
                      ? *preferred
                      : device_default.value_or(*preferred);

// The TrySend fallback is the device-type default, so the OS still plays a
// platform-appropriate waveform if the primary is not in the supported set.
uint16_t fallback = device_default.value_or(target);

boolean sent = false;
manager->TrySendHapticWaveformWithIntensity(target, fallback, intensity, &sent);
```

So there are two platform-aligned tiers: the **semantic** waveform when the device
supports it, otherwise the **device-type default** passed as the `TrySend` fallback.
If the device supports neither, `TrySend` reports `sent == false` and the effect is
silently dropped.

#### D6.5 Semantic mapping table and the SDK interface gap

Per the explainer, the four effects map to these
`KnownSimpleHapticsControllerWaveforms`:

| Web Haptics effect | Windows waveform | WinRT statics interface |
|:------------------:|:----------------:|:------------------------|
| `hint`  | **Hover**   | `IKnownSimpleHapticsControllerWaveformsStatics2` |
| `edge`  | **Collide** | `IKnownSimpleHapticsControllerWaveformsStatics3` |
| `tick`  | **Step**    | `IKnownSimpleHapticsControllerWaveformsStatics3` |
| `align` | **Align**   | `IKnownSimpleHapticsControllerWaveformsStatics3` |

**The build-time SDK gap.** The Windows SDK bundled with the
Chromium toolchain ships the header for `...Statics2`
(Hover, contract 14.0) but not for `...Statics3` (Collide/Align/Step, contract
19.0), while the code must still compile against that SDK.

**Fix:** Declare `IKnownSimpleHapticsControllerWaveformsStatics3` locally
in the `.cc`, replicating the SDK's IID and vtable layout exactly, and obtain it
at runtime via `QueryInterface` from the same activation-factory object. Hover
uses the SDK-provided `Statics2`. The local declaration is a header workaround
only: at runtime, on any 24H2+ machine where the manager resolves, `Statics3` is
present too.

```cpp
MIDL_INTERFACE("ae480ce4-4ab6-5b2f-ad0b-cb52f37d45fb")
IKnownSimpleHapticsControllerWaveformsStatics3 : public IInspectable {
 public:
  virtual HRESULT STDMETHODCALLTYPE get_Collide(UINT16* value) = 0;
  virtual HRESULT STDMETHODCALLTYPE get_Align(UINT16* value) = 0;
  virtual HRESULT STDMETHODCALLTYPE get_Step(UINT16* value) = 0;
};
```

`WaveformForEffect` then uses `Statics2` directly for Hover, and `QueryInterface`s
the locally-declared `Statics3` for the rest:

```cpp
std::optional<uint16_t> HapticsManagerImplWin::WaveformForEffect(
    blink::mojom::HapticEffect effect) {
  UINT16 value = 0;

  if (effect == blink::mojom::HapticEffect::kHint) {           // Hover
    if (!known_waveforms2_) return std::nullopt;
    return SUCCEEDED(known_waveforms2_->get_Hover(&value))
               ? std::optional<uint16_t>(value) : std::nullopt;
  }

  // Collide/Align/Step live on Statics3, absent from the bundled SDK.
  // QI it from the same factory; succeeds only on UniversalApiContract 19.0+.
  if (!known_waveforms2_) return std::nullopt;
  ComPtr<IKnownSimpleHapticsControllerWaveformsStatics3> waveforms3;
  if (FAILED(known_waveforms2_.As(&waveforms3)) || !waveforms3)
    return std::nullopt;

  HRESULT hr = E_FAIL;
  switch (effect) {
    case blink::mojom::HapticEffect::kEdge:  hr = waveforms3->get_Collide(&value); break;
    case blink::mojom::HapticEffect::kTick:  hr = waveforms3->get_Step(&value);    break;
    case blink::mojom::HapticEffect::kAlign: hr = waveforms3->get_Align(&value);   break;
    case blink::mojom::HapticEffect::kHint:  NOTREACHED();  // handled above
  }
  return SUCCEEDED(hr) ? std::optional<uint16_t>(value) : std::nullopt;
}
```

The `Statics3` QueryInterface runs only on 24H2+, the feature's floor, so it
succeeds whenever the manager itself resolved and all four effects get their true
semantic waveform. The QI failure path is defensive: should it ever fail, the
effect degrades to the device-type default fallback ([D6.4](#d64-waveform-detection-semantic-mapping-and-fallback)).
When Chromium's bundled SDK advances to contract 19.0, the local declaration can be
replaced with the SDK interface with no behavior change.

The device-type default fallback ([D6.4](#d64-waveform-detection-semantic-mapping-and-fallback))
fires on *device capability*, not OS version: on a supported machine a given input
device may simply not advertise a semantic waveform, and the OS substitutes the
device-type default. On systems below the 24H2 floor the backend no-ops entirely
([D6.2](#d62-statics-resolution-ensurestatics)).

> The `kHint: NOTREACHED()` case exists only because Chromium switches over mojo
> enums are exhaustive (no `default:`, `-Wswitch` enforced), so every enumerator
> must appear; `kHint` is already handled by the early return.

#### D6.6 Multiple input device handling

The API's target model (shared with the declarative part) is: **dispatch to the
most recent input device; if it is not haptics-capable, play nothing; never
reroute** to some other connected haptics device.

The Windows backend gets this behavior directly from the platform:
`InputHapticsManager::GetForCurrentThread` plus `get_CurrentHapticsController`
returns the controller for the **most recent** input device on the thread's
input queue. The backend therefore does **not** enumerate devices or pick among
them; it always operates on `CurrentHapticsController`:

* If the most-recent device has no haptics controller,
  `GetSupportedWaveforms` returns empty → the effect is dropped (no reroute).
* If the user switches devices (e.g. from a haptic mouse to a plain one), the
  next call naturally sees the new `CurrentHapticsController`.

Not enumerating devices is also a **privacy** property, it avoids exposing how
many haptic devices exist or which one is active.

### D7. Feature gating and registration

| Concern | Location |
|---|---|
| Runtime flag | `runtime_enabled_features.json5` (`WebHaptics`, status `experimental`) |
| Permissions policy | `permissions_policy_features.json5` (`Haptics` → policy name `haptics`, `depends_on: ["WebHaptics"]`, default allowlist `"self"`) |
| Permissions-policy enum | `permissions_policy_feature.mojom` (`kHaptics`) |
| Use counter | `web_feature.mojom` (`kNavigatorPlayHaptics`) |

The `"self"` default allowlist means same-origin iframes inherit access, while
cross-origin iframes must be explicitly delegated with `allow="haptics"`.

## Declarative API: CSS `@haptic` at-rule

> **Design TBD.**

The second half of the Web Haptics feature is a **declarative** CSS surface: a
nested `@haptic <effect-name> <intensity>?;` at-rule that fires a haptic effect
when its containing style rule transitions from not-matching to matching an
element (see the [explainer](./explainer.md) for the proposed syntax, per-rule
tracking, same-element dedup, and cross-element coalescing semantics). As design
is still being finalized, this section is a placeholder and will be filled in later.

---

# **Metrics**

## **Success metrics**

* **Adoption** via the `kNavigatorPlayHaptics` use counter (page-load
  percentage of sites calling `navigator.playHaptics()`).
* Qualitative 1P & 3P partner / origin-trial reports on whether semantic
  effects feel correct across devices.

## **Regression metrics**

* No regression to input-handling latency on the browser UI thread (the backend
  runs there).

## **Experiments**

Waterfall.

---

# **Rollout plan**

Subject to change based on progress of CSS design and partner feedback.

---

# **Core principle considerations**

## **Speed**

* All WinRT calls run on the **browser UI thread**. Each `PlayHaptics` does a
  bounded amount of work (a few WinRT property reads + one send); the supported-
  waveform enumeration is O(small). This must not add measurable input latency.
* Statics and the primed controller are cached, so steady-state calls avoid
  repeated activation-factory resolution.
* No new work happens for pages that never call the API (lazy Mojo bind).

## **Simplicity**

User-visible surface is minimal: one JS method with four named effects and an
optional intensity. We introduce no new UI, no user decisions, and no permission prompt.
The API deliberately hides raw waveforms so developers express *intent*, not device
tuning.

## **Security**

* **Compromised-renderer assumption.** All gating (fenced-frame, active document,
  `"haptics"` permissions policy, user activation) and intensity clamping are re-enforced in
  the browser by `content::HapticsService` ([D4](#d4-process-routing-and-the-contenthapticsservice-intermediary));
  the renderer's own checks are only a fast path. The effect is a closed mojo enum,
  so a malicious renderer can at worst request a valid effect at a valid intensity
  on the current input device, the same thing a legitimate page can do.
* **No direct hardware path.** The renderer holds only a `blink.mojom.HapticsService`
  pipe to the browser intermediary; the platform WinRT API is touched exclusively
  by browser-process code that ran those checks.
* **Low abuse ceiling.** The effect is transient and ends when the user navigates
  away; throttling and origin suppression remain available as follow-ups.
* **Locally-declared WinRT interface** ([D6.5](#d65-semantic-mapping-table-and-the-sdk-interface-gap)): the hand-written vtable must match
  the SDK exactly. It is only ever obtained via `QueryInterface` (which validates
  the IID) and used to read `UINT16` waveform ids, so no untrusted data crosses
  this boundary.

---

# **Privacy considerations**

The API is designed to **minimize fingerprinting**:

* No API to query haptics-capable hardware, the current device, or the available
  waveform set.
* `playHaptics()` returns `undefined` and exposes **no success signal**, so
  content cannot detect whether a device played or even exists.
* The backend never enumerates devices; it only ever uses the platform's
  "current" controller ([D6.6](#d66-multiple-input-device-handling)).
* No new metrics expose per-user device capability.

Features with privacy implications should complete the internal privacy-review
template; this API's posture is "add no new observable device signal."

---

# **Testing plan**

* **Browser re-check test** (`content/browser/haptics/haptics_service_impl_unittest.cc`
  or a `content_browsertest`): drives `content::HapticsService::PlayHaptics`
  through a fake `RenderFrameHost` and asserts the browser **re-enforces** each
  gate (fenced frame, active document, `"haptics"` permissions policy, and user activation) and
  re-clamps intensity to `[0,1]`, independent of what the renderer claimed. The
  one-way call is observed via `remote.FlushForTesting()` and a fake backend that
  records the last effect/intensity; no hardware needed.
* **Web tests** (follow-up): verify the JS surface exists only when `WebHaptics`
  is enabled, that gating (permissions policy, sticky activation, fenced frames)
  silently no-ops, and that the call returns `undefined`.
* **Manual hardware testing (Windows):**

---

# **Followup work**

* **Declarative CSS `@haptic` API ([Declarative API: CSS `@haptic` at-rule](#declarative-api-css-haptic-at-rule))**: design and implement.
* **Device-service backend + second Mojo pipe ([D5](#d5-device-service-backend-non-windows-platforms))**:
  add a device-service `mojom::HapticsManager` and the browser↔device-service pipe
  from `content::HapticsService`, needed for any backend that must run in the
  device service rather than the browser process.
* **Additional native backends**: Linux/macOS/iOS/Android, each hosted where its
  threading requires: in the browser process behind `content::HapticsService`
  (like `HapticsManagerImplWin`) or in the device service behind the deferred pipe
  above.
* **Remove the local `Statics3` declaration** once Chromium's bundled Windows SDK
  reaches UniversalApiContract 19.0.
