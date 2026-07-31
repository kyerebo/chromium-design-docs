# **Web Haptics API Design Doc**

## This Document is Public

*Authors: [akyereboah@microsoft.com](mailto:akyereboah@microsoft.com)*
*July 2026*

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

531787872

### **Code affected**

Blink bindings & modules (`third_party/blink/renderer/modules/haptics/`),
device service mojom & default backend (`services/device/`), browser-process
Windows backend and interface binder (`content/browser/`), permissions-policy
and runtime-flag registration.

---

# **Design**

*The Web Haptics feature is split into the two API parts described above. All
current design work is in **imperative API**.
**Declarative CSS** is a placeholder.*

## Imperative API: `navigator.playHaptics()`

### D0. Background and end-to-end data flow

`navigator.playHaptics(effect, intensity)` is a call that returns `undefined`.
The renderer performs all policy gating, then forwards the request
over a Mojo interface to a browser/device-service backend that talks to the
platform haptics API.

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
    • sticky user-activation check
    • clamp intensity to [0,1]

        │  device.mojom.HapticsManager::PlayHaptics(effect, intensity)
        ▼  (Mojo IPC)
        
Browser process  (content/browser/browser_interface_binders.cc)
  #if IS_WIN → HapticsManagerImplWin   (runs on the UI thread)
  #else      → device service → HapticsManagerImpl (no-op base)
        │
        ▼
Windows.Devices.Haptics.InputHapticsManager (WinRT API)
  → most-recent input device's SimpleHapticsController → hardware vibration
```

### D1. JS surface: IDL, effect enum, runtime flag

The API is a partial `Navigator` interface, gated by the `WebHaptics` runtime
flag and implemented by `HapticsController`.

`third_party/blink/renderer/modules/haptics/navigator_haptics.idl`:

```webidl
// The predefined, semantic haptic effect names.
enum HapticEffect { "hint", "edge", "tick", "align" };

[
    ImplementedAs=HapticsController,
    RuntimeEnabled=WebHaptics
] partial interface Navigator {
    undefined playHaptics(HapticEffect effect, optional double intensity = 1);
};
```

Because `playHaptics()` returns `undefined` and exposes no success signal, web
  content cannot observe whether a device actually played — this is deliberate
  (see [Privacy](#privacy-considerations)).

### D2. Blink renderer: `HapticsController`

`HapticsController` is a `Supplement<Navigator>` that enforces the API's gating
rules and forwards to the browser over Mojo.

`third_party/blink/renderer/modules/haptics/haptics_controller.cc`:

```cpp
void HapticsController::PlayHaptics(const V8HapticEffect& effect,
                                    double intensity) {
  LocalDOMWindow* window = GetSupplementable()->DomWindow();
  if (!window) return;
  UseCounter::Count(window, WebFeature::kNavigatorPlayHaptics);

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

  if (!haptics_manager_.is_bound()) {
    window->GetBrowserInterfaceBroker().GetInterface(
        haptics_manager_.BindNewPipeAndPassReceiver(
            window->GetTaskRunner(TaskType::kMiscPlatformAPI)));
  }
  haptics_manager_->PlayHaptics(ToMojoEffect(effect), intensity,
                                base::DoNothing());
}
```

* **Gating order**: fenced-frame → permissions policy → sticky activation.
Every early-return silently ignores the call, avoiding a fingerprinting vector.
* **Lazy remote**: the `HeapMojoRemote<HapticsManager>` is bound on first use and
  reset in `ContextDestroyed()`.
* **Enum translation**: `ToMojoEffect()` converts the V8 enum to the mojo enum
  1:1 (`kHint`/`kEdge`/`kTick`/`kAlign`).

### D3. Mojo interface: `device.mojom.HapticsManager`

`services/device/public/mojom/haptics_manager.mojom`:

```mojom
module device.mojom;

enum HapticEffect { kHint, kEdge, kTick, kAlign };

interface HapticsManager {
  // Plays |effect| at |intensity| (normalized [0.0, 1.0]) on the most recent
  // input device. No success signal is exposed to avoid fingerprinting.
  PlayHaptics(HapticEffect effect, double intensity) => ();
};
```

The empty `=> ()` reply is intentional to let the renderer know the request was
processed without leaking whether hardware actually played.

### D4. Process routing: browser interface binder

The interface is added to the per-frame binder map. Windows routes to the
browser-process backend; every other platform defers to the device service.

`content/browser/browser_interface_binders.cc`:

```cpp
map->Add<device::mojom::HapticsManager>(
    [](RenderFrameHost* host,
       mojo::PendingReceiver<device::mojom::HapticsManager> receiver) {
#if BUILDFLAG(IS_WIN)
      HapticsManagerImplWin::Create(std::move(receiver));
#else
      GetDeviceService().BindHapticsManager(std::move(receiver));
#endif
    });
