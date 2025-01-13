# Auth

crypto.ts

We enrypt and decrypt things using crypto js Aes using default encyption key.

jwt.ts

We localte the appropriate apprpirate JWKS from a list of keys. A fter that we fiond the appropritae algo for the JWKSAfter that we verify jwt using jwt .verify and retuerns decrypted data in signing we give it some data whoch is encypted and returned.