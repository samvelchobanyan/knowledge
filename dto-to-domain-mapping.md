---
type: principle
severity: required
status: stable
applies_to: any-flutter-project
keywords: [dto, domain, mapping, fromJson, toDomain, data-layer, repository]
related: [[local-state-via-storage-class]]
---

# DTO-to-domain mapping

## Rule

All API responses are parsed via DTOs that live in `data/dto/`. Each DTO provides `fromJson` (constructing the DTO from the raw JSON) and `toDomain()` (mapping the DTO into a domain entity from `domain/entities/`). Repositories return only domain entities. DTOs MUST NOT cross out of the data layer; nothing in `domain/`, `features/`, or `core/` imports from `data/dto/`.

## Rationale

The API and the domain are different things, and they evolve on different schedules. API field renames, casing changes, structural reshufflings, and "this was a number, now it's a string" surprises happen continuously. The domain — what the app actually works with — should not absorb that churn. Without a mapping layer, every backend tweak propagates into features and UI, mock data becomes hard to inject (you'd construct API-shaped JSON to test domain logic), and the domain is silently coupled to whichever backend you happen to have today.

DTOs also localize the validation work: nullable fields, missing keys, type-coercion edge cases, and casing differences (`'Name'` vs `'name'`) live in one obvious place. Domain entities stay strict and clean.

## Implications

- One DTO class per API resource in `data/dto/`, named `<Entity>Dto`
- Each DTO has `factory <Entity>Dto.fromJson(Map<String, dynamic> json)` and `<Entity> toDomain()`
- Domain entities in `domain/entities/` MUST NOT have `fromJson` / `toJson` / API-shaped fields — they are Freezed value objects describing what the app cares about, no more
- Repository implementations (`data/repositories/`) handle JSON → DTO → domain entity internally and return only domain entities
- Features and domain code import from `domain/entities/`, never from `data/dto/`
- DTOs may use private nested classes (`_FileDto`, `_CoverDto`, `_ArtistDto`) for sub-objects of an API response; these stay file-private to the DTO

## When this principle does NOT apply

Does not apply to local, non-secret project state: feature toggles, notification subscriptions, user settings cache, non-authentication local state. Such state goes through a storage class (see [[local-state-via-storage-class]]), not through the repository pattern. The repository pattern, with its DTO layer, is the right tool when the data has an API shape distinct from a domain shape; for local flags and key-value collections, the DTO/repository ceremony adds nothing and obscures the simple thing being stored.
