# Моделі: методи та тригери

Перелік із `docs/MODELS_CORE.md`, `docs/MODELS_MILITARY.md`, `docs/MODELS_COMBAT.md` та `docs/MODELS_PROCESSES.md` (гілка `main`). Без характеристик і описів реалізації.

## Player

**User methods:** немає.

**Domain methods:** `can_pay_global_cost(cost)`, `pay_global_cost(cost)`, `add_coins(amount)`, `create_castle(region, name, founder_knight)`.

**Triggers:** `empty_coins`.

**Trigger methods:** `check_trigger_empty_coins()`, `on_trigger_empty_coins()`.

## Castle

**User methods:** `user_name_next_ready_knight(name)`.

**Domain methods:** `can_pay_local_cost(cost)`, `pay_local_cost(cost)`, `can_house_unit(knight)`, `move_soldiers_between_reserve_and_knight(knight, composition_delta)`, `add_recruited_soldier(type)`, `can_start_building_upgrade(building_type)`, `apply_building_upgrade(building_type, target_level)`, `enqueue_knight_replacement()`, `complete_knight_replacement(replacement)`, `name_next_ready_knight(name)`, `can_annex_region(region, camp)`, `recalculate_region_connections()`.

**Triggers:** `empty_food`, `storage_capacity`.

**Trigger methods:** `check_trigger_empty_food()`, `on_trigger_empty_food()`, `check_trigger_storage_capacity()`, `on_trigger_storage_capacity()`.

## Region

**User methods:** `user_change_allow_transit(value)`, `user_abandon_region()`.

**Domain methods:** `find_active_camp(player)`, `get_or_create_camp(player)`, `register_combat(attacker, registration_reason, defender_player = null)`, `start_next_combat_if_possible()`, `resolve_arrival(army, movement)`, `set_occupied_by(player)`, `restore_owner_control(expected_occupier_player)`, `annex_to(player, castle)`, `become_neutral()`, `become_castle_region(new_castle, player)`, `set_connection_valid(value)`, `apply_resource_site_upgrade(resource_type, from_level, target_level, quantity)`, `can_start_resource_site_upgrade(resource_type, quantity)`, `on_army_presence_changed()`, `destroy_neutral_defense()`.

**Triggers:** `neutral_defense_recovery_complete`.

**Trigger methods:** `check_trigger_neutral_defense_recovery_complete()`, `on_trigger_neutral_defense_recovery_complete()`.

## City

**User methods:** немає.

**Domain methods:** `complete_raid(player)`.

**Triggers:** `active_wealth_recovered`.

**Trigger methods:** `check_trigger_active_wealth_recovered()`, `on_trigger_active_wealth_recovered()`.

## Knight

**User methods:** `user_enter_castle(target_castle)`, `user_leave_castle()`, `user_change_unit_composition(composition_delta)`.

**Domain methods:** `set_location_state(state, target_castle = null)`, `leave_castle_for_movement()`, `set_soldiers(new_composition)`, `set_army(army)`, `add_battle_experience(amount)`, `apply_casualties(...)`, `die()`, `change_home_castle(new_castle)`.

**Triggers:** немає.

**Trigger methods:** немає.

## Army

**User methods:** `user_merge_armies(armies, commander, thresholds)`, `user_split_army(groups)`, `user_change_commander(knight)`, `user_change_combat_thresholds(values)`, `user_attack_player(target_camp)`, `user_raid_city(city)`, `user_attack_neutral_defense()`.

**Domain methods:** `enter_region(region)`, `enter_camp(camp)`, `leave_camp()`, `start_movement(movement)`, `finish_movement()`, `set_combat_waiting(combat)`, `clear_combat_waiting()`, `start_regrouping()`, `finish_regrouping()`, `start_retreat_to(region)`, `start_loss_regrouping_in_current_camp()`, `remove_dead_knights()`, `reassign_commander()`, `ensure_commander_after_casualties()`, `split_for_castle_occupation(castle)`, `block_in_castle(castle)`, `unblock_from_castle(castle)`.

