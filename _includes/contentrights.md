### Content Rights

Content rights determine who can view or modify a document.

Efectively a document can be either:

- Public - document is available to everyone.
- Private - available only to the original publisher of content. 
- Protected - content is available to some parties, but not to others.

Whereas the meaning of "available" MUST be contextualized by one of the two distinct operations:

- Available for "read" or
- Available for "write" i.e. modification (including: removal, archival etc.).

Additionally, Collection.Doc assumes that if you have "write" access to content it implies "read" access, but not: vice versa. 

#### Role-Based Security

Collection.Doc's content rights management system design follows common "role-based" security pattern. Rights to content are granted by enrolling (i.e. listing as items) of users into a document of a "permission group" profile type and assigning permissions to those groups, rather than directly to users.

Both the content item as well as the permission group documents MUST be proper Collection.Doc documents and permissions MUST be granted by pointing "permission" link relation from a content item to the permission group document.

#### Permission Link Relation

![Docs and PermGroups relationship graph](img/docpermissions.png)

The "permission" link relation can be used to establish rights management relationships. Presence of a "permission" link relation indicates that the current resource may have access restricted. By default "read" access is open to anybody and "write" access is restricted to the original publisher of the document ("creator"). A "permission" link relation can alter those defaults and provide much flexibility in rights management.

Links using the "permission" link relation MUST point to a dereferenceable resource that SHOULD provide a resource of a known media type. Permissions link relation MAY point to multiple links.

For example, Collection.doc+JSON media type would represent permissions relationship as: 

```json
{
  "version" : "1.0"
, "attributes"    : {OBJECT}
, "links"   : {
    "profile"      : [ARRAY]
  , "permission"  : [ { "href"      : "http://api.pmp.io/docs/a54dd0e7-2e12-49aa-adf0-373e3873493a"
                       , "operation" : "read"
                       , "blacklist" : false
                       }
                     , { "href"      : "http://api.pmp.io/docs/a54dd0e7-2e12-49aa-adf0-373e3873493a"
                       , "operation" : "read"
                       , "blacklist" : true
                       }
                     , { "href"      : "http://api.pmp.io/docs/a54dd0e7-2e12-49aa-adf0-373e3873493a"
                       , "operation" : "write"
                       }]  
  }  
}
```

where:

- href: points to a dereferenceable URL returning a Collection.Doc describing a permission group.
- operation: can be "read" or "write", correspondingly granting/denying "read" or "write" permission. Any other value is invalid.
- blacklist: indicates whether permission is granted or denied. Default value is "blacklist: false", implying that permission is granted. This is also the implied value if the property is missing altogether. Typically, you would use property only if you are trying to deny access and it would be "blacklist: true". You can indicate "blacklist: false" explicitly, but it's not a recommended practice and you are encouraged to just omit the property instead.

Please notice that the "permission" link relation in the example above contains multiple links. These link relations are additive: the resulting permissions will be determined by adding-up all of the rules defined by each link relation. Following rules are applied to the addition algorithm:

#### Additivity Rules

1. If one link relation grants access, but another one denies it, the resulting access is: denied.
 
   This rule is important to guarantee publisher confidence in the rights management and avoid accidental "security holes". Typically it's more obvious, hence: easily correct-able, if somebody accidentally gets denied access than if somebody accidentally gets the rights they should not have.

1. If a user/organization has "write" access, then it always implies read access, even if another relationship denies "read" access specifically.

1. First all "write" rules are processed separately, then read rules are processed and only in the end the two are combined + conflicts are resolved.

1. Creator user/organization can never be denied either "read" or "write" access to a document. This rule trumps all others.

The resulting additivity rules matrix looks like the following:

```
r(y)        = r(y) and w(undefined)
w(y)        = w(y) and r(y) 
w(y) + r(n) = w(y) and r(y)
w(n) + r(y) = w(n) and r(y)
w(y) + r(y) = w(y) and r(y)
w(n) + r(n) = w(n) and r(n)
```

Where w(y) means: group has write access (y = yes), and r(n) accordingly means: group is denied read access (n = no).

w(undefined) defaults to w(n) if no other rule provides more specific value, but is a weak definition and is overridden by any specific definition whether w(y) or w(n).

In case of multi-level addition, considering rule #3:

```
w(y) + w(n) + r(y) = w(n) and r(y)
```

Please note that the w(y) rule never got a chance to grant read access since it was trumped by w(n).

Another example:

```
w(y) + r(y) + r(n) = w(y) + r(n) = w(y) and r(y)
```

In this case rule #1 governed that `r(y) + r(n) = r(n)`, but it was overridden when added-up by the end-result of w() rules, since w(y) implies r(y) according to rule #2.

#### Defaults.

1. If you omit the permission link relation altogether, it means:
    - Document is accessible for "read" to anybody
    - Document is accessible for "write" only to the creator and distributors.
2. To indicate that a document is view-able only by the creator and distributors, you need to create a group that only contains the creator or distributors, and specify it as a read whitelist:

    ```json
    { "href"      : "https://api.pmp.io/docs/3709eda6-0c57-4f67-ab8f-efddb641297d"
    , "operation" : "read"
    } 
    ```
where `3709eda6-0c57-4f67-ab8f-efddb641297d` is the guid of the document that defines the creator. Creators and distributors always have read and write permissions on a document, so this just serves to prevent others from having access.

#### A Blacklist Without a Whitelist.

This is an important use-case worth discussing separately. 

While technically valid, it rarely makes any sense to define a blacklist without defining a whitelist group regardless of an operation type. 

For "write" operations, not having a "whitelist" defined means that the default write behavior applies, where only the creator is allowed to modify content. Creator cannot be denied access, so why define an extraneous "blacklist"?

For "read" operations, not having an explicit "whitelist" means that the default read behavior applies, where any PMP user with an API Key (or alternative valid authentication token) can "read" a document. By defining a blacklist you can restrict specific users/organizations, but since they can probably easily register as another user - are you really securing anything?

Defining a "blacklist" permission relation without a "whitelist" permission relation also defined on a document is usually a ["smell"](http://en.wikipedia.org/wiki/Code_smell) of potential misconfiguration. It is not invalid, since it is not technically wrong, but publishers should be careful and treat such definitions as indications of a possible error.

#### Rights Management Workflow

To modify permissions on a document, it must be retrieved as a whole, permissions altered and re-saved. There may be a support implemented for PATCH in the future, but no such feature is planned for the current version.


Permissions must be specified on the story itself during a publish PUT or POST. Server performs following processing steps:

1. receive document
1. do immediate validation checks
  * do permission groups referenced exist?
  * do "additive" math of all "permissions" link relations and calculate resulting permissions for each group. 
1. build denormalized list of allowed organizations (if not public)
1. attach list of allowed organizations to the document
1. attach list of allowed organizations to the search document
1. index search document