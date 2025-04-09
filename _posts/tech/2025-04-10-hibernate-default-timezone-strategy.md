---
layout: post
title: Hibernate OffsetDateTime/ZonedDateTime이 UTC로 매핑되는 문제 - 타임존 처리 전략 분석
description: >
  Hibernate 6.2부터 타임존 데이터를 처리하는 기본 전략이 변경되었습니다. 이로 인해 발생할 수 있는 이슈와 디버깅 과정, 그리고 해결 방법을 알아봅니다.
image: /assets/img/blog/common/hibernate.png
category: Tech
tags: [ Hibernate ]
---

- toc
{:toc}

## 요약

Hibernate 6.2부터 타임존 데이터를 처리하는 기본 전략이 변경되었습니다.
기존에는 MySQL, MariaDB, PostgreSQL 같은 벤더를 사용할 때 시스템 타임존을 기준으로 날짜/시간 데이터를 정규화하여 저장 및 조회했지만,
Hibernate 6.2부터는 `UTC` 기준으로 정규화하도록 기본 설정이 변경되었습니다.

이로 인해 `TIMESTAMP WITH TIME ZONE` 타입을 제공하지 않는 MySQL, MariaDB와 같은 데이터베이스를 사용하는 경우
새로 적재되는 데이터가 `UTC` 기준으로 정규화되어 적재될 것이기에 기존 데이터와의 정합성에 유의해야 합니다.

PostgreSQL은 `TIMESTAMP WITH TIME ZONE` 타입을 제공하지만 저장 시 내부적으로 `UTC`로 정규화하고 타임존 정보를 저장하지 않습니다.
즉, 어떤 타임존으로 저장해도 내부적으로 정규화되어 저장되기에 기존 데이터와의 정합성 문제는 없습니다.

그러나 데이터 조회 시에는 `TIMESTAMP WITH TIME ZONE` 타입 제공 여부를 떠나 `OffsetDateTime`/`ZonedDateTime` 타입으로 가져오는 경우
`ZoneOffset`/`ZoneId`가 `UTC` 기준으로 매핑되어 이와 관련한 이슈가 없는지 확인이 필요합니다.

이러한 변경은 다음 설정을 통해 Hibernate 6.2 이전과 같이 동작하도록 할 수 있습니다.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        timezone.default_storage: NORMALIZE
```

---

## 문제 상황

제가 재직 중인 회사는 PostgreSQL을 사용하며 `TIMESTAMP WITH TIME ZONE` 타입으로 날짜/시간 데이터를 저장합니다.
그리고 이는 애플리케이션에서 `OffsetDateTime` 타입에 매핑되도록 구성되어 있습니다.
기존에는 조회된 데이터가 `OffsetDateTime`에 매핑될 때 시스템 기본 타임존인 `Asia/Seoul`에 맞게 `ZoneOffset`이 `+09:00`로 설정되었었습니다.
그런데 Spring Boot 버전을 올린 이후 `ZoneOffset`이 `UTC` 기준으로 설정되었습니다.

```java
// 기존 동작 (Hibernate 6.2 미만)
OffsetDateTime createdAt = entity.getCreatedAt(); // +09:00 Offset

// 변경 후 동작 (Hibernate 6.2 이상)
OffsetDateTime createdAt = entity.getCreatedAt(); // +00:00 Offset (UTC)
```

System, Hibernate, JDBC Connection, Jackson에 타임존 설정을 명시적으로 시도해보았지만 `OffsetDateTime`은 여전히 `UTC` 기준으로 매핑되었습니다.
이로 인해 기존에 Hibernate를 통해 데이터베이스에서 가져온 `OffsetDateTime`을
다른 출처의 `ZoneOffset`이 다른 `OffsetDateTime`과 `equals`로 비교하거나
`toLocalDate()`를 사용하여 날짜만 뽑아내는 경우 또는 그 외 다양한 이슈가 발생할 수 있습니다.

---

## AttributeConverter로 해결?!

일반적으로 많이 사용하는 `AttributeConverter`를 등록하여 데이터베이스에서 가져온 `OffsetDateTime`에 `Offset`을 명시적으로 설정하려 했습니다.
그러나 `AttributeConverter`는 Java 객체와 데이터베이스 스키마 간의 불일치를 해결하는 것이 주 목적이라는 생각이 들었고,
기존에 시스템 타임존을 사용하던 방식이 `UTC`로 고정되도록 바뀌었다는 것이 개인적으로 납득이 잘 되지 않아 Hibernate의 타입 매핑에 대해 자세히 알아보기로 했습니다.

---

## Hibernate의 타입 변환 흐름 분석

> 다음은 디버깅하며 소스 코드를 확인하는 내용입니다. [건너뛰기](#timezonestoragestrategy)

먼저 `ValueExtractor` 인터페이스를 살펴보았습니다. 이 인터페이스는 데이터베이스 조회 결과를 Java 타입으로 변환하는 역할을 합니다.

```java
package org.hibernate.type.descriptor;

