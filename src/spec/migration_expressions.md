https://internetcomputer.org/docs/motoko/language-manual#migration-expressions

Migration expressions
Actors and actor class declaration may specify a migration expression, using an optional, leading <parenthetical> expression with a required field named migration. The value of this field, a function, is applied to the stable variables of an upgraded actor, before initializing any stable fields of the declared actor.

The parenthetical expression must satisfy the following conditions:

It must be static, that is, have no immediate side effects.
Its migration field must be present and have a non-shared function type whose domain and codomain are both record types.
The domain and the codomain must both be stable.
Any field in the codomain must be declared as a stable field in the actor body.
The content type of the codomain field must be a subtype of the content type of the actor's stable field.
The migration expression only affects upgrades of the actor and is otherwise ignored during fresh installation of the actor.

On upgrade, the domain of the migration function is used to construct a record of values containing the current contents of the corresponding stable fields of the retired actor. If one of the fields is absent, the upgrade traps and is aborted.

Otherwise, we obtain an input record of stable values of the appropriate type.

The migration function is applied to the input record. If the application traps, the upgrade is aborted.

Otherwise, the application produces an output record of stable values whose type is the codomain.

The actor's declarations are evaluated in order by evaluating each declaration as usual except that the value of a stable declaration is obtained as follows:

If the stable declaration is present in the codomain, its initial value is obtained from the output record.

Otherwise, if the stable declaration is not present in the domain and is declared stable in the retired actor, then its initial value is obtained from the retired actor.

Otherwise, its value is obtained by evaluating the declaration's initalizer.

Thus a stable variable's initializer is run if the variable is not produced by the migration function and either consumed by the migration function (by appearing in its domain) or absent in the retired actor.

For the upgrade to be safe:

Every stable identifier declared with type U in the domain of the migration function must be declared stable for some type T in the retired actor, with T < U (stable subtyping).

Every stable identifier declared with type T in the retired actor, not present in the domain or codomain, and declared stable and of type U in the replacement actor, must satisfy T < U (stable subtyping).

Thses conditions ensure that every stable variable is either discarded or fresh, requiring initialization, or that its value can be safely consumed from the output of migration or the retired actor without loss of date.

The compiler will issue a warning if a migration function appears to be discarding data by consuming a field and not producing it. The warnings should be carefully considered to verify any data loss is intentional and not accidental.

Example:

// Migration expression to handle stable variable type changes
(with migration = func (old : { var stable_claim_requests : [T.ClaimRequestV1] }) : { var stable_claim_requests : [T.ClaimRequest] } {
  {
    var stable_claim_requests = Array.map<T.ClaimRequestV1, T.ClaimRequest>(
      old.stable_claim_requests,
      func (oldReq : T.ClaimRequestV1) : T.ClaimRequest {
        {
          request_id = oldReq.request_id;
          caller = oldReq.caller;
          swap_canister_id = oldReq.swap_canister_id;
          position_id = oldReq.position_id;
          token0 = oldReq.token0;
          token1 = oldReq.token1;
          status = oldReq.status;
          created_at = oldReq.created_at;
          started_processing_at = oldReq.started_processing_at;
          completed_at = oldReq.completed_at;
          retry_count = 0;  // Initialize new field to 0
          last_attempted_at = null;  // Initialize new field to null
        }
      }
    );
  }
})

shared (deployer) persistent actor class SneedLock() = this {
