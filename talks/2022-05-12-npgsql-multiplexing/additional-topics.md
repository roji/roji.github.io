# Additional, advanced multiplexing topics

* DbDataSource
* Zoom into how the write loop works
  * Never yield for I/O - only when there are no longer any commands
  * Writing completes synchronously under normal circumstances. When it doesn't, we do ContinueWith etc.

## Multiplexing tweaks

* Write Coalescing Buffer Threshold
  * Stop coalescing when we've reached a threshold (1000 bytes).
  * Ethernet MTU is 1500, but jumbo frames are 9000. <!-- .element: class="fragment" -->
* Write Coalescing Timeout <!-- .element: class="fragment" -->
  * Wait a bit even if there aren't any pending commands.
  * Sacrifice latency for throughput
  * Was not effective in TechEmpower <!-- .element: class="fragment" -->
