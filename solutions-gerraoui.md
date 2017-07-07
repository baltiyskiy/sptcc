## Assignment 1

### Does SRSW -> MRSW construction work for multivalued registers?

Yes, since the procedure does not depend on the values.

#### Does SRSW -> MRSW construction  work for regular registers?

Yes. Having single writer is crucial, because all writes happen-before one another. If a read happens not concurrently with that write, all regular SRSW registers will have consistent state equal to the last written value. If a read happens concurrently with a write, then every SRSW register can return either the old or the new value, thus any reader can observe only either the new or the old value, which is consistent with the regular register guarantee.

#### Does SRSW -> MRSW construction  work for atomic registers?

No. That's because each SRSW register has a linearization point on its own, which are not restricted to happen simultaneously. Thus there is no single linearization point for the  resulting MRSW register, thus it's not atomic.

## Assignment 2

