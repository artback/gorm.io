---
title: Предзагрузка (Нетерпеливая загрузка)
layout: страница
---

## Предварительная загрузка

GORM позволяет загружать отношения с помощью `Preload`, например:

```go
type User struct {
  gorm.Model
  Username string
  Orders   []Order
}

type Order struct {
  gorm.Model
  UserID uint
  Price  float64
}

// Предварительная загрузка Заказов (Orders) при поиске пользователей
db.Preload("Orders").Find(&users)
// SELECT * FROM users;
// SELECT * FROM orders WHERE user_id IN (1,2,3,4);

db.Preload("Orders").Preload("Profile").Preload("Role").Find(&users)
// SELECT * FROM users;
// SELECT * FROM orders WHERE user_id IN (1,2,3,4); // has many
// SELECT * FROM profiles WHERE user_id IN (1,2,3,4); // has one
// SELECT * FROM roles WHERE id IN (4,5,6); // belongs to
```

## Join с предварительной загрузкой

`Preload` загружает данные связей в отдельном запросе, `Join Preload` загружает данные связей используя join внутри запроса, например:

```go
db.Joins("Company").Joins("Manager").Joins("Account").First(&user, 1)
db.Joins("Company").Joins("Manager").Joins("Account").First(&user, "users.name = ?", "jinzhu")
db.Joins("Company").Joins("Manager").Joins("Account").Find(&users, "users.id IN ?", []int{1,2,3,4,5})
```

Join with conditions

```go
db.Joins("Company", DB.Where(&Company{Alive: true})).Find(&users)
// SELECT `users`.`id`,`users`.`name`,`users`.`age`,`Company`.`id` AS `Company__id`,`Company`.`name` AS `Company__name` FROM `users` LEFT JOIN `companies` AS `Company` ON `users`.`company_id` = `Company`.`id` AND `Company`.`alive` = true;
```

{% note warn %}
**NOTE** `Join Preload` works with one-to-one relation, e.g: `has one`, `belongs to`
{% endnote %}

## Предзагрузить все

`clause.Associations` can work with `Preload` similar like `Select` when creating/updating, you can use it to `Preload` all associations, for example:

```go
type User struct {
  gorm.Model
  Name       string
  CompanyID  uint
  Company    Company
  Role       Role
  Orders     []Order
}

db.Preload(clause.Associations).Find(&users)
```

`clause.Associations` won't preload nested associations, but you can use it with [Nested Preloading](#nested_preloading) together, e.g:

```go
db.Preload("Orders.OrderItems.Product").Preload(clause.Associations).Find(&users)
```

## Предзагрузка с условиями

GORM allows Preload associations with conditions, it works similar to [Inline Conditions](query.html#inline_conditions)

```go
// Preload Orders with conditions
db.Preload("Orders", "state NOT IN (?)", "cancelled").Find(&users)
// SELECT * FROM users;
// SELECT * FROM orders WHERE user_id IN (1,2,3,4) AND state NOT IN ('cancelled');

db.Where("state = ?", "active").Preload("Orders", "state NOT IN (?)", "cancelled").Find(&users)
// SELECT * FROM users WHERE state = 'active';
// SELECT * FROM orders WHERE user_id IN (1,2) AND state NOT IN ('cancelled');
```

## Пользовательский SQL

You are able to custom preloading SQL by passing in `func(db *gorm.DB) *gorm.DB`, for example:

```go
db.Preload("Orders", func(db *gorm.DB) *gorm.DB {
  return db.Order("orders.amount DESC")
}).Find(&users)
// SELECT * FROM users;
// SELECT * FROM orders WHERE user_id IN (1,2,3,4) order by orders.amount DESC;
```

## <span id="nested_preloading">Nested Preloading</span>

GORM supports nested preloading, for example:

```go
db.Preload("Orders.OrderItems.Product").Preload("CreditCard").Find(&users)

// Customize Preload conditions for `Orders`
// And GORM won't preload unmatched order's OrderItems then
db.Preload("Orders", "state = ?", "paid").Preload("Orders.OrderItems").Find(&users)
```
