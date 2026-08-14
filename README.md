# RedOxyde

RedOxyde is an attempt (maybe overly ambitious?) to fix the underlying issues of the world's dominating CMS (content management system), WordPress, by borrowing and introducing concepts from projects that take security, performance and resource consumption as their top-most priority rather than running after what's the "next shiny thing" we can integrate.

It is a general-purpose content management system for businesses and developers that are looking for a safe and blazzzingly(!) fast CMS stack in their arsenal that doesn't trade away the ergonomics of WordPress-based development.

Calling this overly ambitious is not an understatement because it is trying to go head-on with a project that has been there for 20+ years. In its current state, it is still in the planning and documenting phase and it will be a long LONG way before it becomes useful.

## What is being built?

RedOxyde is a Rust-based CMS stack that utilizes the WebAssembly runtime to allow writing truly extensible applications without trading away performance, security and similar developer ergonomics. Just like plugins and themes, RedOxyde will have a concept of Extensions that can be written in any language that targets the WebAssembly Component Model. Developers can use the familiar concepts of actions and filters borrowed from WordPress, while extensions stay truly isolated, capability-gated and typed end-to-end via compiled wasm modules, keeping things secure.
