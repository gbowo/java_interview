### 一、DTO、DAO、VO、PO的区别


#### 1. DTO


**Data Transfer Object**：数据传输对象，DTO用于在不同层之间传输数据，它通常用于将业务逻辑层（Service层）的数据传输给表示层（Presentation层）或持久化层（Persistence层）。DTO对象通常包含业务领域的数据，但不包含业务逻辑。


#### 2. DAO


**Data Access Objects**：数据访问对象，DAO用于封装数据访问逻辑，它负责与数据库进行交互，执行CRUD（创建、读取、更新、删除）操作。DAO对象通常封装了数据库访问的细节，使业务逻辑层能够更加简洁地操作数据。


#### 3. VO


**Value Object**：值对象，VO也是用于数据传输的对象，类似于DTO，但VO通常更加专注于视图层的数据展示。VO对象通常包含了在前端页面展示所需的数据。屏蔽掉密码、创建时间、状态等敏感信息。


#### 4. PO


**Persistant Object**：持久对象，通常用于表示与数据库中的表（或文档）相映射的Java对象。PO对象的属性对应数据库表的字段，每个PO对象通常表示数据库中的一条记录。PO对象通常用于ORM（对象关系映射）框架中，如Hibernate、MyBatis等。