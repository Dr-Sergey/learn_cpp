# Coroutines (C++20)

Coroutine fundamentals: co_await, co_yield, co_return, promise types, and coroutine-based generators/tasks.

**Topics:** 21

## Contents

- [Customize Coroutine Frame Allocation with promise_type::operator new](Customize_coroutine_frame_allocation_with_promise_operator_new.md)
- [Customize Coroutine Traits with coroutine_traits Specialization](Customize_coroutine_traits_with_coroutine_traits_specialization.md)
- [Debug and inspect coroutine state with custom tooling](Debug_and_inspect_coroutine_state_with_custom_tooling.md)
- [Handle Errors in Coroutines with co_await and std::expected](Handle_errors_in_coroutines_with_co_await_and_stdexpected.md)
- [Implement a task type with structured concurrency semantics](Implement_a_task_type_with_structured_concurrency_semantics.md)
- [Implement a task<T> coroutine type from scratch](Implement_a_taskT_coroutine_type_from_scratch.md)
- [Implement an awaitable timer that suspends and resumes after a delay](Implement_an_awaitable_timer_that_suspends_and_resumes_after_a_delay.md)
- [Implement Async Generator for Asynchronous Element Production](Implement_async_generator_for_asynchronous_element_production.md)
- [Implement co_await for a custom awaitable type](Implement_co_await_for_a_custom_awaitable_type.md)
- [Implement structured concurrency with coroutine task trees](Implement_structured_concurrency_with_coroutine_task_trees.md)
- [Know Stackless vs Stackful Coroutines Tradeoffs](Know_stackless_vs_stackful_coroutines_tradeoffs.md)
- [Survey coroutine libraries: cppcoro, libunifex, folly::coro, stdexec](Survey_coroutine_libraries_cppcoro_libunifex_folly-coro_stdexec.md)
- [Understand co_yield Patterns and Lazy Generator Pipelines](Understand_co_yield_patterns_and_lazy_generator_pipelines.md)
- [Understand coroutine frame allocation and the Heap Allocation Elision Optimization (HALO)](Understand_coroutine_frame_allocation_and_the_Heap_Allocation_Elision_Optimizati.md)
- [Understand coroutine frame lifetime and heap allocation elision](Understand_coroutine_frame_lifetime_and_heap_allocation_elision.md)
- [Understand coroutine interaction with exceptions and stack unwinding](Understand_coroutine_interaction_with_exceptions_and_stack_unwinding.md)
- [Understand final_suspend Customization for Detached vs Awaited Tasks](Understand_final_suspend_customization_for_detached_vs_awaited_tasks.md)
- [Understand symmetric transfer and tail-call optimization in coroutines](Understand_symmetric_transfer_and_tail-call_optimization_in_coroutines.md)
- [Understand the coroutine model: promise_type, coroutine_handle, and suspension](Understand_the_coroutine_model_promise_type_coroutine_handle_and_suspension.md)
- [Use coroutines for async I/O and task chaining](Use_coroutines_for_async_IO_and_task_chaining.md)
- [Write a lazy generator using coroutines](Write_a_lazy_generator_using_coroutines.md)

## Notes

- C++20 coroutines are stackless — each coroutine has a heap-allocated frame
- co_await, co_yield, and co_return are the three coroutine keywords
- A coroutine requires a promise type that controls its behavior (return, suspend, resume)
- std::generator (C++23) is the standard lazy sequence generator built on coroutines
- Coroutine handles (std::coroutine_handle) manage coroutine lifetime — destroy them to avoid leaks
- initial_suspend and inal_suspend control when a coroutine first suspends and finishes
- Custom awaitables implement wait_ready, wait_suspend, wait_resume for custom scheduling
- Symmetric transfer (wait_suspend returning a coroutine_handle) avoids stack overflow in chains
- Coroutine allocations can be elided by the compiler (HALO optimization) in ideal cases
- Error handling in coroutines uses unhandled_exception() in the promise type
