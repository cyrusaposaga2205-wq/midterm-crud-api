# Midterm CRUD API

## Reflection

### 1. What was the most challenging part of building this API?

Getting TypeORM and TypeScript configured correctly was the most challenging part. Small mistakes—such as a typo in `.env` (`DB_PASSWOD` instead of `DB_PASSWORD`), a missing `experimentalDecorators` setting in `tsconfig.json`, or an incorrect import path for `data-source.ts`—caused confusing errors that were difficult to trace. Once the setup was correct, implementing the CRUD operations was straightforward.

### 2. How does TypeORM make database operations easier than raw SQL?

- **Less SQL to write:** Use repository methods such as `find()`, `save()`, and `remove()` instead of writing common `SELECT`, `INSERT`, and `DELETE` statements yourself.
- **Type safety:** TypeScript can catch many mistakes at compile time.
- **Automatic timestamps:** `@CreateDateColumn()` and `@UpdateDateColumn()` manage `createdAt` and `updatedAt` values.
- **Relationships:** Decorators such as `@ManyToOne` and `@OneToMany` describe related entities.
- **Safer queries:** Bound parameters help protect queries from SQL injection.
- **Database portability:** TypeORM supports multiple database engines, though switching databases may require configuration and schema changes.

### 3. Why exclude the password field from API responses, and how is it done?

Even a hashed password is sensitive. If exposed, it could be used in offline cracking attempts. Clients do not need the password returned, so the API should never include it in responses.

The password can be hidden from default queries in the entity:

```ts
@Column({ length: 255, select: false })
password!: string;
```

Then omit it when building the response:

```ts
const { password: _, ...userWithoutPassword } = savedUser;
res.json({ success: true, data: userWithoutPassword });
```

### 4. When would you need a database transaction?

A transaction is useful when several database changes must succeed or fail together. For example, registering a user and creating their default workspace should be atomic: if either step fails, neither should be saved.

```ts
await AppDataSource.transaction(async (manager) => {
  const user = await manager.save(User, { name, email, passwordHash });
  await manager.save(Workspace, {
    name: `${name}'s workspace`,
    ownerId: user.id,
  });
});
```

Other examples include money transfers, creating an order and its payment, and updating stock levels.

### 5. `synchronize: true` versus migrations

| | `synchronize: true` | Migrations |
|---|---|---|
| What it does | Updates the database schema from entities at startup | Applies explicit, versioned schema changes |
| Data safety | Can drop columns and cause data loss | Changes can be reviewed and controlled |
| Versioned in Git | No | Yes |
| Best suited for | Local development | Production |

Use `synchronize: true` only during development. Use migrations in production. For example, renaming an entity column while synchronization is enabled may cause the old column—and its data—to be dropped.

### 6. How would you add a `Product` entity related to `User`?

Create the product entity:

```ts
@Entity('products')
export class Product {
  @PrimaryGeneratedColumn()
  id!: number;

  @Column({ length: 150 })
  name!: string;

  @Column('decimal', { precision: 10, scale: 2 })
  price!: number;

  @ManyToOne(() => User, (user) => user.products, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'userId' })
  user!: User;

  @Column()
  userId!: number;
}
```

Add the inverse relationship to `User`:

```ts
@OneToMany(() => Product, (product) => product.user)
products!: Product[];
```

### 7. What security improvements are needed before production?

- Hash passwords with bcrypt or another suitable password-hashing algorithm. **The current implementation stores passwords as plaintext.**
- Add authentication, such as JWT, so users must sign in before using protected routes.
- Add authorization so users can only edit records they are allowed to access.
- Add rate limiting to slow brute-force attacks.
- Validate request data with a library such as Zod or Joi.
- Restrict CORS to trusted origins instead of allowing every origin with `cors()`.
- Add Helmet to set security-related HTTP headers.
- Disable `synchronize` in production and use migrations.
- Avoid logging secrets or sensitive request data. Error objects may contain more information than intended.
- Hide stack traces in production error responses.
- Serve the app over HTTPS.
- Add centralized error handling rather than relying on separate `try`/`catch` blocks in each route.
- Paginate the `getAll` endpoint to limit how many records it returns at once.
- Make `/health` check database connectivity, for example with `SELECT 1`.
- Rotate the MySQL password if it has ever been committed to Git.
