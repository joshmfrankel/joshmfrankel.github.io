---
layout: post
title: Customizing Database Table, Column, and Association Naming in Prisma ORM
categories:
  - articles
tags:
  - prisma
  - databases
  - javascript
---

I've been working with Prisma as an object-relational mapping tool for my projects. Coming from a background of using raw SQL along with ActiveRecord, I've noticed that default Prisma caters to JavaScript over other established standards. Ensuring database table columns are snake_case along with creating associations that are lowercase and plural doesn't come for free but Prisma does provide a way to configure these within your schema.

<!--excerpt-->

## Prisma Schema

[Prisma](https://www.prisma.io/) provides a schema to help define the model and database layers of your application. By definition it is mapping the concept of a Model to a database table. Now out-of-the-box if you were to create a new table your schema may look something like this. We'll use an example of a blog application where a User can have many Posts.

```js
model User {
  id            String    @id @default(uuid())
  email         String    @unique
  emailVerified DateTime?
  name          String?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
}

model Posts {
  id        String   @id @default(uuid())
  title     String
  content   String?
  publishedAt DateTime?
  authorId  String
  author    User     @relation(fields: [authorId], references: [id])
}
```

Now migrating this schema you'd end up with a **Users** and **Posts** table with that casing. Generally this isn't an issue but the general best practice is to use plural snake_case conventions for table names. Additionally, several of the columns like **publishedAt**, **authorId**, and **emailVerified** follow camelCase conventions which typically isn't found at the database level.

## The @map and @@map directives

Prisma provides two Schema directives that allow you to control the naming conventions of your table names along with column names. We can use the `@map` directive to control the column names to adhere to snake_case conventions.

{% include blockquote.html quote="@map: Maps a field name or enum value from the Prisma schema to a column or document field with a different name in the database. @@map: Maps the Prisma schema model name to a table (relational databases) or collection (MongoDB) with a different name." source_link="https://www.prisma.io/docs/orm/reference/prisma-schema-reference#map" source_text="Prisma Documentation" %}

```js
model User {
  id            String    @id @default(uuid())
  email         String    @unique
  emailVerified DateTime? @map("email_verified")
  name          String?
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")
}

model Posts {
  id        String   @id @default(uuid()) @map("id")
  title     String
  content   String?
  publishedAt DateTime? @map("published_at")
  authorId  String @map("author_id")
  author    User     @relation(fields: [authorId], references: [id])
}
```

This has the benefit of connecting the model to the database table while preserving the correct naming conventions in both use-cases. The model can still utilize camelCase to interact at the JavaScript layer, while the database table uses the more commonly found snake_case equivlent.

For table naming, the `@@map` directive can be used in much the same way. We'll use it to ensure that our database tables are named **users** and **posts** respectively.

```js
model User {
  id            String    @id @default(uuid())

  @@map("users")
}

model Posts {
  id        String   @id @default(uuid())

  @@map("posts")
}
```

We now have SQL that uses database naming conventions while the Prisma models use camelCase conventions to better integrate with the JavaScript language.

## Associations

This same approach can also be used for associations. For example, if there were a join table between Users and Posts, you could name the association with camelCase conventions. In this case, @map and @@map are not needed since Prisma provides a built-in mechanism to define the relationship. This relationship is virtual and does not exist in the database layer.

```js
model User {
  id            String    @id @default(uuid())

  userPosts     Post[]
}

// Prisma used elsewhere in the codebase
// Collection of Posts associated with the User
user.userPosts
```

## Conclusion

And with all of that you can keep both the Database and JavaScript layers focused on their own conventions.

Know about a trick with Prisma? Let me know in the comments below!
