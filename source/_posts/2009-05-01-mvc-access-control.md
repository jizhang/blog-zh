---
title: "MVC 模式中的权限控制"
date: 2009-05-01 13:15:00
categories: [Programming]
tags: [mvc, sql]
---

在 MVC 模式中，通过建立用户表、角色表、控制器表、方法表和权限表来实现权限管理，即根据用户的角色以及所请求的控制器和方法来决定是否有访问权限。该过程放在控制器基类的构造函数中。
1. 用户表 user: user_id, name, password, role_id
2. 角色表 role: role_id, name
3. 控制器表 controller: controller_id, name
4. 方法表 method: method_id, controller_id, name
5. 权限表 role_method: role_id, method_id

使用以下 SQL 语句进行验证：

```sql
SELECT COUNT(*)
FROM role_method
WHERE role_method.role_id = @role_id
AND method_id IN (
    SELECT method_id
    FROM method
    JOIN controller
    USING (controller_id)
    WHERE controller.name = @controller
    AND method.name = @method
)
```
