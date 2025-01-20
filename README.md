# Permission Design

## Permission Granularity and Strategy

Permission Design can be complicated. Most cases, simply applying CRUD permissions to a generic resources is not enough. For example, **member** is a generic resource, a member may have a field called `isFrozen` that cannot be updated by the member herself, but can be updated by the admin. This is a **field-specific** permission. It's inappropriate to use a permission like `["update"] member` to control this field, because you may not want to allow a scope owner to be able to update the `isFrozen` field of members of her scope, which should be controlled by the admin only, imagine a frozen scope owner can unfreeze herself, this is a security issue.

A solution to this issue is to separate permissions into two types: common permissions and sentitive (field-specific) permissions. Common permissions can be implemented by simply applying CRUD permissions to resources, while field-specific permissions need to be treated separately. In the scenario above, we can create a permission called `["update"] member:is-frozen` to control the `isFrozen` field of a member.

Another pitfall you need to be aware of is where bidirectional relationships are involved. For example, someone who has the permission of `["update"] member` can update the member's belonging scopes, but she doesn't have the permission of `["update"] member-scope` to add or remove members from the scope. From the perspective of the member, she can update her own belonging scopes, but from the perspective of the scope, she can't update the scope's members. This makes the permission system inconsistent. To avoid this situation, we must remove the ability of updating the member's belonging scopes from the `["update"] member` permission, and consist with the `["update"] member-scope` permission only to control who belongs to which scope.

We also need something we can used to limit the scope of a permission, **member-scope** therefore comes in handy. For example, `["update"] member:is-frozen` permission can be limited to a member-scope, so that a scope owner can only update the `isFrozen` field of a member of her scope. Logically, the owner should not be able to freeze herself, this can be done by using two strategies: either use a 3rd party permission management library like CASL (conditional permissions), or add condition codes to the service that implements the conditional permission manually, so that the owner can't freeze herself.

There is a rule that should be followed when designing a permission system: members with the same sensitive permissions should not be able to update each other. For example, if two members have the permission of `["update"] member:is-frozen`, they should not be able to update each other's `isFrozen` field.

## Scope vs Hierarchical Permissions

### Scenario

Let's say we have an alpha scope, and Alice is the scope owner. There are several members in alpha scope, Bob is one of them. Meanwhile, Bob is the owner of beta scope, Alice therefore is a supdervisor of beta scope. Charlie is the owner of gamma scope, and Charlie is a member of beta scope as well, however, we wanna make gamma scope independent of alpha scope, only managed by Charlie and admin.

**Why Hierarchical Permissions**? Alice have access to alpha and beta scope, if we use scope only, we will need to create: alpha scope, beta scope, and an alpha-beta scope that contains all members from alpha and beta, and assign Alice to the owner of the scope. Now if a member is removed from beta, we will need to remove the member from alpha-beta scope as well, this is a lot of work and error-prone.

Let's try to resolve this by adding more properties to the scope. First we introduce `superScope`, so that beta scope has alpha scope as its superScope. Then `scopeOwner`. Now we can assign alpha to the superScope of beta, and do not assign alpha to the superScope of gamma. This way, Alice can only manage members in alpha and beta, but not in gamma.

**Why we still need role to manage permissions**? Imagine we have two branch offices, East and West, finance department in East and West share the same role `finance department` which has permission of `["update"] finance:balance`, scopes help us avoiding the finance department in East to update the balance of the finance department in West, if we hope to change the permission of role `finance department`, we don't need to change twice in East and West respectively.

# References

-   [How to Design a Permissions Framework](https://rinaarts.com/how-to-design-a-permissions-framework/)
