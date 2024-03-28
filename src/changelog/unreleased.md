The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

The sections should follow the order `Added`, `Changed`, `Deprecated`,
`Removed`, and `Fixed`.

Platform specific changed should be added to the end of the section and grouped
by platform name. Common API additions should have `, implemented` at the end
for platforms where the API was initially implemented. See the following example
on how to add them:

```md
### Added

- Add `Window::turbo()`, implemented on X11, Wayland, and Web.
- On X11, add `Window::some_rare_api`
- On X11, add `Window::even_more_rare_api`
- On Wayland, add `Window::common_api`
- On Windows, add `Window::some_rare_api`
```

When the change requires non-trivial amount of work for users to comply
with it, the migration guide should be added below the entry, like:

```md
- Deprecate `Window` creation outside of `EventLoop::run`

This was done to simply migration in the future. Consider the
following code:

// Code snippet.


To migrate it we should do X, Y, and then Z, for example:

// Code snippet.

```

The migration guide could reference other migration examples in the current
changelog entry.

## Unreleased

### Added

- Add `OwnedDisplayHandle` type for allowing safe display handle usage outside of
  trivial cases
- Add `ApplicationHandler<T>` trait which mimics `Event<T>`
- Add `WindowBuilder::with_cursor` and `Window::set_cursor` which takes a
  Add `CursorIcon` or `CustomCursor`
- Add `Sync` implementation for `EventLoopProxy<T: Send>`
- Add `Window::default_attributes` to get default `WindowAttributes`
- Add `EventLoop::builder` to get `EventLoopBuilder` without export
- Add `CustomCursor::from_rgba` to allow creating cursor images from RGBA data
- Add `CustomCursorExtWebSys::from_url` to allow loading cursor images from URLs
- Add `CustomCursorExtWebSys::from_animation` to allow creating animated cursors
  Add from other `CustomCursor`s
- Add `{Active,}EventLoop::create_custom_cursor` to load custom cursor image sources
- Add `ActiveEventLoop::create_window` and `EventLoop::create_window`
- Add `CustomCursor` which could be set via `Window::set_cursor`, implemented on
  Windows, macOS, X11, Wayland, and Web
- On Web, add to toggle calling `Event.preventDefault()` on `Window`
- On iOS, add `PinchGesture`, `DoubleTapGesture`, and `RotationGesture`
- On macOS, add services menu.
- On Windows, add `with_title_text_color`, and `with_corner_preference` on
  `WindowAttributesExtWindows`

### Changed

- Bump MSRV from `1.65` to `1.70`.
- Rename `TouchpadMagnify` to `PinchGesture`
- Rename `SmartMagnify` to `DoubleTapGesture`
- Rename `TouchpadRotate` to `RotationGesture`
- Rename `EventLoopWindowTarget` to `ActiveEventLoop`.
- Rename `platform::x11::XWindowType` to `platform::x11::WindowType`.
- Rename `VideoMode` to `VideoModeHandle` to represent that it doesn't hold
  static data.
- Move `dpi` types to its own crate, and re-export it from the root crate
- Replace `log` with `tracing`, use `log` feature on `tracing` to restore old
  behavior
- `EventLoop::with_user_event` now returns `EventLoopBuilder`
- On Web, return `HandleError::Unavailable` when a window handle is not available
- On Web, return `RawWindowHandle::WebCanvas` instead of `RawWindowHandle::Web`
- On Web, remove queuing fullscreen request in absence of transient activation
- On iOS, return `HandleError::Unavailable` when a window handle is not available
- On macOS, return `HandleError::Unavailable` when a window handle is not available
- On Windows, remove `WS_CAPTION`, `WS_BORDER`, and `WS_EX_WINDOWEDGE` styles
  for child windows without decorations

### Deprecated

- Deprecate `EventLoop::run`, use `EventLoop::run_app`
- Deprecate `EventLoopExtRunOnDemand::run_on_demand`, use `EventLoop::run_app_on_demand`
- Deprecate `EventLoopExtPumpEvents::pump_events`, use `EventLoopExtPumpEvents::pump_app_events`

The new `app` APIs accept a newly added `ApplicationHandler<T>` instead of
`Fn`. The semantics are mostly the same given that capture list is your new
`State`. Consider the following code:

