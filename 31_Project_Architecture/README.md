# Project Architecture

Software architecture patterns, project structure, build organization, module design, and system-level architectural decisions for production C++ projects.

**Topics:** 42

## Contents

- [Apply CQRS (Command Query Responsibility Segregation) in C++ systems](Apply_CQRS_Command_Query_Responsibility_Segregation_in_C++_systems.md)
- [Apply layered architecture pattern in C++ projects](Apply_layered_architecture_pattern_in_C++_projects.md)
- [Apply the Mediator pattern to reduce coupling between subsystems](Apply_the_Mediator_pattern_to_reduce_coupling_between_subsystems.md)
- [Apply the Microkernel (plugin) architecture for extensible C++ systems](Apply_the_Microkernel_plugin_architecture_for_extensible_C++_systems.md)
- [Apply the Saga pattern for distributed transactions in C++ microservices](Apply_the_Saga_pattern_for_distributed_transactions_in_C++_microservices.md)
- [Apply the Strangler Fig pattern when modernizing legacy C++ code](Apply_the_Strangler_Fig_pattern_when_modernizing_legacy_C++_code.md)
- [Build observable systems - metrics, tracing, and health checks in C++](Build_observable_systems_-_metrics_tracing_and_health_checks_in_C++.md)
- [Build shared library architecture with proper symbol visibility](Build_shared_library_architecture_with_proper_symbol_visibility.md)
- [Design a clean separation between business logic and infrastructure code](Design_a_clean_separation_between_business_logic_and_infrastructure_code.md)
- [Design a state machine architecture for complex C++ applications](Design_a_state_machine_architecture_for_complex_C++_applications.md)
- [Design a testable architecture with clear seams for mocking](Design_a_testable_architecture_with_clear_seams_for_mocking.md)
- [Design compile-time configuration using constexpr and templates](Design_compile-time_configuration_using_constexpr_and_templates.md)
- [Design configuration management for multi-environment C++ deployments](Design_configuration_management_for_multi-environment_C++_deployments.md)
- [Design data serialization architecture with protobuf, flatbuffers, or msgpack](Design_data_serialization_architecture_with_protobuf_flatbuffers_or_msgpack.md)
- [Design embedded software architecture with layered HAL and BSP](Design_embedded_software_architecture_with_layered_HAL_and_BSP.md)
- [Design event-driven architectures with message queues in C++](Design_event-driven_architectures_with_message_queues_in_C++.md)
- [Design hot-reload and live-patching architectures for C++ applications](Design_hot-reload_and_live-patching_architectures_for_C++_applications.md)
- [Design inter-process communication (IPC) architecture in C++](Design_inter-process_communication_IPC_architecture_in_C++.md)
- [Design logging and telemetry architecture for production C++ systems](Design_logging_and_telemetry_architecture_for_production_C++_systems.md)
- [Design memory management architecture - arenas, pools, and PMR](Design_memory_management_architecture_-_arenas_pools_and_PMR.md)
- [Design module boundaries using C++20 modules or header-only libraries](Design_module_boundaries_using_C++20_modules_or_header-only_libraries.md)
- [Design multi-platform build architecture (Windows, Linux, embedded targets)](Design_multi-platform_build_architecture_Windows_Linux_embedded_targets.md)
- [Design plugin architectures with dynamic loading and abstract interfaces](Design_plugin_architectures_with_dynamic_loading_and_abstract_interfaces.md)
- [Design thread pool architectures for concurrent task processing](Design_thread_pool_architectures_for_concurrent_task_processing.md)
- [Design versioned APIs with backward compatibility in C++](Design_versioned_APIs_with_backward_compatibility_in_C++.md)
- [Implement actor model concurrency with message passing in C++](Implement_actor_model_concurrency_with_message_passing_in_C++.md)
- [Implement error propagation strategies across architectural layers](Implement_error_propagation_strategies_across_architectural_layers.md)
- [Implement feature flags and runtime configuration in C++ projects](Implement_feature_flags_and_runtime_configuration_in_C++_projects.md)
- [Implement graceful shutdown and resource cleanup architecture](Implement_graceful_shutdown_and_resource_cleanup_architecture.md)
- [Implement hexagonal architecture (ports and adapters) in C++](Implement_hexagonal_architecture_ports_and_adapters_in_C++.md)
- [Implement the Entity-Component-System (ECS) pattern for game or simulation engines](Implement_the_Entity-Component-System_ECS_pattern_for_game_or_simulation_engines.md)
- [Implement the Pipeline pattern for data processing tasks](Implement_the_Pipeline_pattern_for_data_processing_tasks.md)
- [Structure a C++ project with clean directory layout and CMake](Structure_a_C++_project_with_clean_directory_layout_and_CMake.md)
- [Structure a monorepo for multiple C++ libraries and applications](Structure_a_monorepo_for_multiple_C++_libraries_and_applications.md)
- [Structure a multi-target embedded project (bootloader, app, tests)](Structure_a_multi-target_embedded_project_bootloader_app_tests.md)
- [Structure safety-critical C++ projects (DO-178C, IEC 61508 guidelines)](Structure_safety-critical_C++_projects_DO-178C_IEC_61508_guidelines.md)
- [Use package managers (Conan, vcpkg) in project architecture](Use_package_managers_Conan_vcpkg_in_project_architecture.md)
- [Use the Abstract Factory pattern for platform-independent code](Use_the_Abstract_Factory_pattern_for_platform-independent_code.md)
- [Use the dependency injection pattern for loosely coupled components](Use_the_dependency_injection_pattern_for_loosely_coupled_components.md)
- [Use the Outbox pattern for reliable event publishing in C++ systems](Use_the_Outbox_pattern_for_reliable_event_publishing_in_C++_systems.md)
- [Use the Repository pattern to abstract data access in C++](Use_the_Repository_pattern_to_abstract_data_access_in_C++.md)
- [Use the Service Locator pattern vs Dependency Injection in large codebases](Use_the_Service_Locator_pattern_vs_Dependency_Injection_in_large_codebases.md)

## Notes

- Separate interface from implementation - use abstract classes, Pimpl, or modules.
- Layer your architecture: platform -> core -> services -> application -> UI.
- Keep compile-time dependencies minimal - forward-declare types in headers when possible.
- Use dependency injection to make components testable and decoupled.
- The SOLID principles guide class design; GRASP patterns guide responsibility assignment.
- Monorepo vs multirepo is a build/CI decision - use Conan/vcpkg for library management.
- Define clear module boundaries - each library should have a single responsibility.
- Document architectural decisions (ADRs) - future developers need to understand the why.
- Use static libraries for internal code and shared libraries for plugin architectures.
- Configuration should be externalized (files, environment) - not hardcoded in source.