```

**Why the browser UI thread on Windows.** `InputHapticsManager::GetForCurrentThread`
returns the haptics manager for the input queue of the *current* thread that owns
the top-level `HWND` receiving pointer input. In Chromium that is the browser
**UI thread**. Running the backend in the device service utility
process (a different thread/queue) would not see the user's input device. Hence
the Windows backend lives in `content/browser/haptics/` and is created directly
on the UI thread, bypassing the device service.

### D5. Device-service default backend (no-op base)

Non-Windows platforms bind the cross-platform base. It is also the class a future
native backend (mac/Android/etc.) would subclass by overriding
`PlatformPlayHaptics`.

`services/device/haptics/haptics_manager_impl.{h,cc}`:

```cpp
class HapticsManagerImpl : public mojom::HapticsManager {
 public:
  static void Create(mojo::PendingReceiver<mojom::HapticsManager> receiver);
  void PlayHaptics(mojom::HapticEffect effect, double intensity,
                   PlayHapticsCallback callback) override;
 protected:
  // Default does nothing; platform subclasses override.
  virtual void PlatformPlayHaptics(mojom::HapticEffect effect,
                                   double intensity);
};

void HapticsManagerImpl::PlayHaptics(mojom::HapticEffect effect,
                                     double intensity,
                                     PlayHapticsCallback callback) {
  intensity = std::clamp(intensity, 0.0, 1.0);  // defense in depth
  PlatformPlayHaptics(effect, intensity);
  last_effect_for_testing_ = effect;
  last_intensity_for_testing_ = intensity;
  std::move(callback).Run();
}
```

### D6. Windows backend: `HapticsManagerImplWin`

`content/browser/haptics/haptics_manager_impl_win.{h,cc}` is the real
implementation, using the `Windows.Devices.Haptics.InputHapticsManager` API.

#### D6.1 Threading model and receiver ownership

The receiver is bound and dispatched on the UI thread. The instance is
self-owned (its lifetime is tied to the Mojo pipe), and a `SEQUENCE_CHECKER`
guards every method:

```cpp
// static
void HapticsManagerImplWin::Create(
    mojo::PendingReceiver<device::mojom::HapticsManager> receiver) {
  DCHECK_CURRENTLY_ON(BrowserThread::UI);
  mojo::MakeSelfOwnedReceiver(std::make_unique<HapticsManagerImplWin>(),
                              std::move(receiver));
}
```

All WinRT calls happen on the one thread that owns the input queue.

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
//    Step/Align). May be null on a pre-19.0 OS where Collide/Step/Align are
//    unavailable (see D6.5).
std::optional<uint16_t> preferred = WaveformForEffect(effect);

// 2) Device-type default: Mouse/Touchpad -> Hover, Pen -> Click.
std::optional<uint16_t> device_default = DefaultWaveformForDevice(device_type);

// Nothing resolved at all -> drop (callback still runs).
if (!preferred && !device_default) { std::move(callback).Run(); return; }

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

So there are two platform-aligned tiers:
1. **Semantic**: the correct Windows waveform for the effect, when the device
   supports it.
2. **Device-type default**: Mouse/Touchpad → Hover, Pen → Click, passed as the
   `TrySend` fallback so the OS substitutes it automatically when the semantic
   waveform is unsupported.

If the device supports neither the semantic waveform nor its device-type default,
`TrySend` reports `sent == false` and the effect is dropped (the callback still
runs).

#### D6.5 Semantic mapping table and the SDK interface gap

Per the explainer, the four effects map to these
`KnownSimpleHapticsControllerWaveforms`:

| Web Haptics effect | Windows waveform | WinRT statics interface |
|:------------------:|:----------------:|:------------------------|
| `hint`  | **Hover**   | `IKnownSimpleHapticsControllerWaveformsStatics2` |
| `edge`  | **Collide** | `IKnownSimpleHapticsControllerWaveformsStatics3` |
| `tick`  | **Step**    | `IKnownSimpleHapticsControllerWaveformsStatics3` |
| `align` | **Align**   | `IKnownSimpleHapticsControllerWaveformsStatics3` |

**The SDK gap.** These named waveforms live on *versioned* interfaces of the same
runtime class, gated by `WINDOWS_FOUNDATION_UNIVERSALAPICONTRACT_VERSION`:

* `...Statics2` (Hover) requires contract **14.0**, **present** in the Windows
  SDK bundled with the Chromium toolchain.
* `...Statics3` (Collide/Align/Step) requires contract **19.0**, **absent** from
  the bundled toolchain SDK.

**Fix:** Declare `IKnownSimpleHapticsControllerWaveformsStatics3` locally
in the `.cc`, replicating the SDK's IID and vtable layout exactly, and obtain it
at runtime via `QueryInterface` from the same activation-factory object. Hover
continues to use the SDK-provided `Statics2`.

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
    device::mojom::HapticEffect effect) {
  UINT16 value = 0;

  if (effect == device::mojom::HapticEffect::kHint) {           // Hover
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
    case device::mojom::HapticEffect::kEdge:  hr = waveforms3->get_Collide(&value); break;
    case device::mojom::HapticEffect::kTick:  hr = waveforms3->get_Step(&value);    break;
    case device::mojom::HapticEffect::kAlign: hr = waveforms3->get_Align(&value);   break;
    case device::mojom::HapticEffect::kHint:  NOTREACHED();  // handled above
  }
  return SUCCEEDED(hr) ? std::optional<uint16_t>(value) : std::nullopt;
}
```

