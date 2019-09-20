A GPUSurface may be allocated in such a way that it is optimized for display to the canvas of the
GPUPresentContext it was created from.

Such optimizations are *unobservable*. In particular, surfaces created from a canvas can be used
in any order, or .destroy()ed instead of presented.

## DXGISwapChain

DXGISwapChain is the most restrictive presentation API across the platforms I investigated.
(Notably, DirectComposition is much less restrictive.)
DXGISwapChain does not provide guarantees about when its contents appear on the screen,
so it can only be used for something like WebGL's `{desynchronized: true}`.

The above rules allow the surface to point at an image in an `IDXGISwapChain` owned by that canvas.
Internally, any necessary copies-on-write or moves-on-write will occur.

On D3D12/DXGI, with a canvas that has had control transferred to an `OffscreenCanvas`,
and a `GPUPresentContext` `ctx` created from it:

- `ctx` may own a DXGISwapChain. The DXGISwapChain has a fixed set of buffers inside which we can allocate.
  Only one is available for the implementation to use at any given time.
- A `GPUSurface` could be allocated inside a swap chain buffer.

If this is done, then:

- If a GPUSurface `s` is created, it can be allocated inside a DXGISwapChain, allowing it to be
  presented using IDXGISwapChain::Present.
  (`s` becomes invalid, so no further access is possible.)
- If two GPUSurfaces `s1` and `s2` are allocated without presenting, `s1` is allocated inside
  the swap chain and `s2` is allocated elsewhere.
- If `s2` is presented, `s1` is relocated (moved-by-copy) out of the swap chain, and `s2` is
  relocated into the swap chain.
- Destroying a GPUSurface frees up the space in the swap chain, so future GPUSurface allocations
  can use it.

Any usage which may cause deoptimization (such as copies) on any platform can cause a warning to
be issued (on all platforms).

(Side note:
[this page]https://docs.microsoft.com/en-us/windows/win32/direct3d12/swap-chains#swap-effects)
says that in D3D12, "the only supported swap effect is `FLIP_SEQUENTIAL`."
However, in practice, `DISCARD` seems to work.)
