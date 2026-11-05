# Learned data is one encrypted file, keyed from the Secret Service

User learning records words the person typed, so it must not sit on disk in plain text. We store it as a single file encrypted with XChaCha20-Poly1305, loaded into memory at startup and rewritten atomically. The key comes from the freedesktop Secret Service when one is running, and otherwise from a 0600 key file in `$XDG_CONFIG_HOME/tessera/`, kept apart from the data directory.

## Considered Options

- **SQLite / SQLCipher**: rejected. The store is a few MB at most, SQLCipher adds a C dependency, and it still leaves the same key-storage question.
- **Plain file with 0600 permissions only**: rejected. Backups, indexers and `grep` over a copied home directory would expose typed text.

## Consequences

Encryption does not protect against other processes running as the same user; they can ask the keyring for the key just as Tessera does. The key-file fallback, common on minimal Hyprland setups with no keyring, is obfuscation only, and user-facing docs must say so.