```rust,no_run
use winit::event::Event;
use winit::event_loop::EventLoop;
use winit::window::Window;

struct MyUserEvent;

let event_loop = EventLoop::<MyUserEvent>::with_user_event().build().unwrap();
let window = event_loop.create_window(Window::default_attributes()).unwrap();
let mut counter = 0;

let _ = event_loop.run(move |event, event_loop| {
    match event {
        Event::AboutToWait => {
            window.request_redraw();
            counter += 1;
        }
        Event::WindowEvent { window_id, event } => {
            // Handle window event.
        }
        Event::UserEvent(event) => {
            // Handle user event.
        }
        Event::DeviceEvent { device_id, event } => {
            // Handle device event.
        }
        _ => (),
    }
});
```

To migrate this code, we should move all the captured values into some `State`
and implement `ApplicationHandler` for this state, then we just move particular
`match event` arms into methods on `ApplicationHandler` , for example:

```rust,no_run
use winit::application::ApplicationHandler;
use winit::event::{Event, WindowEvent, DeviceEvent, DeviceId};
use winit::event_loop::{EventLoop, ActiveEventLoop};
use winit::window::{Window, WindowId};

struct MyUserEvent;

struct State {
    window: Window,
    counter: i32,
}

impl ApplicationHandler<MyUserEvent> for State {
    fn user_event(&mut self, event_loop: &ActiveEventLoop, user_event: MyUserEvent) {
        // Handle user event.
    }

    fn resumed(&mut self, event_loop: &ActiveEventLoop) {
        // Your application got resumed.
    }

    fn window_event(&mut self, event_loop: &ActiveEventLoop, window_id: WindowId, event: WindowEvent) {
        // Handle window event.
    }

    fn device_event(&mut self, event_loop: &ActiveEventLoop, device_id: DeviceId, event: DeviceEvent) {
        // Handle window event.
    }

    fn about_to_wait(&mut self, event_loop: &ActiveEventLoop) {
        self.window.request_redraw();
        self.counter += 1;
    }
}

let event_loop = EventLoop::<MyUserEvent>::with_user_event().build().unwrap();
#[allow(deprecated)]
let window = event_loop.create_window(Window::default_attributes()).unwrap();
let mut state = State { window, counter: 0 };

let _ = event_loop.run_app(&mut state);
```

That should be mostly it, it's just a matter of moving code around.

- Deprecate `EventLoop::create_window`, use `ActiveEventLoop::create_window`

Creating the window without active loop has unexpected issues with mostly all
backends, thus it was decided to require a running loop. This change wasn't
strictly necessary for the current release, but in the future it'll be used to
ensure soundness, better asynchronous startup modeling, and calling back during
window creation to get extra information from the user when this information is
actually _needed_.

Using the migration example from above we can change our code like that:

```rust,no_run
use winit::application::ApplicationHandler;
use winit::event::{Event, WindowEvent, DeviceEvent, DeviceId};
use winit::event_loop::{EventLoop, ActiveEventLoop};
use winit::window::{Window, WindowId};

#[derive(Default)]
struct State {
    window: Option<Window>,
    counter: i32,
}

impl ApplicationHandler for State {
    // This is a common indicator that you can create a window.
    fn resumed(&mut self, event_loop: &ActiveEventLoop) {
      self.window = Some(event_loop.create_window(Window::default_attributes()).unwrap());
    }
    fn window_event(&mut self, event_loop: &ActiveEventLoop, window_id: WindowId, event: WindowEvent) {
        // Handle window event.
    }
    fn device_event(&mut self, event_loop: &ActiveEventLoop, device_id: DeviceId, event: DeviceEvent) {
        // Handle window event.
    }
    fn about_to_wait(&mut self, event_loop: &ActiveEventLoop) {
        if let Some(window) = self.window.as_ref() {
          window.request_redraw();
          self.counter += 1;
        }
    }
}

let event_loop = EventLoop::new().unwrap();
let mut state = State::default();
let _ = event_loop.run_app(&mut state);
```

- Deprecate `Window::set_cursor_icon`, use `Window::set_cursor`

### Removed

- Remove `Window::new`, use `ActiveEventLoop::create_window` and `EventLoop::create_window`
- Remove `Deref` implementation for `EventLoop` that gave `EventLoopWindowTarget`
- Remove `WindowBuilder` in favor of `WindowAttributes`
- Remove Generic parameter `T` from `ActiveEventLoop`
- Remove `EventLoopBuilder::with_user_event`, use `EventLoop::with_user_event`
- Remove Redundant `EventLoopError::AlreadyRunning`
- Remove `WindowAttributes::fullscreen` and expose as field directly
- On X11, remove `platform::x11::XNotSupported` export

### Fixed

- On Web, fix setting cursor icon overriding cursor visibility
- On Windows, fix cursor not confined to center of window when grabbed and hidden
- On macOS, fix sequence of mouse events being out of order when dragging on the trackpad