public interface ValueExtractor<X> {
    X extract(ResultSet rs, int paramIndex, WrapperOptions options) throws SQLException;

    X extract(CallableStatement statement, int paramIndex, WrapperOptions options) throws SQLException;

    X extract(CallableStatement statement, String paramName, WrapperOptions options) throws SQLException;
}
```

디버깅을 통해 `OffsetDateTime` 변환 시 `TimestampUtcAsJdbcTimestampJdbcType` 클래스 내 익명클래스로 구현된 구현체가 사용됨을 확인할 수 있었습니다.

```java
package org.hibernate.type.descriptor.jdbc;

public class TimestampUtcAsJdbcTimestampJdbcType implements JdbcType {
    @Override
    public <X> ValueExtractor<X> getExtractor(final JavaType<X> javaType) {
        return new BasicExtractor<>(javaType, this) {
            @Override
            protected X doExtract(ResultSet rs, int paramIndex, WrapperOptions options) throws SQLException {
                final Timestamp timestamp = rs.getTimestamp(paramIndex, UTC_CALENDAR);
                return javaType.wrap(timestamp == null ? null : timestamp.toInstant(), options);
            }
            // ... 생략 ...
        };
    }
}
```

이 구현체는 데이터베이스 조회 결과인 `ResultSet`에서 `Timestamp`를 가져온 후, 이를 `Instant`로 변환하여 `javaType.wrap()` 메소드에 전달합니다.

이어서 `OffsetDateTimeJavaType` 클래스의 `wrap()` 메소드를 보겠습니다.

```java
package org.hibernate.type.descriptor.java;

