# Verification

On 2026-11-19, the service test suite ran against an isolated PostgreSQL 16 instance with pgvector. **161 tests passed and 1 failed.** Passing coverage included concurrent replacement, effective and recorded time, project visibility, hybrid retrieval, and artifact handling. Retrieval tests used a test embedder. These results measure test behavior and do not establish live ranking quality or storage-provider availability.

A rollback test failed both in the suite and on an isolated rerun. In the tested case, rolling back multiple consecutive replacements left an intermediate version current instead of recovering the earlier valid version. Rollback across such a chain remains an open verification gap.
