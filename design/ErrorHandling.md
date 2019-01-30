# Error Handling

The simplest design for error handling would be to do it synchronously, for example with Javascript exceptions and object creation returning `null`.
However, this would introduce a lot of synchronization points for multi-threaded/multi-process WebGPU implementations, making it too slow to be useful.

There are a number of cases that developers or applications need error handling for:

 - *Debugging*: Getting errors synchronously during development, to break in to the debugger.
 - *Telemetry*: Collecting error logs in deployment, for bug reporting and telemetry.
 - *Fallible Allocation*: Recovering from recoverable errors (like out-of-memory on resource creation).
 - *Fallback*: Tearing down the application and falling back, e.g. to WebGL, 2D Canvas, or static content.

Meanwhile, error handling should not make the API clunky to use.

There are several types of WebGPU calls that get their errors handled differently:

 - *Creation*: WebGPU Object creation.
 - *Encoding*: Recording of GPU commands in `GPUCommandEncoder`.
 - *Operations*: Other API calls, such as `GPUQueue.submit`.

## *Debugging*: Dev Tools

Implementations should provide a way to enable synchronous validation, for example via a debug shim or via the developer tools.
The extra overhead needs to be low enough that applications can still run while being debugged.

## *Telemetry*: Error Logging

```webidl
interface GPULogEntryEvent : Event {
    readonly attribute any object;
    readonly attribute DOMString reason;
};
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

TODO: https://github.com/gpuweb/gpuweb/pull/184#issuecomment-458377539

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

## Open Questions and Considerations

 - WebGPU could guarantee that objects such as `GPUQueue` and `GPUFence` can never be errors.
   If this is true, then the only synchronous API that needs special casing is buffer mapping, where `mapping` is always `null` for an invalid `GPUBuffer`.

 - Should developers be able to self-impose a memory limit (in order to emulate lower-memory devices)?
   Should implementations automatically impose a lower memory limit (to improve portability)?

 - To help developers, `GPULogEntry.reason` could contain some sort of "stack trace" and could take advantage of debug name of objects if that's something that's present in WebGPU.
   For example:

   ```
   Failed <myQueue>.submit because commands[0] (<mainColorPass>) is invalid:
   - <mainColorPass> is invalid because in setIndexBuffer, indexBuffer (<mesh3.indexBuffer>) is invalid
   - <mesh3.indexBuffer> is invalid because it got an unsupported usage flag (0x89)
   ```

 - Should applications be able to intentionally create graphs of potentially-invalid objects, and recover from this late?
   E.g. create a large buffer, create a bind group from that, create a command buffer from that, then choose whether to submit based on whether the buffer was successfully allocated.
    - If yes, `tryCreateBuffer` must return `GPUBuffer` and error log entries must not be generated when creating objects from invalid objects.
      (Only log errors on queue.submit and other device/queue level operations.)
    - If no, `tryCreateBuffer` should return `Promise<GPUBuffer>` and error log entries should be generated when creating objects from invalid objects.

 - How do applications handle the case where they've allocated a lot of optional memory, but want to make another required allocation (which could fail due to OOM)?
   How do they know when to free an optional allocation first?

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
