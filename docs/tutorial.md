
# Tutorial

This guide provides a step-by-step tutorial for setting up automatic API versioning using Cadwyn library. I will illustrate this with an example of a User API, where we will be implementing changes to a User's address.

## Installation

```bash
pip install cadwyn
```

## A dummy setup

Here is an initial API setup where the User has a single address. We will be implementing two routes - one for creating a user and another for retrieving user details. We'll be using "int" for ID for simplicity.

The first API you come up with usually doesn't require more than one address -- why bother?

So we create our file with schemas:

```python
from pydantic import BaseModel


class UserCreateRequest(BaseModel):
    address: str


class UserResource(BaseModel):
    id: int
    address: str
```

And we create our file with routes:

```python
from versions.latest.users import UserCreateRequest, UserResource
from cadwyn import VersionedAPIRouter

router = VersionedAPIRouter()


@router.post("/users", response_model=UserResource)
async def create_user(payload: UserCreateRequest):
    return {
        "id": 83,
        "address": payload.address,
    }


@router.get("/users/{user_id}", response_model=UserResource)
async def get_user(user_id: int):
    return {
        "id": user_id,
        "address": "123 Example St",
    }
```

## Turning address into a list

During our development, we have realized that the initial API design was wrong and that addresses should have always been a list because the user wants to have multiple addresses to choose from so now we have to change the type of the "address" field to the list of strings.

```python
from pydantic import BaseModel, Field


class UserCreateRequest(BaseModel):
    addresses: list[str] = Field(min_items=1)


class UserResource(BaseModel):
    id: int
    addresses: list[str]
```

```python
@router.post("/users", response_model=UserResource)
async def create_user(payload: UserCreateRequest):
    return {
        "id": 83,
        "addresses": payload.addresses,
    }


@router.get("/users/{user_id}", response_model=UserResource)
async def get_user(user_id: int):
    return {
        "id": user_id,
        "addresses": ["123 Example St", "456 Main St"],
    }
```

But every user of ours will now have their API integration broken. To prevent that, we have to introduce API versioning. There aren't many methods of doing that. Most of them force you to either duplicate your schemas, your endpoints, or your entire app instance. And it makes sense, really: duplication is the only way to make sure that you will not break old versions with your new versions; the bigger the piece you duplicating -- the safer. Of course, the safest being duplicating the entire app instance and even having a separate database. But that is expensive and makes it either impossible to make breaking changes often or to support many versions. As a result, either you need infinite resources, very long development cycles, or your users will need to often migrate from version to version.