public class OffsetDateTimeJavaType extends AbstractTemporalJavaType<OffsetDateTime>
        implements VersionJavaType<OffsetDateTime> {
    @Override
    public <X> OffsetDateTime wrap(X value, WrapperOptions options) {
        // ... 생략 ...
        if (value instanceof Instant) {
            Instant instant = (Instant) value;
            return instant.atOffset(ZoneOffset.UTC);
        }
        // ... 생략 ...
    }
}
```

`value`가 `Instant` 타입인 경우 `ZoneOffset.UTC`를 사용하여 `OffsetDateTime`을 생성합니다.
이와 같은 이유로 `OffsetDateTime`의 `Offset`이 `UTC`로 고정된 것입니다.

그런데 `TimestampWithTimeZoneJdbcType`이라는 구현체도 있는데,
왜 이름부터 UTC스러운 `TimestampUtcAsJdbcTimestampJdbcType`이 사용되었는지 알아볼 필요가 있었습니다.

---

## Hibernate의 MetaBuildingProcess

Hibernate는 애플리케이션 실행 시점에 엔티티, 컨버터 등을 읽어들여 메타데이터를 구성합니다.
타입 매핑과 같은 설정들이 메타데이터 구성 과정에서 이루어지기에 `MetadataBuildingProcess`라는 클래스를 살펴보게 되었습니다.

그런데 `handleTypes()`라는 메서드에서 중요한 설정을 발견했습니다.

```java
private static void handleTypes(...) {
    final JdbcType timestampWithTimeZoneOverride = getTimestampWithTimeZoneOverride(options, jdbcTypeRegistry);
    if (timestampWithTimeZoneOverride != null) {
        adaptTimestampTypesToDefaultTimeZoneStorage(typeConfiguration, timestampWithTimeZoneOverride);
    }
}
```

위 코드는 `getTimestampWithTimeZoneOverride()`로 가져온 `JdbcType` 구현체를 `OffsetDateTime` 변환에 사용하도록 Registry에 등록하는 역할을 합니다.

`getTimestampWithTimeZoneOverride()` 메서드는 다음과 같습니다.

```java
private static JdbcType getTimestampWithTimeZoneOverride(MetadataBuildingOptions options, JdbcTypeRegistry jdbcTypeRegistry) {
    switch (options.getDefaultTimeZoneStorage()) {
        case NORMALIZE:
            return jdbcTypeRegistry.getDescriptor(Types.TIMESTAMP);
        case NORMALIZE_UTC:
            return jdbcTypeRegistry.getDescriptor(SqlTypes.TIMESTAMP_UTC);
        default:
            return null;
    }
}
```

만약 `options.getDefaultTimeZoneStorage()`로 가져온 `TimeZoneStorageStrategy` 값이
`NORMALIZE` 또는 `NORMALIZE_UTC`라면 `OffsetDateTime` 변환에 사용할 `JdbcType` 구현체를 반환하고,
이를 Registry에 등록하여 기존 매핑 설정을 Override 하도록 하는 것입니다.

디버깅을 통해 확인해보니 `options.getDefaultTimeZoneStorage()`의 반환 값은 `TimeZoneStorageStrategy.NORMALIZE_UTC`였고,
이로 인해 호출되는 `jdbcTypeRegistry.getDescriptor(SqlTypes.TIMESTAMP_UTC)`의 반환값이 `TimestampUtcAsJdbcTimestampJdbcType`였습니다.

즉 `TimeZoneStorageStrategy.NORMALIZE_UTC`로 인해 `OffsetDateTime` 변환에 `TimestampUtcAsJdbcTimestampJdbcType` 구현체가 사용되도록
Registry에 등록된 것입니다.

---

## TimeZoneStorageStrategy

Hibernate 6에서 `TimeZoneStorage`라는 개념이 새로 도입되었습니다.
이에 대한 `TimeZoneStorageStrategy` 전략 설정에 따라 Hibernate가 날짜/시간 데이터를 데이터베이스에 저장할 때 타임존 정보를 어떻게 처리할지 결정됩니다.

```java
package org.hibernate;

public enum TimeZoneStorageStrategy {
    NATIVE,
    COLUMN,
    NORMALIZE,
    NORMALIZE_UTC
}
```

- `NATIVE`
    - 데이터베이스에 시간대 정보를 온전히 저장합니다.
    - 데이터베이스 `Dialect`의 `TimeZoneSupport` 값이 `NATIVE`가 아닌 경우 실행 시점에 구성 오류를 반환합니다.
- `COLUMN` : 시간대 정보를 별도의 열에 저장합니다.
- `NORMALIZE`
    - 시간대 정보를 저장하지 않습니다.
    - JDBC_TIME_ZONE 설정 또는 시스템 타임존으로 정규화하여 타임스탬프를 저장합니다.
- `NORMALIZE_UTC`
    - 시간대 정보를 저장하지 않습니다.
    - UTC로 정규화하여 타임스탬프를 저장합니다.

이러한 핵심이 되는 `TimeZoneStorageStrategy`를 선택하기 위해 `TimeZoneSupport`, `TimeZoneStorageType`와 같은 열거형 타입들이 사용됩니다.

---

## TimeZoneSupport

이 값은 데이터베이스가 타임존을 지원하는 정도를 나타내며, 총 3가지 값이 존재합니다.

```java
package org.hibernate.dialect;

public enum TimeZoneSupport {
    NATIVE,
    NORMALIZE,
    NONE
}
```

다음 순서대로 지원도가 높음을 의미합니다.

- `NATIVE` : 데이터베이스에서 타임존 정보를 함께 저장 (Oracle, H2, ...)
- `NORMALIZE` : 특정 시간대로 정규화하여 타임스탬프만을 기록 (PostgreSQL, ...)
- `NONE` : `TIMESTAMP WITH TIME ZONE` 타입을 지원하지 않음 (MySQL, MariaDB, ...)

각 데이터베이스 Dialect 구현체마다 이 값을 반환하는 메서드가 있습니다. `PostgreSQLDialect`의 경우 다음과 같이 `NORMALIZE` 값을 반환합니다.

```java
package org.hibernate.dialect;

