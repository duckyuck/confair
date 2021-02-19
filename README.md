# <img align="right" src="conair.jpg" width="200" height="100"> Confair

Confair is a configuration library for Clojure, with some pretty nifty features:

- Config as EDN, in dev and prod.

- Encrypted secrets in the git repo.

- Masked secrets when logging.

- Check in working local config, with easy overrides.

The dream is a world where developers can change configuration options, even
secret ones, without coordinating updates with every other developer on the
team. The dream is easily switching between environments locally, without
commenting in and out bundles of related config options. The dream is checking
in the prod config in a single readable EDN-file, instead of passing it
piecemeal via ENV-vars in Kubernetes secrets. The dream is not accidentally
sending all your secrets to Datadog.

Welcome to the dream.

## Hello, Confair

Okay, that last part might have been too much. I was channeling the spirit of
Nicolas Cage. Anyway, let's take a quick look at how it works.

Create a config file somewhere, for example `./config/dev-config.edn`:

```clj
{:spotify/api-token-url "https://accounts.spotify.com/api/token"
 :spotify/api-url "https://api.spotify.com"
 :spotify/client-id "my-api-client"
 :spotify/client-secret "3abdc"}
```

We want to check the dev config into source control, but we don't want to check
in our client-secret, so we'll encrypt it.

First, create a file with a secret, and make sure we don't check it in:

```sh
echo shhh-dont-tell-anyone > secrets/dev-config-secret.txt
echo "secrets/*.txt" >> .gitignore
```

Second, let confair know about the secret with some metadata:

```clj
^{:config/secrets {:secret/dev [:config/file "./secrets/dev-config-secret.txt"]}}
{:spotify/api-token-url "https://accounts.spotify.com/api/token"
 :spotify/api-url "https://api.spotify.com"
 :spotify/client-id "my-api-client"
 :spotify/client-secret "3abdc"}
```

(A popular option in prod is to replace `:config/file` with `:config/env` to
read from an environment variable instead)

Now that confair knows where to find the secret, it's time to fire up the REPL
to encrypt the client-secret.

```clj
(require '[confair.client :as config])
(require '[confair.client-admin :as ca])

(ca/conceal-value (config/from-file "./config/dev-config.edn")
                  :secret/dev
                  :spotify/client-secret)
```

This loads the configuration (including the metadata we need), and uses the
`:secret/dev` secret to encrypt `:spotify/client-secret`. Our file has now
been updated to look like this:

```clj
^{:config/secrets {:secret/dev [:config/file "./secrets/dev-config-secret.txt"]}}
{:spotify/api-token-url "https://accounts.spotify.com/api/token"
 :spotify/api-url "https://api.spotify.com"
 :spotify/client-id "my-api-client"
 :spotify/client-secret [:secret/dev "TlBZD.....kc="}
```

Our client-secret has been encrypted with high-strength AES128, courtesy of
[Nippy](https://github.com/ptaoussanis/nippy). This file can now be safely
checked into source control. The dev config secret needs to be shared with
other developers out of band, but only once.

In order to use this config in our app, we read it back in like this:

```clj
(require '[confair.client :as config])

(def config (config/from-file "./config/dev-config.edn"))

(:spotify/client-id config) ;; => "my-api-client"
(:spotify/client-secret config) ;; => "3abdc"
```

Note that the secret is decrypted for us.

### Masking

What if you're sending logs to some log aggregation service? Maybe you are
logging your config when starting the process (this is a good idea), or maybe
you're adding config to the `request` map, and some middleware logs it when an
exception occurs?

In either case, you wouldn't want your secrets to be sent verbatim over the net.
Let's mask the config secrets:

```clj
(def config (-> (config/from-file "./config/dev-config.edn")
                (config/mask-config)))

(:spotify/client-id config) ;; => "my-api-client"
(:spotify/client-secret config) ;; => "3abdc"
```

You can still look up config keys individually, but if you turn the config map
into a string with `(str config)` or `(clojure.pprint/pprint config)` or
`(log/info config)` the secrets will be masked:

```clj
{:spotify/api-token-url "https://accounts.spotify.com/api/token"
 :spotify/api-url "https://api.spotify.com"
 :spotify/client-id "my-api-client"
 :spotify/client-secret [:config/masked-string "3*******"}
```

### Local overrides

Instead of checking in `dev-config.edn`, let's add it to `.gitignore`:

```sh
echo "config/dev-config.edn" >> .gitignore
```

We'll move the default configuration to a file that we *do* check in, `./config/default-config.edn`:

```clj
{:spotify/api-token-url "https://accounts.spotify.com/api/token"
 :spotify/api-url "https://api.spotify.com"
 :spotify/client-id "my-api-client"
 :spotify/client-secret "3abdc"}
```

And import the defaults from our `./config/dev-config.edn`:

```clj
^{:config/secrets {:secret/dev [:config/file "./secrets/dev-config-secret.txt"]}
  :dev-config/import [".config/default-config.edn"]}
{:spotify/api-url "https://api-test.spotify.com"}
```

In this example, the default config options will be imported, but
`:spotify/api-url` is overridden.

Add a sample file for new developers for good measure:

```
cp config/dev-config.edn config/dev-config.edn.sample
```

And you are good to go!

## Install

Confair is a stable library - it will never change it's public API in breaking
way, and will never (intentionally) introduce other breaking changes.

With tools.deps:

```clj
com.magnars/confair {:mvn/version "2020-02-18"}
```

With Leiningen:

```clj
[com.magnars/confair "2020-02-18"]
```

## Finally

There's quite a bit more to confair, but this will have to do for now. More docs
coming later, but feel free to check out the tests for more examples.

## License

Copyright © 2021 Magnar Sveen

Distributed under the Eclipse Public License either version 1.0 or (at your
option) any later version.