Stripe has come up [with a solution](https://stripe.com/blog/api-versioning): let's have one latest app version whose responses get migrated to older versions and let's describe changes between these versions using migrations. This approach allows them to keep versions for **years** without dropping them. Obviously, each breaking change is still bad and each version still makes our system more complex and expensive, but their approach gives us a chance to minimize that. Additionally, it allows us backport features and bugfixes to older versions. However, you will also be backporting bugs, which is a sad consequence of eliminating duplication.

Cadwyn is heavily inspired by this approach so let's continue our tutorial and now try to combine the two versions we created using versioning.

## Creating the Migration

We need to create a migration to handle changes between these versions. For every endpoint whose `response_model` is `UserResource`, this migration will convert the list of addresses back to a single address when migrating to the previous version. Yes, migrating **back**: you might be used to database migrations where we write upgrade migration and downgrade migration but here our goal is to have an app of latest version and to describe what older versions looked like in comparison to it. That way the old versions are frozen in migrations and you can **almost** safely forget about them.

```python
from pydantic import Field
from cadwyn.structure import (
    schema,
    VersionChange,
    convert_response_to_previous_version_for,
    RequestInfo,
    ResponseInfo,
)


class ChangeAddressToList(VersionChange):
    description = (
        "Change user address to a list of strings to "
        "allow the user to specify multiple addresses"
    )
    instructions_to_migrate_to_previous_version = [
        # You should use schema inheritance if you don't want to repeat yourself in such cases
        schema(UserCreateRequest).field("addresses").didnt_exist,
        schema(UserCreateRequest).field("address").existed_as(type=str, info=Field()),
        schema(UserResource).field("addresses").didnt_exist,
        schema(UserResource).field("address").existed_as(type=str, info=Field()),
    ]

    @convert_request_to_next_version_for(UserCreateRequest)
    def change_address_to_multiple_items(request: RequestInfo):
        request.body["addresses"] = [request.body.pop("address")]

    @convert_response_to_previous_version_for(UserResource)
    def change_addresses_to_single_item(response: ResponseInfo) -> None:
        response.body["address"] = response.body.pop("addresses")[0]
```

See how we are popping the first address from the list? This is only guaranteed to be possible because we specified earlier that `min_items` for `addresses` must be `1`. If we didn't, then the user would be able to create a user in a newer version that would be impossible to represent in the older version. I.e. If anyone tried to get that user from the older version, they would get a `ResponseValidationError` because the user wouldn't have data for a mandatory `address` field. You need to always keep in mind tht API versioning is only for versioning your **API**, your interface. Your versions must still be completely compatible in terms of data. If they are not, then you are versioning your data and you should really go with a separate app instance. Otherwise, your users will have a hard time migrating back and forth between API versions and so many unexpected errors.

See how we added a migration not only for response but also for request? This will allow our business logic to stay completely the same, no matter which version it was called from. Cadwyn will always give your business logic the request model from the latest version or from a custom schema [if you want to](#internal-request-schemas).

## Grouping Version Changes

Finally, we group the version changes in the `VersionBundle` class. This represents the different versions of your API and the changes between them. You can add any "version changes" to any version. For simplicity, let's use versions 2002 and 2001 which means that we had a single address in API in 2001 and added addresses as a list in 2002's version.

```python
from cadwyn.structure import Version, VersionBundle
from datetime import date
from contextvars import ContextVar

api_version_var = ContextVar("api_version_var")

versions = VersionBundle(
    Version(date(2002, 1, 1), ChangeAddressToList),
    Version(date(2001, 1, 1)),
    api_version_var=api_version_var,
)
```

That's it. You're done with describing things. Now you just gotta ask cadwyn to do the rest for you. We'll need the VersionedAPIRouter we used previously, our API versions, and the module representing the latest versions of our schemas.

```python
from versions import latest, api_version_var
from cadwyn import generate_code_for_versioned_packages, generate_versioned_routers

generate_code_for_versioned_packages(latest, versions)
router_versions = generate_versioned_routers(
    router,
    versions=versions,
    latest_schemas_module=latest,
)
api_version_var.set(date(2002, 1, 1))
uvicorn.run(router_versions[date(2002, 1, 1)])
```

Cadwyn has generated multiple things in this code:

* Two versions of our schemas: one for each API version
* Two versions of our API router: one for each API version

You can now just pick a router by its version and run it separately or use a parent router/app to specify the logic by which you'd like to pick a version. I recommend using a header-based router with version dates as headers. And yes, that's how Stripe does it.

Note that cadwyn migrates your response data based on the `api_version_var` context variable so you must set it with each request. `cadwyn.get_cadwyn_dependency` does that for you automatically on every request based on header value.

Obviously, this was just a simple example and cadwyn has a lot more features so if you're interested -- take a look at the reference.

## Examples

Please, see [tutorial examples](https://github.com/zmievsa/cadwyn/tree/main/tests/test_tutorial) for the fully working version of the project above.

## Important warnings

1. The goal of Cadwyn is to **minimize** the impact of versioning on your business logic. It provides all necessary tools to prevent you from **ever** checking for a concrete version in your code. So please, if you are tempted to check something like `api_version_var.get() >= date(2022, 11, 11)` -- please, take another look into [reference](#version-changes-with-side-effects) section. I am confident that you will find a better solution there.
2. I ask you to be very detailed in your descriptions for version changes. Spending these 5 extra minutes will potentially save you tens of hours in the future when everybody forgets when, how, and why the version change was made.
3. Cadwyn doesn't edit your imports when generating schemas so if you make any imports from versioned code to versioned code, I would suggest using [relative imports](https://docs.python.org/3/reference/import.html#package-relative-imports) to make sure that they will still work as expected after code generation.