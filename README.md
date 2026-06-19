<!-- regenerate: on (set to off if you edit this file) -->

## End Entity Name Restirictions

(yes it should be renamed, please bikeshed)

* [Editor's Copy](https://bob-beck.github.io/ratatouille-leafy-greens/#go.draft-leafy-greens.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-leafy-greens)
* [Individual Draft](https://datatracker.ietf.org/doc/html/draft-leafy-greens)
* [Compare Editor's Copy to Individual Draft](https://bob-beck.github.io/ratatouille-leafy-greens/#go.draft-leafy-greens.diff)

TL;DR - 5280 defines name constrains as a critical extension, which, for wildcards in DNS names, can change meaning over time.

The result of this is that you can not trust that two applications recognizing the critical extension recognize it with
the same semantics.

This affects the usefulness of using name constraints to constain a CA cert from signing for certain SAN DNSnanmes with excluded subtress
in the presence of wildcards.

We fix this by defining a new extension to provide name restriction semantics via a different critical extension, limited in scope
only for TLS, and only for SAN DNS names, with fully defined semantics.


## Contributing

See the
[guidelines for contributions](https://github.com/bob-beck/ratatouille-leafy-greens/blob/main/CONTRIBUTING.md).

The contributing file also has tips on how to make contributions, if you
don't already know how to do that.

## Command Line Usage

Formatted text and HTML versions of the draft can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed.  See
[the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).

