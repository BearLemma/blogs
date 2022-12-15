Inspired by the recent success of tRPC and in preparation of a ChiselStrike example for
Tanstack, we have felt the need to enrich ChiselStrike with an ability to generate heavily
typed TS client code.

The idea is to analyze user-defined route handlers, extract information about what route,
query parameters and JSON body go in and what response goes out. Then use this information to
generate a typed client that can be used to communicate with our backend endpoints. Doing
this in full generality is going to require full-blown type analysis with tsc so as an MVP,
we shall start by generating clients for our CRUD endpoints.

In this series of blog posts, I'll go through the process of
implementing this functionality and together we will examine some of the aspects of this
endavor that I found interesting. If nothing else, one big takeaway of the story is that
code generation sounds scary, but doesn't have to be if done efficiently.

## Target outline

To ilustrate what is the goal, let's consider a ChiselStrike project with two (heavily simplified) entities:

```typescript
// models/blog_post.ts
export class User extends ChiselEntity {
    name: string;
    karma: number;
}
```

```typescript
// models/blog_post.ts
export class BlogPost extends ChiselEntity {
    text: string;
    tags: string[];
    publishedAt: Date;
    author: Id<User>;
}
```

and a routing map exposing CRUD endpoints for both entities `User` and `BlogPost`.

```typescript
// routes/index.ts
export default new RouteMap()
    .prefix("/users", User.crud())
    .prefix("/blog/posts", BlogPost.crud())
```

Note: ChiselStrike provides a convenience static method `crud()` which returns RouteMap
containing all CRUD enpoints for given entity. For more info see (TODO: ADD LINK TO DOCS).

When desining a new interface, I usually start from what the usage should look like for an example app and worry about the implementation afterwards. So let's do that using for our example backend.

First we will need a way of creating the client object:

```typescript
const cli = createChiselClient(serverUrl);
```

`createChiselClient` will take a serverUrl of ChiselStrike backend server with which it will communicate.

Then I would like to use the `client` object to create a new user and then his first
blog post:

```typescript
const user: User = await cli.users.post({
    name: "ILikeTrains",
    age: 17,
});

const post: BlogPost = await cli.blog.posts.post({
    text: "To like or not to like Trains, that is the question!",
    tags: ["lifeStyle"]
    publishedAt: new Date(),
    author: user.id,
});
```

Notice how individual route segments map onto the client fields (`/blog/posts` -> `.blog.
posts`). But our CRUD paths can contain wildcards to capture variables. For example, to patch
a specific blog post, you would issue a patch agains `/blog/posts/:id/`. This functionality
can be implemented by introducing functions into the call chain. Let's ilustrate this
by PATCHing our blog post's tags:

```typescript
const updatedPost: BlogPost = await cli.blog.posts.id(post.id).patch({
    tags: ["lifeStyle", "philosophy"]
});
```

## Models generation

From those examples, it's apparent that if we want type safety, we will need to import
the Entity types (`User` and `BlogPost`) from somewhere and then use them in our client.
But that's easy enough. ChiselStrike already knows everything about Entities and their
fields. That means we can just query the entity information from chisel server and then
store all them in a dedicated file `models.ts`.

The generated entity types will be slightly different than the original ChiselStrike entities
because for the client, we don't need anything as complex as class and you may also notice
the addition of `id` field which is inherited from `ChiselEntity`.

Put together, we can use the following code:

```rust
fn generate_models(version_def: &VersionDefinition) -> Result<String> {
    let mut output = String::new();
    for def in &version_def.type_defs {
        writeln!(output, "export type {} = {{", def.name)?;
        for field in &def.field_defs {
            let field_type = field.field_type()?;
            writeln!(
                output,
                "    {}{}: {};",
                field.name,
                if field.is_optional { "?" } else { "" },
                type_enum_to_code(field_type)?
            )?;
        }
        writeln!(output, "}}")?;
    }
    Ok(output)
}

fn type_enum_to_code(type_enum: &TypeEnum) -> Result<String> {
    let ty_str = match &type_enum {
        TypeEnum::ArrayBuffer(_) => "ArrayBuffer".to_owned(),
        TypeEnum::Bool(_) => "boolean".to_owned(),
        TypeEnum::JsDate(_) => "Date".to_owned(),
        TypeEnum::Number(_) => "number".to_owned(),
        TypeEnum::String(_) | TypeEnum::EntityId(_) => "string".to_owned(),
        TypeEnum::Array(element_type) => {
            format!("{}[]", type_enum_to_code(element_type)?)
        }
        TypeEnum::Entity(entity_name) => entity_name.to_owned(),
    };
    Ok(ty_str)
}
```

