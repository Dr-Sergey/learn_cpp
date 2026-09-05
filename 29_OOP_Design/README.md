# OOP Design

Object-oriented design principles, class hierarchy best practices, design patterns, and proven idioms for building maintainable C++ class architectures. Targeting mid-to-senior developers. The topics here are not about syntax - they are about making decisions: when to use inheritance versus composition, how to keep interfaces focused, and which patterns give you extensibility without sacrificing clarity or performance.

**Topics:** 43

## Contents

- [Apply the Dependency Inversion Principle with abstract interfaces and dependency injection](Apply_the_Dependency_Inversion_Principle_with_abstract_interfaces_and_dependency.md)
- [Design aggregate types and understand aggregate initialization rules](Design_aggregate_types_and_understand_aggregate_initialization_rules.md)
- [Design callback interfaces using std::function, templates, and type erasure](Design_callback_interfaces_using_std-function_templates_and_type_erasure.md)
- [Design class hierarchies that respect the Liskov Substitution Principle](Design_class_hierarchies_that_respect_the_Liskov_Substitution_Principle.md)
- [Design class invariants and validate them with assertions](Design_class_invariants_and_validate_them_with_assertions.md)
- [Design classes with proper const-correctness for member functions and data](Design_classes_with_proper_const-correctness_for_member_functions_and_data.md)
- [Design classes with single responsibility principle in C++](Design_classes_with_single_responsibility_principle_in_C++.md)
- [Design enums and enum classes for type-safe state machines](Design_enums_and_enum_classes_for_type-safe_state_machines.md)
- [Design exception-safe classes with strong and basic guarantees](Design_exception-safe_classes_with_strong_and_basic_guarantees.md)
- [Design fluent interfaces and method chaining in C++](Design_fluent_interfaces_and_method_chaining_in_C++.md)
- [Design immutable classes and understand their thread-safety benefits](Design_immutable_classes_and_understand_their_thread-safety_benefits.md)
- [Design policy-based classes using template parameters](Design_policy-based_classes_using_template_parameters.md)
- [Design proper move semantics for classes with resources](Design_proper_move_semantics_for_classes_with_resources.md)
- [Design RAII wrappers correctly - constructor acquires, destructor releases](Design_RAII_wrappers_correctly_-_constructor_acquires_destructor_releases.md)
- [Design SOA (Structure of Arrays) vs AOS with class wrappers for performance](Design_SOA_Structure_of_Arrays_vs_AOS_with_class_wrappers_for_performance.md)
- [Design thread-safe classes with minimal locking granularity](Design_thread-safe_classes_with_minimal_locking_granularity.md)
- [Design type-erased interfaces for runtime polymorphism without inheritance](Design_type-erased_interfaces_for_runtime_polymorphism_without_inheritance.md)
- [Design value types vs entity types and know when to use each](Design_value_types_vs_entity_types_and_know_when_to_use_each.md)
- [Implement copy-on-write (COW) semantics for expensive-to-copy types](Implement_copy-on-write_COW_semantics_for_expensive-to-copy_types.md)
- [Implement SFINAE-friendly and concept-constrained class interfaces](Implement_SFINAE-friendly_and_concept-constrained_class_interfaces.md)
- [Implement the Builder pattern for complex object construction in C++](Implement_the_Builder_pattern_for_complex_object_construction_in_C++.md)
- [Implement the CRTP for static polymorphism and mixin classes](Implement_the_CRTP_for_static_polymorphism_and_mixin_classes.md)
- [Implement the Observer pattern with modern C++ (signals and slots)](Implement_the_Observer_pattern_with_modern_C++_signals_and_slots.md)
- [Implement the Singleton pattern correctly in modern C++ (Meyers Singleton)](Implement_the_Singleton_pattern_correctly_in_modern_C++_Meyers_Singleton.md)
- [Know hidden friends idiom for operators and ADL-safe free functions](Know_hidden_friends_idiom_for_operators_and_ADL-safe_free_functions.md)
- [Know how to correctly implement comparison operators with spaceship operator](Know_how_to_correctly_implement_comparison_operators_with_spaceship_operator.md)
- [Know how to design classes for testability with dependency injection](Know_how_to_design_classes_for_testability_with_dependency_injection.md)
- [Know when to make member functions static, const, or free functions](Know_when_to_make_member_functions_static_const_or_free_functions.md)
- [Know when to use inheritance vs composition and prefer composition](Know_when_to_use_inheritance_vs_composition_and_prefer_composition.md)
- [Know when to use virtual functions vs std::variant vs CRTP for polymorphism](Know_when_to_use_virtual_functions_vs_std-variant_vs_CRTP_for_polymorphism.md)
- [Master the Rule of Zero, Rule of Five, and Rule of Three](Master_the_Rule_of_Zero_Rule_of_Five_and_Rule_of_Three.md)
- [Prevent object slicing with a clone pattern in polymorphic hierarchies](Understand_object_slicing_and_how_to_prevent_it.md)
- [Understand the four pillars of OOP - encapsulation, inheritance, polymorphism, abstraction](Understand_the_four_pillars_of_OOP_-_encapsulation_inheritance_polymorphism_abst.md)
- [Use access specifiers strategically - public, protected, private semantics](Use_access_specifiers_strategically_-_public_protected_private_semantics.md)
- [Use aggregate initialization vs constructor initialization wisely](Use_aggregate_initialization_vs_constructor_initialization_wisely.md)
- [Use deducing this (C++23) for simplified CRTP and recursive lambdas](Use_deducing_this_C++23_for_simplified_CRTP_and_recursive_lambdas.md)
- [Use factory methods and abstract factories for flexible object creation](Use_factory_methods_and_abstract_factories_for_flexible_object_creation.md)
- [Use final keyword strategically to enable devirtualization](Use_final_keyword_strategically_to_enable_devirtualization.md)
- [Use multiple inheritance safely with virtual inheritance and interfaces](Use_multiple_inheritance_safely_with_virtual_inheritance_and_interfaces.md)
- [Use strong typedefs and phantom types to prevent parameter confusion](Use_strong_typedefs_and_phantom_types_to_prevent_parameter_confusion.md)
- [Use the Interface Segregation Principle to design minimal abstract classes](Use_the_Interface_Segregation_Principle_to_design_minimal_abstract_classes.md)
- [Use the NVI pattern (Non-Virtual Interface) for controlled polymorphism](Use_the_NVI_pattern_Non-Virtual_Interface_for_controlled_polymorphism.md)
- [Use the Pimpl idiom for ABI stability and compilation firewall](Use_the_Pimpl_idiom_for_ABI_stability_and_compilation_firewall.md)

## Notes

- SOLID principles remain relevant in C++ - but template-based designs often replace interface hierarchies.
- Liskov Substitution Principle: derived types must be fully substitutable for their base types.
- Interface Segregation: prefer small, focused abstract interfaces over large monolithic ones.
- Open/Closed Principle: extend behavior through composition and templates, not modification.
- Dependency Inversion: depend on abstractions (concepts, interfaces), not concrete implementations.
- Virtual functions have a cost (indirect call plus cache miss) - consider CRTP or type erasure for hot paths.
- Abstract base classes should have virtual destructors - otherwise polymorphic deletion is UB.
- Multiple inheritance creates diamond problems - use virtual inheritance or composition instead.
- Prefer value semantics when possible - copyable, regular types are easier to reason about.
- Design classes as either leaf (final) or base (virtual destructor) - avoid ambiguous middle ground.
