# Metador

**M**etadata **E**nrichment and **T**ransmission **A**ssistance for **D**igital **O**bjects in **R**esearch

## Overview

Metador is a metadata-aware mailbox for research data.

Like a real mailbox, it should be really simple to set up and use and should not be in your way.

Unlike a real mailbox where any content can be dropped, Metador wants to help you, the
data receiver, to make sense of the data, by requiring the uploader to fill out a form of
your choice for each file to provide all necessary metadata.

**Thereby, Metador faciliates FAIRification of research data and forms a boundary between
the unstructured, unannotated outside world and the FAIR, semantically annotated data
inside your amazing research institution.**

If you **formalize** your metadata requirements in the form of JSON Schemas, then Metador
will **enforce** those requirements, if you let your external partners share their data
with you through it. At the same time, you are **in full control** of the data, because
Metador is simple to set-up locally and eliminates the need of a middle-man service that
assists you with moving larger (i.e., multiple GB) files over the internet.

Using Metador, the sender can upload a dataset (i.e., a set of files) and must annotate
the files according to your requirements. After the upload and annotation is completed, a
hook can be triggered that will get a complete dataset for further processing. For
example, you can put completed and annotated datasets into your in-lab repository. This
makes Metador **easy to integrate into your existing workflows**.

To achieve these goals, Metador combines state-of-the-art resumable file-upload technology
using [Uppy](https://uppy.io) and the [tus](https://tus.io/) protocol 
with a JSON Schema driven metadata editor.

## Why not a different self-hosted file uploader?

Before starting work on Metador, I was not able to find an existing solution checking all
of the following boxes:

* lightweight (easy to deploy and use on a typical institutional Linux server)
* supports upload of large files conveniently (with progress, resumable)
* supports rich and expressive metadata annotation that is **generic** (schema-driven)

If you care about this combination of features, then Metador is for you.
If you do not care about collecting metadata, feel free to pick a different solution.

## Installation

If you do not have a recent version of Python installed, you can use
[`pyenv`](https://github.com/pyenv/pyenv) to install e.g. Python 3.9.0 and enable it.

Download and install [`tusd`](https://github.com/tus/tusd),
the server component for the Tus protocol that will handle the low-level details
of the file uploading process.

Install Metador using `poetry install`, if you use poetry,
or use `pip install --user .` if you use pip (as usual, using a `venv` is recommended).

## Usage

### I want to see it in action, now!

Ensure that tusd and Metador are installed
and that the `tusd` and `metador-cli` scripts are on your path.

Run `tusd` like this: `tusd -hooks-http "$(metador-cli tusd-hook-url)"`

Run Metador like this: `metador-cli run`

Navigate to `http://localhost:8000/` in your browser.


### I want to deploy Metador properly!

For serious deployment into an existing infrastructure, some more steps are required:

* prepare JSON Schemas for the metadata you want to collect for the files.

* think about the way how both `tusd` and Metador will be visible to the outside world.
  I cannot give you assistance with your specific setup that might involve e.g. Apache
  or nginx to serve both applications.

   However your setup might be, you need to make sure that:

   1. both `tusd` and Metador are accessible from the client side (notice that by default
      they run on two different ports, unless you mask that with route rewriting and
      forwarding).

   2. The passed hook endpoint URL of Metador is accessible by `tusd`.

   3. The file upload directory of `tusd` is accessible (read + write) by Metador.

* Use `metador-cli default-conf > metador.toml` to get a copy of the default config file,
  add your JSON schemas (TODO: explain how) and
  at least change the `metadir.site` and `tusd.endpoint` entries according to your
  planned setup (you will probably at least change the domain, and maybe the ports).
  You can delete everything in your config that you do not want to override.

* *Optional:* For ORCID integration, you need access to the ORCID public API.

  Follow instructions given e.g. 
  [here](https://info.orcid.org/documentation/integration-guide/registering-a-public-api-client/)
  As redirect URL you should register the value you get from `metador-cli orcid-redir-url`.
  Afterwards, fill out the `orcid` section of your Metadir configuration accordingly.

* Run `tusd` as required with your setup, passing 
  `-hooks-http "$(metador-cli tusd-hook-url)"` as argument.

* Run Metador with your configuration: `metador-cli run --conf YOUR_CONFIG_FILE`

## FAQ

The following never actually asked questions might be of interest to you.

### Feature: Will there be an API for external tools to automate uploads?

This service is intended for use by human beings, to send you data that originally has
ad-hoc or lacking metadata. If the dataset is already fully annotated, it probably should
be transferred in a different and simpler way. If you insist on using this service
mechanically, in fact it is designed to be as RESTful as possible so you might try to
script an auto-uploader. However, there is no official support for this.

### Feature: Will there be support for e.g. cloud-based storage?

No, this service is meant to bring annotated data to **your** hard drives. Your
post-processing script can do with completed datasets whatever it wants, including
moving it to arbitrary different locations.

### Feature: Will there be support for HTTP-based post-processing hooks?

No, the only kind of hook ever provided will be the automatic calling of a script. The
datasets are located on the hard-drive, your post-processing needs access to it anyway. It
makes no sense to involve networking, in the same way as it would be absurd to make `git`
hooks use the network. Of course, you can write a hook script that uses networking
yourself, e.g. to send a network request to trigger an E-Mail about the new dataset.

### Feature: Will there be support for authentication mechanisms besides ORCID?

ORCID is highly adopted in research and allows to sign in using other mechanisms,
to that in the research domain it should be sufficient. If you or your partners do not
have an ORCID yet, maybe now it is the time!

If you want to restrict access to your instance to a narrow circle of persons, instead of
allowing anyone with an ORCID to use your service, just use the provided **whitelist**
functionality (TODO).

It is **not** planned to add authentication that requires the user to register
specifically to use this service. No one likes to create new accounts and invent new
passwords, and you probably have more important things to do and do not need the
additional responsibility of keeping these credentials secure.

If you, against all advice, want to have a custom authentication mechanism, use 
this with ORCID disabled, i.e. in "open mode", and then restrict access to the service in
a different way.

### Security: Are my datasets secure and cannot be stolen or manipulated?

Short answer: if this service is exposed via HTTPS, for all practical purposes the answer
is **yes**.

OAuth is used to authenticate via ORCID, so authentication is as secure as ORCID is.
After completing authentication, a classical session cookie is stored, proving that the
user is signed in. This cookie is not accessible from JavaScript and is using
all the best practices to counteract typical attacks.

Only with that cookie the user can access the datasets, and only his own datasets.
This is ensured by the fact that the session cookie witnesses an authenticated ORCID
and each dataset is marked with the ORCID of its creator, so it is only accessible by this
user. File uploads are also all marked with the Dataset UUID, which only the user knows.
The risk of abuse can be further minimized if the user signs out after completing their
work.

So if you trust HTTPS, ORCID, and your own server where you host your instance, 
you can be reasonably sure that only the user who created a dataset is able to access it
in any way and that at least no naive attack should succeed.

To gain illicit access to a dataset, an attacker must bypass the 3-legged OAuth procedure,
or must somehow steal the SessionID of a legitimate user, which, as described, seems to be
rather infeasible, unless the attacker has already absolute control over the system of the
uploader. In that case your problem is clearly somewhere else, anyway.

Finally, completed datasets can no longer be accessed by anyone from the client-side and
therefore can be considered to be as secure as the rest of your system.

### Technical: Why do you use old-fashioned session cookies? Why don't you use Redis?

The classical session-cookie approach is easier to implement, handling JWT correctly is
more involved and 
[there are](http://cryto.net/~joepie91/blog/2016/06/13/stop-using-jwt-for-sessions/)
[arguments](https://developer.okta.com/blog/2017/08/17/why-jwts-suck-as-session-tokens)
that JWT should not be used for sessions or at least have no advantages for this usage.

Concerning Redis - it might be used to persist in-memory data between reloads, but only for
this, pulling in this dependency seems to be "overkill". As this service is not intended to
be up-scaled and each instance will probably have probably at most a few dozens of users,
having a shared cache seems not so important. Possibly Redis might be added as an
*optional* dependency at some point in the future, if problems arise, but it will never be
mandatory.

## Development

Before commiting, run `pytest` and make sure you did not break anything.

To generate documentation, run `pdoc -o docs metador`

To check coverage, use `pytest --cov-report term-missing:skip-covered --cov=metador tests/`

Also verify that the pre-commit hooks are run successfully.

## Copyright and Licence

See [LICENSE](./LICENSE).

### Used libraries and tools

#### Libraries

**CLI:** typer (MIT), toml (MIT), colorlog (MIT)

**Backend:** FastAPI (MIT), pydantic (MIT), tusd (MIT), httpx (BSD-3), python-multipart (Apache 2.0)

**Frontend:** uppy (MIT), Picnic CSS (MIT)

...and of course, the Python standard library.

#### Tools

**Checked by:** pytest (MIT), pytest-cov (MIT), pre-commit (MIT), black (MIT), flake8 (MIT), mypy (MIT)

**Packaged by:** poetry (MIT)

More information is in the documentation of the corresponding packages.
