---
title: Is Owner - Strapi Developer Docs
description: Learn in this guide how to restrict content editing to content authors only.
canonicalUrl: https://docs.strapi.io/developer-docs/latest/guides/is-owner.html
---

# Create is owner policy

This guide will explain how to restrict updating and deleting an entry to the User that created it.

You will have to modify the controllers of each Content-Type that you would like to enable owner-restricted API access for.

## Example

For this example, we will create an **article** [Content-Type](/developer-docs/latest/development/backend-customization/models.html).

Add a `text` field named `title` and a `relation` field to this model.

The `relation` field is a **many-to-one** relation with User (from:users-permissions).<br>
A User can have many Articles, but an Article can have only one User.<br>
Name the field `author` for the Article Content-Type and `articles` on the User side.

Now we are ready to start the customization of our [controllers](/developer-docs/latest/development/backend-customization/controllers.html).

## Apply the author by default

When creating a new Article via `POST /api/articles` we must set the authenticated User as the article's author.

We will [customize the `create` controller](/developer-docs/latest/development/backend-customization/controllers.html#extending-core-controllers) function of the Article API to do so.

> You must enable the permissions to **create, update** and **delete** Articles from Settings -> Users & Permissions Plugin -> Roles -> Authenticated

**Path —** `./src/api/article/controllers/article.js`

```js
'use strict';

/**
 *  article controller
 */

const { createCoreController } = require('@strapi/strapi').factories;

module.exports = createCoreController('api::article.article', {
  async create(ctx) {
    var { id } = ctx.state.user; //ctx.state.user contains the current authenticated user
    const response = await super
      .create(ctx)
      .then(article =>
        strapi.entityService.update('api::article.article', article.data.id, {
          data: { author: id },
        })
      );
    return response;
  },
});
```

When a new entry is now created, the authenticated User is automatically set as the author of the article.

## Limit the delete and update

Now we will restrict the update and delete controller of articles to only allow for the author to make changes.

We will use the same file and concepts the previous step.

**Path —** `./src/api/article/controllers/article.js`

```js
'use strict';

/**
 *  article controller
 */

const { createCoreController } = require('@strapi/strapi').factories;

module.exports = createCoreController('api::article.article', {
    async create(ctx) {
        var { id } = ctx.state.user; //ctx.state.user contains the current authenticated user 
        const response = await super.create(ctx)
            .then(article => strapi.entityService
                .update('api::article.article', article.data.id, { data: { author: id } }))
        return response;
    },
    async update(ctx) {
        var { id } = ctx.state.user
        var [article] = await strapi.entityService
            .findMany('api::article.article', {
                filters: {
                    id: ctx.request.params.id,
                    author: id
                }
            })
        if (article) {
            const response = await super.update(ctx);
            return response;
        } else {
            return ctx.unauthorized();
        }
    },
    async delete(ctx) {
        var { id } = ctx.state.user
        var [article] = await strapi.entityService
            .findMany('api::article.article', {
                filters: {
                    id: ctx.request.params.id,
                    author: id
                }
            })
        if (article) {
            const response = await super.delete(ctx);
            return response;
        } else {
            return ctx.unauthorized();
        }
    },
}
);
```

> Remember to replace **api::article.article** with the UID of the content-type that you are looking to customize

And tada! now the User that creates an article, will be set as the author, and is the only one who is able to update/delete the article.