public class PostgreSQLDialect extends Dialect {
    // ...
    @Override
    public TimeZoneSupport getTimeZoneSupport() {
        return TimeZoneSupport.NORMALIZE;
    }
}
```

---

## TimeZoneStorageType

`TimeZoneStorageStrategy`를 설정하기 위한 값입니다. 이 값과 `TimeZoneSupport` 값의 조합으로 `TimeZoneStorageStrategy`가 설정됩니다.

```java
package org.hibernate.annotations;

public enum TimeZoneStorageType {
    NATIVE,
    NORMALIZE,
    NORMALIZE_UTC,
    COLUMN,
    AUTO,
    DEFAULT
}
```

기존 6.2 버전 이전에는 따로 `TimeZoneStorageType`을 설정하지 않았다면 기본 전략으로 `TimeZoneStorageStrategy.NORMALIZE`가 사용되었습니다.
그런데 6.2 버전부터는 `TimeZoneStorageType.DEFAULT`가 기본 설정으로 사용되도록 변경되었습니다.
이 설정은 데이터베이스의 `TimeZoneSupport` 값이 `NATIVE`가 아닌 경우 `TimeZoneStorageStrategy.NORMALIZE_UTC`를 전략으로 사용하도록 합니다.

자세한 내용은 다음 문서에서 확인할 수 있습니다.
[Hibernate 6.2 Migration Guide](https://docs.jboss.org/hibernate/orm/6.2/migration-guide/migration-guide.html#ddl-timezones:~:text=datatype%20by%20default-,Timezone%20and%20offset%20storage,behavior%2C%20set%20the%20configuration%20property%20hibernate.timezone.default_storage%20to%20NORMALIZE.,-Byte%5B%5D/Character%5B%5D%20mapping)

실제로 전략을 선택하는 코드를 보면 다음과 같습니다.

```java
package org.hibernate.boot.internal;

