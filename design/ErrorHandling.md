# Error Handling

The simplest design for error handling would be to do it synchronously, for example with Javascript exceptions and object creation returning `null`.
However, this would introduce a lot of synchronization points for multi-threaded/multi-process WebGPU implementations, making it too slow to be useful.

There are a number of cases that developers or applications need error handling for:

 - *Debugging*: Getting errors synchronously during development, to break in to the debugger.
 - *Telemetry*: Collecting error logs in deployment, for bug reporting and telemetry.
 - *Fallible Allocation*: Recovering from recoverable errors (like out-of-memory on resource creation).
 - *Fatal Errors*: Handling device/adapter loss, either by restoring WebGPU or by fallback to non-WebGPU content.

Meanwhile, error handling should not make the API clunky to use.

There are several types of WebGPU calls that get their errors handled differently:

 - *Creation*: WebGPU Object creation.
 - *Encoding*: Recording of GPU commands in `GPUCommandEncoder`.
 - *Operations*: Other API calls, such as `GPUQueue.submit`.

## *Debugging*: Dev Tools

Implementations should provide a way to enable synchronous validation, for example via a debug shim or via the developer tools.
The extra overhead needs to be low enough that applications can still run while being debugged.

The behavior is implementation-defined. For example, it may throw an exception, or it may just cause the debugger to pause inside the call that fails.

## *Fatal Errors*: Lost/Recovered Events

<!-- calling this revision 5 -->

The `GPUAdapter` and `GPUDevice` are event targets which receive events about adapter and device status.

```webidl
partial interface GPUAdapter {
    Promise<GPUDevice> requestDevice(GPUDeviceDescriptor descriptor,
                                     GPUDeviceLostCallback onDeviceLost);
};

callback GPUDeviceLostCallback = void (GPUDeviceLostInfo info);

interface GPUDeviceLostInfo {
    readonly attribute GPUDevice device;
    readonly attribute DOMString reason;
};
```

`GPUAdapter.requestDevice` requests a device from the adapter.
It returns a Promise which resolves when a device is ready.
The Promise may not resolve for a long time - it resolves when the browser is ready for the application to bring up (or restore) its content.
If the adapter is lost before the Promise resolves, the Promise rejects.

`requestDevice` takes a required `onDeviceLost` callback.
When it is called, the `GPUDevice` cannot be used anymore.
The device and all objects created from the device have become invalid.
All further operations on the device and its objects are errors.
The `"gpulogentry"` event will no longer fire. (This makes all further operations no-ops.)
(TODO: this could probably be an `ondevicelost` event instead.
However, this (1) makes it required and (2) makes it clear that no error can be missed between creation and event listener registration.)

`onDeviceLost` may be called if something goes fatally wrong on the device (e.g. unexpected out-of-memory, crash, or native device loss).
When the application receives this callback, it may immediately request a new device.

### Example Code

```js
class MyRenderer {
  constructor() {
    this.rendering = false;
    this.adapter = null;
    this.device = null;
  }

  async begin() {
    try {
      await initWebGPU();
    } catch (e) {
      console.error(e);
      initFallback();
    }
  }

  async initWebGPU() {
    await ensureDevice();
    // ... Upload resources, etc.
  }

  initFallback() { /* ... */ }

  async ensureDevice() {
    // Stop rendering. (If there was already a device, WebGPU calls made before
    // the app notices the device is lost are okay - they are no-ops.)
    this.rendering = false;

    if (!this.adapter) {
      // If no adapter, get one.
      // (If requestAdapter rejects, no matching adapter is available. Exit to fallback.)
      this.adapter = await gpu.requestAdapter({ /* options */ });
    }

    try {
      this.device = await adapter.requestDevice({ /* options */ }, (info) => {
        info.device.isLost = true; // in case the application wants to know
        // Device was lost.
        console.error("device lost", info);
        // Try to get a device again.
        ensureDevice();
      });
      this.rendering = true;
    } catch (e) {
      // Request failed (likely due to adapter loss).
      console.error("device request failed", e);
      // Try again with a new adapter.
      this.adapter = null;
      await ensureDevice()
    }
  }
}
```

### Case Studies

*What signals should the app get, and when?*

