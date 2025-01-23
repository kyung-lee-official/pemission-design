# Permission Design

## Permission Granularity and Strategy

Permission Design can be complicated. Most cases, simply applying CRUD permissions to a generic resources is not enough. For example, **member** is a generic resource, a member may have a field called `isFrozen` that cannot be updated by the member herself, but can be updated by the admin. This is a **field-specific** permission. It's inappropriate to use a permission like `["update"] member` to control this field, because you may not want to allow a super-role to be able to update the `isFrozen` field of members of its sub-role, which should be controlled by the admin only, imagine a frozen member can unfreeze herself, this is a severe security issue.

A solution to this issue is to separate permissions into two types: common permissions and sentitive (field-specific) permissions. Common permissions can be implemented by simply applying CRUD permissions to resources, while field-specific permissions need to be treated separately. In the scenario above, we can create a permission called `["update"] member:is-frozen` to control the `isFrozen` field of a member.

Another pitfall you need to be aware of is where bidirectional relationships are involved. For example, someone who has the permission of `["update"] member` can update the member's belonging roles, but she doesn't have the permission of `["update"] role` to add or remove members from the role. From the perspective of the member, she can update her own belonging roles, but from the perspective of the role, she can't update the role's members. This makes the permission system inconsistent. To avoid this situation, we should not expose the API to update the member's belonging roles in the `["update"] member` permission, and consist with the `["update"] role` permission only to control who belongs to which role.

Logically, one could not freeze herself, this should be implemented in the PDP side. Your backend code should only provides `principal`, `action`, `resource` and `context (relationship)` to PDP and let PDP to determine if the operation is allowed.

There is a rule we should follow when designing a permission system: members with the same permissions should not be able to update each other. For example, if two members have the permission of `["update"] member:is-frozen`, they should not be able to freeze each other.

## Role Structure

[Discord](https://discord.com/)'s role management system is a typical RBAC system, where a role associates with a set of permissions, and a group of members.

Should we have an owner of a role? No, because it is common to check whether a member is a supervisor of other members, if a role has an owner, we will need to check two things: if the member is the owner of the role, or if the member is a member of the role's super-role. This is redundant and unnecessary. It's recommended to create a super-role if you want to have a role that can be managed by a member.

Let's say we have the following roles: super-alpha, alpha, super-beta, beta, super-gamma, gamma, and admin. The hierarchy is as follows:

```
admin
  |--- super-alpha (Alice)
  | |--- alpha (..., Bob)
  | |--- super-beta (Bob)
  |   |--- beta (..., Charlie)
  |--- super-gamma (Charlie)
    |--- gamma
```

Alice is assigned to super-alpha, there are serveral members in alpha, Bob is one of them, Bob is also assigned to super-beta, which is a super-role of beta. Charlie is a member of beta, and Charlie is also assigned to super-gamma, which is a super-role of gamma.

Regarding to different resource types, we could define permission rules separately.

Say we have a resource `accessment`,

```typescript
type Accessment = {
	id: string;
	score: number;
	/* member id */
	owner: string;
};
```

Everyone has her own accessment, only super-roles can update the accessment, both super-roles and herself can read the accessment.

What we try to achieve is that admin can `["read", "update"] accessment` of super-alpha and its sub-roles. Alice can `["read"] accessment` of super-alpha, and `["read", "update"] accessment` of alpha and its sub-roles, Bob can `["read"] accessment` of super-beta, and `["read", "update"] accessment` of beta and its sub-roles, and same for Charlie.

If Alice try to update Charlie's accessment, the backend will first need to get Charlie's roles (Charlie may have multiple roles), then traverse Charlie's roles to find the super-role of Charlie, and check if Alice belongs to a super-role of Charlie, if so, we can send the following info to the PDP:

```typescript
/* this is a simplified version of the request */
const request = {
	principal: Alice,
	action: "update",
	resource: "accessment",
	context: {
		isPrincipleSuperRole: true,
	},
};
```

## Why Hierarchical Permissions?

Let's assume each role has a manager, and we want David to be the supdervisor of both A-role and B-role, if we use non-hierarchical role only, we will need to create: A-role, B-role, and C-role that contains all members from A-role and B-role, and appoint David as the manager of the C-role. Now if a member was removed from B-role, we will have to to remove the member from C-role as well, this is a lot of work and could be error-prone.

Now we introduce `superRole`, so that A-role and B-role can have C-role as its `superRole`. Now if we make changes to A-role or B-role, the changes will be automatically applied to C-role.

## Permission Inheritance

There is a rule we should carry through when designing hierarchical permissions, that hierarchical permissions only provides relations, it can only answer questions like 'if alpha role is a superRole of beta role?', it doesn't determine 'if Alice can freeze Bob?', just because Bob is a member of beta role doesn't necessarily mean Alice can perform any operations on Bob. Decisions like this should be made by business-specific permissions, which should be defined in the PDP side. For example, if you try to implements this with NestJS and Cerbos, you should send the relations of roles of Alice and Bob to Cerbos, and Cerbos will be responsible for determining if Alice can freeze Bob.

# FAQ

-   Why it's inappropriate to desgin roles same way as the actual organization structure?

    Because the organization structure doesn't always match the practical needs of business, for example, someone may be assigned to a certain confidential and independent project that shouldn't even be known by her direct supdervisor.

-   How to avoid circular superRole?

    Simple answer: No, we don't restrict circular superRole currently, but you should avoid it as possible. Although technically, our data structure can implement circular superRole, and it's possible that Alice leads project alpha where Bob is involved, and Bob leads project beta where Alice is involved. However, circular relations can't be represented in a tree structure, and it may cause infinite loops when traversing the tree. It's recommended to avoid circular superRole.

-   Is it allowed to have multiple superRoles?

    No, `superRole` is a to-one relation, it's not an array. If you need to implement a logic similar to multiple superRoles, consider apply ABAC to it (for example, only Alice and Bob can apply changes to the role), keep in mind that this is a business-specific permission, the permission logic which should be implemented in the PDP side, not hardcoded in the backend.

# References

-   [Role-based access control (RBAC)? | Cerbos](https://www.cerbos.dev/features-benefits-and-use-cases/rbac)

-   [What is Attribute-Based Access Control (ABAC)? | Cerbos](https://www.cerbos.dev/features-benefits-and-use-cases/abac)

-   [How to Design a Permissions Framework](https://rinaarts.com/how-to-design-a-permissions-framework/)
