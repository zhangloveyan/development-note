# next.js

## 前提

js、html、css

## 基本

基于 react ，扩展框架

### 1.安装

2024年01月17日 09:43:26 

使用版本

构建

#### 创建第一个项目

### 2.目录

文件及作用

not-found，全局响应

## 组件

### 1.路由

### 2.导航

## nextui

报错certificate has expired

1、取消ssl验证： npm config set strict-ssl false

## prisma orm框架

mysql 映射

### 从 Prisma 到 MySQL 的原生类型映射

| Prisma     | MySQL            | 注意                                           |
| :--------- | :--------------- | :--------------------------------------------- |
| `String`   | `VARCHAR(191)`   |                                                |
| `Boolean`  | `BOOLEAN`        | 在 MySQL 中 `BOOLEAN` 是 `TINYINT(1)` 的同义词 |
| `Int`      | `INT`            |                                                |
| `BigInt`   | `BIGINT`         |                                                |
| `Float`    | `DOUBLE`         |                                                |
| `Decimal`  | `DECIMAL(65,30)` |                                                |
| `DateTime` | `DATETIME(3)`    |                                                |
| `Json`     | `JSON`           | 仅在 MySQL 5.7+ 中受支持                       |
| `Bytes`    | `LONGBLOB`       |                                                |

### 原生类型映射

内省 MySQL 数据库时，数据库类型根据下表映射到 Prisma：

|                          |               |        |                                                |                                                              |
| :----------------------- | :------------ | :----- | :--------------------------------------------- | :----------------------------------------------------------- |
| MySQL                    | Prisma        | 支持的 | 原生数据库类型属性                             | 注意                                                         |
| `serial`                 | `BigInt`      | ✔️      | `@db.UnsignedBigInt @default(autoincrement())` |                                                              |
| `bigint`                 | `BigInt`      | ✔️      | `@db.BigInt`                                   |                                                              |
| `bigint unsigned`        | `BigInt`      | ✔️      | `@db.UnsignedBigInt`                           |                                                              |
| `bit`                    | `Bytes`       | ✔️      | `@db.Bit(x)`                                   | `bit(1)` maps to `Boolean` - 所有其他 `bit(x)` 映射到 `Bytes` |
| `boolean` | `tinyint(1)` | `Boolean`     | ✔️      | `@db.TinyInt(1)`                               |                                                              |
| `varbinary`              | `Bytes`       | ✔️      | `@db.VarBinary`                                |                                                              |
| `longblob`               | `Bytes`       | ✔️      | `@db.LongBlob`                                 |                                                              |
| `tinyblob`               | `Bytes`       | ✔️      | `@db.TinyBlob`                                 |                                                              |
| `mediumblob`             | `Bytes`       | ✔️      | `@db.MediumBlob`                               |                                                              |
| `blob`                   | `Bytes`       | ✔️      | `@db.Blob`                                     |                                                              |
| `binary`                 | `Bytes`       | ✔️      | `@db.Binary`                                   |                                                              |
| `date`                   | `DateTime`    | ✔️      | `@db.Date`                                     |                                                              |
| `datetime`               | `DateTime`    | ✔️      | `@db.DateTime`                                 |                                                              |
| `timestamp`              | `DateTime`    | ✔️      | `@db.TimeStamp`                                |                                                              |
| `time`                   | `DateTime`    | ✔️      | `@db.Time`                                     |                                                              |
| `decimal(a,b)`           | `Decimal`     | ✔️      | `@db.Decimal(x,y)`                             |                                                              |
| `numeric(a,b)`           | `Decimal`     | ✔️      | `@db.Decimal(x,y)`                             |                                                              |
| `enum`                   | `Enum`        | ✔️      | 不适用                                         |                                                              |
| `float`                  | `Float`       | ✔️      | `@db.Float`                                    |                                                              |
| `double`                 | `Float`       | ✔️      | `@db.Double`                                   |                                                              |
| `smallint`               | `Int`         | ✔️      | `@db.SmallInt`                                 |                                                              |
| `smallint unsigned`      | `Int`         | ✔️      | `@db.UnsignedSmallInt`                         |                                                              |
| `mediumint`              | `Int`         | ✔️      | `@db.MediumInt`                                |                                                              |
| `mediumint unsigned`     | `Int`         | ✔️      | `@db.UnsignedMediumInt`                        |                                                              |
| `int`                    | `Int`         | ✔️      | `@db.Int`                                      |                                                              |
| `int unsigned`           | `Int`         | ✔️      | `@db.UnsignedInt`                              |                                                              |
| `tinyint`                | `Int`         | ✔️      | `@db.TinyInt(x)`                               | `tinyint(1)` 映射到 `Boolean` 所有其他 `tinyint(x)` 映射到 `Int` |
| `tinyint unsigned`       | `Int`         | ✔️      | `@db.UnsignedTinyInt(x)`                       | `tinyint(1) unsigned` **才不是** 映射到 `Boolean`            |
| `year`                   | `Int`         | ✔️      | `@db.Year`                                     |                                                              |
| `json`                   | `Json`        | ✔️      | `@db.Json`                                     | 仅在 MySQL 5.7+ 中受支持                                     |
| `char`                   | `String`      | ✔️      | `@db.Char(x)`                                  |                                                              |
| `varchar`                | `String`      | ✔️      | `@db.VarChar(x)`                               |                                                              |
| `tinytext`               | `String`      | ✔️      | `@db.TinyText`                                 |                                                              |
| `text`                   | `String`      | ✔️      | `@db.Text`                                     |                                                              |
| `mediumtext`             | `String`      | ✔️      | `@db.MediumText`                               |                                                              |
| `longtext`               | `String`      | ✔️      | `@db.LongText`                                 |                                                              |
| `set`                    | `Unsupported` | 还没有 |                                                |                                                              |
| `geometry`               | `Unsupported` | 还没有 |                                                |                                                              |
| `point`                  | `Unsupported` | 还没有 |                                                |                                                              |
| `linestring`             | `Unsupported` | 还没有 |                                                |                                                              |
| `polygon`                | `Unsupported` | 还没有 |                                                |                                                              |
| `multipoint`             | `Unsupported` | 还没有 |                                                |                                                              |
| `multilinestring`        | `Unsupported` | 还没有 |                                                |                                                              |
| `multipolygon`           | `Unsupported` | 还没有 |                                                |                                                              |
| `geometrycollection`     | `Unsupported` | 还没有 |                                                |                                                              |

一些约束

|                 |            |                           |
| --------------- | ---------- | ------------------------- |
| @id             | 主键       |                           |
| @default()      | 默认值     |                           |
| autoincrement() | 自增       | @default(autoincrement()) |
| now()           | 当前时间   | @default(now())           |
| @unique         | 唯一索引   |                           |
| @map            | 别名       | 数据库与实体类不一致      |
| ?               | 可以为null | 不加则不许为null          |

```react
model User {
  @@map("user")
  id          Int      @id @default(autoincrement())
  userId      String?  @db.VarChar(255)
  email       String?  @unique
  createdTime DateTime @default(now()) @map("created_time") @db.DateTime(0)
  updatedTime DateTime @updatedAt @map("updated_time") @db.DateTime()
}
```

- 字段， 字段用 `@` 表示
- 块，用 `@@` 表示