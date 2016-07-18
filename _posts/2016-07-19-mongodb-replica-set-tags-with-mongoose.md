---
layout: post
title: MongoDB replica set tags with mongoose
---

MongoDB replica set tags are a handy way to refer to a subset of nodes. Imagine a scenario where you have three nodes in your replica set and you want to dedicate one server to reporting or background jobs. This can easily be achieved by [replica set tags][1] combined with [preventing that node from becoming primary][2].

After you have set up your MongoDB cluster you also have to configure your clients so they can read from the correct nodes.

Our production API is a node.js application that uses [mongoose][3] as an ORM or ODM if we want to be fancy with abbreviations. Fortunately mongoose [supports replica set tags][4] as `readPreferenceTags`. It claims to support them in the connection URI as well. We are a big fan of 12 factor apps so we pass the connection URI as an environment variable.

Up until this point everything is straightforward and pretty simple. When we configured everything we expected one of the MongoDB nodes to have little to no traffic. To our surprise this was pretty far from the truth. The traffic distribution was the same as before. All nodes got some portion of the traffic. At this point we had to deep dive into the source code of mongoose. I can tell you it wasn't a nice experience.

The problem was pretty apparent from the get go. In the underlying package that handles the actual TCP connection to the database there is a [function called `pickServer`][5] which does what you expect. The strange thing about this is that the read preference tags were not passed down to this function. Every other relevant configuration was available and set properly. I tried to hunt down how the configuration is passed around and where the read preference tags get lost.

I ended up with the realization that the configuration is parsed but it is lost somewhere between the 3-5 underlying packages that are used by mongoose. I [filed an issue][6] on Github, maybe it saves a few hours of debugging for somebody.

Fortunately there is a quick and pretty painless workaround so we didn't have to figure out where it gets actually lost. If the read preference tags are set directly and not via the connection URI then it sticks and gets passed the `pickServer` function.

```js
var mongoUrl = process.env.MONGO_URL;
options = muri(mongoUrl).options;
mongoose.connect(mongoUrl, {
  db: {
    readPreference: {
      preference: options.readPreference,
      tags: options.readPreferenceTags
    }
  }
});
```

This only gets us half-way there. As it turns out there is another bug in one of the underlying packages. It is a pretty trivial bug so as a good open source citizen I opened up my text editor and [fixed the issue][7]. This was the easy part. Now I had to somehow force node.js to load the fixed package instead of the canonical one. This would be super simple in Ruby land. I would just declare that xyz dependency comes from a git repo and be done with it. In Ruby every loaded dependency is global so overwriting it in one place would solve it for every other package. This is not the case for node.js...

npm has a feature called [shrinkwrap][8] which is basically a version lock file. We can specify the exact version of one or more packages which will be consistent across all `npm install`s. Since npm version 3 we can specify only a subset of packages without specifying versions for all packages. We can use this to hack around node.js's questionable module system so we can inject the fixed package.

Our `npm-shrinkwrap.json` is the following:

```json
{
  "dependencies": {
    "mongoose": {
      "version": "4.4.20",
      "from": "mongoose@4.4.20",
      "resolved": "https://registry.npmjs.org/mongoose/-/mongoose-4.4.20.tgz",
      "dependencies": {
        "mongodb": {
          "version": "2.1.18",
          "from": "mongodb@2.1.18",
          "resolved": "https://registry.npmjs.org/mongodb/-/mongodb-2.1.18.tgz",
          "dependencies": {
            "mongodb-core": {
              "version": "1.3.22-alpha4",
              "from": "git+https://github.com/sspinc/mongodb-core.git#fix-tag-selection",
              "resolved": "git+https://github.com/sspinc/mongodb-core.git#e1fdf33a031ae653437aac3d419a81191a777bfa"
            }
          }
        }
      }
    }
  }
}
```

As the time of writing this is the only way to get read preference tags to work with mongoose.

[1]: https://docs.mongodb.com/manual/tutorial/configure-replica-set-tag-sets/
[2]: https://docs.mongodb.com/manual/tutorial/configure-secondary-only-replica-set-member/
[3]: http://mongoosejs.com/
[4]: http://mongoosejs.com/docs/connections.html#connection-string-options
[5]: https://github.com/christkv/mongodb-core/blob/1a3f5aef67a363c1bba4020c352a98b9afd2a277/lib/topologies/replset.js#L985
[6]: https://github.com/Automattic/mongoose/issues/4311
[7]: https://github.com/christkv/mongodb-core/pull/107
[8]: https://docs.npmjs.com/cli/shrinkwrap
