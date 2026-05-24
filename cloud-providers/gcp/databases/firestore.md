# Google Cloud Firestore

> Flexible, scalable NoSQL cloud database for mobile, web, and server development.

## Overview

Cloud Firestore is a NoSQL document database built for automatic scaling, high performance, and ease of application development. It offers real-time listeners and offline support for mobile and web apps.

## Key Concepts

| Term | Definition |
|------|------------|
| Document | Unit of storage containing fields |
| Collection | Container for documents |
| Field | Key-value pair within a document |
| Reference | Pointer to a document or collection |
| Subcollection | Collection within a document |
| Query | Request for documents matching criteria |

## Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **Native** | Real-time, offline, mobile SDKs | Mobile/web apps |
| **Datastore** | Legacy API compatibility | Server-side workloads |

**Note**: Mode is set at database creation and cannot be changed.

## Data Model

```
Firestore Database
|
+-- users (collection)
|   +-- user_123 (document)
|   |   +-- name: "Alice"
|   |   +-- email: "alice@example.com"
|   |   +-- age: 30
|   |   +-- orders (subcollection)
|   |       +-- order_1 (document)
|   |       |   +-- product: "Widget"
|   |       |   +-- price: 29.99
|   |       +-- order_2 (document)
|   |
|   +-- user_456 (document)
|       +-- name: "Bob"
|       +-- email: "bob@example.com"
|
+-- products (collection)
    +-- product_789 (document)
        +-- name: "Widget"
        +-- price: 29.99
        +-- inventory: 100
```

## Document Structure

```python
# Document example
{
    "name": "Alice",
    "email": "alice@example.com",
    "age": 30,
    "address": {                    # Nested object (map)
        "street": "123 Main St",
        "city": "Seattle"
    },
    "tags": ["premium", "active"],  # Array
    "created_at": Timestamp,        # Timestamp
    "location": GeoPoint(47.6, -122.3),  # GeoPoint
    "profile_ref": Reference("/users/user_456")  # Reference
}
```

### Supported Data Types

| Type | Description |
|------|-------------|
| String | UTF-8 text |
| Number | Integer or floating-point |
| Boolean | true/false |
| Timestamp | Date and time |
| GeoPoint | Latitude/longitude |
| Reference | Pointer to another document |
| Array | Ordered list of values |
| Map | Nested object |
| Null | Null value |
| Bytes | Binary data |

## CRUD Operations

### Python SDK

```python
from google.cloud import firestore

db = firestore.Client()

# Create/Set document (overwrites)
doc_ref = db.collection('users').document('user_123')
doc_ref.set({
    'name': 'Alice',
    'email': 'alice@example.com',
    'age': 30
})

# Create with auto-generated ID
doc_ref = db.collection('users').add({
    'name': 'Bob',
    'email': 'bob@example.com'
})

# Update specific fields
doc_ref.update({
    'age': 31,
    'updated_at': firestore.SERVER_TIMESTAMP
})

# Read document
doc = db.collection('users').document('user_123').get()
if doc.exists:
    print(doc.to_dict())

# Delete document
db.collection('users').document('user_123').delete()

# Delete field
doc_ref.update({
    'age': firestore.DELETE_FIELD
})
```

### JavaScript/Web

```javascript
import { getFirestore, collection, doc, setDoc, getDoc, updateDoc, deleteDoc } from 'firebase/firestore';

const db = getFirestore();

// Create/Set
await setDoc(doc(db, 'users', 'user_123'), {
  name: 'Alice',
  email: 'alice@example.com'
});

// Read
const docSnap = await getDoc(doc(db, 'users', 'user_123'));
if (docSnap.exists()) {
  console.log(docSnap.data());
}

// Update
await updateDoc(doc(db, 'users', 'user_123'), {
  age: 31
});

// Delete
await deleteDoc(doc(db, 'users', 'user_123'));
```

## Queries

### Basic Queries

```python
# Get all documents
docs = db.collection('users').stream()
for doc in docs:
    print(doc.id, doc.to_dict())

# Filter with where
docs = db.collection('users').where('age', '>=', 21).stream()

# Multiple conditions (AND)
docs = db.collection('users') \
    .where('age', '>=', 21) \
    .where('age', '<=', 30) \
    .stream()

# Order and limit
docs = db.collection('users') \
    .order_by('age', direction=firestore.Query.DESCENDING) \
    .limit(10) \
    .stream()
```

### Query Operators

| Operator | Description |
|----------|-------------|
| `==` | Equal |
| `!=` | Not equal |
| `<` | Less than |
| `<=` | Less than or equal |
| `>` | Greater than |
| `>=` | Greater than or equal |
| `in` | In list (up to 30) |
| `not-in` | Not in list (up to 10) |
| `array-contains` | Array contains value |
| `array-contains-any` | Array contains any of values |

### Compound Queries

```python
# OR queries (requires composite index)
from google.cloud.firestore_v1.base_query import Or, FieldFilter

filter1 = FieldFilter("status", "==", "active")
filter2 = FieldFilter("status", "==", "pending")

docs = db.collection('orders').where(
    filter=Or([filter1, filter2])
).stream()
```

## Indexes

