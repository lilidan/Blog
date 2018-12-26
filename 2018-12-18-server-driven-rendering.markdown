Server-driven rendering at scale comes with its own set of challenges. Here is a handful we’re solving:

Safely updating our component definitions while maintaining backward compatibility.
Sharing type definitions for our components across platforms.
Responding to events at runtime like button taps or user input.
Transitioning between multiple JSON-driven screens while preserving internal state.
Rendering entirely custom components that don’t have existing implementations at build-time. We’re experimenting with the Lona format for this.


因为已经有view和viewmodel。所以只需要根据viewmodel生成数据json。根据controller生成布局即可。