**Triggers:** `regrouping_complete`.

**Trigger methods:** `check_trigger_regrouping_complete()`, `on_trigger_regrouping_complete()`.

## CampInRegion

**User methods:** `user_annex_region(castle)`.

**Domain methods:** `accept_army(army)`, `on_army_left(army)`, `get_defending_armies()`, `validate_reorganization(armies)`, `can_annex_to(castle)`.

**Triggers:** `can_progress`, `ready_for_annexation`.

**Trigger methods:** `check_trigger_can_progress()`, `on_trigger_can_progress()`, `check_trigger_ready_for_annexation()`, `on_trigger_ready_for_annexation()`.

## Movement

**User methods:** `user_start_movement(army, route[])`, `user_change_planned_route(route[])`, `user_refresh_target_opponent()`.

**Domain methods:** `validate_route(route[])`, `start_phase(...)`, `start_retreat_local(...)`, `determine_target_opponent(final_region)`, `pause_for_combat_waiting()`, `resume_after_combat_waiting()`, `advance_to_next_region()`, `resolve_camp_arrival()`, `continue_after_transit_combat()`.

**Triggers:** `direction_revealed`, `next_region_reached`, `camp_reached`.

**Trigger methods:** `check_trigger_direction_revealed()`, `on_trigger_direction_revealed()`, `check_trigger_next_region_reached()`, `on_trigger_next_region_reached()`, `check_trigger_camp_reached()`, `on_trigger_camp_reached()`.

## CombatSituation

**User methods:** `user_set_transit_decision(decision)`, `user_set_pre_battle_retreat_decision(army, retreat)`, `user_join_defense_from_transit(army)`.

**Domain methods:** `start()`, `determine_transit_mode()`, `collect_potential_defenders()`, `resolve_pre_battle()`, `lock_combat_parameters()`, `get_legal_retreat_regions(role)`, `select_retreat_region(role)`, `resolve_combat()`, `apply_casualties(result)`, `apply_result(result)`, `finish_without_battle(reason)`, `finish()`.

**Triggers:** `battle_start`.

**Trigger methods:** `check_trigger_battle_start()`, `on_trigger_battle_start()`.

## CastleFounding

**User methods:** `user_start_castle_founding(...)`.

**Domain methods:** `validate_start()`, `cancel()`, `complete()`.

**Triggers:** `founder_invalid`, `can_progress`, `complete`.

**Trigger methods:** `check_trigger_founder_invalid()`, `on_trigger_founder_invalid()`, `check_trigger_can_progress()`, `on_trigger_can_progress()`, `check_trigger_complete()`, `on_trigger_complete()`.

## Recruitment

**User methods:** `user_add_recruitment_order(soldier_type, quantity)`.

**Domain methods:** `validate_new_order(type, quantity)`, `finish_current_recruit()`.

**Triggers:** `can_progress`, `current_recruit_finished`.

**Trigger methods:** `check_trigger_can_progress()`, `on_trigger_can_progress()`, `check_trigger_current_recruit_finished()`, `on_trigger_current_recruit_finished()`.

## BuildingUpgrade

**User methods:** `user_start_building_upgrade(castle, building_type)`.

**Domain methods:** `validate_start()`, `complete()`.

**Triggers:** `complete`.

**Trigger methods:** `check_trigger_complete()`, `on_trigger_complete()`.

## ResourceSiteUpgrade

**User methods:** `user_start_resource_site_upgrade(region, resource_type, quantity)`.

**Domain methods:** `validate_start()`, `complete()`.

**Triggers:** `complete`.

**Trigger methods:** `check_trigger_complete()`, `on_trigger_complete()`.

## KnightReplacement

**User methods:** немає.

**Domain methods:** `complete()`.

**Triggers:** `complete`.

**Trigger methods:** `check_trigger_complete()`, `on_trigger_complete()`.