### Types

| Type | Description | Created |
|------|-------------|---------|
| **Single-field** | Index on one field | Automatic |
| **Composite** | Index on multiple fields | Manual |

### Composite Index

```yaml
# firestore.indexes.json
{
  "indexes": [
    {
      "collectionGroup": "users",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "created_at", "order": "DESCENDING" }
      ]
    }
  ]
}
```

```bash
# Deploy indexes
firebase deploy --only firestore:indexes
```

## Real-Time Listeners

```python
# Listen to document changes
def on_snapshot(doc_snapshot, changes, read_time):
    for doc in doc_snapshot:
        print(f'Document {doc.id} data: {doc.to_dict()}')

doc_ref = db.collection('users').document('user_123')
doc_watch = doc_ref.on_snapshot(on_snapshot)

# Listen to collection
col_ref = db.collection('users')
col_watch = col_ref.on_snapshot(on_snapshot)

# Stop listening
doc_watch.unsubscribe()
```

```javascript
// JavaScript
import { onSnapshot, doc } from 'firebase/firestore';

const unsubscribe = onSnapshot(doc(db, 'users', 'user_123'), (doc) => {
  console.log('Current data:', doc.data());
});

// Stop listening
unsubscribe();
```

## Transactions and Batches

### Transaction

```python
@firestore.transactional
def update_balance(transaction, account_ref, amount):
    snapshot = account_ref.get(transaction=transaction)
    new_balance = snapshot.get('balance') + amount
    transaction.update(account_ref, {'balance': new_balance})

account_ref = db.collection('accounts').document('account_1')
update_balance(db.transaction(), account_ref, 100)
```

### Batch Write

```python
batch = db.batch()

# Add operations
ref1 = db.collection('users').document('user_1')
batch.set(ref1, {'name': 'Alice'})

ref2 = db.collection('users').document('user_2')
batch.update(ref2, {'status': 'active'})

ref3 = db.collection('users').document('user_3')
batch.delete(ref3)

# Commit (atomic)
batch.commit()
```

## Security Rules

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Users can only read/write their own data
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }

    // Public read, authenticated write
    match /products/{productId} {
      allow read: if true;
      allow write: if request.auth != null;
    }

    // Validate data
    match /orders/{orderId} {
      allow create: if request.auth != null
        && request.resource.data.total > 0
        && request.resource.data.items.size() > 0;
    }

    // Subcollections
    match /users/{userId}/orders/{orderId} {
      allow read, write: if request.auth.uid == userId;
    }
  }
}
```

## Multi-Region

| Location | Regions |
|----------|---------|
| nam5 | us-central1, us-east1 |
| eur3 | europe-west1, europe-west4 |

```bash
# Create multi-region database
gcloud firestore databases create \
  --location=nam5 \
  --type=firestore-native
```

## CLI Quick Reference

```bash
# Create database
gcloud firestore databases create --location=us-central1

# Export data
gcloud firestore export gs://my-bucket/firestore-backup

# Import data
gcloud firestore import gs://my-bucket/firestore-backup

# List indexes
gcloud firestore indexes composite list

# Create index
gcloud firestore indexes composite create \
  --collection-group=users \
  --field-config=field-path=status,order=ascending \
  --field-config=field-path=created_at,order=descending

# Delete collection (use firebase-tools)
firebase firestore:delete --all-collections
```

## Pricing

| Component | Cost |
|-----------|------|
| Document reads | $0.06 per 100K |
| Document writes | $0.18 per 100K |
| Document deletes | $0.02 per 100K |
| Storage | $0.18 per GB/month |
| Network egress | Standard GCP rates |
| Free tier | 50K reads, 20K writes, 20K deletes, 1 GB storage/day |

## Exam Tips (Associate Cloud Engineer, Professional Cloud Architect)

1. **Native vs Datastore**: Mode set at creation, cannot change
2. **Real-time listeners**: For live updates in apps
3. **Composite indexes**: Required for complex queries
4. **Transactions**: Up to 500 documents
5. **Batch writes**: Up to 500 operations, atomic
6. **Security rules**: Client-side access control
7. **Subcollections**: Collections within documents
8. **Multi-region**: Higher availability, more cost
9. **TTL**: Auto-delete documents after time
10. **Export/Import**: Use GCS for backups

## Gotchas

- Mode (Native/Datastore) cannot be changed
- Composite indexes required for complex queries
- Document size limit: 1 MiB
- Maximum subcollection depth: 100
- Transactions limited to 500 documents
- OR queries require composite indexes
- Real-time listeners count as reads
- Deleting a document doesn't delete subcollections
- Indexes have creation time (can be slow)
- Security rules don't apply to server-side access

## Limits

| Resource | Limit |
|----------|-------|
| Document size | 1 MiB |
| Document nested depth | 20 levels |
| Field name length | 1,500 bytes |
| Field path length | 1,500 bytes |
| Subcollection depth | 100 |
| Writes per document | 1 per second |
| Transaction documents | 500 |
| Batch operations | 500 |
| Composite indexes per database | 200 |
| Maximum index entries per document | 40,000 |
| Array/map elements | 20,000 |
| IN/not-in values | 30/10 |