On a pre-19.0 OS the `QueryInterface` fails and the effect degrades to the
device-type default fallback ([D6.4](#d64-per-device-waveform-detection-semantic-mapping-and-fallback)). When Chromium's bundled SDK advances to
contract 19.0, the local declaration can be deleted and replaced with the SDK
interface with no behavior change.

These contract levels map to specific OS builds.
`Statics2` (Hover, contract 14.0) is present on essentially all supported
Windows 11 builds, so `hint` gets its true semantic waveform broadly.
`Statics3` (Collide/Step/Align, contract 19.0) only arrived
with Windows 11 **24H2** (Oct 2024), so `edge`/`tick`/`align` get their true
semantic waveform only on 24H2-or-newer machines; on older Windows 11 builds
(21H2/22H2/23H2) and on Windows 10 they degrade to the device-type default
fallback ([D6.4](#d64-per-device-waveform-detection-semantic-mapping-and-fallback)). This is why `Statics3` is best-effort and runtime-probed rather than
assumed from the OS family.

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
them — it always operates on `CurrentHapticsController`:

* If the most-recent device has no haptics controller,
  `GetSupportedWaveforms` returns empty → the effect is dropped (no reroute).
* If the user switches devices (e.g. from a haptic mouse to a plain one), the
  next call naturally sees the new `CurrentHapticsController`.

Not enumerating devices is also a **privacy** property: it avoids exposing how
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

## Part B — Declarative API: CSS `@haptic` at-rule

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

* **Compromised-renderer assumption.** The browser trusts nothing from the
  renderer: intensity is re-clamped in the browser/device-service backend, and
  the effect enum is a closed mojo enum. A malicious renderer can at worst
  request a valid effect at a valid intensity on the current input device — the
  same thing a legitimate page can do.
* **Gating** (fenced-frame suppression, `"haptics"` permissions policy, sticky
  user activation) is enforced in the renderer; the effect is transient and the
  user can end it by navigating away, so the abuse ceiling is low. Anti-abuse
  tools (throttling, origin suppression) remain available as follow-ups.
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

* **Unit test** (`services/device/haptics/haptics_manager_impl_unittest.cc`):
  binds the device-service `HapticsManager` and asserts the effect/intensity are
  received and that intensity is clamped to `[0,1]` — via the static
  `last_*_for_testing_` hooks, no hardware needed.
* **Web tests** (follow-up): verify the JS surface exists only when `WebHaptics`
  is enabled, that gating (permissions policy, sticky activation, fenced frames)
  silently no-ops, and that the call returns `undefined`.
* **Manual hardware testing (Windows):**

---

# **Followup work**

* **Declarative CSS `@haptic` API ([Part B](#part-b--declarative-api-css-haptic-at-rule))** — design and implement.
* **Additional native backends** — Linux/macOS/iOS/Android by subclassing
  `HapticsManagerImpl::PlatformPlayHaptics`.
* **Remove the local `Statics3` declaration** once Chromium's bundled Windows SDK
  reaches UniversalApiContract 19.0.
