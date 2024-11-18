## How to store passwords in the database

Do not store as plain text. OWASP recommendations.

1. hash the password. Do not use MD5, SHA1 even though they are fast. bcrypt: slow.
2. Salt the password. Unique and random generated string.

Precomputation attacks: rainbow tables and database lookups are used to defeat the one-way hashed passwords.

hash(password+salt): store in database (id,salt,hash)

## References

1. byte byte go YouTube video
2. https://en.wikipedia.org/wiki/Salt_(cryptography)
3. https://newsletter.systemdesign.one/p/how-to-store-passwords-in-database
