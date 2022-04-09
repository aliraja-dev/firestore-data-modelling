# Firestore Data Modelling

Firestore data modelling is based on multiple decisions:
relationships between entity fields, optimizing reads and writes, security of the data and reducing the cost of the firestore fee.
Firestore is a NOSQL like database, in which the entire db is stored as a one large JSON object.

Embed. Model data directly on a document.
Root collection. Normalize data into separate collections, then reference document IDs.
Subcollection. Nest data in a collection under a document.
Bucket. Separate data into multiple documents, but embed as much as possible.
