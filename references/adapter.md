# Adapter

Make a vendor SDK, third-party library, or legacy component conform to an interface your domain code controls.

## Pressure

A vendor SDK's types, errors, or calling conventions are spreading through your domain code. Swapping the implementation would require rewriting everything. The vendor's interface does not match the interface your domain code wants.

## Solution

Define an interface in your domain that expresses the operation in your terms. Implement it with an adapter that translates between the two. This is the highest-value structural pattern -- it makes provider swapping possible and is the concrete form of the "port" in Hexagonal Architecture.

## Implementation

```ts
// Before: domain code calls S3 directly
class DocumentService {
  async upload(id: string, content: Buffer) {
    await s3.putObject({ Bucket: "docs", Key: id, Body: content });
  }
}

// After: domain depends on an interface you control
interface DocumentStore {
  save(id: string, content: Buffer): Promise<void>;
  load(id: string): Promise<Buffer>;
}

class S3DocumentStore implements DocumentStore {
  async save(id: string, content: Buffer) {
    try {
      await s3.putObject({ Bucket: "docs", Key: id, Body: content });
    } catch (err) {
      // Translate vendor errors to your own taxonomy
      if (err instanceof S3ServiceException) {
        throw new StorageError("s3", err.message);
      }
      throw err;
    }
  }

  async load(id: string): Promise<Buffer> {
    const result = await s3.getObject({ Bucket: "docs", Key: id });
    return result.Body as Buffer;
  }
}
```

```go
// Go: interface at the consumer, satisfied implicitly
type DocumentStore interface {
    Save(ctx context.Context, id string, content io.Reader) error
    Load(ctx context.Context, id string) (io.ReadCloser, error)
}

type S3Store struct {
    client *s3.Client
    bucket string
}

func (s *S3Store) Save(ctx context.Context, id string, content io.Reader) error {
    _, err := s.client.PutObject(ctx, &s3.PutObjectInput{
        Bucket: &s.bucket, Key: &id, Body: content,
    })
    if err != nil {
        return fmt.Errorf("s3: %w", err)  // wrap, don't expose raw S3 error
    }
    return nil
}
```

## What a proper adapter must do (the 4 translations)

1. **Translate method names** -- `putObject()` becomes `save()`
2. **Translate types** -- vendor's `PutObjectOutput` becomes your return type
3. **Translate errors** -- vendor exceptions become your error taxonomy. This is the most commonly skipped step and the one that makes adapters real. If `S3ServiceException` escapes, nothing has been adapted.
4. **Translate semantics** -- vendor's eventual consistency becomes your consistency guarantee

## Common mistake: the leaky adapter

```ts
// Bad: returns vendor types -- callers still depend on S3 types
class S3DocumentStore implements DocumentStore {
  async save(id: string, content: Buffer): Promise<PutObjectOutput> {
    return s3.putObject({/* ... */});
  }
}
```

## Minimal cut

One interface, one adapter. Do not create an abstract base class or adapter registry. Only add multiple adapters when there is actually a second implementation.

## Cost

One more indirection layer. If you have only one provider and no plan for a second, the adapter is speculative. Document the exit plan or defer the adapter.

## Adapter vs Facade

Adapter changes an interface to match an expectation. Facade simplifies an interface nobody wants to use directly. Adapter is about compatibility, Facade about complexity.

## Related patterns

- **Hexagonal Architecture** -- Adapters are the concrete form of ports
- **Facade** -- Facade simplifies, Adapter converts
