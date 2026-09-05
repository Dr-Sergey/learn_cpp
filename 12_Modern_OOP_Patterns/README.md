# Modern OOP Patterns

Inheritance, virtual functions, CRTP, type erasure, mixins, and modern OOP design patterns.

**Topics:** 29

## Contents

- [Apply the Builder pattern using method chaining and C++ return *this idiom](Apply_the_Builder_pattern_using_method_chaining_and_C++_return_this_idiom.md)
- [Apply the Liskov Substitution Principle and verify it in code](Apply_the_Liskov_Substitution_Principle_and_verify_it_in_code.md)
- [Apply the Observer pattern with type-safe signal/slot using templates](Apply_the_Observer_pattern_with_type-safe_signalslot_using_templates.md)
- [Apply the policy-based design pattern with templates](Apply_the_policy-based_design_pattern_with_templates.md)
- [Apply the Strategy pattern using templates and std::function](Apply_the_Strategy_pattern_using_templates_and_stdfunction.md)
- [Apply the type-erasure pattern using virtual tables and concepts](Apply_the_type-erasure_pattern_using_virtual_tables_and_concepts.md)
- [Implement an RAII lock with trylock semantics for optional ownership](Implement_an_RAII_lock_with_trylock_semantics_for_optional_ownership.md)
- [Implement copy-on-write semantics for large shared objects](Implement_copy-on-write_semantics_for_large_shared_objects.md)
- [Implement the Decorator pattern using composition and interface forwarding](Implement_the_Decorator_pattern_using_composition_and_interface_forwarding.md)
- [Implement the Observer pattern with type-safe signal/slot semantics](Implement_the_Observer_pattern_with_type-safe_signalslot_semantics.md)
- [Implement the Prototype pattern using virtual clone methods](Implement_the_Prototype_pattern_using_virtual_clone_methods.md)
- [Implement type-safe event systems using std::variant and visitor dispatch](Implement_type-safe_event_systems_using_stdvariant_and_visitor_dispatch.md)
- [Understand covariant return types and their limitations with smart pointers](Understand_covariant_return_types_and_their_limitations_with_smart_pointers.md)
- [Understand covariant return types in virtual functions](Understand_covariant_return_types_in_virtual_functions.md)
- [Understand interface segregation: prefer small focused interfaces](Understand_interface_segregation_prefer_small_focused_interfaces.md)
- [Understand object slicing and how to prevent it](Understand_object_slicing_and_how_to_prevent_it.md)
- [Understand three-way comparison operator <=> (C++20)](Understand_three-way_comparison_operator_C++20.md)
- [Understand virtual dispatch and its cost; prefer non-virtual interfaces](Understand_virtual_dispatch_and_its_cost_prefer_non-virtual_interfaces.md)
- [Use abstract interfaces with pure virtual destructors correctly](Use_abstract_interfaces_with_pure_virtual_destructors_correctly.md)
- [Use concepts to model behavioral requirements for template parameters](Use_concepts_to_model_behavioral_requirements_for_template_parameters.md)
- [Use deducing-this (explicit object parameter, C++23) for CRTP without templates](Use_deducing-this_explicit_object_parameter_C++23_for_CRTP_without_templates.md)
- [Use mixins through CRTP or inheritance to compose behavior](Use_mixins_through_CRTP_or_inheritance_to_compose_behavior.md)
- [Use override and final explicitly on virtual functions](Use_override_and_final_explicitly_on_virtual_functions.md)
- [Use std::enable_shared_from_this for safe shared_ptr from this](Use_stdenable_shared_from_this_for_safe_shared_ptr_from_this.md)
- [Use the Null Object pattern to avoid null pointer checks throughout code](Use_the_Null_Object_pattern_to_avoid_null_pointer_checks_throughout_code.md)
- [Use the State pattern with std::variant for finite state machines](Use_the_State_pattern_with_stdvariant_for_finite_state_machines.md)
- [Use the Template Method pattern to define algorithm skeletons in base classes](Use_the_Template_Method_pattern_to_define_algorithm_skeletons_in_base_classes.md)
- [Use the visitor pattern with std::variant instead of double dispatch](Use_the_visitor_pattern_with_stdvariant_instead_of_double_dispatch.md)
- [Use virtual base classes to resolve diamond inheritance](Use_virtual_base_classes_to_resolve_diamond_inheritance.md)

## Notes

- Prefer composition over inheritance - deep hierarchies are fragile and hard to modify.
- The NVI (Non-Virtual Interface) pattern separates public interface from virtual implementation, letting the base class control pre/post-conditions while derived classes provide only the customization points.
- CRTP (Curiously Recurring Template Pattern) enables static polymorphism without vtable overhead - useful when the full type hierarchy is known at compile time.
- Virtual destructors are required in base classes intended for polymorphic deletion; without one, deleting a derived object through a base pointer is undefined behavior.
- The `override` and `final` keywords prevent accidental hiding and give the compiler enough information to devirtualize calls.
- The Pimpl idiom (pointer to implementation) hides implementation details and reduces compile-time dependencies between translation units.
- Mixin classes via CRTP or template parameters add reusable behavior without requiring a deep inheritance hierarchy.
- Type erasure (e.g., `std::function`, `std::any`) provides runtime polymorphism without inheritance - useful when the set of types is open but the interface is fixed.
- Multiple inheritance in C++ requires careful handling of the diamond problem; virtual inheritance resolves the ambiguity at the cost of slightly more complex object layout.
- Prefer strong types (wrapper classes) over primitive types to enforce semantics at compile time and make function signatures self-documenting.