Two independent applications are running on the same webpage against two devices on the same adapter.
The tab is in the background, and one device is using a lot of resources.
 - The browser chooses to lose the heavier device.
    - `onDeviceLost`, reason = recovering device resources
    - (App calls `createDevice` on any adapter, but it doesn't resolve yet.)
 - Later, the browser might choose to lose the smaller device too.
    - `onDeviceLost`, reason = recovering device resources
    - (App calls `createDevice` on any adapter, but it doesn't resolve yet.)
 - Later, the tab is brought to the foreground. 
    - Both `createDevice` Promises resolve.
      (Unless the adapter was lost, in which case they would have rejected.)

A page begins loading in a tab, but then the tab is backgrounded.
 - On load, the page attempts creation of a device.
    - `createDevice` Promise will resolve.

A device's adapter is physically unplugged from the system (but an integrated GPU is still available).
 - The same adapter, or a new adapter, is plugged back in.
    - A later `requestAdapters` call may return the new adapter. TODO: `"gpuadapterschanged"`?

An app is running on an integrated adapter.
 - A new, discrete adapter is plugged in.
    - A later `requestAdapters` call may return the new adapter. TODO: `"gpuadapterschanged"`?

An app is running on a discrete adapter.
 - The adapter is physically unplugged from the system. An integrated GPU is still available.
    - `onDeviceLost` is called, `requestDevice` on same adapter rejects, `requestAdapters` gives the new adapter.
 - The same adapter, or a new adapter, is plugged back in.
    - A later `requestAdapters` call may return the new adapter. TODO: `"gpuadapterschanged"`?

The device is lost because of an unexpected error in the implementation.
 - `onDeviceLost`, reason = whatever the unexpected thing was.

A TDR-like scenario occurs.
 - The adapter is lost, which loses all devices on the adapter.
   `onDeviceLost` on every device, reason = adapter reset. Application must request adapter again.
 - (TODO: alternatively, adapter could be retained, but all devices on it are lost.)

All devices and adapters are lost (except for software?) because GPU access has been disabled by the browser (for this page or globally).
 - `onDeviceLost` on every device, reason = whatever

WebGPU access has been disabled for the page.
 - `requestAdapters` rejects (or returns a software adapter).

The device is lost right as it's being returned by requestDevice.
 - `onDeviceLost`.

## *Telemetry*: Error Logging

```webidl
interface GPULogEntryEvent : Event {
    readonly attribute any object;
    readonly attribute DOMString reason;
};

partial interface GPUDevice : EventTarget {};
```

`WebGPUDevice` `"gpulogentry"` -> `GPULogEntryEvent`:
Fires when a non-fatal error occurs in the API, i.e. a validation error (including if an "invalid" object is used).

When there is a validation error in the API (including operations on "invalid" WebGPU objects), an error is logged.
When a validation error is discovered by the WebGPU implementation, it may fire `"gpulogentry"` event on the `GPUDevice`.
These events should not be used by applications to recover from expected, recoverable errors.
Instead, the error log may be used for handling unexpected errors in deployment, for bug reporting and telemetry.

`"gpulogentry"` events are all delivered through the device.
They are not directly associated with the objects or operations that produced them.

The `"gpulogentry"` event always fires on the event loop where the error occurred.
Event listeners must be set per event loop (i.e. on the main thread and on each worker).
(TODO: This is necessary because of the `object` field.
But it seems nicer to always fire on the main thread.
Should the object be removed?
Its debug label can be included in the `reason` instead.)

For creation errors, the `object` attribute holds the object handle that was created.
(It will always point to an "invalid" object.)
It preserves the JavaScript object wrapper of that handle (including any extra JavaScript properties attached to that wrapper).

The `reason` is a human-readable string, provided for debugging/reporting/telemetry.

The WebGPU implementation may choose not to fire the `"gpulogentry"` event for a given log entry if there have been too many errors, too many errors in a row, or too many errors of the same kind.
(In badly-formed applications, this mechanism can prevent the `"gpulogentry"` events from having a significant performance impact on the system.)

No `"gpulogentry"` events will be fired after the device is lost.
(Though a there may be one "just before" the device is lost, if the error would be useful for telemetry.)

## Object Creation

WebGPU objects are opaque handles.
On creation, such a handle is "pending" until the backing object is created by the implementation.
After that, a handle may refer to a successfully created object (called a "valid object"), or an error that occured during creation (called an "invalid object").

When a WebGPU object handle is passed to an operation, the object will resolve (to "valid" or "invalid") before it is actually used by that operation.

### Error propagation of invalid objects

Using any invalid object in a WebGPU operation produces a validation error.
The effect of an error depends on the type of a call:

 - For object creation, the call produces a new invalid object.
 - For `GPUCommandEncoder` encoding methods, the `GPUCommandEncoder.finishEncoding` method will return an invalid object.
 - For other WebGPU calls, the call becomes a no-op.

In each case, an error is logged to the error log.

## *Fallible Allocation*: Out-of-memory in object creation

Recoverable fallible allocations are exposed as Promise versions of the `createBuffer`/`createTexture` entry points.

```webidl
partial interface GPUDevice {
    Promise<GPUBuffer> tryCreateBuffer(GPUBufferDescriptor descriptor);
    Promise<GPUTexture> tryCreateTexture(GPUTextureDescriptor descriptor);
};
```

If an application wants to allocate with fatal out-of-memory, it uses createBuffer/createTexture.
Just like with any the creation of smaller objects, an out-of-memory condition will be treated as a fatal error: the device is lost.

The `tryCreate*` entry points return Promises.
 - If creation succeeds, the Promise resolves to a *valid* object.
 - If there is a validation error, the Promise resolves to an *invalid* object (and produces a log entry).
 - If the device is lost, the Promise resolves to an *invalid* object.
 - If the resource allocation runs out of memory, the Promise rejects.
    - (The application can assume that rejection *always* means it should recover.)

 ### Relation to `createReady*Pipeline`

TODO: https://github.com/gpuweb/gpuweb/pull/184#issuecomment-458377539

`tryCreate*` have some similarity to `createReady*Pipeline`, which also return a `Promise`.
The resolve(invalid), resolve(valid), and reject cases should match as much as possible between these two pairs of functions.

## Open Questions and Considerations

 - Should developers be able to self-impose a memory limit (in order to emulate lower-memory devices)?
   Should implementations automatically impose a lower memory limit (to improve portability)?

 - To help developers, `GPULogEntry.reason` could contain some sort of "stack trace" and could take advantage of debug name of objects if that's something that's present in WebGPU.
   For example:

   ```
   Failed <myQueue>.submit because commands[0] (<mainColorPass>) is invalid:
   - <mainColorPass> is invalid because in setIndexBuffer, indexBuffer (<mesh3.indexBuffer>) is invalid
   - <mesh3.indexBuffer> is invalid because it got an unsupported usage flag (0x89)
   ```

 - How do applications handle the case where they've allocated a lot of optional memory, but want to make another required allocation (which could fail due to OOM)?
   How do they know when to free an optional allocation first?
    - (We will likely solve this with `GPUResourceHeap`, once we figure out what that looks like.)

## Resolved Questions
   
 - Should there be a mode/flag which causes OOM errors to trigger context loss?
    - Resolved: Not necessary, since an application can manually destroy the context based on entries in the error log.

 - In a world with persistent object "usage" state:
   If an invalid command buffer is submitted, and its transitions becomes no-ops, the usage state won't update.
   Will this cause future command buffer submits to become invalid because of a usage validation error?
    - Tentatively resolved: WebGPU is expected not to require explicit usage transitions.

 - Should an object creation error immediately log an error to the error log?
   Or should it only log if the error propagates to a device-level operation?
    - Tentatively resolved: errors should be logged immediately.

 - Should applications be able to intentionally create graphs of potentially-invalid objects, and recover from this late?
   E.g. create a large buffer, create a bind group from that, create a command buffer from that, then choose whether to submit based on whether the buffer was successfully allocated.
    - If yes, `tryCreateBuffer` must return `GPUBuffer` and error log entries must not be generated when creating objects from invalid objects.
      (Only log errors on queue.submit and other device/queue level operations.)
    - If no, `tryCreateBuffer` should return `Promise<GPUBuffer>` and error log entries should be generated when creating objects from invalid objects.
    - Resolved: no.
