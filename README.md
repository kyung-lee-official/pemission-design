# Permission Design

## Permission Granularity and Strategy

Permission Design can be complicated. Most cases, simply applying CRUD permissions to a generic resources is not enough. For example, **member** is a generic resource, a member may have a field called `isFrozen` that cannot be updated by the member herself, but can be updated by the admin. This is a **field-specific** permission. It's inappropriate to use a permission like `["update"] member` to control this field, because you may not want to allow a scope owner to be able to update the `isFrozen` field of members of her scope, which should be controlled by the admin only, imagine a frozen scope owner can unfreeze herself, this is a security issue.

A solution to this issue is to separate permissions into two types: common permissions and sentitive (field-specific) permissions. Common permissions can be implemented by simply applying CRUD permissions to resources, while field-specific permissions need to be treated separately. In the scenario above, we can create a permission called `["update"] member:is-frozen` to control the `isFrozen` field of a member.

Another pitfall you need to be aware of is where bidirectional relationships are involved. For example, someone who has the permission of `["update"] member` can update the member's belonging scopes, but she doesn't have the permission of `["update"] member-scope` to add or remove members from the scope. From the perspective of the member, she can update her own belonging scopes, but from the perspective of the scope, she can't update the scope's members. This makes the permission system inconsistent. To avoid this situation, we must remove the ability of updating the member's belonging scopes from the `["update"] member` permission, and consist with the `["update"] member-scope` permission only to control who belongs to which scope.

We also need something we can used to limit the scope of a permission, **member-scope** therefore comes in handy. For example, `["update"] member:is-frozen` permission can be limited to a member-scope, so that a scope owner can only update the `isFrozen` field of a member of her scope. Logically, the owner should not be able to freeze herself, this can be done by using two strategies: either use a 3rd party permission management library like CASL (conditional permissions), or add condition codes to the service that implements the conditional permission manually, so that the owner can't freeze herself.

There is a rule that should be followed when designing a permission system: members with the same sensitive permissions should not be able to update each other. For example, if two members have the permission of `["update"] member:is-frozen`, they should not be able to update each other's `isFrozen` field.

## We Already Have Scope, Why still Need Role?

Imagine we have two branch offices, East and West, finance department in East and West share the same role `finance department` which has permission of `["update"] finance:balance`, scopes help us avoiding the finance department in East to update the balance of the finance department in West, if we hope to change the permission of role `finance department`, we don't need to change twice in East and West respectively.

Basically 'role' defines a set of permissions, it doesn't have an owner. 'scope' defines a set of members, it has an owner. 'role' can be assigned to 'scope'.

|                          | Scope: West            | Scope: East            |
| ------------------------ | ---------------------- | ---------------------- |
| Role: finance department | Can apply to West only | Can apply to East only |

## Scenario

Let's say we have an alpha scope, and Alice is the scope owner. There are several members in alpha scope, Bob is one of them. Meanwhile, Bob is the owner of beta scope, Alice therefore is a supdervisor of beta scope. Charlie is the owner of gamma scope, and Charlie is a member of beta scope as well, however, we wanna make gamma scope independent of alpha scope, only managed by Charlie and admin.

```
alpha (Alice)
  |--- beta (Bob)
        |--- gamma (Charlie), independent of alpha
```

## Why Hierarchical Permissions?

Alice is the supdervisor of both alpha and beta scope, if we use non-hierarchical scopes only, we will need to create: alpha scope, beta scope, and an alpha-beta scope that contains all members from alpha and beta, and assign Alice to the owner of the scope. Now if a member is removed from beta, we will need to remove the member from alpha-beta scope as well, this is a lot of work and could be error-prone.

Let's try to resolve this by adding more properties to the scope. First we introduce `superScope`, so that beta scope can has alpha scope as its superScope. Then `scopeOwner`. Now we can assign alpha to the superScope of beta, but do not assign beta to the superScope of gamma. This way, Alice can only manage members in alpha and beta, but not in gamma.

## Permission Inheritance

There is a rule we should carry through when designing hierarchical permissions, that hierarchical permissions only provides relations, it can only answer questions like 'if alpha scope is a superScope of beta scope?', it doesn't determine 'if Alice can freeze Bob?', just because Bob is a member of alpha scope doesn't necessarily mean Alice can perform any operations on Bob. Decisions like this should be made by business-specific permissions. For example, if you try to implements this with NestJS and Cerbos, you should send the relations of scopes of Alice and Bob to Cerbos, and Cerbos will be responsible for determining if Alice can freeze Bob.

# FAQ

-   Why it's inappropriate to desgin roles same way as the actual organization structure?

    Because the organization structure doesn't always match the practical needs of business, for example, a manager may be assigned to a certain confidential and independent project that shouldn't even be known by her supdervisor.

-   How to avoid circular superScope?

    Simple answer: No. Although our data structure can implement circular superScope technically, and it's possible that Alice leads project alpha where Bob is involved, and Bob leads project beta where Alice is involved. However, circular relations can't be represented in a tree structure, which should be avoided currently.

-   Is it allowed to have multiple superScopes?

    No, `superScope` is a to-one relation, it's not an array. If you need to implement a logic similar to multiple superScopes, consider creating a new scope and apply ABAC to it (for example, only Alice and Bob can apply changes to the new scope), keep in mind that this is a business-specific permission, the permission logic which should be implemented in the PDP side, not hardcoded in the NestJS's guard. Also, circular relations should be avoided as possible.

# References

-   [Role-based access control (RBAC)? | Cerbos](https://www.cerbos.dev/features-benefits-and-use-cases/rbac)

-   [What is Attribute-Based Access Control (ABAC)? | Cerbos](https://www.cerbos.dev/features-benefits-and-use-cases/abac)

-   [How to Design a Permissions Framework](https://rinaarts.com/how-to-design-a-permissions-framework/)