public class MetadataBuilderImpl implements MetadataBuilderImplementor, TypeContributions {
    // ...
    private TimeZoneStorageStrategy toTimeZoneStorageStrategy(TimeZoneSupport timeZoneSupport) {
        switch (defaultTimezoneStorage) {
            case NATIVE:
                if (timeZoneSupport != TimeZoneSupport.NATIVE) {
                    throw new HibernateException("The configured time zone storage type NATIVE is not supported with the configured dialect");
                }
                return TimeZoneStorageStrategy.NATIVE;
            case COLUMN:
                return TimeZoneStorageStrategy.COLUMN;
            case NORMALIZE:
                return TimeZoneStorageStrategy.NORMALIZE;
            case NORMALIZE_UTC:
                return TimeZoneStorageStrategy.NORMALIZE_UTC;
            case AUTO:
                switch (timeZoneSupport) {
                    case NATIVE:
                        return TimeZoneStorageStrategy.NATIVE;
                    case NORMALIZE:
                    case NONE:
                        return TimeZoneStorageStrategy.COLUMN;
                    default:
                        throw new HibernateException("Unsupported time zone support: " + timeZoneSupport);
                }
            case DEFAULT:
                switch (timeZoneSupport) {
                    case NATIVE:
                        return TimeZoneStorageStrategy.NATIVE;
                    case NORMALIZE:
                    case NONE:
                        return TimeZoneStorageStrategy.NORMALIZE_UTC;
                    default:
                        throw new HibernateException("Unsupported time zone support: " + timeZoneSupport);
                }
            default:
                throw new HibernateException("Unsupported time zone storage type: " + defaultTimezoneStorage);
        }
        // ...
    }
```

여기서 `defaultTimezoneStorage`는 `TimeZoneStorageType`의 기본값인 `DEFAULT`이고, `timeZoneSupport`는 PostgreSQL을 사용하기 때문에
`NORMALIZE`입니다.
이러한 이유로 `TimeZoneStorage` 전략으로 `TimeZoneStorageStrategy.NORMALIZE_UTC`가 사용되게 된 것입니다.

---

## 정리

Hibernate 6.2 이상을 사용하며, `TimeZoneSupport`가 `NATIVE`가 아닌 데이터베이스(MySQL, MariaDB, PostgreSQL 등...)를 사용하는 경우
`OffsetDateTime`/`ZonedDateTime`으로 매핑된 데이터 조회 시 `Offset`/`TimeZone`이 `UTC`로 고정됩니다.

> 현 포스트에서는 조회에 관련된 디버깅 과정만 담았지만, 데이터를 저장하는 경우에도 UTC 기준으로 정규화되어 데이터베이스로 넘어간다고 보시면 됩니다.

---

## 해결 1 : TimeZoneStorageType 설정 변경

`TimeZoneStorageType`을 `NORMALIZE`로 설정하여 `TimeZoneStorageStrategy`를 `NORMALIZE`로 변경할 수 있습니다.
이를 통해 기존처럼 시스템 타임존 기준으로 정규화되도록 할 수 있습니다.

### 글로벌 설정

다음과 같이 프로퍼티 설정으로 `TimeZoneStorageType`을 글로벌 설정할 수 있습니다.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        timezone.default_storage: NORMALIZE
```

자세한 내용은 다음 문서에서 확인할 수 있습니다.
[Hibernate 6.2 User Guide](https://docs.jboss.org/hibernate/orm/6.2/userguide/html_single/Hibernate_User_Guide.html#_time_zone_storage:~:text=implements%20CurrentTenantIdentifierResolver.-,hibernate.timezone.default_storage,-See%3A%20AvailableSettings)

### 개별 필드 설정

`@TimeZoneStorage`를 사용하여 개별 필드에 설정하는 방식입니다.

```java

@TimeZoneStorage(TimeZoneStorageType.NORMALIZE)
private OffsetDateTime normalized;

@TimeZoneStorage(TimeZoneStorageType.COLUMN) // 별도의 컬럼에 타임존 저장
@TimeZoneColumn(name = "column_offset")
private OffsetDateTime offsetDateTime;
```

자세한 내용은 다음 문서에서 확인할 수 있습니다.
[Hibernate 6.2 User Guide](https://docs.jboss.org/hibernate/orm/6.2/userguide/html_single/Hibernate_User_Guide.html#settings-hibernate.timezone.default_storage)

---

## 해결 2 : Hibernate의 제안 받아들이기

Hibernate가 제시하는 이러한 접근 방식은 일관된 시간 처리를 위한 것으로 이해됩니다.
내부 시스템에서는 UTC 시간대만을 사용하고, 외부로 데이터를 노출할 때만 해당 지역의 로컬 시간대로 변환하여 표현하는 것이 핵심입니다.
이를 구현하기 위해 내부 시스템 전반은 UTC 기준으로 관리하되,
외부 인터페이스에서는 Formatter 등록이나 Jackson 글로벌 타임존 설정 등을 통해 시간대 변환 계층을 둘 수 있습니다.
결과적으로 이는 내부 시스템 전체에 걸쳐 UTC를 기준으로 하는 명확한 규칙을 수립하는 것입니다.
이처럼 기존의 방식을 고수하기보다 새로운 접근 방식을 수용하는 것이 더 나은 해결책이 될 수 있습니다.

---
## 느낀 점

이전 [Spring RestClient/RestTemplate 요청이 실패하는 이유](https://nilgil.com/blog/spring-http-transfer-method-changes/)
포스트처럼 이번에도 버전 업그레이드로 인한 이슈를 다뤘습니다. 외부 라이브러리의 변경으로 인한 문제는 인지, 원인 파악이 쉽지 않다는 것을 다시 한번 경험했고,
이러한 상황에서 테스트 코드가 얼마나 중요한 역할을 하는지 다시금 깨닫게 되었습니다.

또한 코드의 의도를 이해하는 것의 중요성을 배웠습니다. '기존과 달라졌으니 원래대로 돌려놓자'라는 단순한 접근보다는
'이러한 변경의 의도는 무엇일까?'라는 질문을 통해 더 나은 시스템을 위한 고민으로 발전시킬 수 있었던 유익한 경험이었습니다.

이번 이슈와 관련하여 이미 여러 포스팅이 있어 쉽게 해결할 수도 있었지만, 직접 디버깅하고 소스 코드를 분석해 본 것이 Hibernate에 대한 이해도를 높이는 좋은 기회가 되었습니다.
앞으로도 이러한 학습 자세를 유지해야겠다고 다짐하게 되었습니다.
