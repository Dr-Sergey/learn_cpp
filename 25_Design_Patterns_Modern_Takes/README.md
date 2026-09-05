# Design Patterns - Modern Takes

Classic design patterns reimagined with modern C++ features: CRTP, type erasure, variant visitors, etc.

**Topics:** 25

## Contents

- [Apply the Memento pattern for snapshotting object state](Apply_the_Memento_pattern_for_snapshotting_object_state.md)
- [Design a multi-stage std::expected pipeline architecture](Implement_monadic_error_handling_pipelines_with_stdexpected_2.md)
- [Extend a std::variant state machine with entry actions and extended state](Implement_a_type-safe_state_machine_with_stdvariant_and_transition_tables_2.md)
- [Implement a type-safe discriminated union message bus with std::variant](Implement_a_type-safe_discriminated_union_message_bus_with_stdvariant.md)
- [Implement a type-safe state machine with std::variant and a transition table](Implement_a_type-safe_state_machine_with_stdvariant_and_a_transition_table.md)
- [Implement a type-safe state machine with std::variant and transition tables](Implement_a_type-safe_state_machine_with_stdvariant_and_transition_tables.md)
- [Implement an Entity-Component-System (ECS) architecture](Implement_an_Entity-Component-System_ECS_architecture.md)
- [Implement an Entity-Component-System (ECS) architecture in C++](Implement_an_Entity-Component-System_ECS_architecture_in_C++.md)
- [Implement an event bus using std::variant and type-indexed dispatch](Implement_an_event_bus_using_stdvariant_and_type-indexed_dispatch.md)
- [Implement Command undo/redo with a deque-based bounded history](Implement_the_Command_pattern_with_undoredo_using_move_semantics_2.md)
- [Implement Command undo/redo with lambda commands instead of virtual interfaces](Implement_the_Command_pattern_with_undoredo_using_move_semantics_3.md)
- [Implement monadic error handling pipelines beyond std::expected basics](Implement_monadic_error_handling_pipelines_beyond_stdexpected_basics.md)
- [Implement monadic error handling pipelines with std::expected](Implement_monadic_error_handling_pipelines_with_stdexpected.md)
- [Implement property-based testing with RapidCheck](Implement_property-based_testing_with_RapidCheck.md)
- [Implement the Command pattern with undo/redo using move semantics](Implement_the_Command_pattern_with_undoredo_using_move_semantics.md)
- [Implement the Entity-Component-System (ECS) architecture pattern](Implement_the_Entity-Component-System_ECS_architecture_pattern.md)
- [Implement the Memento pattern for state snapshots with value semantics](Implement_the_Memento_pattern_for_state_snapshots_with_value_semantics.md)
- [Implement the Proxy pattern for lazy initialization and access control](Implement_the_Proxy_pattern_for_lazy_initialization_and_access_control.md)
- [Implement the Specification pattern for composable business rules](Implement_the_Specification_pattern_for_composable_business_rules.md)
- [Implement the Strategy pattern using std::function and template policies](Implement_the_Strategy_pattern_using_stdfunction_and_template_policies.md)
- [Use the async_scope pattern for structured concurrency in async code](Use_the_async_scope_pattern_for_structured_concurrency_in_async_code.md)
- [Use the Interpreter pattern with std::variant for an expression tree](Use_the_Interpreter_pattern_with_stdvariant_for_an_expression_tree.md)
- [Use the Opaque Typedef pattern for domain-driven type safety](Use_the_Opaque_Typedef_pattern_for_domain-driven_type_safety.md)
- [Use the Pipeline pattern with coroutines for streaming data processing](Use_the_Pipeline_pattern_with_coroutines_for_streaming_data_processing.md)
- [Use the Strategy pattern with templates for zero-overhead policy injection](Use_the_Strategy_pattern_with_templates_for_zero-overhead_policy_injection.md)

## Notes

- Modern C++ replaces many GoF patterns with language features - lambdas replace Strategy, variants replace Visitor.
- Type erasure (like `std::function`) provides runtime polymorphism without inheritance hierarchies.
- CRTP replaces virtual dispatch for static polymorphism, giving you zero overhead at runtime.
- `std::variant` + `std::visit` implements the Visitor pattern without double dispatch.
- The Singleton pattern is often an anti-pattern - prefer dependency injection.
- Policy-based design (templates as mixins) replaces deep inheritance for configurable behavior.
- `std::unique_ptr` with custom deleters implements the RAII-based Handle pattern.
- The Builder pattern benefits from C++20 designated initializers and aggregate init.
- `std::expected` chains replace the traditional error-handling Command pattern.
- Event-driven systems use signal/slot or observer patterns - consider `std::function` + containers.
