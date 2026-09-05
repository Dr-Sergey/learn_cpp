# std::execution & Senders/Receivers

The execution model for async work: schedulers, senders, receivers, and structured concurrency.

**Topics:** 36

## Contents

- [Chain senders with then, upon_error, and upon_stopped](Chain_senders_with_then_upon_error_and_upon_stopped.md)
- [Compare std::execution with coroutines and know when to combine them](Compare_stdexecution_with_coroutines_and_know_when_to_combine_them.md)
- [Implement a custom sender from scratch](Implement_a_custom_sender_from_scratch.md)
- [Implement a custom sender type satisfying the sender concept](Implement_a_custom_sender_type_satisfying_the_sender_concept.md)
- [Implement a custom sender type with the sender concept](Implement_a_custom_sender_type_with_the_sender_concept.md)
- [Integrate Asio with P2300 senders via asio::execution](Integrate_Asio_with_P2300_senders_via_asioexecution.md)
- [Understand cancellation in P2300 via stop tokens and stop callbacks](Understand_cancellation_in_P2300_via_stop_tokens_and_stop_callbacks.md)
- [Understand schedulers and execution contexts in std::execution](Understand_schedulers_and_execution_contexts_in_stdexecution.md)
- [Understand stop tokens and cancellation in sender pipelines](Understand_stop_tokens_and_cancellation_in_sender_pipelines.md)
- [Understand structured concurrency and async_scope for safe task lifetimes](Understand_structured_concurrency_and_async_scope_for_safe_task_lifetimes.md)
- [Understand structured concurrency with async_scope](Understand_structured_concurrency_with_async_scope.md)
- [Understand the sender/receiver execution model (P2300)](Understand_the_senderreceiver_execution_model_P2300.md)
- [Understand the sender/receiver model and why it replaces futures and callbacks](Understand_the_senderreceiver_model_and_why_it_replaces_futures_and_callbacks.md)
- [Understand the sender/receiver model: what a sender is and what a receiver is](Understand_the_senderreceiver_model_what_a_sender_is_and_what_a_receiver_is.md)
- [Use async_scope for structured concurrency and lifetime management](Use_async_scope_for_structured_concurrency_and_lifetime_management.md)
- [Use schedulers as sender factories and switch context with continues_on](Understand_schedulers_and_execution_contexts_in_stdexecution_2.md)
- [Use schedulers to control where senders execute](Use_schedulers_to_control_where_senders_execute.md)
- [Use std::execution::bulk for parallel loop execution over a sender](Use_stdexecutionbulk_for_parallel_loop_execution_over_a_sender.md)
- [Use std::execution::just and just_error to create leaf senders](Use_stdexecutionjust_and_just_error_to_create_leaf_senders.md)
- [Use std::execution::just to create a sender that emits a value](Use_stdexecutionjust_to_create_a_sender_that_emits_a_value.md)
- [Use std::execution::let_value and let_error for monadic sender chaining](Use_stdexecutionlet_value_and_let_error_for_monadic_sender_chaining.md)
- [Use std::execution::let_value, let_error, and let_stopped for dependent senders](Use_stdexecutionlet_value_let_error_and_let_stopped_for_dependent_senders.md)
- [Use std::execution::on to run a sender on a specific scheduler](Use_stdexecutionon_to_run_a_sender_on_a_specific_scheduler.md)
- [Use std::execution::split to multicast a sender to multiple receivers](Use_stdexecutionsplit_to_multicast_a_sender_to_multiple_receivers.md)
- [Use std::execution::sync_wait to block until a sender completes](Use_stdexecutionsync_wait_to_block_until_a_sender_completes.md)
- [Use std::execution::then to chain sender transformations](Use_stdexecutionthen_to_chain_sender_transformations.md)
- [Use std::execution::when_all and when_any for concurrent fan-out](Use_stdexecutionwhen_all_and_when_any_for_concurrent_fan-out.md)
- [Use std::execution::when_all to join multiple senders](Use_stdexecutionwhen_all_to_join_multiple_senders.md)
- [Use std::this_thread::sync_wait to drive a sender pipeline synchronously](Use_stdthis_threadsync_wait_to_drive_a_sender_pipeline_synchronously.md)
- [Use stdexec::let_value and let_error for monadic sender chaining](Use_stdexeclet_value_and_let_error_for_monadic_sender_chaining.md)
- [Use stdexec::on and stdexec::starts_on for scheduler affinity](Use_stdexecon_and_stdexecstarts_on_for_scheduler_affinity.md)
- [Use stdexec::split to multicast a sender to multiple receivers](Use_stdexecsplit_to_multicast_a_sender_to_multiple_receivers.md)
- [Use stdexec::sync_wait to drive a sender pipeline from synchronous code](Use_stdexecsync_wait_to_drive_a_sender_pipeline_from_synchronous_code.md)
- [Use stdexec::then to chain sender transformations](Use_stdexecthen_to_chain_sender_transformations.md)
- [Use stdexec::when_all to join multiple senders](Use_stdexecwhen_all_to_join_multiple_senders.md)
- [Write a P2300-compatible thread pool scheduler from scratch](Write_a_P2300-compatible_thread_pool_scheduler_from_scratch.md)

## Notes

- `std::execution` (P2300) provides a structured concurrency model for C++26. It is a significant shift from older async patterns, so the learning curve is real - but the payoff is composability, type safety, and built-in cancellation.
- Senders represent lazy asynchronous work - they describe what to do, not when to do it. Nothing runs until you connect and start them.
- Receivers consume the result of a sender via one of three channels: value, error, or stopped. Every sender must notify its receiver on exactly one of those channels.
- `std::execution::then` chains async operations in a style similar to continuations. Each step transforms the value type, and the entire type chain is checked at compile time.
- Operation states hold the state of an in-flight async operation. They must remain stable in memory from the point `start()` is called until the receiver is notified - do not move them.
- Schedulers represent execution contexts such as thread pools, event loops, and GPU queues. They are the answer to the question "where should this work run?"
- `std::execution::when_all` composes multiple concurrent senders into a single awaitable that delivers all results together.
- Structured concurrency ensures child operations complete before their parent scope exits - there are no "fire and forget" leaks.
- stdexec (from NVIDIA) is the reference implementation ahead of standardization. The namespace is `stdexec::` rather than `std::execution::` in current code.
- The sender/receiver model is designed to replace callbacks, futures, and ad-hoc coroutine wiring for high-performance async code.