For those of you thinking "Whoooha, that's a Rust code! I haven't signed up for that!"
don't worry if you don't understand everything. We will write surprisingly little amount
of Rust to do the generation.

In any case, the above code will generate the following types:

```typescript
// models.ts
export type User = {
    id: string;
    name: string;
    karma: number;
}

export type BlogPost = {
    id: string;
    text: string;
    tags: string[];
    publishedAt: Date;
    author: string;
}
```

That wasn't so bad, was it? Addmitedly, we saved a lot of work by pulling the already parsed
types from chiseld backend. In the future, I want to drop that dependency, but it will do
for now.

## Client object

Now that we have the types, we can focus on the client itself. To make it work as ilustrated
by our example above, I think we can leverage TS's object types which we can nest together
and create the required structe. It's simple, it's typed and it will be easy to generate
compared to a bunch of classes. Let's do it:

```typescript
const client = {
    users: {
        post: (user: Omit<User, "id">) => Promise<User> {...}, ...
        id: (id: string) => {
            return {
                patch: (user: Partial<User>) => Promise<User> {...}, ...
            }
        }
    }
    blog: {
        posts: {
            post: (blogPost: Omit<BlogPost, "id">) => Promise<BlogPost> {...},
            ...
            id: (id: string) => {
                return {
                    patch: (blogPost: Partial<BlogPost>) => Promise<BlogPost> {...}, ...
                }
            }
        }
    }
}
```

Woooha, that's a lot of code! We haven't implemented any of those functions and that's not
even all of the CRUD methods! We are still missing `get`, `delete`, `put` for both variants
(singular - with id) and plain (without `id`). We will probably want to have some convenience
methods for bulk retrieval like `getIter` and `getAll` as well.

Even if we are planning on generating the code, it needs more structure. Ok, let's start off
by focusing on replacing `post: (user: Omit<User, "id">) => Promise<User> {...},` with something more reasonable.

## Function generation in TypeScript

How about instead of generating the function in Rust, we wrote a TS function that would
generate the handler for us? It's very easy to do that using arrow functions:

```typescript
function makePostOne<Entity extends Record<string, unknown>>(
    url: URL,
    entityType: reflect.Entity,
): (entity: OmitRecursively<Entity, "id">) => Promise<Entity> {
    return async (entity: OmitRecursively<Entity, "id">) => {
        const entityJson = entityToJson(entityType, entity);
        const resp = await sendJson(url, "POST", entityJson);
        await throwOnError(resp);
        return entityFromJson<Entity>(entityType, await resp.json());
    };
}
```

From the code it's perfectly obvious what doe function does, isn't? Ha! Of course not!
So let's explain what's going on here.

The `makePostOne` function takes `url` and `entityType` parameters. `url` tells us where
is the request going to be sent and we need `entityType` reflection parameter to convert
our entity to and from JSON. More on that later.

`makePostOne` returns an async arrow function which has a signiture of `(entity: OmitRecursively<Entity, "id">) => Promise<Entity>`. The `entity` parameter needs to be
`OmitRecursively<Entity, "id">` because when we are POSTing (creating) a new entity, we
don't have an ID yet. The database will provide it for us. But all of our Entity types
contain the `id` field so we need to omit it. We need to do so in a recursive manner because
our entities can be nested. We will describe `OmitRecursively` in more detail later.
The return type doesn't need such a restriction because the function returns the newly
created entity including the `id` field.

Now to the body of the function. You can see that I first convert the entity to JSON using
`entityToJson`. This is one of those funny little details that look innocent but boy are they
complicated. The reason why I can't simply do `JSON.stringify` is that we support types like
`Date` or `ArrayBuffer` and our CRUD API expects `Date` fields to be unix timestamp and
`ArrayBuffer` to be base64 encoded etc. To do the conversion, we need a reflection object
`entityType` describing the type. The function will also do some basic sanity checking
to make sure that the user hasn't bypassed the TypeSystem and hasn't smuggled some type missmatching values in there.

We we have the Json-like object in hand, we simply send it down the wire using `sendJson`,
check the response for errors (`throwOnError`) and do the inverse of `entityToJson` - `entityFromJson` for analogous reasons which were described earlier.

Now let's use it:

```typescript
const client = {
    users: {
        post: makePostOne<User>(url(`/users`), reflection.User), ...
    }
    blog: {
        posts: {
            post: makePostOne<BlogPost>(url(`/blog/posts`), reflection.BlogPost), ...
        }
    }
}
```

Awesome! This will save us a lot of generation as we can put the function `makePostOne` to a
`client_lib.ts` library file which would be just a regular TS file which can be linted, analyzed
or anything that we fancy.