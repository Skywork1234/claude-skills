---
name: layer4-data-source-model
description: Generate a new data-layer datasource + model (and optional entity) for the usb-oda-data-experience-module-order-fulfillment Flutter package, following that repo's existing conventions in lib/src/data. Use when adding a new API endpoint or data source under lib/src/data/ in that package specifically (depends on next_core's odaCore.coreNetworkHttp).
---

## Scope check

This skill is global but its conventions (`next_core`, `context.odaCore.coreNetworkHttp`, the specific barrel files below) are specific to the `usb_oda_data_experience_module_order_fulfillment` package. Before generating, confirm the current project's `pubspec.yaml` name matches `usb_oda_data_experience_module_order_fulfillment` and depends on `next_core`. If it doesn't, this skill's patterns likely don't apply — ask the user before forcing them onto an unrelated project.

## What this module's "layer 4" (data layer) looks like

Only two layers exist as folders: `data` and `domain` (no presentation/core here — this is a pure data package consumed by app-level code). Grouping is **feature-first**, not type-first:

```
lib/src/data/
├── data_sources/<feature>/<name>_(remote_)data(source|Source).dart   # abstract + Impl in one file
├── models/<feature>/<name>_model.dart                                 # extends Entity OR standalone envelope
└── repositories/   # EMPTY — not implemented, do not generate here unless asked
lib/src/domain/
└── entities/<feature>/<name>_entity.dart   # only when a model needs to extend an entity
```

Repository/usecase layer is unimplemented (empty barrel files). Datasources are consumed directly by app code. **Do not scaffold a repository or usecase** unless the user explicitly asks — there's no existing pattern to follow for it yet.

## Before generating, confirm with the user

1. Feature name (folder under `data_sources/` and `models/`) and operation name (e.g. `getPaymentQueue`)
2. HTTP method + `urlName` (MDM symbolic key) or a `legacy` full URL if no MDM mapping exists yet
3. Request fields (query params / body) and response fields (for the model)
4. Does the response need a matching domain **Entity** (i.e. will it be consumed as a domain type elsewhere), or is it just a response DTO with no entity counterpart?
5. Error handling style: **simple** (no try/catch, just status-code check) or **robust** (try/catch with `CoreNetworkHttpException` + resultCode fallback)?

## Datasource pattern

No DI (no get_it/riverpod/provider) and no shared base class — each file is self-contained: one abstract interface + one `Impl`. Methods take `BuildContext` or a `XxxNetworkRequestContext` (wraps `BuildContext` + `token`, see `lib/src/data/models/network_request_context.dart`) as a parameter instead of injecting a client.

API calls go through `next_core`'s `CoreHttpBaseOption` via `context.odaCore.coreNetworkHttp.request(...)`. Prefer `urlName` (resolved via MDM); use `legacy: 'https://...'` only as an escape hatch when no MDM mapping exists yet (comment the intended `urlName` above it, e.g. `// urlName: 'getPaymentQueue',`).

**Simple style** (reference: `lib/src/data/data_sources/cart/checkout_remote_datasource.dart`):
```dart
abstract class XxxRemoteDatasource {
  Future<XxxResponseModel> doThing({required String id, required XxxNetworkRequestContext requestContext});
}

class XxxRemoteDatasourceImpl implements XxxRemoteDatasource {
  Map<String, dynamic>? _responseBody(dynamic data) => data is Map<String, dynamic> ? data : null;

  @override
  Future<XxxResponseModel> doThing({required String id, required XxxNetworkRequestContext requestContext}) async {
    final response = await requestContext.context.odaCore.coreNetworkHttp.request(
      CoreHttpBaseOption(urlName: 'xxx', method: Method.get),
    );
    final body = _responseBody(response.data);
    if (response.statusCode == 200 && body != null) {
      return XxxResponseModel.fromJson(body);
    }
    return XxxResponseModel.empty();
  }
}
```

**Robust style** (reference: `lib/src/data/data_sources/payment_queue/get_payment_queue_data_source.dart`) adds:
```dart
GetXxxResponseModel _responseModel({required dynamic data, int? statusCode}) {
  final body = _responseBody(data);
  if (body == null) return GetXxxResponseModel(resultCode: statusCode?.toString());
  body.putIfAbsent('resultCode', () => statusCode?.toString() ?? '');
  return GetXxxResponseModel.fromJson(body);
}

dynamic _errorResponse(CoreNetworkHttpException exception) {
  try { return (exception.cause as dynamic).response; } catch (_) { return null; }
}
```
wrapped as: `try { ...request... } on CoreNetworkHttpException catch (e) { final r = _errorResponse(e); if (r != null) return _responseModel(data: r.data, statusCode: r.statusCode); throw Exception('Failed to ...'); } catch (e) { throw Exception('Failed to ...'); }`

## Model pattern

No freezed/json_serializable — 100% manual `fromJson`/`toJson`. Explicit casts (`as String?`, `(x as num?)?.toDouble()`); lists via `(json['x'] as List<dynamic>?)?.map((e) => Model.fromJson(e as Map<String, dynamic>)).toList()`. Always add a `factory Xxx.empty()` fallback.

**Extends Entity** (reference: `lib/src/data/models/cart/validate_imei_model.dart` + `lib/src/domain/entities/cart/validate_imei_entity.dart`) — model adds no new fields, forwards everything via `super.xxx`, and IS the entity subtype (no `toEntity()` conversion needed):
```dart
class XxxModel extends XxxEntity {
  const XxxModel({super.fieldA, super.fieldB});

  factory XxxModel.fromJson(Map<String, dynamic> json) => XxxModel(
    fieldA: json['fieldA'] as String?,
    fieldB: json['fieldB'] as String?,
  );

  Map<String, dynamic> toJson() => {'fieldA': fieldA, 'fieldB': fieldB};
}
```
The paired entity is a plain class with `final` fields + `const` constructor, nothing else.

**Standalone envelope** (reference: `lib/src/data/models/payment_queue/get_payment_queue_model.dart`) — standard 3-level response wrapper used across the module:
```
XxxResponseModel { resultCode, resultDescription, developerMessage, data: XxxDataModel }
  → XxxDataModel { ...fields, nested list of XxxItemModel }
    → XxxItemModel { ...fields }
```
Each level: `const` constructor with named/optional fields, `fromJson` factory, `toJson` method. Top-level `XxxResponseModel` also gets `factory .empty()`.

## After generating files

Update both barrel files to export the new file(s):
- `lib/src/data/data_sources/data_source.dart`
- `lib/src/data/models/models.dart`

If you notice existing files missing from these barrels while you're in there, check *why* before adding them — `dart analyze` after editing. Confirmed case: `models/payment_queue/get_status_payment_queue_model.dart` is deliberately NOT exported because it declares its own `GetPaymentQueueDataModel` class that collides (`ambiguous_export`) with the unrelated `GetPaymentQueueDataModel` in `get_payment_queue_model.dart` — that's a pre-existing duplicate-name bug in the source files, not something to paper over by adding the export. Leave it out unless asked to fix the underlying duplicate.
