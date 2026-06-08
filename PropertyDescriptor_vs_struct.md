# PropertyDescriptor vs google.protobuf.Struct

`google.protobuf.Struct` может быть полезен в этой задаче, но он не должен полностью заменять типизированный контракт настроек вроде `PropertyDescriptor`.

`Struct` предназначен для произвольных JSON payload, когда схема заранее неизвестна. Поэтому он хорошо подходит для plugin manifest, JSON Schema, vendor-specific UI hints или точек расширения. Но модель настроек плагина — это не произвольные данные: она описывает поля, значения, ограничения и поведение, которые host/UI должны понимать надежно.

## Где Struct Подходит

`Struct` хорошо подходит для статической JSON-метаинформации или как escape hatch:

```proto
import "google/protobuf/struct.proto";

message StaticSettingsSchema {
    google.protobuf.Struct schema = 1;
}
```

Он также может подойти, если UI уже умеет напрямую потреблять JSON Schema:

```proto
message AcquireSettingsResponse {
    google.protobuf.Struct settings_schema = 1;
    google.protobuf.Struct current_values = 2;
}
```

Это гибко и позволяет авторам плагинов описывать UI-формы без изменения proto-контракта.

## Где Struct Слаб

Для основного settings API замена типизированных полей на `google.protobuf.Value` приведет к потере важных гарантий.

Текущая типизированная модель различает:

```proto
oneof value {
    string value_string = 20;
    int32 value_int32 = 21;
    double value_double = 22;
    bool value_bool = 23;
    int64 value_int64 = 28;
    uint64 value_uint64 = 29;
}
```

В `google.protobuf.Value` числа представлены как JSON numbers, фактически как `double`. Это значит, что большие `int64`/`uint64` значения могут терять точность. В официальных комментариях `struct.proto` также указано, что JSON numbers не могут безопасно представлять большие `int64` значения.

Использование только `Struct` также ослабляет:

- Compile-time проверки в сгенерированном C++/Go коде.
- Явные ограничения вроде ranges и enumerations.
- Понятную API-документацию для авторов плагинов.
- Стабильные проверки совместимости при эволюции proto.

## RangeConstraint

`RangeConstraint` — хороший пример того, почему полностью заменять модель на `Struct` рискованно.

Через `Struct` диапазон можно передать как JSON-объект:

```json
{
    "min": 1,
    "max": 100,
    "default": 10
}
```

Но это уже не строгий proto-контракт, а соглашение на уровне JSON-ключей. Proto не проверит:

- Что `min`, `max` и `default` существуют.
- Что они одного типа.
- Что это именно `int32`, `int64`, `uint64` или `double`.
- Что `min <= max`.
- Что `default` попадает в диапазон.

Типизированный `RangeConstraint` явно описывает допустимые варианты `min`, `max` и `default`, сохраняет различие между `int32`, `int64`, `uint64` и `double`, а также лучше документирует контракт для UI и авторов плагинов.

## Рекомендуемый Гибрид

Основной контракт настроек лучше оставить типизированным, а `Struct` использовать только для extension-полей.

```proto
message SettingValue {
    oneof value {
        string string_value = 1;
        int32 int32_value = 2;
        int64 int64_value = 3;
        uint64 uint64_value = 4;
        double double_value = 5;
        bool bool_value = 6;
        StringList string_list_value = 7;
    }
}

message SettingDescriptor {
    string id = 1;
    SettingType type = 2;
    SettingConstraints constraints = 3;

    google.protobuf.Struct ui_hints = 20;
    google.protobuf.Struct vendor_extensions = 21;
}
```

Так важная часть контракта остается типизированной, но при этом остаётся возможность добавлять plugin-specific UI hints и будущие расширения без изменения основной схемы.

## Вывод

`google.protobuf.Struct` хорошо подходит для произвольного JSON и статических schema payload, но он слишком свободный для основного контракта настроек плагина. Типизированная proto-модель должна оставаться source of truth для settings, values и constraints. `Struct` лучше использовать как escape hatch для опциональной UI-метаинформации или vendor-specific расширений.
